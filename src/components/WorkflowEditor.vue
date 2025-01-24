<template>
    <div class="relative h-full">
      <VueFlow
        v-model:nodes="nodes"
        v-model:edges="edges"
        :class="{ 'simulation-mode': simulationMode }"
        @connect="onConnect"
        @dragover.prevent
        @drop="onDrop"
      >
        <Controls
          @simulate="startSimulation"
          @export="exportFlow"
          @import="importFlow"
          @clear="clearLogs"
          @toggle-logs="toggleLogs"
        />
        <Background />

        <SimulationLogs
          :logs="simulationLogs"
          :simulation-mode="simulationMode"
          @clear="clearLogs"
          v-if="showLogs"
        />

        <template
          v-for="(nodeComponent, nodeType) in nodeComponents"
          :key="nodeType"
          #[`node-${nodeType}`]="nodeProps"
        >
          <component
            :is="nodeComponent"
            v-bind="nodeProps"
            :class="{
              'processing': getSimulationStep(nodeProps.id)?.status === 'processing',
              'completed': getSimulationStep(nodeProps.id)?.status === 'completed'
            }"
            :simulation-step="getSimulationStep(nodeProps.id)"
            @delete="() => removeNode(nodeProps.id)"
          />
        </template>
      </VueFlow>
    </div>
  </template>

<script setup>
import { ref, reactive, onMounted, markRaw } from 'vue'
import { VueFlow, useVueFlow } from '@vue-flow/core'
import { Background } from '@vue-flow/background'
import SimulationLogs from './SimulationLogs.vue'
import Controls from './Controls.vue'


const showLogs = ref(true)

function toggleLogs() {
  showLogs.value = !showLogs.value
}

const nodes = ref([])
const edges = ref([])
const { addEdges, removeNodes, addNodes, toObject } = useVueFlow()

const nodeComponents = reactive({})
const nodeFiles = import.meta.glob('./nodes/*.vue')
const simulationMode = ref(false)
const simulationSteps = ref([])

onMounted(async () => {
    for (const path in nodeFiles) {
        const fileName = path.split('/').pop().replace('.vue', '')
        const nodeType = fileName.toLowerCase().replace('node', '')
        const module = await nodeFiles[path]()
        nodeComponents[nodeType] = markRaw(module.default)
    }
})

function onConnect(params) {
    addEdges([params])
}

function removeNode(nodeId) {
    removeNodes([nodeId])
}

function onDrop(event) {
    event.preventDefault()
    const nodeTypeData = event.dataTransfer.getData('application/nodeType')
    if (!nodeTypeData) return

    const { type, initialData } = JSON.parse(nodeTypeData)
    const newNode = {
        id: `${type}-${Date.now()}`,
        type: type.toLowerCase(),
        position: {
            x: event.clientX - event.target.getBoundingClientRect().left,
            y: event.clientY - event.target.getBoundingClientRect().top
        },
        data: initialData,
        class: ''
    }
    addNodes([newNode])
}

async function startSimulation() {
    simulationLogs.value = []
    const workflowData = toObject()
    const { nodes: workflowNodes, edges: workflowEdges } = workflowData

    if (workflowNodes.length === 0) {
        alert('No nodes to simulate')
        return
    }

    simulationMode.value = true
    simulationSteps.value = []

    nodes.value = nodes.value.map(node => ({
        ...node,
        class: ''
    }))

    const startNodes = workflowNodes.filter(node =>
        !workflowEdges.some(edge => edge.target === node.id)
    )

    for (const startNode of startNodes) {
        await processNodeSequentially(startNode, workflowNodes, workflowEdges)
    }

    setTimeout(() => {
        simulationMode.value = false
        simulationSteps.value = []
        nodes.value = nodes.value.map(node => ({
            ...node,
            class: ''
        }))
    }, 1000)
}

