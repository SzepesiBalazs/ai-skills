---
name: npm-dependency-graph
description: "Build a dependency graph visualizer for npm/yarn/pnpm projects in a Vue 3 app. Use when: visualizing package dependencies, building a dep graph explorer, teaching npm/yarn/pnpm concepts, showing transitive dependencies, detecting duplicate or conflicting packages, exploring lockfile structure, understanding hoisting and workspace dependencies."
argument-hint: "package.json content or list of packages to visualize, plus the package manager (npm/yarn/pnpm)"
---

# Dependency Graph Visualizer — npm/yarn/pnpm

## When to Use

- Visualizing direct and transitive dependencies from a `package.json` or lockfile
- Teaching npm/yarn/pnpm concepts: hoisting, peer dependencies, workspaces, lockfiles
- Detecting duplicate packages, version conflicts, or circular dependencies
- Building an interactive explorer that lets users drill into dependency trees

## Architecture Overview

```
┌────────────────────────────────────────────────────────────────────────┐
│  DependencyGraphView (page-level view)                                 │
│                                                                        │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │  ToolBar                                                        │  │
│  │  [npm ▾]  [Search...]  [Show dev deps ☑]  [Depth: 3 ▾]         │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│  ┌──────────────────────────────────┐  ┌───────────────────────────┐  │
│  │  GraphCanvas                     │  │  PackageDetailPanel       │  │
│  │                                  │  │                           │  │
│  │   [root]──▶[react]──▶[loose-env] │  │  react@18.2.0             │  │
│  │        │                         │  │  ─────────────────────    │  │
│  │        └──▶[react-dom]           │  │  License: MIT             │  │
│  │                  │               │  │  Size: 6.9 kB             │  │
│  │                  └──▶[scheduler] │  │  Required by: 2 packages  │  │
│  │                                  │  │                           │  │
│  │  (SVG radial tier layout)        │  │  Dependencies (3)         │  │
│  │                                  │  │  > loose-envify ^1.1.0    │  │
│  └──────────────────────────────────┘  │  > object-assign ^4.1.1   │  │
│                                        │  > scheduler ^0.23.0      │  │
│  ┌─────────────────────────────────┐   │                           │  │
│  │  LegendBar                      │   │  [View on npm ↗]          │  │
│  │  ● Direct  ● Transitive  ● Dev  │   └───────────────────────────┘  │
│  └─────────────────────────────────┘                                   │
└────────────────────────────────────────────────────────────────────────┘
```

Five pieces:

1. **GraphCanvas** — SVG radial-tier graph that renders package nodes and dependency edges
2. **PackageDetailPanel** — sidebar showing metadata, version, license, and dependents for the selected node
3. **ToolBar** — package manager selector, search, depth limiter, dev-dep toggle
4. **LegendBar** — color key for node kinds (direct, transitive, dev, peer)
5. **useDependencyGraph composable** — parses package data, builds the graph, manages selection, filtering, and layout

## Package Manager Concepts

### npm
- `package.json` declares `dependencies`, `devDependencies`, `peerDependencies`, `optionalDependencies`
- `package-lock.json` (v3) records exact resolved versions and integrity hashes for the full tree
- **Hoisting**: npm flattens nested deps into `node_modules/` root when versions are compatible; only one copy is installed
- `npm ls --json` outputs the full dependency tree with version resolution

### yarn (classic v1 / berry v2+)
- `yarn.lock` uses a deterministic, content-addressed format
- **Zero-installs (berry)**: `.yarn/cache` stores zip archives; no `node_modules` required
- `yarn why <package>` explains why a package is installed and which packages required it
- Workspaces link packages in a monorepo via `"workspaces": ["packages/*"]`

### pnpm
- Uses a **content-addressable store** (`~/.pnpm-store`): each version stored once globally, symlinked
- `node_modules/.pnpm/<pkg>@<version>/node_modules/` — nested layout prevents phantom dependencies
- `pnpm-lock.yaml` records importers (workspaces) and snapshots separately
- `pnpm why <package>` and `pnpm ls --depth Infinity --json` expose the full graph

