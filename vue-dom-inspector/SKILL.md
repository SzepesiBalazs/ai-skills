---
name: vue-dom-inspector
description: "Add a simulated DOM inspector overlay to a Vue 3 app. Use when: building a DOM inspector, highlighting elements on hover/click, showing element details in a panel, creating a devtools-like element picker, implementing an inspect-mode toggle, visualizing the DOM tree with highlights."
argument-hint: "Description of the inspector behavior or which elements to inspect"
---

# Simulated DOM Inspector — Element Highlight in UI

## When to Use

- Adding a "pick element" / inspect mode to a Vue app
- Highlighting a DOM element with an overlay when hovered or selected
- Showing element tag, classes, dimensions, or computed styles in a side panel
- Building a lightweight in-app DevTools experience

## Architecture Overview

```
┌─────────────────────────────────────────────┐
│  App                                        │
│  ┌────────────────────┐  ┌───────────────┐  │
│  │   Page Content     │  │  Inspector    │  │
│  │                    │  │  Panel        │  │
│  │  ┌──────────┐     │  │  tag: <div>   │  │
│  │  │ hovered  │─────│──│▶ classes: ... │  │
│  │  │ element  │     │  │  size: WxH    │  │
│  │  └──────────┘     │  │  ...          │  │
│  │                    │  └───────────────┘  │
│  └────────────────────┘                     │
│  ┌──────────────────────────────────────┐   │
│  │  Highlight Overlay (absolute/fixed)  │   │
│  └──────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
```

Three pieces:
1. **InspectorOverlay** — a positioned `<div>` drawn over the hovered/selected element
2. **InspectorPanel** — shows metadata (tag, id, classes, size) for the selected element
3. **useInspector composable** — manages inspect mode, hover tracking, and element selection

## Composable — `useInspector`

Create `src/composables/useInspector.ts`:

```ts
import { ref, readonly, onBeforeUnmount } from 'vue'

export interface ElementInfo {
  tagName: string
  id: string
  classList: string[]
  rect: DOMRect
  computedStyle: CSSStyleDeclaration
}

export function useInspector() {
  const inspecting = ref(false)
  const hoveredInfo = ref<ElementInfo | null>(null)
  const selectedInfo = ref<ElementInfo | null>(null)
  const overlayRect = ref<DOMRect | null>(null)

  function getElementInfo(el: HTMLElement): ElementInfo {
    return {
      tagName: el.tagName.toLowerCase(),
      id: el.id,
      classList: Array.from(el.classList),
      rect: el.getBoundingClientRect(),
      computedStyle: window.getComputedStyle(el),
    }
  }

  function onMouseMove(e: MouseEvent) {
    if (!inspecting.value) return
    const el = document.elementFromPoint(e.clientX, e.clientY) as HTMLElement | null
    if (!el || el.closest('[data-inspector-ignore]')) {
      hoveredInfo.value = null
      overlayRect.value = null
      return
    }
    const info = getElementInfo(el)
    hoveredInfo.value = info
    overlayRect.value = info.rect
  }

  function onMouseDown(e: MouseEvent) {
    if (!inspecting.value) return
    e.preventDefault()
    e.stopPropagation()
    const el = document.elementFromPoint(e.clientX, e.clientY) as HTMLElement | null
    if (!el || el.closest('[data-inspector-ignore]')) return
    selectedInfo.value = getElementInfo(el)
  }

  function startInspecting() {
    inspecting.value = true
    document.addEventListener('mousemove', onMouseMove, true)
    document.addEventListener('mousedown', onMouseDown, true)
  }

  function stopInspecting() {
    inspecting.value = false
    hoveredInfo.value = null
    overlayRect.value = null
    document.removeEventListener('mousemove', onMouseMove, true)
    document.removeEventListener('mousedown', onMouseDown, true)
  }

  function toggleInspecting() {
    inspecting.value ? stopInspecting() : startInspecting()
  }

  onBeforeUnmount(() => stopInspecting())

  return {
    inspecting: readonly(inspecting),
    hoveredInfo: readonly(hoveredInfo),
    selectedInfo: readonly(selectedInfo),
    overlayRect: readonly(overlayRect),
    startInspecting,
    stopInspecting,
    toggleInspecting,
  }
}
```

**Key conventions:**
- Use `data-inspector-ignore` attribute on inspector UI itself to avoid self-inspection
- Capture-phase listeners (`true`) so inspect clicks don't trigger app actions
- `onBeforeUnmount` cleanup to remove global listeners
- `readonly()` on exposed refs to prevent external mutation

## Overlay Component — `InspectorOverlay.vue`