async function processNodeSequentially(node, allNodes, edges) {
    addLog(`Processing node: ${node.type} (${node.id})`, 'info')

    nodes.value = nodes.value.map(n => ({
        ...n,
        class: n.id === node.id ? 'processing' : n.class
    }))

    simulationSteps.value = [...simulationSteps.value, {
        nodeId: node.id,
        status: 'processing',
        data: node.data
    }]

    await new Promise(resolve => setTimeout(resolve, 1000))

    let output
    switch (node.type) {
        case 'modelselect':
            addLog(`Model event: ${node.data.modelName} ${node.data.eventType}`, 'info')
            output = {
                [node.data.outputKey]: node.data.outputValue,
                modelName: node.data.modelName,
                eventType: node.data.eventType,
                fields: node.data.watchFields
            };
            break;
        case 'variable':
            addLog(`Variable set: ${node.data.outputKey} = ${node.data.initialValue}`, 'info')
            output = {
                [node.data.outputKey]: node.data.initialValue
            };
            break;
        case 'if':
            addLog(`Evaluating condition: ${node.data.conditionType}`, 'info')
            const incomingEdge = edges.find(e => e.target === node.id)
            const previousNode = allNodes.find(n => n.id === incomingEdge?.source)
            const inputValue = previousNode?.data?.initialValue || previousNode?.output
            output = evaluateCondition(node.data, inputValue)
            console.log('Condition evaluation:', { inputValue, condition: node.data, output })
            break
        case 'api':
            output = await simulateApiCall(node.data)
            break
        case 'notification':
            const templateVars = getPreviousNodeOutputs(node, edges, allNodes);
            const message = node.data.message.replace(/\{\{(\w+)\}\}/g, (_, key) => {
                return templateVars[key] || `{{${key}}}`;
            });
            addLog(`Showing notification: ${message}`, node.data.type)
            showNotification({ ...node.data, message });
            output = null;
            break;
    }

    node.output = output

    nodes.value = nodes.value.map(n => ({
        ...n,
        class: n.id === node.id ? 'completed' : n.class
    }))

    simulationSteps.value = simulationSteps.value.map(step =>
        step.nodeId === node.id ? { ...step, status: 'completed', output } : step
    )

    await new Promise(resolve => setTimeout(resolve, 1000))

    const outgoingEdges = edges.filter(edge => edge.source === node.id)
    for (const edge of outgoingEdges) {
        const nextNode = allNodes.find(n => n.id === edge.target)
        if (nextNode) {
            if (node.type === 'if') {
                const shouldFollow = output ? edge.sourceHandle === 'true' : edge.sourceHandle === 'false'
                if (shouldFollow) {
                    await processNodeSequentially(nextNode, allNodes, edges)
                }
            } else {
                await processNodeSequentially(nextNode, allNodes, edges)
            }
        }
    }
}

function getPreviousNodeOutputs(node, edges, allNodes) {
    const vars = {};
    const incomingEdges = edges.filter(e => e.target === node.id);

    incomingEdges.forEach(edge => {
        const sourceNode = allNodes.find(n => n.id === edge.source);
        if (sourceNode?.output) {
            Object.assign(vars, sourceNode.output);
        }
    });

    return vars;
}

function evaluateCondition(conditionData, inputValue) {
    console.log('Input Value:', inputValue);

    // Extract value based on Model Event node's output
    let valueToCompare = inputValue;
    if (typeof inputValue === 'object' && inputValue !== null) {
        valueToCompare = inputValue.outputValue;
    }

    console.log('Comparing:', { expected: conditionData.expectedValue, actual: valueToCompare });

    switch (conditionData.conditionType) {
        case 'equals':
            return String(valueToCompare) === String(conditionData.expectedValue);
        case 'notEquals':
            return String(valueToCompare) !== String(conditionData.expectedValue);
        case 'contains':
            return String(valueToCompare).includes(String(conditionData.expectedValue));
        default:
            return false;
    }
}

async function simulateApiCall(apiData) {
    await new Promise(resolve => setTimeout(resolve, 1000))
    return { status: 'success', data: {} }
}

function showNotification(notificationData) {
    console.log('Showing notification:', notificationData)
    alert(`${notificationData.type.toUpperCase()}: ${notificationData.message}`)
}

function getSimulationStep(nodeId) {
    return simulationSteps.value.find(step => step.nodeId === nodeId)
}

const fileInput = ref(null)

function exportFlow() {
    const flowData = {
        nodes: nodes.value,
        edges: edges.value
    }
    const blob = new Blob([JSON.stringify(flowData, null, 2)], { type: 'application/json' })
    const url = URL.createObjectURL(blob)
    const a = document.createElement('a')
    a.href = url
    a.download = 'workflow.json'
    a.click()
    URL.revokeObjectURL(url)
}

async function importFlow() {
    const input = document.createElement('input')
    input.type = 'file'
    input.accept = '.json'

    input.onchange = async (e) => {
        const file = e.target.files[0]
        if (!file) return

        try {
            const text = await file.text()
            const data = JSON.parse(text)
            nodes.value = data.nodes
            edges.value = data.edges
        } catch (error) {
            console.error('Import error:', error)
            alert('Error importing workflow')
        }
    }

    input.click()
}

const simulationLogs = ref([])

function addLog(message, type = 'info') {
  simulationLogs.value.push({
    message,
    type,
    timestamp: new Date()
  })
}

function getLogClass(type) {
  switch (type) {
    case 'error': return 'bg-red-100 text-red-800'
    case 'success': return 'bg-green-100 text-green-800'
    case 'warning': return 'bg-yellow-100 text-yellow-800'
    default: return 'bg-blue-100 text-blue-800'
  }
}

function formatTime(date) {
  return new Date(date).toLocaleTimeString()
}

function clearLogs() {
  simulationLogs.value = []
}

</script>

<style>
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
