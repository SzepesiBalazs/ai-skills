---
name: vue-css-playground
description: "Build a live CSS style editor (mini CodePen) in a Vue 3 app. Use when: creating a CSS playground, building a live style editor, teaching flexbox or grid layout, demonstrating responsive design, experimenting with CSS properties interactively, building a CodePen-like editor, visualizing CSS layout models."
argument-hint: "Description of the CSS concepts or layout to demonstrate"
---

# Live CSS Style Editor — Mini CodePen

## When to Use

- Building an interactive CSS playground where users edit styles and see results live
- Teaching CSS layout concepts (flexbox, grid) with visual feedback
- Demonstrating responsive design with breakpoint previews
- Creating a CodePen-like split-pane editor for HTML + CSS

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────────┐
│  CssPlaygroundView (page-level view)                             │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │  PresetPicker                                            │    │
│  │  [Flexbox] [Grid] [Responsive] [Custom]                  │    │
│  └──────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌──────────────────────────┐  ┌─────────────────────────────┐  │
│  │  EditorPane              │  │  PreviewPane                 │  │
│  │  ┌─────────────────────┐ │  │  ┌───────────────────────┐  │  │
│  │  │  HTML Editor        │ │  │  │  Live Preview         │  │  │
│  │  │  <textarea>         │ │  │  │  (rendered output)    │  │  │
│  │  └─────────────────────┘ │  │  │                       │  │  │
│  │  ┌─────────────────────┐ │  │  └───────────────────────┘  │  │
│  │  │  CSS Editor         │ │  │                             │  │
│  │  │  <textarea>         │ │  │  BreakpointBar              │  │
│  │  └─────────────────────┘ │  │  [Mobile] [Tablet] [Desktop]│  │
│  └──────────────────────────┘  └─────────────────────────────┘  │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │  PropertyPanel (contextual CSS property quick-controls)  │    │
│  │  display: [flex ▾]  flex-direction: [row ▾]  gap: [___]  │    │
│  └──────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────┘
```

Five pieces:

1. **EditorPane** — side-by-side HTML and CSS `<textarea>` editors with syntax-highlighted labels
2. **PreviewPane** — sandboxed live preview that renders user HTML + CSS, with resizable width
3. **PresetPicker** — buttons to load preset layout scenarios (flexbox, grid, responsive)
4. **PropertyPanel** — contextual quick-controls for common CSS properties (dropdowns, sliders)
5. **useCssPlayground composable** — manages editor state, presets, preview rendering, breakpoint simulation

## Data Model

Create `src/types/css-playground.ts`:

```ts
export interface CssPreset {
  name: string
  description: string
  html: string
  css: string
  /** Which property panel to show: 'flex' | 'grid' | 'responsive' | 'none' */
  panelType: 'flex' | 'grid' | 'responsive' | 'none'
}

export interface BreakpointOption {
  name: string
  label: string
  width: number
}
```

**Key conventions:**

- `panelType` drives which quick-controls appear in the PropertyPanel
- Presets contain both the starting HTML and CSS — loading a preset replaces both editors

## Preset Scenarios

Create `src/data/css-presets.ts`:

```ts
import type { CssPreset, BreakpointOption } from '@/types/css-playground'

export const flexboxPreset: CssPreset = {
  name: 'Flexbox',
  description: 'Explore flexbox alignment, direction, wrapping, and gap',
  panelType: 'flex',
  html: `<div class="container">
  <div class="box">1</div>
  <div class="box">2</div>
  <div class="box">3</div>
  <div class="box">4</div>
  <div class="box">5</div>
</div>`,
  css: `.container {
  display: flex;
  flex-direction: row;
  justify-content: flex-start;
  align-items: stretch;
  flex-wrap: nowrap;
  gap: 8px;
  padding: 16px;
  min-height: 200px;
  background: #f1f5f9;
  border-radius: 8px;
}

.box {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 60px;
  height: 60px;
  background: #6366f1;
  color: white;
  font-weight: bold;
  font-size: 1.2rem;
  border-radius: 6px;
}`,
}

