---
name: vue-git-graph
description: "Build a visual Git history graph in a Vue 3 app. Use when: visualizing git history, simulating commits and branches, rendering a commit graph, building a git timeline, creating a branch visualization, showing version control history as an interactive SVG graph, simulating merge workflows."
argument-hint: "Description of the git graph behavior or which scenarios to simulate"
---

# Visual Git History Graph — Simulated Commits & Branches

## When to Use

- Visualizing a git commit history as an interactive graph
- Simulating commits, branches, merges in the browser (no real `.git`)
- Building an educational tool to teach branching strategies
- Rendering an SVG-based commit timeline with branch lanes

## Architecture Overview

```
┌──────────────────────────────────────────────────────────┐
│  GitGraphView (page-level view)                          │
│  ┌─────────────────────────┐  ┌───────────────────────┐  │
│  │  ControlPanel            │  │  CommitDetailPanel    │  │
│  │  [Add Commit]            │  │  id: abc123           │  │
│  │  [Create Branch]         │  │  message: "feat: ..." │  │
│  │  [Merge Branch]          │  │  branch: main         │  │
│  │  [Checkout ▾]            │  │  parents: [def456]    │  │
│  └─────────────────────────┘  └───────────────────────┘  │
│  ┌──────────────────────────────────────────────────────┐ │
│  │  GitGraph (SVG)                                      │ │
│  │                                                      │ │
│  │  ● main ──●──●──●──────●  ← CommitNode + lines      │ │
│  │                \       /                             │ │
│  │                 ●──●──●   feature                    │ │
│  │                                                      │ │
│  │  BranchLabel tags shown at branch heads              │ │
│  └──────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘
```

Four pieces:
1. **GitGraph** — SVG container that renders commit nodes and connecting lines
2. **CommitNode** — a circle + label for a single commit
3. **BranchLabel** — colored tag at the head of each branch
4. **useGitGraph composable** — manages simulated commit/branch state and graph layout

## Data Model

Create `src/types/git-graph.ts`:

```ts
export interface Commit {
  id: string
  message: string
  branchName: string
  parentIds: string[]
  timestamp: number
}

export interface Branch {
  name: string
  color: string
  headCommitId: string
}

export interface GitGraphState {
  commits: Commit[]
  branches: Branch[]
  head: string          // current branch name
  selectedCommitId: string | null
}
```

**Key conventions:**
- `id` is a short random hex string (use `crypto.randomUUID().slice(0, 7)`)
- `parentIds` is an array — merge commits have two parents, normal commits have one
- `branchName` on a commit records which branch was active when the commit was created
- `head` tracks the currently checked-out branch name

## Composable — `useGitGraph`

Create `src/composables/useGitGraph.ts`:

