<template>
  <div
    ref="sortableRef"
    @wheel="onWheel"
  >
    <TransitionGroup
      :name="transitionGroupName"
    >
      <div
        v-for="(item, index) in items"
        :key="itemKey(item, index)"
        :style="styles[index]"
        :class="itemClass"
        @mousedown="onDragStart($event, index)"
        @touchstart="onDragStart($event, index)"
      >
        <slot
          name="item"
          :item="item"
          :index="index"
        />
      </div>
    </TransitionGroup>
  </div>
</template>

<script setup>
import { computed, ref, watch } from 'vue'
import { clamp, getEventPosition } from '../modules/utils/mouse.js'
import { getRelativeEventPosition } from '../modules/utils/mouse.js'
import { hasClassUpToParent, isBetween, moveArrayElement } from '../modules/utils/utils.js'

const props = defineProps({
  direction: {
    type: String,
    default: 'vertical'
  },
  handle: {
    type: String,
    default: null
  },
  itemClass: {
    type: String,
    default: null
  },
  animationDuration: {
    type: Number,
    default: 200
  },
  animationEasing: {
    type: String,
    default: 'ease'
  },
  autosScrollEnabled: {
    type: Boolean,
    default: true
  },
  disabled: {
    type: Boolean,
    default: false
  },
  transitionGroupName: {
    type: String,
    default: null
  },
  itemKey: {
    type: Function,
    default: (item, index) => `index-${index}`
  }
})

const emits = defineEmits([
  'start',
  'end',
  'change'
])

const items = defineModel()

const isVertical = computed(() => props.direction == 'vertical')

const styles = ref(items.value.map(() => ({})))

const sortableRef = ref(null)

const sortableHeight = ref(0)
const sortableWidth = ref(0)
const sortableSize = computed(() => isVertical.value ? sortableHeight.value : sortableWidth.value)

const position = ref({
  x: 0,
  y: 0
})

const isDragging = ref(false)

const initialIndex = ref(null)
const scrollIndex = ref(null)
const elementDragOffset = ref(0)

let target = null
let animationRequest = null

// Relative to the viewport.
let initialDragPosition = null
let currentDragPosition = null

function onWheel(event) {
  if (isDragging.value) {
    event.preventDefault()
  }
}

const transitionStyle = computed(() => ({
  transition: `transform ${props.animationDuration}ms ${props.animationEasing}`
}))

function getStyle(index) {
  if (initialIndex.value === null) {
    return {}
  }

  if (index == initialIndex.value) {
    return {
      position: 'relative',
      zIndex: 1000
    }
  }

  const style = {
    ...transitionStyle.value,
    transform: 'translate3d(0, 0, 0)'
  }

  if (isBetween(index, initialIndex.value, scrollIndex.value)) {
    const transform = (initialIndex.value - scrollIndex.value < 0) ? '-100%' : '100%'
    const transformX = isVertical.value ? 0 : transform
    const transformY = isVertical.value ? transform : 0
    style.transform = `translate3d(${transformX}, ${transformY}, 0)`
  }

  return style
}

watch(scrollIndex, () => {
  styles.value = items.value.map((item, index) => getStyle(index))
  emits('change', { oldIndex: initialIndex.value, newIndex: scrollIndex.value })
})

function onDragStart(event, index) {
  if (props.disabled) {
    return
  }

  // Ignore right click.
  if (event.button && event.button != 0) {
    return
  }

  if (props.handle && !hasClassUpToParent(event.target, event.currentTarget, props.handle)) {
    return
  }

  event.preventDefault()

  document.addEventListener('mousemove', onDrag)
  document.addEventListener('mouseup', onDragStop)

  document.addEventListener('touchmove', onDrag)
  document.addEventListener('touchend', onDragStop)

  isDragging.value = true
  initialIndex.value = index

  sortableHeight.value = sortableRef.value.scrollHeight
  sortableWidth.value = sortableRef.value.scrollWidth

  target = event.currentTarget

  if (props.autosScrollEnabled) {
    animationRequest = requestAnimationFrame(animate)
  }

  initialDragPosition = getEventPosition(event)

  const { x, y } = getRelativeEventPosition(event, target)
  elementDragOffset.value = isVertical.value ? y : x

  onDrag(event)

  emits('start', { index })
}