### Key shared concepts

| Concept | Explanation |
|---|---|
| **Direct dependency** | Listed in your `package.json` |
| **Transitive dependency** | Required by one of your dependencies, not by you directly |
| **Peer dependency** | Expected to be provided by the consuming project, not installed automatically |
| **Hoisting** | Moving nested deps up to `node_modules/` root to avoid duplication |
| **Phantom dependency** | A package used in code that is not declared in `package.json` (pnpm prevents this) |
| **Lockfile** | Exact snapshot of resolved versions; always commit it |
| **Workspace** | Monorepo feature where multiple local packages share a single `node_modules/` |

## Data Model

Create `src/types/dependency-graph.ts`:

```ts
export type PackageManager = 'npm' | 'yarn' | 'pnpm'

export type DepKind = 'direct' | 'transitive' | 'dev' | 'peer' | 'optional'

export interface PackageNode {
  id: string          // "<name>@<version>"
  name: string
  version: string
  kind: DepKind
  /** depth from the root package (0 = root, 1 = direct dep) */
  depth: number
  license?: string
  /** estimated install size in bytes */
  size?: number
  /** IDs of PackageNodes that directly require this node */
  requiredBy: string[]
}

export interface DepEdge {
  source: string   // PackageNode.id
  target: string   // PackageNode.id
  kind: DepKind
}

export interface GraphData {
  nodes: PackageNode[]
  edges: DepEdge[]
  root: string     // PackageNode.id of the root package
}

export interface LayoutNode extends PackageNode {
  x: number
  y: number
  vx: number
  vy: number
}
```

**Key conventions:**
- Node `id` is `"<name>@<version>"` — unique and stable across re-renders
- `depth` drives the radial layout tier; nodes at depth 0 are drawn in the center
- `requiredBy` enables rendering "who needs this?" in the detail panel without re-traversing edges

## Sample Data Builder

Create `src/data/buildSampleGraph.ts`:

```ts
import type { GraphData, PackageNode, DepEdge } from '@/types/dependency-graph'

interface PackageJsonDep {
  [name: string]: string  // name → version range
}

interface PackageJsonLike {
  name: string
  version: string
  dependencies?: PackageJsonDep
  devDependencies?: PackageJsonDep
  peerDependencies?: PackageJsonDep
}

/**
 * Builds a two-level GraphData from a package.json-like object.
 * In a real app, replace with an actual lockfile parser or `npm ls --json` output.
 */
export function buildGraphFromPackageJson(pkg: PackageJsonLike): GraphData {
  const nodes: PackageNode[] = []
  const edges: DepEdge[] = []
  const rootId = `${pkg.name}@${pkg.version}`

  nodes.push({
    id: rootId,
    name: pkg.name,
    version: pkg.version,
    kind: 'direct',
    depth: 0,
    requiredBy: [],
  })

  function addDeps(
    deps: PackageJsonDep | undefined,
    kind: 'direct' | 'dev' | 'peer',
    depth: number,
  ) {
    if (!deps) return
    for (const [name, versionRange] of Object.entries(deps)) {
      const version = versionRange.replace(/^[\^~>=<]/, '')
      const nodeId = `${name}@${version}`
      if (!nodes.find((n) => n.id === nodeId)) {
        nodes.push({
          id: nodeId,
          name,
          version,
          kind,
          depth,
          requiredBy: [rootId],
        })
      } else {
        const existing = nodes.find((n) => n.id === nodeId)!
        if (!existing.requiredBy.includes(rootId)) {
          existing.requiredBy.push(rootId)
        }
      }
      edges.push({ source: rootId, target: nodeId, kind })
    }
  }

  addDeps(pkg.dependencies, 'direct', 1)
  addDeps(pkg.devDependencies, 'dev', 1)
  addDeps(pkg.peerDependencies, 'peer', 1)

  return { nodes, edges, root: rootId }
}

/** Realistic sample graph matching a small React app */
export const sampleGraph: GraphData = buildGraphFromPackageJson({
  name: 'my-app',
  version: '1.0.0',
  dependencies: {
    react: '18.2.0',
    'react-dom': '18.2.0',
    axios: '1.6.7',
    zustand: '4.5.2',
  },
  devDependencies: {
    vite: '5.2.0',
    typescript: '5.4.4',
    vitest: '1.4.0',
    '@vitejs/plugin-react': '4.2.1',
  },
  peerDependencies: {
    react: '>=18',
  },
})
```

