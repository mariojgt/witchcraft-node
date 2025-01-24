<template>
    <div
        class="bg-gray-900/95 backdrop-blur border-[1px] border-purple-500/30 rounded-xl p-4 min-w-[300px] relative text-gray-100">
        <!-- Header -->
        <div class="flex justify-between items-center mb-4 pb-2 border-b border-purple-500/30">
            <div class="flex items-center gap-2">
                <BellIcon class="w-5 h-5 text-purple-400" />
                <h3 class="font-bold text-purple-400">Model Events</h3>
            </div>
            <button @click="$emit('delete')" class="hover:bg-red-500/20 p-1 rounded transition-colors">
                <XIcon class="w-4 h-4 text-red-400" />
            </button>
        </div>

        <!-- Content -->
        <div class="space-y-4">
            <div class="space-y-2">
                <label class="text-xs uppercase tracking-wider text-purple-400">Model</label>
                <select v-model="data.modelName"
                    class="w-full bg-gray-800 border-0 rounded-lg focus:ring-2 focus:ring-purple-500 text-gray-100">
                    <option value="User">User</option>
                    <option value="Product">Product</option>
                    <option value="Order">Order</option>
                </select>
            </div>

            <div class="space-y-2">
                <label class="text-xs uppercase tracking-wider text-purple-400">Event Type</label>
                <select v-model="data.eventType"
                    class="w-full bg-gray-800 border-0 rounded-lg focus:ring-2 focus:ring-purple-500 text-gray-100">
                    <option value="created">Created</option>
                    <option value="updated">Updated</option>
                    <option value="deleted">Deleted</option>
                    <option value="restored">Restored</option>
                </select>
            </div>

            <div class="space-y-2">
                <label class="text-xs uppercase tracking-wider text-purple-400">Watch Fields</label>
                <div class="bg-gray-800 rounded-lg p-3 space-y-2">
                    <label v-for="field in availableFields" :key="field"
                        class="flex items-center gap-2 text-gray-300 hover:text-purple-400 transition-colors cursor-pointer">
                        <input type="checkbox" :value="field" v-model="data.watchFields"
                            class="rounded border-0 bg-gray-700 text-purple-500 focus:ring-purple-500" />
                        <span class="text-sm">{{ field }}</span>
                    </label>
                </div>
            </div>

            <div class="space-y-2">
                <label class="text-xs uppercase tracking-wider text-purple-400">Output Key</label>
                <input v-model="data.outputKey" placeholder="Variable output key"
                    class="w-full bg-gray-800 border-0 rounded-lg focus:ring-2 focus:ring-purple-500 text-gray-100" />
            </div>

            <div class="space-y-2">
                <label class="text-xs uppercase tracking-wider text-purple-400">Output Value</label>
                <input v-model="data.outputValue" :placeholder="getOutputPlaceholder"
                    class="w-full bg-gray-800 border-0 rounded-lg focus:ring-2 focus:ring-purple-500 text-gray-100" />
            </div>
        </div>

        <!-- Connection Point -->
        <div
            class="absolute top-1/2 -right-3 w-3 h-6 bg-gray-800 rounded-r-lg border-r-2 border-y-2 border-purple-500/50">
            <Handle type="source" position="right" id="event" class="!bg-purple-500" />
        </div>
    </div>
</template>

<script setup>
import { Handle } from '@vue-flow/core'
import { XIcon, BellIcon } from 'lucide-vue-next'
import { defineOptions, computed } from 'vue'

const props = defineProps(['data'])

const availableFields = computed(() => {
    switch (props.data.modelName) {
        case 'User': return ['status', 'email', 'role', 'last_login']
        case 'Product': return ['status', 'price', 'stock', 'category']
        case 'Order': return ['status', 'total', 'payment_status', 'shipping_status']
        default: return []
    }
})

defineOptions({
    nodeMetadata: {
        category: 'Events',
        icon: BellIcon,
        label: 'Model Events',
        initialData: {
            modelName: 'User',
            eventType: 'updated',
            watchFields: [],
            outputKey: 'modelEvent',
            outputValue: ''
        }
    }
})

defineEmits(['delete'])
</script>