export const gridPreset: CssPreset = {
  name: 'Grid',
  description: 'Explore CSS Grid columns, rows, gap, and placement',
  panelType: 'grid',
  html: `<div class="container">
  <div class="box">1</div>
  <div class="box">2</div>
  <div class="box">3</div>
  <div class="box">4</div>
  <div class="box">5</div>
  <div class="box">6</div>
</div>`,
  css: `.container {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  grid-template-rows: auto;
  gap: 8px;
  padding: 16px;
  min-height: 200px;
  background: #f1f5f9;
  border-radius: 8px;
}

.box {
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 16px;
  background: #10b981;
  color: white;
  font-weight: bold;
  font-size: 1.2rem;
  border-radius: 6px;
}`,
}

export const responsivePreset: CssPreset = {
  name: 'Responsive',
  description: 'Media queries that adapt layout across breakpoints',
  panelType: 'responsive',
  html: `<div class="container">
  <header class="header">Header</header>
  <nav class="sidebar">Sidebar</nav>
  <main class="content">Main Content</main>
  <footer class="footer">Footer</footer>
</div>`,
  css: `.container {
  display: grid;
  grid-template-columns: 1fr;
  grid-template-rows: auto;
  gap: 8px;
  padding: 16px;
  min-height: 300px;
  font-family: sans-serif;
}

.header, .sidebar, .content, .footer {
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 16px;
  border-radius: 6px;
  font-weight: bold;
  color: white;
}

.header  { background: #6366f1; }
.sidebar { background: #f59e0b; }
.content { background: #10b981; min-height: 120px; }
.footer  { background: #ef4444; }

/* Tablet */
@media (min-width: 640px) {
  .container {
    grid-template-columns: 200px 1fr;
    grid-template-rows: auto 1fr auto;
  }
  .header  { grid-column: 1 / -1; }
  .footer  { grid-column: 1 / -1; }
}

/* Desktop */
@media (min-width: 1024px) {
  .container {
    grid-template-columns: 250px 1fr;
  }
}`,
}

export const allPresets: CssPreset[] = [flexboxPreset, gridPreset, responsivePreset]

export const breakpointOptions: BreakpointOption[] = [
  { name: 'mobile', label: 'Mobile', width: 375 },
  { name: 'tablet', label: 'Tablet', width: 768 },
  { name: 'desktop', label: 'Desktop', width: 1200 },
  { name: 'full', label: 'Full Width', width: 0 },
]
```

## Composable — `useCssPlayground`

Create `src/composables/useCssPlayground.ts`:

```ts
import { ref, readonly, computed, watch } from 'vue'
import type { CssPreset, BreakpointOption } from '@/types/css-playground'
import { flexboxPreset, breakpointOptions } from '@/data/css-presets'

