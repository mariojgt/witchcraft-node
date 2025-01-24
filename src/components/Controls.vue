<template>
    <Panel position="top-right">
        <div class="bg-gray-800 p-3 rounded-lg shadow-lg">
            <div class="space-y-2">
                <!-- Categories section -->
                <div v-for="(category, categoryName) in categorizedNodes" :key="categoryName"
                    class="border-b border-gray-700 last:border-b-0">
                    <div @click="toggleCategory(categoryName)"
                        class="flex justify-between items-center cursor-pointer py-2 px-2 hover:bg-gray-700 transition-colors">
                        <h3 class="text-sm uppercase text-gray-300">{{ categoryName }}</h3>
                        <ChevronDownIcon
                            :class="`w-4 h-4 transform transition-transform ${expandedCategories[categoryName] ? 'rotate-180' : ''}`" />
                    </div>
                    <div v-if="expandedCategories[categoryName]" class="grid grid-cols-2 gap-2 p-2">
                        <div v-for="type in category" :key="type.id" draggable="true"
                            @dragstart="onDragStart($event, type)"
                            class="bg-gray-700 hover:bg-blue-600 text-white p-2 rounded-md transition-colors flex flex-col items-center justify-center space-y-1 cursor-move">
                            <component :is="type.icon" class="w-6 h-6" />
                            <span class="text-xs">{{ type.label }}</span>
                        </div>
                    </div>
                </div>
            </div>

            <!-- Action buttons -->
            <!-- Action buttons -->
            <div class="mt-4 flex flex-col gap-2">
                <button @click="$emit('simulate')"
                    class="bg-blue-700 hover:bg-blue-600 text-white px-4 py-2 rounded-md transition-colors flex items-center gap-2 w-full justify-center">
                    <PlayIcon class="w-5 h-5" />
                    Simulate Flow
                </button>

                <!-- Import/Export buttons -->
                <div class="flex gap-2">
                    <button @click="$emit('export')"
                        class="bg-green-700 hover:bg-green-600 text-white px-4 py-2 rounded-md transition-colors flex items-center gap-2 flex-1 justify-center">
                        <DownloadIcon class="w-5 h-5" />
                        Export
                    </button>
                    <button @click="$emit('import')"
                        class="bg-yellow-700 hover:bg-yellow-600 text-white px-4 py-2 rounded-md transition-colors flex items-center gap-2 flex-1 justify-center">
                        <UploadIcon class="w-5 h-5" />
                        Import
                    </button>
                </div>

                <!-- Utility buttons -->
                <div class="flex gap-2">
                    <button @click="$emit('toggle-logs')"
                        class="bg-gray-700 hover:bg-gray-600 text-white px-4 py-2 rounded-md transition-colors flex items-center gap-2 flex-1 justify-center">
                        <ListIcon class="w-5 h-5" />
                        Toggle Logs
                    </button>
                    <button @click="$emit('clear')"
                        class="bg-red-700 hover:bg-red-600 text-white px-4 py-2 rounded-md transition-colors flex items-center gap-2 flex-1 justify-center">
                        <TrashIcon class="w-5 h-5" />
                        Clear Log
                    </button>
                </div>
            </div>
        </div>
    </Panel>
</template>

<script setup>
import { ref, reactive, onMounted, computed } from 'vue'
import { Panel } from '@vue-flow/core'
import {
    ChevronDownIcon,
    PlayIcon,
    DownloadIcon,
    UploadIcon,
    ListIcon,
    TrashIcon,
    ScanEye
} from 'lucide-vue-next'

const nodeTypes = reactive([])
const nodeFiles = import.meta.glob('./nodes/*.vue')
const expandedCategories = reactive({})

defineEmits(['simulate', 'export', 'import', 'clear', 'toggle-logs'])

onMounted(async () => {
    for (const path in nodeFiles) {
        const fileName = path.split('/').pop().replace('Node.vue', '')
        const nodeType = fileName.toLowerCase()
        const module = await nodeFiles[path]()
        const metadata = module.default?.nodeMetadata || module.nodeMetadata || {
            category: 'Other',
            icon: ScanEye,
            label: nodeType.replace(/^\w/, c => c.toUpperCase()),
            initialData: { label: 'New Node' }
        }
        nodeTypes.push({
            id: nodeType,
            ...metadata
        })
    }
    const categories = nodeTypes.reduce((cats, node) => {
        cats[node.category] = true
        return cats
    }, {})
    Object.assign(expandedCategories, categories)
})

const categorizedNodes = computed(() => {
    return nodeTypes.reduce((categories, node) => {
        if (!categories[node.category]) {
            categories[node.category] = []
        }
        categories[node.category].push(node)
        return categories
    }, {})
})

function toggleCategory(categoryName) {
    expandedCategories[categoryName] = !expandedCategories[categoryName]
}

function onDragStart(event, type) {
    event.dataTransfer.setData('application/nodeType', JSON.stringify({
        type: type.id,
        initialData: type.initialData
    }))
    event.dataTransfer.effectAllowed = 'copy'
}
</script>
