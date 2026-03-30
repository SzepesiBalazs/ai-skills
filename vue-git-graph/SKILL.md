---
name: vue-git-graph
description: "Build a visual Git history graph simulator in a Vue 3 app. Use when: visualizing git history, simulating a real git workflow step-by-step, rendering a commit graph, building a git timeline, creating a branch visualization, showing version control history as an interactive SVG graph, simulating merge workflows, teaching git with a command log."
argument-hint: "Description of the git workflow scenario to simulate"
---

# Visual Git History Graph — Workflow Simulator

## When to Use

- Simulating a real git workflow step-by-step with visual feedback
- Teaching git branching strategies with a command log showing what happened
- Rendering an SVG-based commit graph that builds up as steps play
- Building an educational tool where users watch a workflow unfold, not type commands manually

## Architecture Overview

```
┌────────────────────────────────────────────────────────────────┐
│  GitGraphView (page-level view)                                │
│                                                                │
│  ┌─────────────────────────────────────────┐  ┌────────────┐  │
│  │  ScenarioPicker                         │  │ Controls   │  │
│  │  [Feature Branch] [Gitflow] [Hotfix]    │  │ [▶ Next]   │  │
│  │                                         │  │ [⏭ All]    │  │
│  └─────────────────────────────────────────┘  │ [↺ Reset]  │  │
│                                               └────────────┘  │
│  ┌──────────────────────────────┐  ┌──────────────────────┐   │
│  │  GitGraph (SVG)              │  │  CommandLog           │   │
│  │                              │  │  ┌──────────────────┐ │   │
│  │  ● main ──●──●──●──────●    │  │  │ $ git init       │ │   │
│  │                \       /     │  │  │ $ git commit ... │ │   │
│  │                 ●──●──●      │  │  │ $ git checkout -b│ │   │
│  │                 feature      │  │  │ $ git commit ... │ │   │
│  │                              │  │  │ $ git merge ...  │ │   │
│  │  BranchLabel tags at heads   │  │  │ ← current step   │ │   │
│  └──────────────────────────────┘  │  └──────────────────┘ │   │
│                                    │                        │   │
│                                    │  CommitDetailPanel     │   │
│                                    │  (click a node)        │   │
│                                    └──────────────────────┘   │
└────────────────────────────────────────────────────────────────┘
```

Five pieces:

1. **GitGraph** — SVG container that renders commit nodes and connecting lines
2. **CommandLog** — terminal-style box showing the git commands executed so far
3. **ScenarioPicker** — buttons to load a preset workflow scenario
4. **useGitGraph composable** — manages simulated state, command log, step-through playback
5. **GitGraphView** — page view wiring it all together

## Data Model

Create `src/types/git-graph.ts`:

```ts
export interface Commit {
  id: string;
  message: string;
  branchName: string;
  parentIds: string[];
  timestamp: number;
}

export interface Branch {
  name: string;
  color: string;
  headCommitId: string;
}

/** A single step in a workflow scenario */
export interface WorkflowStep {
  /** The git command to display, e.g. 'git commit -m "Initial commit"' */
  command: string;
  /** The action to execute: 'init' | 'commit' | 'branch' | 'checkout' | 'merge' */
  action: "init" | "commit" | "branch" | "checkout" | "merge";
  /** Arguments for the action */
  args: {
    message?: string;
    branchName?: string;
    sourceBranch?: string;
  };
}

/** A named workflow scenario with an ordered list of steps */
export interface WorkflowScenario {
  name: string;
  description: string;
  steps: WorkflowStep[];
}
```

**Key conventions:**

- Every mutation to the graph is driven by a `WorkflowStep` — this keeps the command log in sync
- `command` is the human-readable git CLI string shown in the log box
- `action` + `args` tell the composable what to actually execute on the simulated state

## Preset Workflow Scenarios

Create `src/data/git-scenarios.ts`:

```ts
import type { WorkflowScenario } from "@/types/git-graph";

export const featureBranchWorkflow: WorkflowScenario = {
  name: "Feature Branch",
  description: "Create a feature branch, make commits, merge back to main",
  steps: [
    { command: "git init", action: "init", args: {} },
    {
      command: 'git commit -m "Initial commit"',
      action: "commit",
      args: { message: "Initial commit" },
    },
    {
      command: 'git commit -m "Add project structure"',
      action: "commit",
      args: { message: "Add project structure" },
    },
    {
      command: "git checkout -b feature/login",
      action: "branch",
      args: { branchName: "feature/login" },
    },
    {
      command: 'git commit -m "Add login form"',
      action: "commit",
      args: { message: "Add login form" },
    },
    {
      command: 'git commit -m "Add validation"',
      action: "commit",
      args: { message: "Add validation" },
    },
    {
      command: 'git commit -m "Add auth API call"',
      action: "commit",
      args: { message: "Add auth API call" },
    },
    {
      command: "git checkout main",
      action: "checkout",
      args: { branchName: "main" },
    },
    {
      command: "git merge feature/login",
      action: "merge",
      args: { sourceBranch: "feature/login" },
    },
    {
      command: 'git commit -m "Update README"',
      action: "commit",
      args: { message: "Update README" },
    },
  ],
};

export const gitflowWorkflow: WorkflowScenario = {
  name: "Gitflow",
  description: "Develop branch, feature branches, release branch, hotfix",
  steps: [
    { command: "git init", action: "init", args: {} },
    {
      command: 'git commit -m "Initial commit"',
      action: "commit",
      args: { message: "Initial commit" },
    },
    {
      command: "git checkout -b develop",
      action: "branch",
      args: { branchName: "develop" },
    },
    {
      command: 'git commit -m "Setup dev tooling"',
      action: "commit",
      args: { message: "Setup dev tooling" },
    },
    {
      command: "git checkout -b feature/auth",
      action: "branch",
      args: { branchName: "feature/auth" },
    },
    {
      command: 'git commit -m "Add auth module"',
      action: "commit",
      args: { message: "Add auth module" },
    },
    {
      command: 'git commit -m "Add JWT handling"',
      action: "commit",
      args: { message: "Add JWT handling" },
    },
    {
      command: "git checkout develop",
      action: "checkout",
      args: { branchName: "develop" },
    },
    {
      command: "git merge feature/auth",
      action: "merge",
      args: { sourceBranch: "feature/auth" },
    },
    {
      command: "git checkout -b release/1.0",
      action: "branch",
      args: { branchName: "release/1.0" },
    },
    {
      command: 'git commit -m "Bump version to 1.0"',
      action: "commit",
      args: { message: "Bump version to 1.0" },
    },
    {
      command: "git checkout main",
      action: "checkout",
      args: { branchName: "main" },
    },
    {
      command: "git merge release/1.0",
      action: "merge",
      args: { sourceBranch: "release/1.0" },
    },
  ],
};

export const hotfixWorkflow: WorkflowScenario = {
  name: "Hotfix",
  description: "Production hotfix branched from main, merged back",
  steps: [
    { command: "git init", action: "init", args: {} },
    {
      command: 'git commit -m "Initial commit"',
      action: "commit",
      args: { message: "Initial commit" },
    },
    {
      command: 'git commit -m "Add user dashboard"',
      action: "commit",
      args: { message: "Add user dashboard" },
    },
    {
      command: 'git commit -m "Release v1.0"',
      action: "commit",
      args: { message: "Release v1.0" },
    },
    {
      command: "git checkout -b hotfix/fix-login",
      action: "branch",
      args: { branchName: "hotfix/fix-login" },
    },
    {
      command: 'git commit -m "Fix login crash on empty email"',
      action: "commit",
      args: { message: "Fix login crash on empty email" },
    },
    {
      command: 'git commit -m "Add regression test"',
      action: "commit",
      args: { message: "Add regression test" },
    },
    {
      command: "git checkout main",
      action: "checkout",
      args: { branchName: "main" },
    },
    {
      command: "git merge hotfix/fix-login",
      action: "merge",
      args: { sourceBranch: "hotfix/fix-login" },
    },
    {
      command: 'git commit -m "Release v1.0.1"',
      action: "commit",
      args: { message: "Release v1.0.1" },
    },
  ],
};

export const allScenarios: WorkflowScenario[] = [
  featureBranchWorkflow,
  gitflowWorkflow,
  hotfixWorkflow,
];
```

## Composable — `useGitGraph`

Create `src/composables/useGitGraph.ts`:

```ts
import { ref, readonly, computed } from "vue";
import type {
  Commit,
  Branch,
  WorkflowScenario,
  WorkflowStep,
} from "@/types/git-graph";
import { featureBranchWorkflow } from "@/data/git-scenarios";

const BRANCH_COLORS = [
  "#6366f1", // indigo
  "#10b981", // emerald
  "#f59e0b", // amber
  "#ef4444", // red
  "#8b5cf6", // violet
  "#06b6d4", // cyan
  "#ec4899", // pink
  "#84cc16", // lime
];

function generateId(): string {
  return crypto.randomUUID().slice(0, 7);
}

export interface LayoutNode {
  commit: Commit;
  x: number;
  y: number;
  color: string;
}

export interface LayoutEdge {
  fromX: number;
  fromY: number;
  toX: number;
  toY: number;
  color: string;
}

export function useGitGraph() {
  const commits = ref<Commit[]>([]);
  const branches = ref<Branch[]>([]);
  const head = ref<string>("main");
  const selectedCommitId = ref<string | null>(null);

  // --- Command log & step playback ---
  const commandLog = ref<string[]>([]);
  const currentScenario = ref<WorkflowScenario>(featureBranchWorkflow);
  const currentStepIndex = ref<number>(0);
  const isFinished = computed(
    () => currentStepIndex.value >= currentScenario.value.steps.length,
  );

  // --- Graph layout constants ---
  const NODE_RADIUS = 12;
  const ROW_HEIGHT = 60;
  const COL_WIDTH = 80;
  const PADDING = 40;

  // --- Computed: branch column assignments ---
  const branchColumns = computed<Record<string, number>>(() => {
    const cols: Record<string, number> = {};
    branches.value.forEach((b, i) => {
      cols[b.name] = i;
    });
    return cols;
  });

  // --- Computed: color map ---
  const branchColorMap = computed<Record<string, string>>(() => {
    const map: Record<string, string> = {};
    branches.value.forEach((b) => {
      map[b.name] = b.color;
    });
    return map;
  });

  // --- Computed: layout nodes ---
  const layoutNodes = computed<LayoutNode[]>(() => {
    return commits.value.map((commit, index) => {
      const col = branchColumns.value[commit.branchName] ?? 0;
      return {
        commit,
        x: PADDING + col * COL_WIDTH,
        y: PADDING + index * ROW_HEIGHT,
        color: branchColorMap.value[commit.branchName] ?? BRANCH_COLORS[0],
      };
    });
  });

  // --- Computed: layout edges ---
  const layoutEdges = computed<LayoutEdge[]>(() => {
    const commitIndexMap = new Map<string, number>();
    commits.value.forEach((c, i) => commitIndexMap.set(c.id, i));

    const edges: LayoutEdge[] = [];
    for (const node of layoutNodes.value) {
      for (const parentId of node.commit.parentIds) {
        const parentIdx = commitIndexMap.get(parentId);
        if (parentIdx === undefined) continue;
        const parentNode = layoutNodes.value[parentIdx];
        edges.push({
          fromX: node.x,
          fromY: node.y,
          toX: parentNode.x,
          toY: parentNode.y,
          color: node.color,
        });
      }
    }
    return edges;
  });

  // --- Computed: selected commit details ---
  const selectedCommit = computed<Commit | null>(() => {
    if (!selectedCommitId.value) return null;
    return commits.value.find((c) => c.id === selectedCommitId.value) ?? null;
  });

  // --- Computed: SVG canvas dimensions ---
  const graphWidth = computed(() => {
    const maxCol = Math.max(branches.value.length - 1, 0);
    return PADDING * 2 + maxCol * COL_WIDTH + NODE_RADIUS * 2;
  });

  const graphHeight = computed(() => {
    return (
      PADDING * 2 +
      Math.max(commits.value.length - 1, 0) * ROW_HEIGHT +
      NODE_RADIUS * 2
    );
  });

  // --- Internal actions (not called directly — driven by executeStep) ---

  function doInit() {
    commits.value = [];
    branches.value = [
      { name: "main", color: BRANCH_COLORS[0], headCommitId: "" },
    ];
    head.value = "main";
    selectedCommitId.value = null;
  }

  function doCommit(message: string) {
    const currentBranch = branches.value.find((b) => b.name === head.value);
    if (!currentBranch) return;

    const newId = generateId();
    const parentIds = currentBranch.headCommitId
      ? [currentBranch.headCommitId]
      : [];
    commits.value.push({
      id: newId,
      message,
      branchName: head.value,
      parentIds,
      timestamp: Date.now(),
    });
    currentBranch.headCommitId = newId;
  }

  function doBranch(name: string) {
    const exists = branches.value.find((b) => b.name === name);
    if (exists) return;

    const currentBranch = branches.value.find((b) => b.name === head.value);
    if (!currentBranch) return;

    const colorIndex = branches.value.length % BRANCH_COLORS.length;
    branches.value.push({
      name,
      color: BRANCH_COLORS[colorIndex],
      headCommitId: currentBranch.headCommitId,
    });
    head.value = name;
  }

  function doCheckout(branchName: string) {
    const branch = branches.value.find((b) => b.name === branchName);
    if (!branch) return;
    head.value = branchName;
  }

  function doMerge(sourceBranchName: string) {
    const currentBranch = branches.value.find((b) => b.name === head.value);
    const sourceBranch = branches.value.find(
      (b) => b.name === sourceBranchName,
    );
    if (!currentBranch || !sourceBranch) return;
    if (currentBranch.name === sourceBranch.name) return;

    const newId = generateId();
    commits.value.push({
      id: newId,
      message: `Merge '${sourceBranchName}' into '${head.value}'`,
      branchName: head.value,
      parentIds: [currentBranch.headCommitId, sourceBranch.headCommitId],
      timestamp: Date.now(),
    });
    currentBranch.headCommitId = newId;
  }

  // --- Execute a single workflow step ---
  function executeStep(step: WorkflowStep) {
    commandLog.value.push("$ " + step.command);

    switch (step.action) {
      case "init":
        doInit();
        break;
      case "commit":
        doCommit(step.args.message ?? "untitled commit");
        break;
      case "branch":
        if (step.args.branchName) doBranch(step.args.branchName);
        break;
      case "checkout":
        if (step.args.branchName) doCheckout(step.args.branchName);
        break;
      case "merge":
        if (step.args.sourceBranch) doMerge(step.args.sourceBranch);
        break;
    }
  }

  // --- Step forward one step ---
  function nextStep() {
    if (isFinished.value) return;
    executeStep(currentScenario.value.steps[currentStepIndex.value]);
    currentStepIndex.value++;
  }

  // --- Play all remaining steps ---
  function playAll() {
    while (!isFinished.value) {
      nextStep();
    }
  }

  // --- Load a scenario and reset ---
  function loadScenario(scenario: WorkflowScenario) {
    currentScenario.value = scenario;
    currentStepIndex.value = 0;
    commits.value = [];
    branches.value = [];
    head.value = "main";
    commandLog.value = [];
    selectedCommitId.value = null;
  }

  // --- Select a commit for detail view ---
  function selectCommit(commitId: string | null) {
    selectedCommitId.value = commitId;
  }

  return {
    commits: readonly(commits),
    branches: readonly(branches),
    head: readonly(head),
    selectedCommitId: readonly(selectedCommitId),
    selectedCommit,
    commandLog: readonly(commandLog),
    currentScenario: readonly(currentScenario),
    currentStepIndex: readonly(currentStepIndex),
    isFinished,
    layoutNodes,
    layoutEdges,
    graphWidth,
    graphHeight,
    NODE_RADIUS,
    branchColorMap,
    nextStep,
    playAll,
    loadScenario,
    selectCommit,
  };
}
```