export function useCssPlayground() {
  const htmlSource = ref('')
  const cssSource = ref('')
  const activePreset = ref<CssPreset>(flexboxPreset)
  const activeBreakpoint = ref<BreakpointOption>(
    breakpointOptions.find((b) => b.name === 'full')!,
  )

  // --- Computed: combined preview document ---
  const previewDocument = computed(() => {
    return `<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <style>${cssSource.value}</style>
</head>
<body style="margin:0;padding:0;">
  ${htmlSource.value}
</body>
</html>`
  })

  // --- Computed: preview container width ---
  const previewWidth = computed(() => {
    if (activeBreakpoint.value.width === 0) return '100%'
    return `${activeBreakpoint.value.width}px`
  })

  // --- Load a preset ---
  function loadPreset(preset: CssPreset) {
    activePreset.value = preset
    htmlSource.value = preset.html
    cssSource.value = preset.css
  }

  // --- Update CSS from property panel controls ---
  function updateCssProperty(property: string, value: string) {
    const regex = new RegExp(`(${escapeRegExp(property)}\\s*:\\s*)([^;]+)(;)`)
    if (regex.test(cssSource.value)) {
      cssSource.value = cssSource.value.replace(regex, `$1${value}$3`)
    }
  }

  function escapeRegExp(str: string): string {
    return str.replace(/[.*+?^${}()|[\]\\]/g, '\\$&')
  }

  // --- Extract current value of a CSS property from .container rule ---
  function getCssProperty(property: string): string {
    const regex = new RegExp(`${escapeRegExp(property)}\\s*:\\s*([^;]+);`)
    const match = cssSource.value.match(regex)
    return match ? match[1].trim() : ''
  }

  // --- Set breakpoint ---
  function setBreakpoint(bp: BreakpointOption) {
    activeBreakpoint.value = bp
  }

  // --- Initialize with default preset ---
  function init() {
    loadPreset(flexboxPreset)
  }

  return {
    htmlSource,
    cssSource,
    activePreset: readonly(activePreset),
    activeBreakpoint: readonly(activeBreakpoint),
    previewDocument,
    previewWidth,
    loadPreset,
    updateCssProperty,
    getCssProperty,
    setBreakpoint,
    init,
  }
}
```

**Key conventions:**

- `htmlSource` and `cssSource` are writable refs — bound directly to `<textarea v-model>`
- `previewDocument` builds a full HTML document string used as the `srcdoc` of an `<iframe>`
- `updateCssProperty()` does regex replacement in the CSS string — used by the PropertyPanel quick-controls
- `getCssProperty()` extracts the current value so dropdowns/sliders stay in sync with manual edits
- `readonly()` on preset/breakpoint refs to prevent external mutation

## Preview Pane Component — `PreviewPane.vue`

Create `src/components/PreviewPane.vue`:

```vue
<template>
  <div class="flex flex-col items-center">
    <div
      class="overflow-hidden rounded-lg border border-gray-300 bg-white transition-all duration-300"
      :style="{ width: previewWidth }"
    >
      <iframe
        ref="previewFrame"
        :srcdoc="previewDocument"
        sandbox="allow-scripts"
        class="h-96 w-full border-0"
        title="CSS Preview"
      ></iframe>
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent } from 'vue'

export default defineComponent({
  props: {
    previewDocument: { type: String, required: true },
    previewWidth: { type: String, required: true },
  },
  setup() {
    return {}
  },
})
</script>
```

**Key conventions:**

- Render preview in an `<iframe>` with `srcdoc` — isolates user CSS from the app's Tailwind styles
- `sandbox="allow-scripts"` restricts the iframe but still allows CSS animations/transitions
- Container width is driven by the breakpoint selection, enabling responsive previews
- Never use `allow-same-origin` in sandbox to prevent iframe from accessing parent DOM

## Editor Pane Component — `EditorPane.vue`

Create `src/components/EditorPane.vue`:

```vue
<template>
  <div class="flex flex-col gap-4">
    <div>
      <label class="mb-1 block text-sm font-semibold text-gray-700">HTML</label>
      <textarea
        :value="htmlSource"
        @input="$emit('update:htmlSource', ($event.target as HTMLTextAreaElement).value)"
        class="h-40 w-full rounded-lg border border-gray-300 bg-gray-900 p-3 font-mono text-sm text-green-400 focus:border-indigo-500 focus:outline-none focus:ring-1 focus:ring-indigo-500"
        spellcheck="false"
      ></textarea>
    </div>
    <div>
      <label class="mb-1 block text-sm font-semibold text-gray-700">CSS</label>
      <textarea
        :value="cssSource"
        @input="$emit('update:cssSource', ($event.target as HTMLTextAreaElement).value)"
        class="h-56 w-full rounded-lg border border-gray-300 bg-gray-900 p-3 font-mono text-sm text-blue-400 focus:border-indigo-500 focus:outline-none focus:ring-1 focus:ring-indigo-500"
        spellcheck="false"
      ></textarea>
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent } from 'vue'

