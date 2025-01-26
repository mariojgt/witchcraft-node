<template>
    <div class="relative h-full">
        <VueFlow v-model:nodes="nodes" v-model:edges="edges" :class="{ 'simulation-mode': simulationMode }"
            @connect="onConnect" @dragover.prevent @drop="onDrop">
            <Controls @simulate="startSimulation" @export="exportFlow" @import="importFlow" @clear="clearLogs"
                @toggle-logs="toggleLogs" :speed="simulationSpeed" @change-speed="changeSimulationSpeed" />
            <Background />

            <SimulationLogs :logs="simulationLogs" :simulation-mode="simulationMode" @clear="clearLogs"
                v-if="showLogs" />

            <template v-for="(nodeComponent, nodeType) in nodeComponents" :key="nodeType"
                #[`node-${nodeType}`]="nodeProps">
                <component :is="nodeComponent" v-bind="nodeProps" :class="{
                    'processing': getSimulationStep(nodeProps.id)?.status === 'processing',
                    'completed': getSimulationStep(nodeProps.id)?.status === 'completed'
                }" :simulation-step="getSimulationStep(nodeProps.id)" :output="nodeOutputs[nodeProps.id]"
                    @delete="() => removeNode(nodeProps.id)" />
            </template>
        </VueFlow>
    </div>
</template>


<script setup>
import { ref, reactive, onMounted, markRaw } from "vue";
import { VueFlow, useVueFlow } from "@vue-flow/core";
import { Background } from "@vue-flow/background";
import SimulationLogs from "./SimulationLogs.vue";
import Controls from "./Controls.vue";

const showLogs = ref(true);
const simulationSpeed = ref(1000); // Adjustable simulation speed (default: 1 second per step)
const simulationMode = ref(false);
const simulationSteps = reactive([]); // Change to `reactive` for consistency
const nodes = ref([]);
const edges = ref([]);
const simulationLogs = ref([]);
const nodeOutputs = reactive({});

const { addEdges, removeNodes, addNodes, toObject } = useVueFlow();
const nodeComponents = reactive({});
const nodeFiles = import.meta.glob("./nodes/*.vue");

onMounted(async () => {
    for (const path in nodeFiles) {
        const fileName = path.split("/").pop().replace(".vue", "");
        const nodeType = fileName.toLowerCase().replace("node", "");
        const module = await nodeFiles[path]();
        nodeComponents[nodeType] = markRaw(module.default);
    }
});

/**
 * Retrieves the simulation step for a given node ID.
 * Returns `null` if no simulation step exists for the given node.
 * @param {string} nodeId - The ID of the node.
 * @returns {Object|null} Simulation step or null if not found.
 */
function getSimulationStep(nodeId) {
    return simulationSteps.find((step) => step.id === nodeId) || null;
}

function toggleLogs() {
    showLogs.value = !showLogs.value;
}

function onConnect(params) {
    addEdges([params]);
}

function removeNode(nodeId) {
    removeNodes([nodeId]);
    delete nodeOutputs[nodeId];
}

function onDrop(event) {
    event.preventDefault();
    const nodeTypeData = event.dataTransfer.getData("application/nodeType");
    if (!nodeTypeData) return;

    const { type, initialData } = JSON.parse(nodeTypeData);
    const newNode = {
        id: `${type}-${Date.now()}`,
        type: type.toLowerCase(),
        position: {
            x: event.clientX - event.target.getBoundingClientRect().left,
            y: event.clientY - event.target.getBoundingClientRect().top,
        },
        data: initialData,
        class: "",
    };
    addNodes([newNode]);
}

async function startSimulation() {
    simulationLogs.value = [];
    const workflowData = toObject();
    const { nodes: workflowNodes, edges: workflowEdges } = workflowData;

    if (workflowNodes.length === 0) {
        alert("No nodes to simulate");
        return;
    }

    simulationMode.value = true;
    simulationSteps.value = [];
    nodes.value = nodes.value.map((node) => ({ ...node, class: "" }));

    const startNodes = workflowNodes.filter(
        (node) => !workflowEdges.some((edge) => edge.target === node.id)
    );

    for (const startNode of startNodes) {
        await processNodeSequentially(startNode, workflowNodes, workflowEdges);
    }

    simulationMode.value = false;
}