## Composable — `useDependencyGraph`

Create `src/composables/useDependencyGraph.ts`:

```ts
import { ref, computed, readonly } from 'vue'
import type { GraphData, PackageNode, LayoutNode, DepKind, PackageManager } from '@/types/dependency-graph'
import { sampleGraph } from '@/data/buildSampleGraph'

export function useDependencyGraph() {
  const graph = ref<GraphData>(sampleGraph)
  const selectedNodeId = ref<string | null>(null)
  const packageManager = ref<PackageManager>('npm')
  const showDevDeps = ref(true)
  const maxDepth = ref(3)
  const searchQuery = ref('')

  // ── Filtered nodes ─────────────────────────────────────────────────────
  const filteredNodes = computed<PackageNode[]>(() =>
    graph.value.nodes.filter((n) => {
      if (!showDevDeps.value && n.kind === 'dev') return false
      if (n.depth > maxDepth.value) return false
      if (searchQuery.value) {
        return n.name.toLowerCase().includes(searchQuery.value.toLowerCase())
      }
      return true
    }),
  )

  const filteredNodeIds = computed(() => new Set(filteredNodes.value.map((n) => n.id)))

  const filteredEdges = computed(() =>
    graph.value.edges.filter(
      (e) => filteredNodeIds.value.has(e.source) && filteredNodeIds.value.has(e.target),
    ),
  )

  // ── Layout: radial tiers ───────────────────────────────────────────────
  const layoutNodes = computed<LayoutNode[]>(() => {
    const tiers: Map<number, PackageNode[]> = new Map()
    for (const node of filteredNodes.value) {
      const tier = tiers.get(node.depth) ?? []
      tier.push(node)
      tiers.set(node.depth, tier)
    }

    const centerX = 400
    const centerY = 300
    const radiusStep = 130
    const result: LayoutNode[] = []

    for (const [depth, nodes] of tiers) {
      const radius = depth * radiusStep
      nodes.forEach((node, i) => {
        const angle = (2 * Math.PI * i) / nodes.length - Math.PI / 2
        result.push({
          ...node,
          x: depth === 0 ? centerX : centerX + radius * Math.cos(angle),
          y: depth === 0 ? centerY : centerY + radius * Math.sin(angle),
          vx: 0,
          vy: 0,
        })
      })
    }
    return result
  })

  // ── Selection ──────────────────────────────────────────────────────────
  const selectedNode = computed<PackageNode | null>(() =>
    selectedNodeId.value
      ? (graph.value.nodes.find((n) => n.id === selectedNodeId.value) ?? null)
      : null,
  )

  function selectNode(id: string | null) {
    selectedNodeId.value = id
  }

  function loadGraph(data: GraphData) {
    graph.value = data
    selectedNodeId.value = null
  }

  // ── npm registry URL helper ────────────────────────────────────────────
  function npmUrl(name: string): string {
    // Allowlist: only valid npm package name characters to prevent open-redirect
    const safeName = name.replace(/[^a-zA-Z0-9@/._-]/g, '')
    return `https://www.npmjs.com/package/${safeName}`
  }

  return {
    graph: readonly(graph),
    filteredNodes,
    filteredEdges,
    layoutNodes,
    selectedNode,
    selectedNodeId: readonly(selectedNodeId),
    packageManager,
    showDevDeps,
    maxDepth,
    searchQuery,
    selectNode,
    loadGraph,
    npmUrl,
  }
}
```

**Key conventions:**
- `filteredNodes` / `filteredEdges` are derived — never mutate `graph` directly
- Radial tier layout: root at center, direct deps on ring 1, transitives on ring 2, etc.
- `npmUrl()` sanitizes the package name with a regex allowlist before building a URL — prevents open-redirect or XSS via crafted package names
- `readonly()` on `graph` and `selectedNodeId` to prevent external mutation

## GraphCanvas Component — `GraphCanvas.vue`

Create `src/components/GraphCanvas.vue`:

```vue
<template>
  <svg
    class="h-full w-full cursor-default select-none"
    viewBox="0 0 800 600"
    @click.self="selectNode(null)"
  >
    <!-- Edges -->
    <g class="edges">
      <line
        v-for="edge in edges"
        :key="`${edge.source}->${edge.target}`"
        :x1="nodePos(edge.source)?.x"
        :y1="nodePos(edge.source)?.y"
        :x2="nodePos(edge.target)?.x"
        :y2="nodePos(edge.target)?.y"
        :class="edgeClass(edge.kind)"
        stroke-width="1.5"
      />
    </g>

    <!-- Nodes -->
    <g class="nodes">
      <g
        v-for="node in nodes"
        :key="node.id"
        :transform="`translate(${node.x}, ${node.y})`"
        class="cursor-pointer"
        @click.stop="selectNode(node.id)"
      >
        <circle
          :r="node.depth === 0 ? 28 : 18"
          :class="nodeClass(node)"
          :stroke="selectedId === node.id ? '#f59e0b' : 'transparent'"
          stroke-width="3"
        />
        <text
          class="fill-white text-[9px] font-medium pointer-events-none"
          text-anchor="middle"
          dy="3"
        >
          {{ node.name.length > 10 ? node.name.slice(0, 9) + '…' : node.name }}
        </text>
      </g>
    </g>
  </svg>