```ts
import { ref, readonly, computed } from 'vue'
import type { Commit, Branch, GitGraphState } from '@/types/git-graph'

const BRANCH_COLORS = [
  '#6366f1', // indigo
  '#10b981', // emerald
  '#f59e0b', // amber
  '#ef4444', // red
  '#8b5cf6', // violet
  '#06b6d4', // cyan
  '#ec4899', // pink
  '#84cc16', // lime
]

function generateId(): string {
  return crypto.randomUUID().slice(0, 7)
}

export interface LayoutNode {
  commit: Commit
  x: number
  y: number
  color: string
}

export interface LayoutEdge {
  fromX: number
  fromY: number
  toX: number
  toY: number
  color: string
}

export function useGitGraph() {
  const commits = ref<Commit[]>([])
  const branches = ref<Branch[]>([])
  const head = ref<string>('main')
  const selectedCommitId = ref<string | null>(null)

  // --- Graph layout constants ---
  const NODE_RADIUS = 12
  const ROW_HEIGHT = 60
  const COL_WIDTH = 80
  const PADDING = 40

  // --- Computed: branch column assignments ---
  const branchColumns = computed<Record<string, number>>(() => {
    const cols: Record<string, number> = {}
    branches.value.forEach((b, i) => {
      cols[b.name] = i
    })
    return cols
  })

  // --- Computed: color map ---
  const branchColorMap = computed<Record<string, string>>(() => {
    const map: Record<string, string> = {}
    branches.value.forEach((b) => {
      map[b.name] = b.color
    })
    return map
  })

  // --- Computed: layout nodes ---
  const layoutNodes = computed<LayoutNode[]>(() => {
    return commits.value.map((commit, index) => {
      const col = branchColumns.value[commit.branchName] ?? 0
      return {
        commit,
        x: PADDING + col * COL_WIDTH,
        y: PADDING + index * ROW_HEIGHT,
        color: branchColorMap.value[commit.branchName] ?? BRANCH_COLORS[0],
      }
    })
  })

  // --- Computed: layout edges ---
  const layoutEdges = computed<LayoutEdge[]>(() => {
    const commitIndexMap = new Map<string, number>()
    commits.value.forEach((c, i) => commitIndexMap.set(c.id, i))

    const edges: LayoutEdge[] = []
    for (const node of layoutNodes.value) {
      for (const parentId of node.commit.parentIds) {
        const parentIdx = commitIndexMap.get(parentId)
        if (parentIdx === undefined) continue
        const parentNode = layoutNodes.value[parentIdx]
        edges.push({
          fromX: node.x,
          fromY: node.y,
          toX: parentNode.x,
          toY: parentNode.y,
          color: node.color,
        })
      }
    }
    return edges
  })

  // --- Computed: selected commit details ---
  const selectedCommit = computed<Commit | null>(() => {
    if (!selectedCommitId.value) return null
    return commits.value.find((c) => c.id === selectedCommitId.value) ?? null
  })

  // --- Computed: SVG canvas dimensions ---
  const graphWidth = computed(() => {
    const maxCol = Math.max(branches.value.length - 1, 0)
    return PADDING * 2 + maxCol * COL_WIDTH + NODE_RADIUS * 2
  })

  const graphHeight = computed(() => {
    return PADDING * 2 + Math.max(commits.value.length - 1, 0) * ROW_HEIGHT + NODE_RADIUS * 2
  })

  // --- Init with a default main branch + initial commit ---
  function init() {
    const initialId = generateId()
    branches.value = [
      { name: 'main', color: BRANCH_COLORS[0], headCommitId: initialId },
    ]
    commits.value = [
      {
        id: initialId,
        message: 'Initial commit',
        branchName: 'main',
        parentIds: [],
        timestamp: Date.now(),
      },
    ]
    head.value = 'main'
    selectedCommitId.value = null
  }

  // --- Add a commit to the current branch ---
  function addCommit(message: string) {
    const currentBranch = branches.value.find((b) => b.name === head.value)
    if (!currentBranch) return

    const newId = generateId()
    const newCommit: Commit = {
      id: newId,
      message,
      branchName: head.value,
      parentIds: [currentBranch.headCommitId],
      timestamp: Date.now(),
    }
    commits.value.push(newCommit)
    currentBranch.headCommitId = newId
  }

  // --- Create a new branch from current HEAD ---
  function createBranch(name: string) {
    const exists = branches.value.find((b) => b.name === name)
    if (exists) return

    const currentBranch = branches.value.find((b) => b.name === head.value)
    if (!currentBranch) return

    const colorIndex = branches.value.length % BRANCH_COLORS.length
    branches.value.push({
      name,
      color: BRANCH_COLORS[colorIndex],
      headCommitId: currentBranch.headCommitId,
    })
    head.value = name
  }

  // --- Merge a source branch into the current branch ---
  function mergeBranch(sourceBranchName: string) {
    const currentBranch = branches.value.find((b) => b.name === head.value)
    const sourceBranch = branches.value.find((b) => b.name === sourceBranchName)
    if (!currentBranch || !sourceBranch) return
    if (currentBranch.name === sourceBranch.name) return

    const newId = generateId()
    const mergeCommit: Commit = {
      id: newId,
      message: `Merge '${sourceBranchName}' into '${head.value}'`,
      branchName: head.value,
      parentIds: [currentBranch.headCommitId, sourceBranch.headCommitId],
      timestamp: Date.now(),
    }
    commits.value.push(mergeCommit)
    currentBranch.headCommitId = newId
  }

  // --- Checkout an existing branch ---
  function checkout(branchName: string) {
    const branch = branches.value.find((b) => b.name === branchName)
    if (!branch) return
    head.value = branchName
  }

  // --- Select a commit for detail view ---
  function selectCommit(commitId: string | null) {
    selectedCommitId.value = commitId
  }

  return {
    commits: readonly(commits),
    branches: readonly(branches),
    head: readonly(head),
    selectedCommitId: readonly(selectedCommitId),
    selectedCommit,
    layoutNodes,
    layoutEdges,
    graphWidth,
    graphHeight,
    NODE_RADIUS,
    init,
    addCommit,
    createBranch,
    mergeBranch,
    checkout,
    selectCommit,
  }
}
```