async function processNodeSequentially(node, allNodes, edges) {
    addLog(`Processing node: ${node.type} (${node.id})`, "info");
    updateNodeStatus(node.id, "processing");

    await delay(simulationSpeed.value);

    // Fetch input from connected source nodes
    const inputValue = getInputValue(node, edges, allNodes);

    // Process the node with its data and input
    const output = await processNode(node, inputValue);
    nodeOutputs[node.id] = output;

    updateNodeStatus(node.id, "completed");

    // Propagate the output to connected nodes
    const outgoingEdges = edges.filter((edge) => edge.source === node.id);
    for (const edge of outgoingEdges) {
        const nextNode = allNodes.find((n) => n.id === edge.target);
        if (nextNode) {
            const shouldFollow = node.type === "if"
                ? edge.sourceHandle === (output ? "true" : "false")
                : true; // Default true for non-condition nodes

            if (shouldFollow) {
                await processNodeSequentially(nextNode, allNodes, edges);
            }
        }
    }
}

function getInputValue(node, edges, allNodes) {
    const incomingEdges = edges.filter((edge) => edge.target === node.id);
    const inputValues = {};

    incomingEdges.forEach((edge) => {
        const sourceNode = allNodes.find((n) => n.id === edge.source);
        if (sourceNode && nodeOutputs[sourceNode.id]) {
            Object.assign(inputValues, nodeOutputs[sourceNode.id]);
        }
    });

    // For "if" node, return the relevant input key or fallback to the full object
    return inputValues;
}

async function processNode(node, inputValue) {
    let output = {};
    switch (node.type) {
        case "variable":
            output = { [node.data.outputKey]: node.data.initialValue };
            break;
        case "if":
            output = evaluateCondition(node.data, inputValue);
            break;
        case "api":
            output = await simulateApiCall(node.data);
            break;
        case "notification":
            const message = node.data.message.replace(/\{\{(\w+)\}\}/g, (_, key) => inputValue[key] || `{{${key}}}`);
            addLog(`Notification: ${message}`, node.data.type);
            output = null;
            break;
        default:
            output = node.data || {};
    }
    addLog(`Node output: ${JSON.stringify(output)}`, "success");
    return output;
}


function updateNodeStatus(nodeId, status) {
    nodes.value = nodes.value.map((n) =>
        n.id === nodeId ? { ...n, class: status } : n
    );
}

function addLog(message, type = "info") {
    simulationLogs.value.push({ message, type, timestamp: new Date() });
}

function changeSimulationSpeed(newSpeed) {
    simulationSpeed.value = newSpeed;
}

function delay(ms) {
    return new Promise((resolve) => setTimeout(resolve, ms));
}

function simulateApiCall(apiData) {
    return new Promise((resolve) =>
        setTimeout(() => resolve({ status: "success", data: apiData }), 1000)
    );
}

function evaluateCondition(nodeData, inputValue) {
    const { conditionType, expectedValue } = nodeData;
    const actualValue = inputValue?.status || inputValue; // Adjust based on your input data structure

    switch (conditionType) {
        case "equals":
            return String(actualValue) === String(expectedValue);
        case "notEquals":
            return String(actualValue) !== String(expectedValue);
        case "contains":
            return String(actualValue).includes(String(expectedValue));
        case "changed":
            return actualValue !== nodeData.previousValue;
        default:
            console.error(`Unknown condition type: ${conditionType}`);
            return false;
    }
}

function exportFlow() {
    const flowData = { nodes: nodes.value, edges: edges.value };
    const blob = new Blob([JSON.stringify(flowData, null, 2)], {
        type: "application/json",
    });
    const url = URL.createObjectURL(blob);
    const a = document.createElement("a");
    a.href = url;
    a.download = "workflow.json";
    a.click();
    URL.revokeObjectURL(url);
}

function importFlow() {
    const input = document.createElement("input");
    input.type = "file";
    input.accept = ".json";
    input.onchange = async (e) => {
        const file = e.target.files[0];
        if (!file) return;
        try {
            const text = await file.text();
            const data = JSON.parse(text);
            nodes.value = data.nodes;
            edges.value = data.edges;
        } catch (error) {
            console.error("Import error:", error);
        }
    };
    input.click();
}

function clearLogs() {
    simulationLogs.value = [];
}
</script>

<style>
/* Improved styles for better simulation visibility */
.vue-flow__node {
    transition: all 0.3s ease;
}

.simulation-mode .vue-flow__node {
    opacity: 0.5;
}

.vue-flow__node.processing {
    opacity: 1 !important;
    border: 2px solid #4299e1 !important;
    box-shadow: 0 0 20px rgba(74, 144, 226, 0.7) !important;
}

.vue-flow__node.completed {
    opacity: 1 !important;
    border: 2px solid #48bb78 !important;
    box-shadow: 0 0 20px rgba(39, 174, 96, 0.7) !important;
}

.vue-flow__edge.active path {
    stroke: #4299e1;
    stroke-width: 3;
}
</style>