export default defineComponent({
  props: {
    htmlSource: { type: String, required: true },
    cssSource: { type: String, required: true },
  },
  emits: ['update:htmlSource', 'update:cssSource'],
  setup() {
    return {}
  },
})
</script>
```

**Key conventions:**

- Use `:value` + `@input` emit pattern (not `v-model`) for parent-controlled state
- Dark themed textareas with monospace font for a code-editor feel
- `spellcheck="false"` to avoid red underlines in code

## Property Panel Component — `PropertyPanel.vue`

Create `src/components/PropertyPanel.vue`:

```vue
<template>
  <div
    v-if="panelType !== 'none'"
    class="rounded-lg border border-gray-200 bg-white p-4 shadow-sm"
  >
    <h3 class="mb-3 text-sm font-semibold text-gray-600 uppercase tracking-wide">
      Quick Controls
    </h3>

    <!-- Flex controls -->
    <div v-if="panelType === 'flex'" class="flex flex-wrap gap-4">
      <div>
        <label class="mb-1 block text-xs text-gray-500">flex-direction</label>
        <select
          :value="getCssProperty('flex-direction')"
          @change="updateCssProperty('flex-direction', ($event.target as HTMLSelectElement).value)"
          class="rounded border border-gray-300 px-2 py-1 text-sm"
        >
          <option value="row">row</option>
          <option value="row-reverse">row-reverse</option>
          <option value="column">column</option>
          <option value="column-reverse">column-reverse</option>
        </select>
      </div>
      <div>
        <label class="mb-1 block text-xs text-gray-500">justify-content</label>
        <select
          :value="getCssProperty('justify-content')"
          @change="updateCssProperty('justify-content', ($event.target as HTMLSelectElement).value)"
          class="rounded border border-gray-300 px-2 py-1 text-sm"
        >
          <option value="flex-start">flex-start</option>
          <option value="flex-end">flex-end</option>
          <option value="center">center</option>
          <option value="space-between">space-between</option>
          <option value="space-around">space-around</option>
          <option value="space-evenly">space-evenly</option>
        </select>
      </div>
      <div>
        <label class="mb-1 block text-xs text-gray-500">align-items</label>
        <select
          :value="getCssProperty('align-items')"
          @change="updateCssProperty('align-items', ($event.target as HTMLSelectElement).value)"
          class="rounded border border-gray-300 px-2 py-1 text-sm"
        >
          <option value="stretch">stretch</option>
          <option value="flex-start">flex-start</option>
          <option value="flex-end">flex-end</option>
          <option value="center">center</option>
          <option value="baseline">baseline</option>
        </select>
      </div>
      <div>
        <label class="mb-1 block text-xs text-gray-500">flex-wrap</label>
        <select
          :value="getCssProperty('flex-wrap')"
          @change="updateCssProperty('flex-wrap', ($event.target as HTMLSelectElement).value)"
          class="rounded border border-gray-300 px-2 py-1 text-sm"
        >
          <option value="nowrap">nowrap</option>
          <option value="wrap">wrap</option>
          <option value="wrap-reverse">wrap-reverse</option>
        </select>
      </div>
      <div>
        <label class="mb-1 block text-xs text-gray-500">gap</label>
        <input
          type="range"
          min="0"
          max="40"
          :value="parseInt(getCssProperty('gap')) || 0"
          @input="updateCssProperty('gap', ($event.target as HTMLInputElement).value + 'px')"
          class="w-24"
        />
        <span class="ml-1 text-xs text-gray-500">{{ getCssProperty('gap') }}</span>
      </div>
    </div>

    <!-- Grid controls -->
    <div v-if="panelType === 'grid'" class="flex flex-wrap gap-4">
      <div>
        <label class="mb-1 block text-xs text-gray-500">grid-template-columns</label>
        <select
          :value="getCssProperty('grid-template-columns')"
          @change="updateCssProperty('grid-template-columns', ($event.target as HTMLSelectElement).value)"
          class="rounded border border-gray-300 px-2 py-1 text-sm"
        >
          <option value="repeat(2, 1fr)">2 columns</option>
          <option value="repeat(3, 1fr)">3 columns</option>
          <option value="repeat(4, 1fr)">4 columns</option>
          <option value="1fr 2fr">1fr 2fr</option>
          <option value="2fr 1fr">2fr 1fr</option>
          <option value="1fr 1fr 2fr">1fr 1fr 2fr</option>
        </select>
      </div>
      <div>
        <label class="mb-1 block text-xs text-gray-500">gap</label>
        <input
          type="range"
          min="0"
          max="40"
          :value="parseInt(getCssProperty('gap')) || 0"
          @input="updateCssProperty('gap', ($event.target as HTMLInputElement).value + 'px')"
          class="w-24"
        />
        <span class="ml-1 text-xs text-gray-500">{{ getCssProperty('gap') }}</span>
      </div>
    </div>

    <!-- Responsive controls: just breakpoint switcher info -->
    <div v-if="panelType === 'responsive'" class="text-sm text-gray-600">
      Use the breakpoint bar above the preview to test different screen widths.
      Edit the <code class="rounded bg-gray-100 px-1">@media</code> rules in the CSS editor.
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent, type PropType } from 'vue'