**Key conventions:**
- `readonly()` on exposed refs to prevent external mutation
- Layout computed properties derive positions from branch column index and commit order
- `init()` must be called in the view's `onBeforeMount` to seed default state
- Merge commits have two `parentIds`, rendered as two incoming edges in the graph

## Graph Component — `GitGraph.vue`

Create `src/components/GitGraph.vue`:

```vue
<template>
  <svg
    :width="graphWidth"
    :height="graphHeight"
    class="block"
  >
    <!-- Edges (lines between commits) -->
    <path
      v-for="(edge, i) in layoutEdges"
      :key="'edge-' + i"
      :d="edgePath(edge)"
      :stroke="edge.color"
      stroke-width="2"
      fill="none"
    />

    <!-- Commit nodes -->
    <g
      v-for="node in layoutNodes"
      :key="node.commit.id"
      class="cursor-pointer"
      @click="$emit('select-commit', node.commit.id)"
    >
      <circle
        :cx="node.x"
        :cy="node.y"
        :r="nodeRadius"
        :fill="node.color"
        :stroke="node.commit.id === selectedCommitId ? '#fff' : 'transparent'"
        stroke-width="3"
      />
      <text
        :x="node.x + nodeRadius + 8"
        :y="node.y + 4"
        class="text-xs fill-current text-gray-700"
        font-size="12"
      >
        {{ node.commit.id }} — {{ node.commit.message }}
      </text>
    </g>

    <!-- Branch labels at head commits -->
    <g v-for="branch in branches" :key="'label-' + branch.name">
      <template v-if="branchHeadNode(branch.name)">
        <rect
          :x="branchHeadNode(branch.name)!.x - 30"
          :y="branchHeadNode(branch.name)!.y - nodeRadius - 22"
          rx="4"
          ry="4"
          width="60"
          height="18"
          :fill="branchColorMap[branch.name]"
        />
        <text
          :x="branchHeadNode(branch.name)!.x"
          :y="branchHeadNode(branch.name)!.y - nodeRadius - 9"
          text-anchor="middle"
          font-size="10"
          fill="white"
          font-weight="bold"
        >
          {{ branch.name }}
        </text>
      </template>
    </g>
  </svg>
</template>

<script lang="ts">
import type { LayoutEdge, LayoutNode } from '@/composables/useGitGraph'
import type { Branch } from '@/types/git-graph'

export default {
  props: {
    layoutNodes: { type: Array as () => LayoutNode[], required: true },
    layoutEdges: { type: Array as () => LayoutEdge[], required: true },
    branches: { type: Array as () => Branch[], required: true },
    branchColorMap: { type: Object as () => Record<string, string>, required: true },
    graphWidth: { type: Number, required: true },
    graphHeight: { type: Number, required: true },
    nodeRadius: { type: Number, required: true },
    selectedCommitId: { type: String, default: null },
  },
  emits: ['select-commit'],
  setup(props) {
    function edgePath(edge: LayoutEdge): string {
      if (edge.fromX === edge.toX) {
        return `M ${edge.fromX} ${edge.fromY} L ${edge.toX} ${edge.toY}`
      }
      const midY = (edge.fromY + edge.toY) / 2
      return `M ${edge.fromX} ${edge.fromY} C ${edge.fromX} ${midY}, ${edge.toX} ${midY}, ${edge.toX} ${edge.toY}`
    }

    function branchHeadNode(branchName: string): LayoutNode | undefined {
      const branch = props.branches.find((b) => b.name === branchName)
      if (!branch) return undefined
      return props.layoutNodes.find((n) => n.commit.id === branch.headCommitId)
    }

    return { edgePath, branchHeadNode }
  },
}
</script>
```