function onDrag(event) {
  currentDragPosition = getEventPosition(event)
  const { x, y } = getRelativeEventPosition(event, sortableRef.value)
  const size = isVertical.value ? target.offsetHeight : target.offsetWidth
  const padding = size / 2

  const offset = padding - elementDragOffset.value
  const xOffset = isVertical.value ? 0 : offset
  const yOffset = isVertical.value ? offset : 0

  position.value.x = clamp(x + xOffset, sortableRef.value.scrollLeft + padding, sortableRef.value.offsetWidth + sortableRef.value.scrollLeft - padding)
  position.value.y = clamp(y + yOffset, sortableRef.value.scrollTop + padding, sortableRef.value.offsetHeight + sortableRef.value.scrollTop - padding)

  moveTarget()
}

function onDragStop() {
  document.removeEventListener('mousemove', onDrag)
  document.removeEventListener('mouseup', onDragStop)

  document.removeEventListener('touchmove', onDrag)
  document.removeEventListener('touchend', onDragStop)

  target.style.transform = ''
  target = null

  if (animationRequest) {
    cancelAnimationFrame(animationRequest)
    animationRequest = null
  }

  items.value = initialIndex.value != scrollIndex.value
    ? moveArrayElement(items.value, initialIndex.value, scrollIndex.value)
    : items.value

  emits('end', {
    oldIndex: initialIndex.value,
    newIndex: scrollIndex.value,
    value: items.value
  })

  isDragging.value = false
  initialIndex.value = null
  scrollIndex.value = null
}

function moveTarget() {
  const size = isVertical.value ? target.offsetHeight : target.offsetWidth
  const coordinate = isVertical.value ? position.value.y : position.value.x

  scrollIndex.value = Math.floor(coordinate / size)

  const transform = coordinate - size * initialIndex.value - size / 2
  const transformX = isVertical.value ? 0 : transform
  const transformY = isVertical.value ? transform : 0

  target.style.transform = `translate3d(${transformX}px, ${transformY}px, 0)`
}

let previousTimeStamp = 0
const checkInterval = 10

function animate(timestamp) {
  const elapsed = timestamp - previousTimeStamp

  if (elapsed >= checkInterval) {
    autosScroll()
    previousTimeStamp = timestamp
  }

  animationRequest = requestAnimationFrame(animate)
}

function autosScroll() {
  if (!isDragging.value) {
    return
  }

  const size = isVertical.value ? target.offsetHeight : target.offsetWidth
  const padding = size / 2

  const SCROLL_MARGIN = 50
  const SCROLL_PADDING = SCROLL_MARGIN + size / 2
  const SCROLL_SPEED = 10

  // Relative to the view (top = 0 and left = 0, even if scrolled).
  const relativeCoordinate = isVertical.value
    ? position.value.y - sortableRef.value.scrollTop
    : position.value.x - sortableRef.value.scrollLeft

  const scrollKey = isVertical.value
    ? 'scrollTop'
    : 'scrollLeft'

  const viewportSize = isVertical.value
    ? sortableRef.value.offsetHeight
    : sortableRef.value.offsetWidth

  const initialScroll = sortableRef.value[scrollKey]

  let direction = 1
  let acceleration = 1

  if ((sortableSize.value - sortableRef.value[scrollKey]) != viewportSize && relativeCoordinate > (viewportSize - SCROLL_PADDING)) {
    acceleration = clamp(relativeCoordinate + padding - (viewportSize - SCROLL_MARGIN), 0, SCROLL_MARGIN) / SCROLL_MARGIN
    direction = 1
  } else if (sortableRef.value[scrollKey] > 0 && relativeCoordinate < SCROLL_PADDING) {
    acceleration = 1 - clamp(relativeCoordinate - padding, 0, SCROLL_MARGIN) / SCROLL_MARGIN
    direction = -1
  } else {
    return
  }

  const dragDelta = isVertical.value
    ? currentDragPosition.y - initialDragPosition.y
    : currentDragPosition.x - initialDragPosition.x

  const dragDirection = dragDelta > 0 ? 1 : -1

  if (Math.abs(dragDelta) < 5 || dragDirection != direction) {
    return
  }

  sortableRef.value[scrollKey] += direction * acceleration * SCROLL_SPEED
  const scrollOffset = direction * Math.abs(initialScroll - sortableRef.value[scrollKey])

  const xScrollOffset = isVertical.value ? 0 : scrollOffset
  const yScrollOffset = isVertical.value ? scrollOffset : 0
  position.value.x = clamp(position.value.x + xScrollOffset, sortableRef.value.scrollLeft + padding, sortableRef.value.offsetWidth + sortableRef.value.scrollLeft - padding)
  position.value.y = clamp(position.value.y + yScrollOffset, sortableRef.value.scrollTop + padding, sortableRef.value.offsetHeight + sortableRef.value.scrollTop - padding)
  moveTarget()
}
</script>