export default defineComponent({
  props: {
    panelType: { type: String as PropType<'flex' | 'grid' | 'responsive' | 'none'>, required: true },
    getCssProperty: { type: Function as PropType<(prop: string) => string>, required: true },
    updateCssProperty: { type: Function as PropType<(prop: string, val: string) => void>, required: true },
  },
  setup() {
    return {}
  },
})
</script>
```

**Key conventions:**

- Panel content switches based on `panelType` from the active preset
- Dropdowns use `:value` + `@change` bound to `getCssProperty` / `updateCssProperty` from the composable
- Range sliders provide tactile control for numeric properties like `gap`
- Controls read from and write back to the CSS source string — always in sync with manual editor changes

## Breakpoint Bar Component — `BreakpointBar.vue`

Create `src/components/BreakpointBar.vue`:

```vue
<template>
  <div class="flex items-center gap-2">
    <span class="text-xs font-medium text-gray-500">Preview width:</span>
    <button
      v-for="bp in breakpoints"
      :key="bp.name"
      class="rounded-md px-3 py-1 text-xs font-medium transition-colors"
      :class="
        activeName === bp.name
          ? 'bg-indigo-600 text-white'
          : 'bg-gray-200 text-gray-700 hover:bg-gray-300'
      "
      @click="$emit('select', bp)"
    >
      {{ bp.label }}
      <span v-if="bp.width > 0" class="ml-1 text-[10px] opacity-70">{{ bp.width }}px</span>
    </button>
  </div>
</template>

<script lang="ts">
import { defineComponent, type PropType } from 'vue'
import type { BreakpointOption } from '@/types/css-playground'

export default defineComponent({
  props: {
    breakpoints: { type: Array as PropType<BreakpointOption[]>, required: true },
    activeName: { type: String, required: true },
  },
  emits: ['select'],
  setup() {
    return {}
  },
})
</script>
```

## Preset Picker Component — `PresetPicker.vue`

Create `src/components/PresetPicker.vue`:

```vue
<template>
  <div class="flex flex-wrap items-center gap-2">
    <span class="text-sm font-medium text-gray-600">Presets:</span>
    <button
      v-for="preset in presets"
      :key="preset.name"
      class="rounded-lg px-4 py-2 text-sm font-medium transition-colors"
      :class="
        activeName === preset.name
          ? 'bg-indigo-600 text-white shadow-md'
          : 'bg-white text-gray-700 border border-gray-300 hover:bg-gray-50'
      "
      @click="$emit('select', preset)"
    >
      {{ preset.name }}
    </button>
  </div>
</template>

<script lang="ts">
import { defineComponent, type PropType } from 'vue'
import type { CssPreset } from '@/types/css-playground'

export default defineComponent({
  props: {
    presets: { type: Array as PropType<CssPreset[]>, required: true },
    activeName: { type: String, required: true },
  },
  emits: ['select'],
  setup() {
    return {}
  },
})
</script>
```

## Page View — `CssPlaygroundView.vue`

Create `src/views/CssPlaygroundView.vue`:

```vue
<template>
  <div class="mx-auto max-w-7xl space-y-6">
    <h1 class="text-2xl font-bold text-gray-800">Live CSS Playground</h1>

    <PresetPicker
      :presets="allPresets"
      :activeName="activePreset.name"
      @select="loadPreset"
    />

    <div class="grid grid-cols-1 gap-6 lg:grid-cols-2">
      <!-- Editor side -->
      <EditorPane
        :htmlSource="htmlSource"
        :cssSource="cssSource"
        @update:htmlSource="htmlSource = $event"
        @update:cssSource="cssSource = $event"
      />

      <!-- Preview side -->
      <div class="space-y-3">
        <BreakpointBar
          :breakpoints="breakpointOptions"
          :activeName="activeBreakpoint.name"
          @select="setBreakpoint"
        />
        <PreviewPane
          :previewDocument="previewDocument"
          :previewWidth="previewWidth"
        />
      </div>
    </div>

    <PropertyPanel
      :panelType="activePreset.panelType"
      :getCssProperty="getCssProperty"
      :updateCssProperty="updateCssProperty"
    />
  </div>
