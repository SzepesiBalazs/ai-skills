---
name: vue-html-builder
description: "Build a semantic HTML builder with drag-and-drop elements in a Vue 3 app. Use when: creating an HTML builder, building a drag-and-drop page editor, teaching semantic HTML structure, visualizing DOM trees, auditing accessibility live, generating accessible HTML snippets, building a visual HTML composer."
argument-hint: "Description of the HTML layout or content to build"
---

# Semantic HTML Builder — Drag & Drop with Accessibility Audit

## When to Use

- Building an interactive tool where users compose HTML by dragging elements onto a canvas
- Teaching semantic HTML structure (landmarks, heading hierarchy, content vs. layout tags)
- Live accessibility auditing as elements are placed (missing alt, unlabelled inputs, etc.)
- Generating clean, accessible HTML output from a visual editor

## Architecture Overview

```
┌────────────────────────────────────────────────────────────────────────┐
│  HtmlBuilderView (page-level view)                                     │
│                                                                        │
│  ┌──────────────────┐  ┌──────────────────────┐  ┌─────────────────┐  │
│  │  ElementPalette  │  │  CanvasArea           │  │ PropertiesPanel │  │
│  │                  │  │                       │  │                 │  │
│  │  Structure       │  │  ┌─────────────────┐  │  │ tag: <header>   │  │
│  │  [header]        │  │  │ <header>        │  │  │ id: ________    │  │
│  │  [nav]           │  │  │   <nav>         │  │  │ class: ______   │  │
│  │  [main]          │  │  │   </nav>        │  │  │ aria-label: ___ │  │
│  │  [article]  ──▶  │  │  │   <main>        │  │  │ role: _______   │  │
│  │  [section]  drag │  │  │     <h1>        │  │  │                 │  │
│  │  [aside]         │  │  │     <p>         │  │  │ [Delete node]   │  │
│  │  [footer]        │  │  │   </main>       │  │  └─────────────────┘  │
│  │                  │  │  │ </header>       │  │                       │
│  │  Content         │  │  └─────────────────┘  │  ┌─────────────────┐  │
│  │  [h1..h6]        │  │                       │  │ A11yPanel       │  │
│  │  [p]  [a]        │  │  [Drop here ▼]        │  │                 │  │
│  │  [ul] [ol]       │  └──────────────────────┘  │ ⚠ img missing   │  │
│  │  [blockquote]    │                             │   alt attr      │  │
│  │                  │  ┌──────────────────────┐  │ ⚠ heading order │  │
│  │  Interactive     │  │  HtmlOutput           │  │   h1 → h3 skip  │  │
│  │  [button]        │  │  (generated HTML)     │  │ ✓ landmarks ok  │  │
│  │  [form] [input]  │  └──────────────────────┘  └─────────────────┘  │
│  │  [label]         │                                                  │
│  │  [textarea]      │                                                  │
│  │                  │                                                  │
│  │  Media           │                                                  │
│  │  [img] [figure]  │                                                  │
│  └──────────────────┘                                                  │
└────────────────────────────────────────────────────────────────────────┘
```

Six pieces:

1. **ElementPalette** — categorized list of draggable semantic HTML elements
2. **CanvasArea** — recursive drop zone that renders the live node tree and accepts drops
3. **PropertiesPanel** — edit tag attributes, `aria-*` values, and text content for the selected node
4. **A11yPanel** — live accessibility audit results derived from the current tree
5. **HtmlOutput** — read-only code view of the serialized HTML string
6. **useHtmlBuilder composable** — manages the node tree, selection, drag state, HTML serialization, and a11y audit

## Data Model

Create `src/types/html-builder.ts`:

```ts
export interface HtmlNode {
  id: string;
  tag: string;
  attributes: Record<string, string>;
  textContent: string;
  children: HtmlNode[];
}

export interface ElementDefinition {
  tag: string;
  label: string;
  /** Category shown in the palette */
  category: "structure" | "content" | "interactive" | "media";
  /** Default attributes applied when the element is first dropped */
  defaultAttributes: Record<string, string>;
  /** Default text content */
  defaultText: string;
  /** Whether this element can have child nodes dropped into it */
  allowChildren: boolean;
  /** Accessibility hint shown in the palette tooltip */
  a11yHint: string;
}

export interface A11yIssue {
  nodeId: string;
  severity: "error" | "warning";
  message: string;
  fix: string;
}
```

**Key conventions:**

- `HtmlNode.id` is a UUID short-slice — used as drag handle and Vue `:key`
- `attributes` is a flat `Record<string, string>` — includes `id`, `class`, `aria-*`, `role`, `alt`, etc.
- `allowChildren` drives whether a `CanvasNode` renders a child drop zone
- `A11yIssue.nodeId` links each issue back to the exact node in the tree

## Element Definitions

Create `src/data/html-elements.ts`:

