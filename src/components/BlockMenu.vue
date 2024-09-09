<template>
  <div ref="container" as="div" class="relative w-max h-max">
    <div @click="open = !open" class="handle" data-test-id="block-menu">
      <Tooltip value="<span class='text-neutral-400'><span class='text-white'>Drag</span> to move<br/><span class='text-white'>Click</span> to open menu</span>">
        <svg 
            @mouseup="$event.stopPropagation()"
            class="size-6 text-zinc-800 dark:text-zinc-200 hover:bg-neutral-100 dark:hover:bg-zinc-600 dark:hover:text-zinc-200 p-1 rounded group-hover:opacity-100 opacity-0 duration-200"
            :class="open ? 'opacity-100' : ''"
            xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><circle cx="9" cy="12" r="1"/><circle cx="9" cy="5" r="1"/><circle cx="9" cy="19" r="1"/><circle cx="15" cy="12" r="1"/><circle cx="15" cy="5" r="1"/><circle cx="15" cy="19" r="1"
        /></svg>
      </Tooltip>
    </div>
    <div v-show="open" class="block-menu">
      <div ref="menu"
        class="w-[10rem] lg:w-[12rem] xl:w-[16rem] absolute z-10 rounded bg-white dark:bg-zinc-900 border border-solid border-zinc-200 dark:border-zinc-700 shadow-lg py-1 text-sm right-full max-h-[24rem] overflow-auto focus-visible:outline-none top-0">
        <div class="text-left divide-y">
          <!-- Search term -->
          <div v-if="searchTerm" class="block-menu-search px-4 py-2 flex gap-2 w-full">
            <div class="size-2">
                <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><circle cx="11" cy="11" r="8"/><path d="m21 21-4.3-4.3"/></svg>
            </div>
            <div class="truncate text-zinc-700 dark:text-zinc-200">
              {{ searchTerm }}
            </div>
          </div>
          <!-- Turn into another block like Text, Heading or Divider -->
          <div class="p-2" v-if="options.filter(option => option.type === 'Turn into').length">
            <div class="px-2 pb-2 font-semibold uppercase text-xs tracking-wide text-zinc-500">Turn into</div>
            <div v-for="option, i in options.filter(option => option.type === 'Turn into')"
              class="p-2 bg-white text-zinc-700 dark:text-zinc-200 dark:bg-zinc-900 rounded hover:bg-zinc-100 dark:hover:bg-zinc-800 duration-300 flex items-center gap-2"
              :class="[active === (i + options.filter(option => option.type !== 'Turn into').length) ? 'bg-neutral-100' : '']"
              @click.stop="setBlockType(option.blockType);"
              @mouseup.stop="() => {}"
              @mouseover="active = (i + options.filter(option => option.type !== 'Turn into').length)">
              <v-icon v-if="option.icon"
                :name="option.icon" class="w-5 h-5"/>
              <span class="truncate">{{ option.label }}</span>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, watch, PropType } from 'vue'
import Fuse from 'fuse.js'
import { BlockType, availableBlockTypes } from '@/utils/types'
import Tooltip from './elements/Tooltip.vue'

const props = defineProps({
  blockTypes: {
    type: Object as PropType<null|(string|BlockType)[]>,
    default: null,
  },
})

const emit = defineEmits([
  'setBlockType',
  'clearSearch',
])

const open = ref(false)
let openedWithSlash = false
const container = ref<HTMLDivElement|null>(null)
const menu = ref<HTMLDivElement|null>(null)

watch(open, isOpen => {
  if (!isOpen) {
    openedWithSlash = false
  }
})

document.addEventListener('click', (event:Event) => {
  // Close menu on click outside of menu
  if (!open.value) return
  if (!(container.value && container.value.contains(event.target as Node))) {
    open.value = false
    searchTerm.value = ''
    active.value = 0
  }
})

/*
Support keyboard navigation
*/
const active = ref(0)
const searchTerm = ref('')

document.addEventListener('keydown', (event:KeyboardEvent) => {
  if (!open.value) return
  if (['ArrowUp', 'ArrowDown'].includes(event.key)) {
    // Support up/down navigation with keyboard
    if (event.key === 'ArrowUp') {
      // Move up
      // Scroll to bottom of menu if at top
      active.value = active.value - 1 >= 0 ? active.value - 1 : options.value.length - 1
    } else {
      // Move down
      // Scroll to top of menu if at bottom
      active.value = active.value + 1 <= options.value.length - 1 ? active.value + 1 : 0
    }
  } else if (event.key === 'ArrowLeft' || event.key === 'ArrowRight') {
    // Left/right will exit menu
    if (searchTerm.value.length === 0) open.value = false
  } else if (event.key === 'Enter') {
    // Enter selects menu option
    event.preventDefault()
    setBlockType(options.value[active.value].blockType)
  } else if (event.key === 'Escape') {
    // Escape closes menu
    open.value = false
    searchTerm.value = ''
    active.value = 0
  } else if (event.key.match(/^([a-zA-Z]|[0-9]| )$/)) {
    // Alphanumeric searches menu
    searchTerm.value += event.key
    active.value = 0
  } else if (event.key === 'Backspace') {
    // Backspace closes menu if searchTerm is empty
    if (searchTerm.value.length === 0) open.value = false
    else searchTerm.value = searchTerm.value.slice(0, -1)
    active.value = 0
  }
})

document.addEventListener('keyup', (event:KeyboardEvent) => {
  if (!open.value) return
  if (event.key === 'Enter') {
    // Enter selects menu option
    event.preventDefault()
    event.stopPropagation()
  }
})

/*
Menu options
*/

const fuzzySearch = new Fuse(availableBlockTypes, {
  keys: ['label']
})

const options = computed(() => {
  const options = searchTerm.value === ''
    ? availableBlockTypes
    : fuzzySearch.search(searchTerm.value).map(result => result.item)
  if (props.blockTypes) return options.filter(option => props.blockTypes?.includes(option.blockType))
  else return options
})



function setBlockType (blockType:BlockType|string) {
  emit('setBlockType', blockType, searchTerm.value.length, openedWithSlash)

  searchTerm.value = ''
  open.value = false
}

defineExpose({
  open,
  openedWithSlash,
})
</script>