```vue
<template>
  <div
    v-if="rect"
    data-inspector-ignore
    class="pointer-events-none fixed z-[9999] border-2 border-blue-500 bg-blue-500/10 transition-all duration-75"
    :style="overlayStyle"
  >
    <span
      class="absolute -top-6 left-0 rounded bg-blue-600 px-1.5 py-0.5 text-xs text-white whitespace-nowrap"
    >
      {{ label }}
    </span>
  </div>
</template>

<script lang="ts">
import { computed } from 'vue'

export default {
  props: {
    rect: { type: DOMRect, default: null },
    label: { type: String, default: '' },
  },
  setup(props) {
    const overlayStyle = computed(() => {
      if (!props.rect) return {}
      return {
        top: `${props.rect.top}px`,
        left: `${props.rect.left}px`,
        width: `${props.rect.width}px`,
        height: `${props.rect.height}px`,
      }
    })

    return { overlayStyle }
  },
}
</script>
```

**Key conventions:**
- `pointer-events-none` so the overlay doesn't intercept mouse events
- `fixed` positioning to align with `getBoundingClientRect()` values
- `z-[9999]` to sit above all app content
- A floating tag-name badge above the highlight box
- `transition-all duration-75` for smooth movement between elements

## Inspector Panel — `InspectorPanel.vue`

```vue
<template>
  <aside
    v-if="info"
    data-inspector-ignore
    class="fixed bottom-4 right-4 z-[9999] w-72 rounded-lg border border-gray-200 bg-white p-4 shadow-lg text-sm font-mono"
  >
    <h3 class="mb-2 text-xs font-semibold uppercase tracking-wide text-gray-500">
      Inspector
    </h3>
    <p>
      <span class="text-purple-600">&lt;{{ info.tagName }}&gt;</span>
      <span v-if="info.id" class="ml-1 text-orange-500">#{{ info.id }}</span>
    </p>
    <p v-if="info.classList.length" class="mt-1 text-green-600">
      .{{ info.classList.join(' .') }}
    </p>
    <p class="mt-2 text-gray-500">
      {{ Math.round(info.rect.width) }} × {{ Math.round(info.rect.height) }}px
    </p>
  </aside>
</template>

<script lang="ts">
import type { ElementInfo } from '../composables/useInspector'

export default {
  props: {
    info: { type: Object as () => ElementInfo | null, default: null },
  },
  setup() {
    return {}
  },
}
</script>
```

## Wiring It Together

In the view or layout that enables inspection:

```vue
<template>
  <div>
    <!-- Toggle button -->
    <button
      data-inspector-ignore
      class="fixed bottom-4 left-4 z-[9999] rounded-full p-2 shadow-lg"
      :class="inspecting ? 'bg-blue-600 text-white' : 'bg-white text-gray-700'"
      @click="toggleInspecting"
    >
      <!-- Crosshair / inspect icon -->
      <svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5" fill="none" viewBox="0 0 24 24" stroke="currentColor">
        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
          d="M15 15l-2 5L9 9l11 4-5 2zm0 0l5 5M7.188 2.239l.777 2.897M5.136 7.965l-2.898-.777M13.95 4.05l-2.122 2.122m-5.657 5.656l-2.12 2.122" />
      </svg>
    </button>

    <!-- App content goes here -->
    <RouterView />

    <!-- Inspector UI -->
    <InspectorOverlay
      :rect="overlayRect"
      :label="hoveredInfo?.tagName ?? ''"
    />
    <InspectorPanel :info="selectedInfo" />
  </div>
</template>

<script lang="ts">
import { useInspector } from '../composables/useInspector'
import InspectorOverlay from '../components/InspectorOverlay.vue'
import InspectorPanel from '../components/InspectorPanel.vue'

export default {
  components: { InspectorOverlay, InspectorPanel },
  setup() {
    const {
      inspecting,
      hoveredInfo,
      selectedInfo,
      overlayRect,
      toggleInspecting,
    } = useInspector()

    return { inspecting, hoveredInfo, selectedInfo, overlayRect, toggleInspecting }
  },
}
</script>
```

## Handling Scroll

`getBoundingClientRect()` returns viewport-relative values which pair correctly with `position: fixed`. If the app scrolls while inspecting, update the overlay on scroll:

```ts
// Inside useInspector — add to startInspecting / stopInspecting
document.addEventListener('scroll', onMouseMove, true)
document.removeEventListener('scroll', onMouseMove, true)
```

## Keyboard Shortcut

Toggle inspect mode with a keyboard shortcut (e.g., `Ctrl+Shift+I` or a custom key):

```ts
function onKeyDown(e: KeyboardEvent) {
  if (e.ctrlKey && e.shiftKey && e.key === 'C') {
    e.preventDefault()
    toggleInspecting()
  }
}

// Add/remove in startInspecting/stopInspecting or mount/unmount
document.addEventListener('keydown', onKeyDown)
```

## Checklist

- [ ] `data-inspector-ignore` on all inspector UI elements
- [ ] Capture-phase listeners to intercept before app handlers
- [ ] Cleanup listeners in `onBeforeUnmount`
- [ ] `pointer-events-none` on the overlay
- [ ] `position: fixed` + `z-[9999]` for overlay and panel
- [ ] Handle scroll updates if the page scrolls during inspection
