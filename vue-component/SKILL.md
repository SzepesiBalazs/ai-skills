---
name: vue-component
description: "Create Vue 3 components and views with Composition API and Tailwind CSS. Use when: creating Vue components, building Vue views, writing Vue templates, creating Vue page layouts, building a Vue navbar."
argument-hint: "Description of the component or view to create"
---

# Vue 3 Component Patterns

## When to Use

- Creating new Vue views (pages) or reusable components
- Need the Composition API pattern with TypeScript

## Component Pattern

All components use `<script lang="ts">` with the Composition API `setup()` function:

```vue
<template>
  <!-- Template content -->
</template>

<script lang="ts">
import { ref, computed, onBeforeMount } from 'vue'

export default {
  setup() {
    // Reactive state
    const data = ref(null)

    // Computed properties
    const derivedValue = computed(() => /* ... */)

    // Lifecycle
    onBeforeMount(() => {
      // Load data
    })

    // Methods
    const handleAction = () => { /* ... */ }

    // Expose to template
    return { data, derivedValue, handleAction }
  },
}
</script>
```

**Key conventions:**
- Use `<script lang="ts">` (NOT `<script setup>`)
- Export default with `setup()` function
- Use `ref()` for reactive state, `computed()` for derived values
- Use `onBeforeMount()` for data loading
- Explicitly return all template-bound values

## Styling

- Use Tailwind CSS utility classes directly in templates
- No `<style>` blocks — all styling via Tailwind classes
- Responsive design: `sm:`, `md:`, `lg:` prefixes

## App.vue Layout Pattern

The root App.vue contains:
1. A responsive nav bar with `RouterLink` navigation
2. A main content area with `RouterView`

```vue
<template>
  <header>
    <div class="wrapper">
      <nav class="bg-indigo-800">
        <div class="mx-auto max-w-7xl px-2 sm:px-6 lg:px-8">
          <div class="relative flex h-16 items-center justify-between">
            <!-- Mobile menu button -->
            <div class="absolute inset-y-0 left-0 flex items-center sm:hidden">
              <button type="button" class="relative inline-flex items-center justify-center rounded-md p-2 text-gray-400 hover:bg-gray-700 hover:text-white">
                <span class="sr-only">Menu</span>
                <!-- Hamburger SVG icon -->
              </button>
            </div>
            <!-- Desktop nav links -->
            <div class="flex flex-1 items-center justify-center sm:items-stretch sm:justify-start">
              <div class="hidden sm:ml-6 sm:block">
                <div class="flex space-x-4">
                  <RouterLink class="rounded-md px-3 py-2 text-sm font-medium text-white" :to="{ name: 'home' }">Home</RouterLink>
                  <!-- Add more RouterLinks as needed -->
                </div>
              </div>
            </div>
          </div>
        </div>
        <!-- Mobile menu -->
        <div class="sm:hidden" id="mobile-menu">
          <div class="space-y-1 px-2 pb-3 pt-2">
            <RouterLink class="block rounded-md px-3 py-2 text-base font-medium text-white" to="/">Home</RouterLink>
          </div>
        </div>
      </nav>
    </div>
  </header>
  <div class="pt-6 pl-10 pr-10 pb-8 bg-indigo-100 min-h-screen">
    <RouterView></RouterView>
  </div>
</template>

<script lang="ts">
import { RouterLink, RouterView } from 'vue-router'

export default {
  components: { RouterLink, RouterView },
  setup() {
    return {}
  },
}
</script>
```

## Testing Pattern

Tests use Vitest + @vue/test-utils in `src/components/__tests__/`:

```ts
import { describe, it, expect } from 'vitest'
import { mount } from '@vue/test-utils'
import MyComponent from '../../views/MyComponent.vue'

describe('MyComponent.vue', () => {
  it('renders correctly', () => {
    const wrapper = mount(MyComponent)
    expect(wrapper.vm.someProperty).toBeDefined()
  })
})
```