</template>

<script lang="ts">
import { computed, defineComponent, type PropType } from 'vue'
import type { LayoutNode, DepEdge, DepKind } from '@/types/dependency-graph'

export default defineComponent({
  props: {
    nodes: { type: Array as PropType<LayoutNode[]>, required: true },
    edges: { type: Array as PropType<DepEdge[]>, required: true },
    selectedId: { type: String as PropType<string | null>, default: null },
  },
  emits: ['select'],
  setup(props, { emit }) {
    const posMap = computed(() => {
      const m = new Map<string, { x: number; y: number }>()
      for (const n of props.nodes) m.set(n.id, { x: n.x, y: n.y })
      return m
    })

    function nodePos(id: string) {
      return posMap.value.get(id)
    }

    function nodeClass(node: LayoutNode): string {
      const base = 'transition-colors'
      const colors: Record<DepKind | 'root', string> = {
        root: 'fill-slate-800',
        direct: 'fill-indigo-500',
        transitive: 'fill-sky-400',
        dev: 'fill-emerald-500',
        peer: 'fill-amber-500',
        optional: 'fill-rose-400',
      }
      const kind = node.depth === 0 ? 'root' : node.kind
      return `${base} ${colors[kind] ?? colors.direct}`
    }

    function edgeClass(kind: DepKind): string {
      const colors: Record<DepKind, string> = {
        direct: 'stroke-indigo-300',
        transitive: 'stroke-sky-200',
        dev: 'stroke-emerald-300',
        peer: 'stroke-amber-300',
        optional: 'stroke-rose-300',
      }
      return `opacity-60 ${colors[kind] ?? colors.direct}`
    }

    function selectNode(id: string | null) {
      emit('select', id)
    }

    return { nodePos, nodeClass, edgeClass, selectNode }
  },
})
</script>
```

**Key conventions:**
- Pure SVG — no external graph library dependency
- `posMap` computed from `LayoutNode[]` for O(1) edge endpoint lookup
- Root node rendered larger (`r=28`) to visually anchor the graph
- `@click.self` on `<svg>` deselects without blocking node clicks

## PackageDetailPanel Component — `PackageDetailPanel.vue`

Create `src/components/PackageDetailPanel.vue`:

```vue
<template>
  <aside
    v-if="node"
    class="flex flex-col gap-3 rounded-lg border border-gray-200 bg-white p-5 shadow-sm"
  >
    <div>
      <h2 class="text-lg font-bold text-gray-800">{{ node.name }}</h2>
      <p class="text-sm text-gray-500">{{ node.version }}</p>
    </div>

    <div class="flex flex-wrap gap-2">
      <span :class="kindBadge(node.kind)" class="rounded-full px-2.5 py-0.5 text-xs font-medium">
        {{ node.kind }}
      </span>
      <span v-if="node.license" class="rounded-full bg-gray-100 px-2.5 py-0.5 text-xs text-gray-600">
        {{ node.license }}
      </span>
    </div>

    <div v-if="node.requiredBy.length">
      <p class="mb-1 text-xs font-semibold uppercase tracking-wide text-gray-400">Required by</p>
      <ul class="space-y-0.5">
        <li
          v-for="dep in node.requiredBy"
          :key="dep"
          class="cursor-pointer text-sm text-indigo-600 hover:underline"
          @click="$emit('select', dep)"
        >
          {{ dep }}
        </li>
      </ul>
    </div>

    <div v-if="node.size" class="text-sm text-gray-500">
      Install size: {{ formatSize(node.size) }}
    </div>

    <a
      :href="npmUrl(node.name)"
      target="_blank"
      rel="noopener noreferrer"
      class="mt-auto inline-flex items-center gap-1 text-sm text-indigo-600 hover:underline"
    >
      View on npm ↗
    </a>
  </aside>

  <div
    v-else
    class="flex h-full items-center justify-center rounded-lg border border-dashed border-gray-300 p-8 text-sm text-gray-400"
  >
    Click a node to inspect it
  </div>