</template>

<script lang="ts">
import { onBeforeMount } from 'vue'
import { useCssPlayground } from '@/composables/useCssPlayground'
import { allPresets, breakpointOptions } from '@/data/css-presets'
import EditorPane from '@/components/EditorPane.vue'
import PreviewPane from '@/components/PreviewPane.vue'
import PresetPicker from '@/components/PresetPicker.vue'
import BreakpointBar from '@/components/BreakpointBar.vue'
import PropertyPanel from '@/components/PropertyPanel.vue'

export default {
  components: { EditorPane, PreviewPane, PresetPicker, BreakpointBar, PropertyPanel },
  setup() {
    const {
      htmlSource,
      cssSource,
      activePreset,
      activeBreakpoint,
      previewDocument,
      previewWidth,
      loadPreset,
      updateCssProperty,
      getCssProperty,
      setBreakpoint,
      init,
    } = useCssPlayground()

    onBeforeMount(() => init())

    return {
      htmlSource,
      cssSource,
      activePreset,
      activeBreakpoint,
      previewDocument,
      previewWidth,
      loadPreset,
      updateCssProperty,
      getCssProperty,
      setBreakpoint,
      allPresets,
      breakpointOptions,
    }
  },
}
</script>
```

## Routing

Add to `src/router/index.ts`:

```ts
{
  path: '/css-playground',
  name: 'css-playground',
  component: () => import('../views/CssPlaygroundView.vue'),
},
```

Add a nav link in `App.vue`:

```vue
<RouterLink class="rounded-md px-3 py-2 text-sm font-medium text-white" :to="{ name: 'css-playground' }">CSS Playground</RouterLink>
```

## Testing Pattern

Tests in `src/components/__tests__/CssPlayground.test.ts`:

```ts
import { describe, it, expect } from 'vitest'
import { useCssPlayground } from '@/composables/useCssPlayground'

describe('useCssPlayground', () => {
  it('initializes with the flexbox preset', () => {
    const { htmlSource, cssSource, activePreset, init } = useCssPlayground()
    init()
    expect(activePreset.value.name).toBe('Flexbox')
    expect(htmlSource.value).toContain('container')
    expect(cssSource.value).toContain('display: flex')
  })

  it('switches presets', () => {
    const { cssSource, activePreset, loadPreset, init } = useCssPlayground()
    init()
    const gridPreset = { name: 'Grid', description: '', html: '<div></div>', css: 'display: grid;', panelType: 'grid' as const }
    loadPreset(gridPreset)
    expect(activePreset.value.name).toBe('Grid')
    expect(cssSource.value).toContain('grid')
  })

  it('updates a CSS property value', () => {
    const { cssSource, updateCssProperty, init } = useCssPlayground()
    init()
    updateCssProperty('flex-direction', 'column')
    expect(cssSource.value).toContain('flex-direction: column;')
  })

  it('reads a CSS property value', () => {
    const { getCssProperty, init } = useCssPlayground()
    init()
    expect(getCssProperty('flex-direction')).toBe('row')
  })

  it('builds a complete preview document', () => {
    const { previewDocument, init } = useCssPlayground()
    init()
    expect(previewDocument.value).toContain('<!DOCTYPE html>')
    expect(previewDocument.value).toContain('<style>')
    expect(previewDocument.value).toContain('display: flex')
  })
})
```

## Security Notes

- Preview uses `<iframe srcdoc>` with `sandbox="allow-scripts"` — **never** add `allow-same-origin` as it would let user CSS/JS access the parent Vue app's DOM
- User input is rendered only inside the sandboxed iframe, not in the parent document
- No `eval()` or `innerHTML` in the parent app — all user content stays in the iframe sandbox