```ts
import type { ElementDefinition } from "@/types/html-builder";

export const elementDefinitions: ElementDefinition[] = [
  // Structure
  {
    tag: "header",
    label: "Header",
    category: "structure",
    defaultAttributes: { role: "banner" },
    defaultText: "",
    allowChildren: true,
    a11yHint: "Page-level banner landmark. Use once per page.",
  },
  {
    tag: "nav",
    label: "Nav",
    category: "structure",
    defaultAttributes: { "aria-label": "Primary navigation" },
    defaultText: "",
    allowChildren: true,
    a11yHint: "Navigation landmark. Add aria-label when multiple navs exist.",
  },
  {
    tag: "main",
    label: "Main",
    category: "structure",
    defaultAttributes: {},
    defaultText: "",
    allowChildren: true,
    a11yHint: "Main content landmark. Use exactly once per page.",
  },
  {
    tag: "article",
    label: "Article",
    category: "structure",
    defaultAttributes: {},
    defaultText: "",
    allowChildren: true,
    a11yHint: "Self-contained content. Should have a heading inside.",
  },
  {
    tag: "section",
    label: "Section",
    category: "structure",
    defaultAttributes: { "aria-labelledby": "" },
    defaultText: "",
    allowChildren: true,
    a11yHint:
      "Thematic grouping. Needs an accessible name via aria-labelledby.",
  },
  {
    tag: "aside",
    label: "Aside",
    category: "structure",
    defaultAttributes: { "aria-label": "Supplementary content" },
    defaultText: "",
    allowChildren: true,
    a11yHint: "Complementary landmark for tangentially related content.",
  },
  {
    tag: "footer",
    label: "Footer",
    category: "structure",
    defaultAttributes: { role: "contentinfo" },
    defaultText: "",
    allowChildren: true,
    a11yHint: "Page-level contentinfo landmark. Use once per page.",
  },
  // Content
  {
    tag: "h1",
    label: "H1",
    category: "content",
    defaultAttributes: {},
    defaultText: "Page title",
    allowChildren: false,
    a11yHint: "Top-level heading. Use exactly one h1 per page.",
  },
  {
    tag: "h2",
    label: "H2",
    category: "content",
    defaultAttributes: {},
    defaultText: "Section heading",
    allowChildren: false,
    a11yHint: "Second-level heading. Must follow an h1.",
  },
  {
    tag: "h3",
    label: "H3",
    category: "content",
    defaultAttributes: {},
    defaultText: "Sub-section heading",
    allowChildren: false,
    a11yHint: "Third-level heading. Must follow an h2.",
  },
  {
    tag: "p",
    label: "Paragraph",
    category: "content",
    defaultAttributes: {},
    defaultText: "Paragraph text.",
    allowChildren: false,
    a11yHint: "Block of text. Prefer p over generic divs for prose.",
  },
  {
    tag: "a",
    label: "Link",
    category: "content",
    defaultAttributes: { href: "#" },
    defaultText: "Link text",
    allowChildren: false,
    a11yHint:
      'Hyperlink. Text must describe the destination, not "click here".',
  },
  {
    tag: "ul",
    label: "List (ul)",
    category: "content",
    defaultAttributes: {},
    defaultText: "",
    allowChildren: true,
    a11yHint: "Unordered list. Children should be li elements.",
  },
  {
    tag: "ol",
    label: "List (ol)",
    category: "content",
    defaultAttributes: {},
    defaultText: "",
    allowChildren: true,
    a11yHint: "Ordered list. Children should be li elements.",
  },
  {
    tag: "li",
    label: "List Item",
    category: "content",
    defaultAttributes: {},
    defaultText: "List item",
    allowChildren: false,
    a11yHint: "Must be a direct child of ul or ol.",
  },
  {
    tag: "blockquote",
    label: "Blockquote",
    category: "content",
    defaultAttributes: { cite: "" },
    defaultText: "Quoted text",
    allowChildren: false,
    a11yHint: "Extended quotation. Use cite attribute for source URL.",
  },
  // Interactive
  {
    tag: "button",
    label: "Button",
    category: "interactive",
    defaultAttributes: { type: "button" },
    defaultText: "Click me",
    allowChildren: false,
    a11yHint: "Use for actions. Text or aria-label required.",
  },
  {
    tag: "form",
    label: "Form",
    category: "interactive",
    defaultAttributes: { "aria-label": "Form" },
    defaultText: "",
    allowChildren: true,
    a11yHint: "Must have an accessible name via aria-label or aria-labelledby.",
  },
  {
    tag: "label",
    label: "Label",
    category: "interactive",
    defaultAttributes: { for: "" },
    defaultText: "Label text",
    allowChildren: false,
    a11yHint:
      "Associate with an input via the for attribute matching input id.",
  },
  {
    tag: "input",
    label: "Input",
    category: "interactive",
    defaultAttributes: { type: "text", id: "", name: "" },
    defaultText: "",
    allowChildren: false,
    a11yHint: "Must have an associated label via id + for, or aria-label.",
  },
  {
    tag: "textarea",
    label: "Textarea",
    category: "interactive",
    defaultAttributes: { id: "", name: "", rows: "4" },
    defaultText: "",
    allowChildren: false,
    a11yHint: "Multi-line text input. Requires an associated label.",
  },
  {
    tag: "select",
    label: "Select",
    category: "interactive",
    defaultAttributes: { id: "", name: "" },
    defaultText: "",
    allowChildren: false,
    a11yHint: "Dropdown control. Requires an associated label.",
  },
  // Media
  {
    tag: "img",
    label: "Image",
    category: "media",
    defaultAttributes: { src: "", alt: "" },
    defaultText: "",
    allowChildren: false,
    a11yHint: 'Always provide alt text. Use alt="" for decorative images.',
  },
  {
    tag: "figure",
    label: "Figure",
    category: "media",
    defaultAttributes: {},
    defaultText: "",
    allowChildren: true,
    a11yHint: "Groups media with its caption. Use with figcaption.",
  },
  {
    tag: "figcaption",
    label: "Figcaption",
    category: "media",
    defaultAttributes: {},
    defaultText: "Caption text",
    allowChildren: false,
    a11yHint: "Caption for a figure element. Must be inside a figure.",
  },
];

export const elementsByCategory = elementDefinitions.reduce(
  (acc, el) => {
    (acc[el.category] ??= []).push(el);
    return acc;
  },
  {} as Record<ElementDefinition["category"], ElementDefinition[]>,
);
```