**Key conventions:**
- SVG `<path>` with cubic Bézier curves for cross-branch edges, straight lines for same-branch
- Emit `select-commit` event upward — parent view handles selection state
- Branch labels render as `<rect>` + `<text>` positioned above the head commit node
- No `<style>` block — text styling via SVG attributes and Tailwind classes

## Commit Detail Panel — `CommitDetailPanel.vue`

```vue
<template>
  <div
    v-if="commit"
    class="rounded-lg border border-gray-200 bg-white p-4 shadow-sm"
  >
    <h3 class="mb-2 text-sm font-semibold text-gray-500 uppercase tracking-wide">Commit Details</h3>
    <dl class="space-y-2 text-sm">
      <div>
        <dt class="font-medium text-gray-600">ID</dt>
        <dd class="font-mono text-indigo-600">{{ commit.id }}</dd>
      </div>
      <div>
        <dt class="font-medium text-gray-600">Message</dt>
        <dd>{{ commit.message }}</dd>
      </div>
      <div>
        <dt class="font-medium text-gray-600">Branch</dt>
        <dd>{{ commit.branchName }}</dd>
      </div>
      <div>
        <dt class="font-medium text-gray-600">Parents</dt>
        <dd class="font-mono">{{ commit.parentIds.length ? commit.parentIds.join(', ') : '(root)' }}</dd>
      </div>
      <div>
        <dt class="font-medium text-gray-600">Time</dt>
        <dd>{{ new Date(commit.timestamp).toLocaleString() }}</dd>
      </div>
    </dl>
  </div>
</template>

<script lang="ts">
import type { Commit } from '@/types/git-graph'

export default {
  props: {
    commit: { type: Object as () => Commit | null, default: null },
  },
  setup() {
    return {}
  },
}
</script>
```

## Page View — `GitGraphView.vue`

Create `src/views/GitGraphView.vue`:

```vue
<template>
  <div class="mx-auto max-w-6xl">
    <h1 class="mb-6 text-2xl font-bold text-gray-800">Git History Graph</h1>

    <div class="mb-4 flex flex-wrap items-end gap-4">
      <!-- Add Commit -->
      <div>
        <label class="mb-1 block text-xs font-medium text-gray-600">Commit message</label>
        <div class="flex gap-2">
          <input
            v-model="commitMessage"
            type="text"
            class="rounded border border-gray-300 px-3 py-1.5 text-sm focus:border-indigo-500 focus:outline-none"
            placeholder="feat: add feature"
            @keyup.enter="handleAddCommit"
          />
          <button
            class="rounded bg-indigo-600 px-3 py-1.5 text-sm font-medium text-white hover:bg-indigo-700"
            @click="handleAddCommit"
          >
            Commit
          </button>
        </div>
      </div>

      <!-- Create Branch -->
      <div>
        <label class="mb-1 block text-xs font-medium text-gray-600">New branch name</label>
        <div class="flex gap-2">
          <input
            v-model="newBranchName"
            type="text"
            class="rounded border border-gray-300 px-3 py-1.5 text-sm focus:border-indigo-500 focus:outline-none"
            placeholder="feature/login"
            @keyup.enter="handleCreateBranch"
          />
          <button
            class="rounded bg-emerald-600 px-3 py-1.5 text-sm font-medium text-white hover:bg-emerald-700"
            @click="handleCreateBranch"
          >
            Branch
          </button>
        </div>
      </div>

      <!-- Checkout -->
      <div>
        <label class="mb-1 block text-xs font-medium text-gray-600">Checkout</label>
        <select
          :value="head"
          class="rounded border border-gray-300 px-3 py-1.5 text-sm focus:border-indigo-500 focus:outline-none"
          @change="checkout(($event.target as HTMLSelectElement).value)"
        >
          <option v-for="b in branches" :key="b.name" :value="b.name">
            {{ b.name }}
          </option>
        </select>
      </div>

      <!-- Merge -->
      <div>
        <label class="mb-1 block text-xs font-medium text-gray-600">Merge into {{ head }}</label>
        <div class="flex gap-2">
          <select
            v-model="mergeBranchName"
            class="rounded border border-gray-300 px-3 py-1.5 text-sm focus:border-indigo-500 focus:outline-none"
          >
            <option v-for="b in otherBranches" :key="b.name" :value="b.name">
              {{ b.name }}
            </option>
          </select>
          <button
            class="rounded bg-amber-600 px-3 py-1.5 text-sm font-medium text-white hover:bg-amber-700"
            :disabled="!mergeBranchName"
            @click="handleMerge"
          >
            Merge
          </button>
        </div>
      </div>
    </div>

    <!-- Current branch indicator -->
    <p class="mb-4 text-sm text-gray-500">
      HEAD →
      <span class="font-semibold text-indigo-600">{{ head }}</span>
    </p>

    <!-- Graph + Detail side-by-side -->
    <div class="flex gap-6">
      <div class="overflow-auto rounded-lg border border-gray-200 bg-white p-4">
        <GitGraph
          :layout-nodes="layoutNodes"
          :layout-edges="layoutEdges"
          :branches="branches"
          :branch-color-map="branchColorMap"
          :graph-width="graphWidth"
          :graph-height="graphHeight"
          :node-radius="NODE_RADIUS"
          :selected-commit-id="selectedCommitId"
          @select-commit="selectCommit"
        />
      </div>
      <div class="w-64 shrink-0">
        <CommitDetailPanel :commit="selectedCommit" />
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import { ref, computed, onBeforeMount } from 'vue'
import { useGitGraph } from '@/composables/useGitGraph'
import GitGraph from '@/components/GitGraph.vue'
import CommitDetailPanel from '@/components/CommitDetailPanel.vue'

export default {
  components: { GitGraph, CommitDetailPanel },
  setup() {
    const {
      commits,
      branches,
      head,
      selectedCommitId,
      selectedCommit,
      layoutNodes,
      layoutEdges,
      graphWidth,
      graphHeight,
      NODE_RADIUS,
      init,
      addCommit,
      createBranch,
      mergeBranch,
      checkout,
      selectCommit,
    } = useGitGraph()

    const commitMessage = ref('')
    const newBranchName = ref('')
    const mergeBranchName = ref('')

    const branchColorMap = computed<Record<string, string>>(() => {
      const map: Record<string, string> = {}
      branches.value.forEach((b) => { map[b.name] = b.color })
      return map
    })

    const otherBranches = computed(() =>
      branches.value.filter((b) => b.name !== head.value),
    )

    function handleAddCommit() {
      if (!commitMessage.value.trim()) return
      addCommit(commitMessage.value.trim())
      commitMessage.value = ''
    }

    function handleCreateBranch() {
      if (!newBranchName.value.trim()) return
      createBranch(newBranchName.value.trim())
      newBranchName.value = ''
    }

    function handleMerge() {
      if (!mergeBranchName.value) return
      mergeBranch(mergeBranchName.value)
      mergeBranchName.value = ''
    }

    onBeforeMount(() => {
      init()
    })

    return {
      commits,
      branches,
      head,
      selectedCommitId,
      selectedCommit,
      layoutNodes,
      layoutEdges,
      graphWidth,
      graphHeight,
      NODE_RADIUS,
      branchColorMap,
      otherBranches,
      commitMessage,
      newBranchName,
      mergeBranchName,
      handleAddCommit,
      handleCreateBranch,
      handleMerge,
      checkout,
      selectCommit,
    }
  },
}
</script>
```

## Integration with Vue Router

Add a route in `src/router/index.ts`:

```ts
import GitGraphView from '../views/GitGraphView.vue'

// Inside the routes array:
{
  path: '/git-graph',
  name: 'git-graph',
  component: GitGraphView,
},
```

Add a nav link in `App.vue`:

```vue
<RouterLink class="rounded-md px-3 py-2 text-sm font-medium text-white" :to="{ name: 'git-graph' }">
  Git Graph
</RouterLink>
```

## File Checklist

| File | Purpose |
|------|---------|
| `src/types/git-graph.ts` | `Commit`, `Branch`, `GitGraphState` interfaces |
| `src/composables/useGitGraph.ts` | State management, layout computation, actions |
| `src/components/GitGraph.vue` | SVG graph renderer (nodes + edges + branch labels) |
| `src/components/CommitDetailPanel.vue` | Selected commit metadata display |
| `src/views/GitGraphView.vue` | Page view with controls + graph + detail panel |
| `src/router/index.ts` | Add `/git-graph` route |
| `App.vue` | Add nav link |