**Key conventions:**

- `readonly()` on all exposed refs to prevent external mutation
- Every graph mutation goes through `executeStep()` which logs the command first — this guarantees the command log and graph are always in sync
- `nextStep()` advances one step; `playAll()` runs all remaining steps instantly
- `loadScenario()` resets everything and loads a new scenario
- Layout computed properties derive positions from branch column index and commit order

## Command Log Component — `CommandLog.vue`

Create `src/components/CommandLog.vue`:

```vue
<template>
  <div
    class="rounded-lg border border-gray-700 bg-gray-900 p-4 font-mono text-sm text-green-400"
  >
    <div class="mb-2 flex items-center gap-2 border-b border-gray-700 pb-2">
      <span class="h-3 w-3 rounded-full bg-red-500"></span>
      <span class="h-3 w-3 rounded-full bg-yellow-500"></span>
      <span class="h-3 w-3 rounded-full bg-green-500"></span>
      <span class="ml-2 text-xs text-gray-400">Terminal</span>
    </div>
    <div ref="logContainer" class="max-h-64 overflow-y-auto">
      <div v-if="log.length === 0" class="text-gray-500">
        Click "Next Step" to start the workflow...
      </div>
      <div
        v-for="(line, i) in log"
        :key="i"
        :class="[
          'py-0.5 leading-relaxed',
          i === log.length - 1 ? 'text-white font-bold' : 'text-green-400',
        ]"
      >
        {{ line }}
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import { ref, watch, nextTick } from "vue";

export default {
  props: {
    log: { type: Array as () => string[], required: true },
  },
  setup(props) {
    const logContainer = ref<HTMLElement | null>(null);

    watch(
      () => props.log.length,
      () => {
        nextTick(() => {
          if (logContainer.value) {
            logContainer.value.scrollTop = logContainer.value.scrollHeight;
          }
        });
      },
    );

    return { logContainer };
  },
};
</script>
```