## Composable — `useHtmlBuilder`

Create `src/composables/useHtmlBuilder.ts`:

```ts
import { ref, readonly, computed } from "vue";
import type { HtmlNode, A11yIssue } from "@/types/html-builder";
import { elementDefinitions } from "@/data/html-elements";

function generateId(): string {
  return crypto.randomUUID().slice(0, 8);
}

export function useHtmlBuilder() {
  const nodes = ref<HtmlNode[]>([]);
  const selectedNodeId = ref<string | null>(null);
  const draggedTag = ref<string | null>(null);

  // --- Selected node (resolved) ---
  const selectedNode = computed<HtmlNode | null>(() => {
    if (!selectedNodeId.value) return null;
    return findNode(nodes.value, selectedNodeId.value);
  });

  // --- A11y audit ---
  const a11yIssues = computed<A11yIssue[]>(() => auditTree(nodes.value));

  // --- Serialized HTML output ---
  const htmlOutput = computed<string>(() => serializeNodes(nodes.value, 0));

  // --- Find a node anywhere in the tree ---
  function findNode(list: HtmlNode[], id: string): HtmlNode | null {
    for (const node of list) {
      if (node.id === id) return node;
      const found = findNode(node.children, id);
      if (found) return found;
    }
    return null;
  }

  // --- Drop an element onto the root or into a parent ---
  function dropElement(tag: string, parentId: string | null) {
    const def = elementDefinitions.find((d) => d.tag === tag);
    if (!def) return;

    const newNode: HtmlNode = {
      id: generateId(),
      tag,
      attributes: { ...def.defaultAttributes },
      textContent: def.defaultText,
      children: [],
    };

    if (parentId === null) {
      nodes.value.push(newNode);
    } else {
      const parent = findNode(nodes.value, parentId);
      if (parent) parent.children.push(newNode);
    }
    selectedNodeId.value = newNode.id;
  }

  // --- Remove a node from the tree ---
  function removeNode(id: string) {
    if (selectedNodeId.value === id) selectedNodeId.value = null;
    nodes.value = removeFromList(nodes.value, id);
  }

  function removeFromList(list: HtmlNode[], id: string): HtmlNode[] {
    return list
      .filter((n) => n.id !== id)
      .map((n) => ({ ...n, children: removeFromList(n.children, id) }));
  }

  // --- Update attribute on selected node ---
  function updateAttribute(nodeId: string, key: string, value: string) {
    const node = findNode(nodes.value, nodeId);
    if (!node) return;
    if (value === "") {
      delete node.attributes[key];
    } else {
      node.attributes[key] = value;
    }
  }

  // --- Update text content ---
  function updateTextContent(nodeId: string, text: string) {
    const node = findNode(nodes.value, nodeId);
    if (node) node.textContent = text;
  }

  // --- Move a node up or down within its parent's children list ---
  function moveNode(id: string, direction: "up" | "down") {
    moveInList(nodes.value, id, direction);
  }

  function moveInList(
    list: HtmlNode[],
    id: string,
    direction: "up" | "down",
  ): boolean {
    const idx = list.findIndex((n) => n.id === id);
    if (idx !== -1) {
      const target = direction === "up" ? idx - 1 : idx + 1;
      if (target < 0 || target >= list.length) return true;
      [list[idx], list[target]] = [list[target], list[idx]];
      return true;
    }
    for (const node of list) {
      if (moveInList(node.children, id, direction)) return true;
    }
    return false;
  }

  // --- Select a node ---
  function selectNode(id: string | null) {
    selectedNodeId.value = id;
  }

  // --- Set drag data ---
  function startDrag(tag: string) {
    draggedTag.value = tag;
  }

  function endDrag() {
    draggedTag.value = null;
  }

  // --- Clear the canvas ---
  function clearCanvas() {
    nodes.value = [];
    selectedNodeId.value = null;
  }

  // --- HTML serialization ---
  function serializeNodes(list: HtmlNode[], depth: number): string {
    const indent = "  ".repeat(depth);
    return list
      .map((node) => {
        const attrs = Object.entries(node.attributes)
          .filter(([, v]) => v !== "")
          .map(([k, v]) => ` ${k}="${v}"`)
          .join("");
        const voidTags = ["input", "img", "br", "hr", "meta", "link"];
        if (voidTags.includes(node.tag)) {
          return `${indent}<${node.tag}${attrs} />`;
        }
        const inner =
          node.children.length > 0
            ? `\n${serializeNodes(node.children, depth + 1)}\n${indent}`
            : node.textContent;
        return `${indent}<${node.tag}${attrs}>${inner}</${node.tag}>`;
      })
      .join("\n");
  }

  // --- Accessibility audit ---
  function auditTree(list: HtmlNode[]): A11yIssue[] {
    const issues: A11yIssue[] = [];
    const headingsFound: number[] = [];

    function walk(nodeList: HtmlNode[]) {
      for (const node of nodeList) {
        // img must have alt
        if (node.tag === "img" && node.attributes["alt"] === undefined) {
          issues.push({
            nodeId: node.id,
            severity: "error",
            message: `<img> is missing an alt attribute`,
            fix: 'Add alt="" for decorative images or a descriptive alt for meaningful ones.',
          });
        }

        // input/textarea/select must have a label
        if (["input", "textarea", "select"].includes(node.tag)) {
          const hasId = !!node.attributes["id"];
          const hasAriaLabel = !!node.attributes["aria-label"];
          const hasAriaLabelledBy = !!node.attributes["aria-labelledby"];
          if (!hasId && !hasAriaLabel && !hasAriaLabelledBy) {
            issues.push({
              nodeId: node.id,
              severity: "error",
              message: `<${node.tag}> has no accessible label`,
              fix: 'Add an id and pair with a <label for="...">, or add aria-label.',
            });
          }
        }

        // button must have text or aria-label
        if (
          node.tag === "button" &&
          !node.textContent.trim() &&
          !node.attributes["aria-label"]
        ) {
          issues.push({
            nodeId: node.id,
            severity: "error",
            message: "<button> has no accessible name",
            fix: "Add visible text content or an aria-label attribute.",
          });
        }

        // a must have text or aria-label
        if (
          node.tag === "a" &&
          !node.textContent.trim() &&
          !node.attributes["aria-label"]
        ) {
          issues.push({
            nodeId: node.id,
            severity: "error",
            message: "<a> link has no accessible name",
            fix: "Add link text or an aria-label.",
          });
        }

        // heading order check
        const headingMatch = node.tag.match(/^h([1-6])$/);
        if (headingMatch) {
          const level = Number(headingMatch[1]);
          const last = headingsFound[headingsFound.length - 1];
          if (last !== undefined && level > last + 1) {
            issues.push({
              nodeId: node.id,
              severity: "warning",
              message: `<${node.tag}> skips a heading level (previous was h${last})`,
              fix: `Use h${last + 1} before h${level}, or restructure content.`,
            });
          }
          headingsFound.push(level);
        }

        // section should have aria-labelledby or aria-label
        if (
          node.tag === "section" &&
          !node.attributes["aria-labelledby"] &&
          !node.attributes["aria-label"]
        ) {
          issues.push({
            nodeId: node.id,
            severity: "warning",
            message: "<section> has no accessible name",
            fix: "Add aria-labelledby pointing to a heading inside, or aria-label.",
          });
        }

        walk(node.children);
      }
    }

    walk(list);
    return issues;
  }

  return {
    nodes: readonly(nodes),
    selectedNodeId: readonly(selectedNodeId),
    selectedNode,
    a11yIssues,
    htmlOutput,
    draggedTag: readonly(draggedTag),
    dropElement,
    removeNode,
    updateAttribute,
    updateTextContent,
    moveNode,
    selectNode,
    startDrag,
    endDrag,
    clearCanvas,
  };
}
```