</template>

<script lang="ts">
import { defineComponent, type PropType } from 'vue'
import type { PackageNode, DepKind } from '@/types/dependency-graph'

export default defineComponent({
  props: {
    node: { type: Object as PropType<PackageNode | null>, default: null },
    npmUrl: { type: Function as PropType<(name: string) => string>, required: true },
  },
  emits: ['select'],
  setup() {
    function kindBadge(kind: DepKind): string {
      const map: Record<DepKind, string> = {
        direct: 'bg-indigo-100 text-indigo-700',
        transitive: 'bg-sky-100 text-sky-700',
        dev: 'bg-emerald-100 text-emerald-700',
        peer: 'bg-amber-100 text-amber-700',
        optional: 'bg-rose-100 text-rose-700',
      }
      return map[kind] ?? map.direct
    }

    function formatSize(bytes: number): string {
      if (bytes < 1024) return `${bytes} B`
      if (bytes < 1024 * 1024) return `${(bytes / 1024).toFixed(1)} kB`
      return `${(bytes / (1024 * 1024)).toFixed(1)} MB`
    }

    return { kindBadge, formatSize }
  },
})
</script>
```

## ToolBar Component — `ToolBar.vue`

Create `src/components/ToolBar.vue`:

```vue
<template>
  <div class="flex flex-wrap items-center gap-3">
    <select
      :value="packageManager"
      @change="$emit('update:packageManager', ($event.target as HTMLSelectElement).value)"
      class="rounded-lg border border-gray-300 px-3 py-1.5 text-sm"
    >
      <option value="npm">npm</option>
      <option value="yarn">yarn</option>
      <option value="pnpm">pnpm</option>
    </select>

    <input
      type="search"
      :value="searchQuery"
      placeholder="Search packages..."
      @input="$emit('update:searchQuery', ($event.target as HTMLInputElement).value)"
      class="w-48 rounded-lg border border-gray-300 px-3 py-1.5 text-sm focus:border-indigo-500 focus:outline-none focus:ring-1 focus:ring-indigo-500"
    />

    <label class="flex items-center gap-2 text-sm text-gray-600">
      Depth
      <select
        :value="maxDepth"
        @change="$emit('update:maxDepth', Number(($event.target as HTMLSelectElement).value))"
        class="rounded-lg border border-gray-300 px-2 py-1.5 text-sm"
      >
        <option :value="1">1</option>
        <option :value="2">2</option>
        <option :value="3">3</option>
        <option :value="99">All</option>
      </select>
    </label>

    <label class="flex cursor-pointer items-center gap-2 text-sm text-gray-600">
      <input
        type="checkbox"
        :checked="showDevDeps"
        @change="$emit('update:showDevDeps', ($event.target as HTMLInputElement).checked)"
        class="rounded"
      />
      Show dev deps
    </label>
  </div>