**Key conventions:**

- Terminal-style dark background with green monospace text
- macOS-style traffic light dots in the header for visual polish
- The latest command is highlighted in white + bold so you see what just happened
- Auto-scrolls to bottom when new commands are added via `watch` + `nextTick`
- No `<style>` block — all Tailwind

## Graph Component — `GitGraph.vue`

Create `src/components/GitGraph.vue`:

```vue
<template>
  <svg :width="graphWidth" :height="graphHeight" class="block">
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
import type { LayoutEdge, LayoutNode } from "@/composables/useGitGraph";
import type { Branch } from "@/types/git-graph";

export default {
  props: {
    layoutNodes: { type: Array as () => LayoutNode[], required: true },
    layoutEdges: { type: Array as () => LayoutEdge[], required: true },
    branches: { type: Array as () => Branch[], required: true },
    branchColorMap: {
      type: Object as () => Record<string, string>,
      required: true,
    },
    graphWidth: { type: Number, required: true },
    graphHeight: { type: Number, required: true },
    nodeRadius: { type: Number, required: true },
    selectedCommitId: { type: String, default: null },
  },
  emits: ["select-commit"],
  setup(props) {
    function edgePath(edge: LayoutEdge): string {
      if (edge.fromX === edge.toX) {
        return `M ${edge.fromX} ${edge.fromY} L ${edge.toX} ${edge.toY}`;
      }
      const midY = (edge.fromY + edge.toY) / 2;
      return `M ${edge.fromX} ${edge.fromY} C ${edge.fromX} ${midY}, ${edge.toX} ${midY}, ${edge.toX} ${edge.toY}`;
    }

    function branchHeadNode(branchName: string): LayoutNode | undefined {
      const branch = props.branches.find((b) => b.name === branchName);
      if (!branch) return undefined;
      return props.layoutNodes.find((n) => n.commit.id === branch.headCommitId);
    }

    return { edgePath, branchHeadNode };
  },
};
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
    <h3
      class="mb-2 text-sm font-semibold text-gray-500 uppercase tracking-wide"
    >
      Commit Details
    </h3>
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
        <dd class="font-mono">
          {{ commit.parentIds.length ? commit.parentIds.join(", ") : "(root)" }}
        </dd>
      </div>
      <div>
        <dt class="font-medium text-gray-600">Time</dt>
        <dd>{{ new Date(commit.timestamp).toLocaleString() }}</dd>
      </div>
    </dl>
  </div>
</template>

<script lang="ts">
import type { Commit } from "@/types/git-graph";

export default {
  props: {
    commit: { type: Object as () => Commit | null, default: null },
  },
  setup() {
    return {};
  },
};
</script>
```

## Page View — `GitGraphView.vue`

Create `src/views/GitGraphView.vue`:

```vue
<template>
  <div class="mx-auto max-w-6xl">
    <h1 class="mb-4 text-2xl font-bold text-gray-800">
      Git Workflow Simulator
    </h1>

    <!-- Scenario picker -->
    <div class="mb-4">
      <span class="mr-2 text-sm font-medium text-gray-600">Scenario:</span>
      <button
        v-for="scenario in scenarios"
        :key="scenario.name"
        class="mr-2 rounded-full px-4 py-1.5 text-sm font-medium transition"
        :class="
          currentScenario.name === scenario.name
            ? 'bg-indigo-600 text-white'
            : 'bg-gray-200 text-gray-700 hover:bg-gray-300'
        "
        @click="loadScenario(scenario)"
      >
        {{ scenario.name }}
      </button>
    </div>

    <!-- Description + controls -->
    <div class="mb-4 flex items-center justify-between">
      <p class="text-sm text-gray-500">{{ currentScenario.description }}</p>
      <div class="flex gap-2">
        <button
          class="rounded bg-indigo-600 px-4 py-1.5 text-sm font-medium text-white hover:bg-indigo-700 disabled:opacity-40"
          :disabled="isFinished"
          @click="nextStep"
        >
          ▶ Next Step
        </button>
        <button
          class="rounded bg-emerald-600 px-4 py-1.5 text-sm font-medium text-white hover:bg-emerald-700 disabled:opacity-40"
          :disabled="isFinished"
          @click="playAll"
        >
          ⏭ Play All
        </button>
        <button
          class="rounded bg-gray-500 px-4 py-1.5 text-sm font-medium text-white hover:bg-gray-600"
          @click="loadScenario(currentScenario)"
        >
          ↺ Reset
        </button>
      </div>
    </div>

    <!-- Progress -->
    <p class="mb-4 text-sm text-gray-500">
      Step {{ currentStepIndex }} / {{ currentScenario.steps.length }}
      <span class="ml-3">HEAD →</span>
      <span class="ml-1 font-semibold text-indigo-600">{{ head || "—" }}</span>
      <span v-if="isFinished" class="ml-3 font-semibold text-emerald-600"
        >✓ Workflow complete</span
      >
    </p>

    <!-- Graph + sidebar (command log & detail) -->
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

      <div class="flex w-80 shrink-0 flex-col gap-4">
        <CommandLog :log="commandLog" />
        <CommitDetailPanel :commit="selectedCommit" />
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import { onBeforeMount } from "vue";
import { useGitGraph } from "@/composables/useGitGraph";
import { allScenarios } from "@/data/git-scenarios";
import GitGraph from "@/components/GitGraph.vue";
import CommandLog from "@/components/CommandLog.vue";
import CommitDetailPanel from "@/components/CommitDetailPanel.vue";

export default {
  components: { GitGraph, CommandLog, CommitDetailPanel },
  setup() {
    const {
      commits,
      branches,
      head,
      selectedCommitId,
      selectedCommit,
      commandLog,
      currentScenario,
      currentStepIndex,
      isFinished,
      layoutNodes,
      layoutEdges,
      graphWidth,
      graphHeight,
      NODE_RADIUS,
      branchColorMap,
      nextStep,
      playAll,
      loadScenario,
      selectCommit,
    } = useGitGraph();

    const scenarios = allScenarios;

    onBeforeMount(() => {
      loadScenario(allScenarios[0]);
    });

    return {
      commits,
      branches,
      head,
      selectedCommitId,
      selectedCommit,
      commandLog,
      currentScenario,
      currentStepIndex,
      isFinished,
      layoutNodes,
      layoutEdges,
      graphWidth,
      graphHeight,
      NODE_RADIUS,
      branchColorMap,
      scenarios,
      nextStep,
      playAll,
      loadScenario,
      selectCommit,
    };
  },
};
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
<RouterLink
  class="rounded-md px-3 py-2 text-sm font-medium text-white"
  :to="{ name: 'git-graph' }"
>
  Git Graph
</RouterLink>
```

## File Checklist

| File                                   | Purpose                                                           |
| -------------------------------------- | ----------------------------------------------------------------- |
| `src/types/git-graph.ts`               | `Commit`, `Branch`, `WorkflowStep`, `WorkflowScenario` interfaces |
| `src/data/git-scenarios.ts`            | Preset scenarios: Feature Branch, Gitflow, Hotfix                 |
| `src/composables/useGitGraph.ts`       | State, layout, step playback, command logging                     |
| `src/components/GitGraph.vue`          | SVG graph renderer (nodes + edges + branch labels)                |
| `src/components/CommandLog.vue`        | Terminal-style box showing executed git commands                  |
| `src/components/CommitDetailPanel.vue` | Selected commit metadata display                                  |
| `src/views/GitGraphView.vue`           | Page view with scenario picker, controls, graph, log              |
| `src/router/index.ts`                  | Add `/git-graph` route                                            |
| `App.vue`                              | Add nav link                                                      |