**Key conventions:**

- `nodes` is a reactive tree — every mutation (add, remove, move) updates it in place
- `htmlOutput` is a computed string — always in sync with the tree, no manual refresh
- `a11yIssues` is a computed audit — runs on every tree change with zero extra calls
- `readonly()` on all exposed refs to prevent external mutation
- `dropElement(tag, parentId)` — `parentId: null` appends to root, otherwise appends to parent's children
- The audit (`auditTree`) is pure — it receives the node list and returns issues without side effects

## Element Palette Component — `ElementPalette.vue`

Create `src/components/ElementPalette.vue`:

```vue
<template>
  <aside class="flex flex-col gap-4 overflow-y-auto">
    <div v-for="(elements, category) in elementsByCategory" :key="category">
      <h3
        class="mb-2 text-xs font-semibold uppercase tracking-wider text-gray-400"
      >
        {{ category }}
      </h3>
      <div class="flex flex-col gap-1">
        <div
          v-for="el in elements"
          :key="el.tag"
          draggable="true"
          :title="el.a11yHint"
          class="flex cursor-grab items-center gap-2 rounded-md border border-gray-200 bg-white px-3 py-2 text-sm font-mono text-gray-700 shadow-sm select-none hover:border-indigo-400 hover:bg-indigo-50 active:cursor-grabbing"
          @dragstart="onDragStart(el.tag)"
          @dragend="$emit('drag-end')"
        >
          <span class="text-indigo-500">&lt;{{ el.tag }}&gt;</span>
          <span class="text-xs text-gray-500">{{ el.label }}</span>
        </div>
      </div>
    </div>
  </aside>
</template>

<script lang="ts">
import { defineComponent } from "vue";
import { elementsByCategory } from "@/data/html-elements";

export default defineComponent({
  emits: ["drag-start", "drag-end"],
  setup(_, { emit }) {
    function onDragStart(tag: string) {
      emit("drag-start", tag);
    }
    return { elementsByCategory, onDragStart };
  },
});
</script>
```