</template>

<script lang="ts">
import { defineComponent, type PropType } from 'vue'
import type { PackageManager } from '@/types/dependency-graph'

export default defineComponent({
  props: {
    packageManager: { type: String as PropType<PackageManager>, required: true },
    searchQuery: { type: String, required: true },
    maxDepth: { type: Number, required: true },
    showDevDeps: { type: Boolean, required: true },
  },
  emits: ['update:packageManager', 'update:searchQuery', 'update:maxDepth', 'update:showDevDeps'],
})
</script>
```

## LegendBar Component — `LegendBar.vue`

```vue
<template>
  <div class="flex flex-wrap items-center gap-4 text-xs">
    <span v-for="item in items" :key="item.label" class="flex items-center gap-1.5">
      <span class="inline-block h-3 w-3 rounded-full" :class="item.color"></span>
      {{ item.label }}
    </span>
  </div>
</template>

<script lang="ts">
export default {
  setup() {
    const items = [
      { label: 'Root', color: 'bg-slate-800' },
      { label: 'Direct', color: 'bg-indigo-500' },
      { label: 'Transitive', color: 'bg-sky-400' },
      { label: 'Dev', color: 'bg-emerald-500' },
      { label: 'Peer', color: 'bg-amber-500' },
      { label: 'Optional', color: 'bg-rose-400' },
    ]
    return { items }
  },
}
</script>
```

## Page View — `DependencyGraphView.vue`

Create `src/views/DependencyGraphView.vue`:

```vue
<template>
  <div class="mx-auto max-w-screen-xl space-y-4">
    <h1 class="text-2xl font-bold text-gray-800">Dependency Graph</h1>

    <ToolBar
      v-model:packageManager="packageManager"
      v-model:searchQuery="searchQuery"
      v-model:maxDepth="maxDepth"
      v-model:showDevDeps="showDevDeps"
    />

    <div class="grid grid-cols-1 gap-4 lg:grid-cols-[1fr_280px]">
      <div class="h-[560px] overflow-hidden rounded-xl border border-gray-200 bg-slate-50">
        <GraphCanvas
          :nodes="layoutNodes"
          :edges="filteredEdges"
          :selectedId="selectedNodeId"
          @select="selectNode"
        />
      </div>

      <PackageDetailPanel
        :node="selectedNode"
        :npmUrl="npmUrl"
        @select="selectNode"
      />
    </div>

    <LegendBar />
  </div>
</template>

<script lang="ts">
import { useDependencyGraph } from '@/composables/useDependencyGraph'
import GraphCanvas from '@/components/GraphCanvas.vue'
import PackageDetailPanel from '@/components/PackageDetailPanel.vue'
import ToolBar from '@/components/ToolBar.vue'
import LegendBar from '@/components/LegendBar.vue'

export default {
  components: { GraphCanvas, PackageDetailPanel, ToolBar, LegendBar },
  setup() {
    const {
      filteredEdges,
      layoutNodes,
      selectedNode,
      selectedNodeId,
      packageManager,
      showDevDeps,
      maxDepth,
      searchQuery,
      selectNode,
      npmUrl,
    } = useDependencyGraph()

    return {
      filteredEdges,
      layoutNodes,
      selectedNode,
      selectedNodeId,
      packageManager,
      showDevDeps,
      maxDepth,
      searchQuery,
      selectNode,
      npmUrl,
    }
  },
}
</script>
```

## Routing

Add to `src/router/index.ts`:

```ts
{
  path: '/dependency-graph',
  name: 'dependency-graph',
  component: () => import('../views/DependencyGraphView.vue'),
},
```

Add a nav link in `App.vue`:

```vue
<RouterLink class="rounded-md px-3 py-2 text-sm font-medium text-white" :to="{ name: 'dependency-graph' }">Dep Graph</RouterLink>
```

## Parsing Real Lockfile Data

To replace the sample data with real package trees:

### npm (`package-lock.json` v3)
```ts
import lockfile from './package-lock.json'