**Key conventions:**

- Native HTML5 `draggable="true"` + `dragstart` event — no third-party DnD library needed
- `title` attribute shows the `a11yHint` as a native tooltip
- `select-none` prevents text selection while dragging
- Emits `drag-start(tag)` upward so the composable's `startDrag()` can track what is being dragged

## Canvas Node Component — `CanvasNode.vue`

Create `src/components/CanvasNode.vue` (recursive):

```vue
<template>
  <div
    class="group relative rounded border-2 transition-colors"
    :class="[
      isSelected
        ? 'border-indigo-500 bg-indigo-50'
        : 'border-gray-200 bg-white hover:border-indigo-300',
    ]"
    @click.stop="$emit('select', node.id)"
  >
    <!-- Node header bar -->
    <div class="flex items-center gap-2 px-2 py-1 text-xs font-mono">
      <span class="text-indigo-600 font-semibold">&lt;{{ node.tag }}&gt;</span>
      <span v-if="node.attributes['id']" class="text-orange-500"
        >#{{ node.attributes["id"] }}</span
      >
      <span v-if="node.attributes['class']" class="text-green-600">
        .{{ node.attributes["class"].split(" ").join(" .") }}
      </span>
      <span
        v-if="node.attributes['aria-label']"
        class="ml-auto text-blue-500 text-xs"
      >
        aria-label
      </span>
      <!-- Up / Down / Remove controls -->
      <div class="ml-auto hidden group-hover:flex items-center gap-1">
        <button
          type="button"
          class="text-gray-400 hover:text-gray-700"
          title="Move up"
          @click.stop="$emit('move', node.id, 'up')"
        >
          ↑
        </button>
        <button
          type="button"
          class="text-gray-400 hover:text-gray-700"
          title="Move down"
          @click.stop="$emit('move', node.id, 'down')"
        >
          ↓
        </button>
        <button
          type="button"
          class="text-red-400 hover:text-red-600"
          title="Remove element"
          @click.stop="$emit('remove', node.id)"
        >
          ✕
        </button>
      </div>
    </div>

    <!-- Text content preview -->
    <p v-if="node.textContent" class="px-3 pb-1 text-xs text-gray-500 italic">
      {{ node.textContent }}
    </p>

    <!-- Children + drop zone -->
    <div
      v-if="def?.allowChildren"
      class="mx-2 mb-2 min-h-[40px] rounded border border-dashed border-gray-300 p-2"
      :class="{ 'border-indigo-400 bg-indigo-50': isDragOver }"
      @dragover.prevent="isDragOver = true"
      @dragleave="isDragOver = false"
      @drop.stop="onDrop"
    >
      <CanvasNode
        v-for="child in node.children"
        :key="child.id"
        :node="child"
        :selectedNodeId="selectedNodeId"
        :draggedTag="draggedTag"
        @select="$emit('select', $event)"
        @remove="$emit('remove', $event)"
        @move="$emit('move', $event, $event)"
        @drop-into="$emit('drop-into', $event)"
      />
      <p
        v-if="!node.children.length"
        class="text-center text-xs text-gray-400 py-2"
      >
        Drop inside &lt;{{ node.tag }}&gt;
      </p>
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent, computed, ref, type PropType } from "vue";
import type { HtmlNode } from "@/types/html-builder";
import { elementDefinitions } from "@/data/html-elements";

export default defineComponent({
  name: "CanvasNode",
  props: {
    node: { type: Object as PropType<HtmlNode>, required: true },
    selectedNodeId: { type: String as PropType<string | null>, default: null },
    draggedTag: { type: String as PropType<string | null>, default: null },
  },
  emits: ["select", "remove", "move", "drop-into"],
  setup(props, { emit }) {
    const isDragOver = ref(false);

    const isSelected = computed(() => props.node.id === props.selectedNodeId);
    const def = computed(() =>
      elementDefinitions.find((d) => d.tag === props.node.tag),
    );

    function onDrop() {
      isDragOver.value = false;
      if (props.draggedTag) {
        emit("drop-into", { tag: props.draggedTag, parentId: props.node.id });
      }
    }

    return { isSelected, def, isDragOver, onDrop };
  },
});
</script>
```

**Key conventions:**

- Self-referencing component (`name: 'CanvasNode'`) for recursive child rendering
- Drop zone only renders when `def.allowChildren` is true
- `@dragover.prevent` is required to allow dropping (browser default is to deny)
- `@drop.stop` prevents the event from bubbling to a parent drop zone
- Move controls only show on hover (`group-hover:flex`) to keep the UI clean

## Canvas Area Component — `CanvasArea.vue`

Create `src/components/CanvasArea.vue`:

```vue
<template>
  <div
    class="relative min-h-[400px] rounded-lg border-2 border-dashed border-gray-300 bg-gray-50 p-4 transition-colors"
    :class="{ 'border-indigo-400 bg-indigo-50': isDragOver && !nodes.length }"
    @dragover.prevent="isDragOver = true"
    @dragleave="isDragOver = false"
    @drop="onRootDrop"
  >
    <div
      v-if="!nodes.length"
      class="flex h-full min-h-[360px] items-center justify-center"
    >
      <p class="text-sm text-gray-400">
        Drag elements from the palette to start building
      </p>
    </div>

    <div v-else class="flex flex-col gap-2">
      <CanvasNode
        v-for="node in nodes"
        :key="node.id"
        :node="node"
        :selectedNodeId="selectedNodeId"
        :draggedTag="draggedTag"
        @select="$emit('select', $event)"
        @remove="$emit('remove', $event)"
        @move="$emit('move', $event[0], $event[1])"
        @drop-into="$emit('drop-into', $event)"
      />
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent, ref, type PropType } from "vue";
import type { HtmlNode } from "@/types/html-builder";
import CanvasNode from "./CanvasNode.vue";

export default defineComponent({
  components: { CanvasNode },
  props: {
    nodes: { type: Array as PropType<HtmlNode[]>, required: true },
    selectedNodeId: { type: String as PropType<string | null>, default: null },
    draggedTag: { type: String as PropType<string | null>, default: null },
  },
  emits: ["select", "remove", "move", "drop-into", "root-drop"],
  setup(_, { emit }) {
    const isDragOver = ref(false);

    function onRootDrop() {
      isDragOver.value = false;
      emit("root-drop");
    }

    return { isDragOver, onRootDrop };
  },
});
</script>
```

## Properties Panel Component — `PropertiesPanel.vue`

Create `src/components/PropertiesPanel.vue`:

```vue
<template>
  <aside class="flex flex-col gap-4">
    <div
      v-if="!node"
      class="rounded-lg border border-gray-200 bg-white p-4 text-sm text-gray-400"
    >
      Select an element to edit its properties.
    </div>

    <div
      v-else
      class="rounded-lg border border-gray-200 bg-white p-4 shadow-sm"
    >
      <h3
        class="mb-3 text-xs font-semibold uppercase tracking-wide text-gray-500"
      >
        &lt;{{ node.tag }}&gt; Properties
      </h3>

      <!-- Text content -->
      <div v-if="node.textContent !== undefined" class="mb-3">
        <label class="block text-xs font-medium text-gray-600 mb-1"
          >Text content</label
        >
        <input
          :value="node.textContent"
          type="text"
          class="w-full rounded border border-gray-300 px-2 py-1 text-sm font-mono"
          @input="
            $emit(
              'update-text',
              node.id,
              ($event.target as HTMLInputElement).value,
            )
          "
        />
      </div>

      <!-- Attributes -->
      <div class="flex flex-col gap-2">
        <div
          v-for="(value, key) in node.attributes"
          :key="key"
          class="flex items-center gap-2"
        >
          <label class="w-28 shrink-0 text-xs font-mono text-indigo-600">{{
            key
          }}</label>
          <input
            :value="value"
            type="text"
            class="flex-1 rounded border border-gray-300 px-2 py-1 text-sm font-mono"
            @input="
              $emit(
                'update-attr',
                node.id,
                key,
                ($event.target as HTMLInputElement).value,
              )
            "
          />
        </div>
      </div>

      <!-- Add new attribute -->
      <div class="mt-3 flex gap-2">
        <input
          v-model="newAttrKey"
          type="text"
          placeholder="aria-label, role…"
          class="flex-1 rounded border border-gray-300 px-2 py-1 text-xs font-mono"
        />
        <button
          type="button"
          class="rounded bg-indigo-600 px-2 py-1 text-xs font-medium text-white hover:bg-indigo-700"
          @click="addAttribute"
        >
          + Attr
        </button>
      </div>

      <!-- Delete -->
      <button
        type="button"
        class="mt-4 w-full rounded border border-red-300 px-3 py-1.5 text-xs font-medium text-red-600 hover:bg-red-50"
        @click="$emit('remove', node.id)"
      >
        Remove &lt;{{ node.tag }}&gt;
      </button>
    </div>
  </aside>
</template>

<script lang="ts">
import { defineComponent, ref, type PropType } from "vue";
import type { HtmlNode } from "@/types/html-builder";

export default defineComponent({
  props: {
    node: { type: Object as PropType<HtmlNode | null>, default: null },
  },
  emits: ["update-attr", "update-text", "remove"],
  setup(props, { emit }) {
    const newAttrKey = ref("");

    function addAttribute() {
      if (!props.node || !newAttrKey.value.trim()) return;
      emit("update-attr", props.node.id, newAttrKey.value.trim(), "");
      newAttrKey.value = "";
    }

    return { newAttrKey, addAttribute };
  },
});
</script>
```

**Key conventions:**

- Uses `:value` + `@input` emit pattern — parent controls state, no local v-model
- Attribute keys rendered via `v-for` over `node.attributes` — automatically shows all current attrs
- "Add attribute" input lets users add any `aria-*`, `role`, `data-*`, or custom attribute freely

## Accessibility Panel Component — `A11yPanel.vue`

Create `src/components/A11yPanel.vue`:

```vue
<template>
  <aside class="rounded-lg border border-gray-200 bg-white shadow-sm">
    <div
      class="border-b border-gray-100 px-4 py-3 flex items-center justify-between"
    >
      <h3 class="text-xs font-semibold uppercase tracking-wide text-gray-500">
        Accessibility Audit
      </h3>
      <span
        class="rounded-full px-2 py-0.5 text-xs font-medium"
        :class="
          issues.length
            ? 'bg-red-100 text-red-600'
            : 'bg-green-100 text-green-600'
        "
      >
        {{
          issues.length
            ? `${issues.length} issue${issues.length > 1 ? "s" : ""}`
            : "All clear"
        }}
      </span>
    </div>

    <div class="divide-y divide-gray-100 max-h-64 overflow-y-auto">
      <div
        v-for="issue in issues"
        :key="issue.nodeId + issue.message"
        class="cursor-pointer px-4 py-3 hover:bg-gray-50"
        @click="$emit('select-node', issue.nodeId)"
      >
        <div class="flex items-start gap-2">
          <span
            class="mt-0.5 shrink-0 text-base"
            :class="
              issue.severity === 'error' ? 'text-red-500' : 'text-amber-500'
            "
          >
            {{ issue.severity === "error" ? "✕" : "⚠" }}
          </span>
          <div>
            <p class="text-xs font-medium text-gray-800">{{ issue.message }}</p>
            <p class="mt-0.5 text-xs text-gray-500">{{ issue.fix }}</p>
          </div>
        </div>
      </div>

      <div
        v-if="!issues.length"
        class="px-4 py-6 text-center text-xs text-green-600"
      >
        No accessibility issues detected.
      </div>
    </div>
  </aside>
</template>

<script lang="ts">
import { defineComponent, type PropType } from "vue";
import type { A11yIssue } from "@/types/html-builder";

export default defineComponent({
  props: {
    issues: { type: Array as PropType<A11yIssue[]>, required: true },
  },
  emits: ["select-node"],
  setup() {
    return {};
  },
});
</script>
```

**Key conventions:**

- Clicking an issue emits `select-node(nodeId)` — the view calls `selectNode()` to highlight the offending element
- `severity: 'error'` shown with ✕ red, `'warning'` with ⚠ amber
- Panel scrolls independently when issue list is long — fixed `max-h-64 overflow-y-auto`

## HTML Output Component — `HtmlOutput.vue`

Create `src/components/HtmlOutput.vue`:

```vue
<template>
  <div class="rounded-lg border border-gray-200 bg-gray-900 shadow-sm">
    <div
      class="flex items-center justify-between border-b border-gray-700 px-4 py-2"
    >
      <span class="text-xs font-medium text-gray-400">Generated HTML</span>
      <button
        type="button"
        class="rounded px-2 py-1 text-xs text-gray-400 hover:bg-gray-700 hover:text-white"
        @click="copy"
      >
        {{ copied ? "Copied!" : "Copy" }}
      </button>
    </div>
    <pre
      class="overflow-x-auto p-4 text-xs text-green-400 font-mono leading-relaxed whitespace-pre"
    >{{ html || '<!-- start dragging elements -->' }}</pre>
  </div>
</template>

<script lang="ts">
import { defineComponent, ref } from "vue";

export default defineComponent({
  props: {
    html: { type: String, required: true },
  },
  setup(props) {
    const copied = ref(false);

    async function copy() {
      if (!props.html) return;
      await navigator.clipboard.writeText(props.html);
      copied.value = true;
      setTimeout(() => (copied.value = false), 2000);
    }

    return { copied, copy };
  },
});
</script>
```

## Page View — `HtmlBuilderView.vue`

Create `src/views/HtmlBuilderView.vue`:

```vue
<template>
  <div class="mx-auto max-w-screen-2xl space-y-4">
    <div class="flex items-center justify-between">
      <h1 class="text-2xl font-bold text-gray-800">Semantic HTML Builder</h1>
      <button
        type="button"
        class="rounded-md border border-gray-300 px-3 py-1.5 text-sm text-gray-600 hover:bg-gray-100"
        @click="clearCanvas"
      >
        Clear canvas
      </button>
    </div>

    <div class="grid grid-cols-[220px_1fr_280px] gap-4">
      <!-- Left: palette -->
      <ElementPalette @drag-start="startDrag" @drag-end="endDrag" />

      <!-- Center: canvas + output -->
      <div class="flex flex-col gap-4">
        <CanvasArea
          :nodes="nodes"
          :selectedNodeId="selectedNodeId"
          :draggedTag="draggedTag"
          @select="selectNode"
          @remove="removeNode"
          @move="moveNode"
          @drop-into="({ tag, parentId }) => dropElement(tag, parentId)"
          @root-drop="() => draggedTag && dropElement(draggedTag, null)"
        />
        <HtmlOutput :html="htmlOutput" />
      </div>

      <!-- Right: properties + a11y -->
      <div class="flex flex-col gap-4">
        <PropertiesPanel
          :node="selectedNode"
          @update-attr="updateAttribute"
          @update-text="updateTextContent"
          @remove="removeNode"
        />
        <A11yPanel :issues="a11yIssues" @select-node="selectNode" />
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import { useHtmlBuilder } from "@/composables/useHtmlBuilder";
import ElementPalette from "@/components/ElementPalette.vue";
import CanvasArea from "@/components/CanvasArea.vue";
import PropertiesPanel from "@/components/PropertiesPanel.vue";
import A11yPanel from "@/components/A11yPanel.vue";
import HtmlOutput from "@/components/HtmlOutput.vue";

export default {
  components: {
    ElementPalette,
    CanvasArea,
    PropertiesPanel,
    A11yPanel,
    HtmlOutput,
  },
  setup() {
    const {
      nodes,
      selectedNodeId,
      selectedNode,
      a11yIssues,
      htmlOutput,
      draggedTag,
      dropElement,
      removeNode,
      updateAttribute,
      updateTextContent,
      moveNode,
      selectNode,
      startDrag,
      endDrag,
      clearCanvas,
    } = useHtmlBuilder();

    return {
      nodes,
      selectedNodeId,
      selectedNode,
      a11yIssues,
      htmlOutput,
      draggedTag,
      dropElement,
      removeNode,
      updateAttribute,
      updateTextContent,
      moveNode,
      selectNode,
      startDrag,
      endDrag,
      clearCanvas,
    };
  },
};
</script>
```