// lockfile.packages is a flat map: { "node_modules/react": { version, dependencies, ... } }
// Walk it to build PackageNode[] and DepEdge[]
function parseNpmLockfile(lock: typeof lockfile): GraphData { /* ... */ }
```

### pnpm (`pnpm ls --json`)
```bash
pnpm ls --depth Infinity --json > deps.json
```

### yarn berry (`yarn info --all --json`)
```bash
yarn info --all --json > deps.jsonl  # newline-delimited JSON
```

## Testing Pattern

Tests in `src/composables/__tests__/useDependencyGraph.test.ts`:

```ts
import { describe, it, expect } from 'vitest'
import { useDependencyGraph } from '@/composables/useDependencyGraph'

describe('useDependencyGraph', () => {
  it('returns all nodes from sample graph by default', () => {
    const { filteredNodes } = useDependencyGraph()
    expect(filteredNodes.value.length).toBeGreaterThan(0)
  })

  it('hides dev deps when showDevDeps is false', () => {
    const { filteredNodes, showDevDeps } = useDependencyGraph()
    showDevDeps.value = false
    expect(filteredNodes.value.every((n) => n.kind !== 'dev')).toBe(true)
  })

  it('limits nodes by maxDepth', () => {
    const { filteredNodes, maxDepth } = useDependencyGraph()
    maxDepth.value = 1
    expect(filteredNodes.value.every((n) => n.depth <= 1)).toBe(true)
  })

  it('filters nodes by search query', () => {
    const { filteredNodes, searchQuery } = useDependencyGraph()
    searchQuery.value = 'react'
    expect(filteredNodes.value.every((n) => n.name.includes('react'))).toBe(true)
  })

  it('selects and deselects a node', () => {
    const { selectedNode, selectNode } = useDependencyGraph()
    expect(selectedNode.value).toBeNull()
    selectNode('react@18.2.0')
    expect(selectedNode.value?.name).toBe('react')
    selectNode(null)
    expect(selectedNode.value).toBeNull()
  })

  it('generates a safe npm URL', () => {
    const { npmUrl } = useDependencyGraph()
    expect(npmUrl('react')).toBe('https://www.npmjs.com/package/react')
  })

  it('strips dangerous characters from npm URL', () => {
    const { npmUrl } = useDependencyGraph()
    expect(npmUrl('react<script>')).not.toContain('<script>')
  })
})
```

## Security Notes

- `npmUrl()` sanitizes package names with a regex allowlist — prevents open-redirect or XSS via crafted package names
- Never `eval()` or `innerHTML` lockfile content — parse as plain JSON only
- Package names from lockfiles are untrusted input; display them via Vue's `{{ }}` interpolation (auto-escaped)
- Do not expose file system paths in the UI

## Package Manager Cheat Sheet

| Task | npm | yarn | pnpm |
|---|---|---|---|
| Install all | `npm install` | `yarn` | `pnpm install` |
| Add package | `npm i <pkg>` | `yarn add <pkg>` | `pnpm add <pkg>` |
| Add dev dep | `npm i -D <pkg>` | `yarn add -D <pkg>` | `pnpm add -D <pkg>` |
| Remove | `npm uninstall <pkg>` | `yarn remove <pkg>` | `pnpm remove <pkg>` |
| Why installed | `npm explain <pkg>` | `yarn why <pkg>` | `pnpm why <pkg>` |
| List tree | `npm ls --depth 2` | `yarn list --depth 2` | `pnpm ls --depth 2` |
| Audit | `npm audit` | `yarn audit` | `pnpm audit` |
| Dedupe | `npm dedupe` | `yarn dedupe` | `pnpm dedupe` |
| Workspaces | `npm workspaces` | `yarn workspaces` | `pnpm -r` |