## Routing

Add to `src/router/index.ts`:

```ts
{
  path: '/html-builder',
  name: 'html-builder',
  component: () => import('../views/HtmlBuilderView.vue'),
},
```

Add a nav link in `App.vue`:

```vue
<RouterLink
  class="rounded-md px-3 py-2 text-sm font-medium text-white"
  :to="{ name: 'html-builder' }"
>HTML Builder</RouterLink>
```

## Testing Pattern

Tests in `src/components/__tests__/HtmlBuilder.test.ts`:

```ts
import { describe, it, expect } from "vitest";
import { useHtmlBuilder } from "@/composables/useHtmlBuilder";

describe("useHtmlBuilder", () => {
  it("drops an element onto the root", () => {
    const { nodes, dropElement } = useHtmlBuilder();
    dropElement("header", null);
    expect(nodes.value).toHaveLength(1);
    expect(nodes.value[0].tag).toBe("header");
  });

  it("drops an element into a parent", () => {
    const { nodes, dropElement } = useHtmlBuilder();
    dropElement("main", null);
    const parentId = nodes.value[0].id;
    dropElement("h1", parentId);
    expect(nodes.value[0].children).toHaveLength(1);
    expect(nodes.value[0].children[0].tag).toBe("h1");
  });

  it("removes a node", () => {
    const { nodes, dropElement, removeNode } = useHtmlBuilder();
    dropElement("header", null);
    const id = nodes.value[0].id;
    removeNode(id);
    expect(nodes.value).toHaveLength(0);
  });

  it("updates an attribute", () => {
    const { nodes, dropElement, updateAttribute } = useHtmlBuilder();
    dropElement("img", null);
    const id = nodes.value[0].id;
    updateAttribute(id, "alt", "A photo of a cat");
    expect(nodes.value[0].attributes["alt"]).toBe("A photo of a cat");
  });

  it("flags img missing alt as an error", () => {
    const { nodes, dropElement, a11yIssues } = useHtmlBuilder();
    dropElement("img", null);
    // Clear the default alt so it's truly missing
    delete nodes.value[0].attributes["alt"];
    expect(
      a11yIssues.value.some((i) => i.message.includes("missing an alt")),
    ).toBe(true);
  });

  it("flags a heading skip as a warning", () => {
    const { nodes, dropElement, a11yIssues } = useHtmlBuilder();
    dropElement("h1", null);
    dropElement("h3", null); // skips h2
    expect(
      a11yIssues.value.some((i) => i.message.includes("skips a heading level")),
    ).toBe(true);
  });

  it("serializes nodes to HTML", () => {
    const { dropElement, htmlOutput } = useHtmlBuilder();
    dropElement("main", null);
    expect(htmlOutput.value).toContain("<main>");
    expect(htmlOutput.value).toContain("</main>");
  });
});
```

## File Checklist

| File                                 | Purpose                                                 |
| ------------------------------------ | ------------------------------------------------------- |
| `src/types/html-builder.ts`          | `HtmlNode`, `ElementDefinition`, `A11yIssue` interfaces |
| `src/data/html-elements.ts`          | All element definitions with defaults and a11y hints    |
| `src/composables/useHtmlBuilder.ts`  | Tree state, drag tracking, serialization, audit         |
| `src/components/ElementPalette.vue`  | Categorized draggable element list                      |
| `src/components/CanvasNode.vue`      | Recursive node renderer with drop zone                  |
| `src/components/CanvasArea.vue`      | Root drop zone and node list                            |
| `src/components/PropertiesPanel.vue` | Live attribute and text content editor                  |
| `src/components/A11yPanel.vue`       | Live accessibility issue list                           |
| `src/components/HtmlOutput.vue`      | Generated HTML code view with copy button               |
| `src/views/HtmlBuilderView.vue`      | Page view wiring all components together                |
| `src/router/index.ts`                | Add `/html-builder` route                               |
| `App.vue`                            | Add nav link                                            |

## Accessibility & Security Notes

- The `htmlOutput` string is only ever shown in a `<pre>` block or copied to clipboard — it is **never** injected via `innerHTML` or `v-html` in the parent app
- If a live preview iframe is added in the future, use `srcdoc` with `sandbox="allow-scripts"` and **never** `allow-same-origin`, consistent with the CSS Playground skill
- The composable's audit runs on `computed()` — it is a pure function with no DOM access, making it safe and testable without a browser
- `navigator.clipboard.writeText()` in `HtmlOutput.vue` is guarded by the button click — no automatic clipboard writes
