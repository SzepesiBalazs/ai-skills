---
name: pair-programming
description: "Add an AI pair programmer panel to a Vue 3 app. Use when: building an AI-assisted coding panel, creating a pair programming interface, adding a code explanation tool, building a prompt assistant for asking good dev questions, creating a side panel that helps users think through problems with AI."
argument-hint: "The coding context or problem domain the pair programmer should assist with"
---

# AI Pair Programmer Panel

## When to Use

- Adding an AI pair programmer side panel to a Vue app
- Building a prompt assistant that helps users ask better technical questions
- Creating a code explanation workflow where users paste code and get step-by-step breakdowns
- Teaching developers how to collaborate effectively with AI through structured prompts

## Core Functionality

The panel provides two interaction modes:

1. **Ask Mode** — guides the user to formulate a high-quality question using a structured prompt builder (context + problem + expected behavior + what was tried)
2. **Explain Mode** — accepts a code snippet and produces a structured explanation request, broken down by: what the code does, why it works, and what edge cases exist

## Input / Output

| Input | Output |
|---|---|
| Raw question or vague problem description | Structured prompt ready to send to AI |
| Code snippet pasted into the panel | Explanation request with context framing |
| Selected interaction mode (Ask / Explain) | Formatted prompt string + copy button |

## Logic

### Asking Good Questions — Prompt Template

A good pair programming question follows this structure:

```
Context:    I am working on [what you're building / the codebase].
Problem:    [Describe what is going wrong or what you need to achieve].
Expected:   [What you expected to happen].
Tried:      [What you already tried and why it didn't work].
Question:   [The specific thing you need help with].
```

**Example — bad question:**
> "Why doesn't my Vue component update?"

**Example — good question (built with the template):**
> Context: I'm building a Vue 3 dashboard with Pinia. Problem: A child component doesn't re-render when I update a nested object inside the store. Expected: The component should reflect the updated value immediately. Tried: I called `store.item.value = newValue` directly — the ref didn't trigger reactivity. Question: How do I correctly mutate nested reactive state in Pinia to trigger component updates?

---

### Explaining Code — Prompt Template

Paste code into the panel. The template wraps it in a structured explanation request:

```
Please explain the following code:

\`\`\`[language]
[code snippet]
\`\`\`

Cover:
1. What this code does (high-level summary)
2. How it works step by step
3. Why each key decision was made (patterns, tradeoffs)
4. Edge cases or potential issues to watch out for
5. How I could improve or extend it
```

**Example output prompt:**
> Please explain the following code:
> ```ts
> const result = items
>   .filter(item => item.active)
>   .reduce((acc, item) => ({ ...acc, [item.id]: item }), {})
> ```
> Cover: 1. What this code does, 2. How it works step by step, 3. Why each decision was made, 4. Edge cases, 5. How to improve or extend it.

---

## UI

```
┌─────────────────────────────────────────┐
│  🤝 Pair Programmer              [×]    │
│                                         │
│  Mode: [Ask a Question] [Explain Code]  │
│                                         │
│  ── Ask a Question ──────────────────   │
│  Context (what are you building?)       │
│  ┌─────────────────────────────────┐   │
│  │ I'm building a Vue 3 dashboard…│   │
│  └─────────────────────────────────┘   │
│  Problem                                │
│  ┌─────────────────────────────────┐   │
│  │                                 │   │
│  └─────────────────────────────────┘   │
│  Expected behavior                      │
│  ┌─────────────────────────────────┐   │
│  │                                 │   │
│  └─────────────────────────────────┘   │
│  What have you tried?                   │
│  ┌─────────────────────────────────┐   │
│  │                                 │   │
│  └─────────────────────────────────┘   │
│                                         │
│  ┌─────────────────────────────────┐   │
│  │  Generated Prompt               │   │
│  │  Context: I'm building a Vue 3  │   │
│  │  dashboard…                     │   │
│  └─────────────────────────────────┘   │
│  [📋 Copy Prompt]                       │
└─────────────────────────────────────────┘
```

## Component — `PairProgrammerPanel.vue`

```vue
<template>
  <aside
    class="fixed right-0 top-0 h-full w-96 border-l border-gray-200 bg-white shadow-xl flex flex-col z-50"
    v-if="open"
  >
    <!-- Header -->
    <div class="flex items-center justify-between border-b border-gray-200 px-4 py-3">
      <h2 class="text-sm font-semibold text-gray-800">🤝 Pair Programmer</h2>
      <button @click="$emit('close')" class="text-gray-400 hover:text-gray-600">✕</button>
    </div>

    <!-- Mode tabs -->
    <div class="flex border-b border-gray-200">
      <button
        v-for="m in modes"
        :key="m.id"
        class="flex-1 py-2 text-sm font-medium transition-colors"
        :class="mode === m.id
          ? 'border-b-2 border-indigo-600 text-indigo-600'
          : 'text-gray-500 hover:text-gray-700'"
        @click="mode = m.id"
      >
        {{ m.label }}
      </button>
    </div>

    <!-- Ask mode -->
    <div v-if="mode === 'ask'" class="flex flex-col gap-3 overflow-y-auto p-4 flex-1">
      <div v-for="field in askFields" :key="field.key" class="flex flex-col gap-1">
        <label class="text-xs font-medium text-gray-600">{{ field.label }}</label>
        <textarea
          v-model="askForm[field.key]"
          :placeholder="field.placeholder"
          rows="2"
          class="rounded-md border border-gray-300 p-2 text-sm focus:border-indigo-400 focus:outline-none resize-none"
        />
      </div>
    </div>

    <!-- Explain mode -->
    <div v-if="mode === 'explain'" class="flex flex-col gap-3 overflow-y-auto p-4 flex-1">
      <div class="flex flex-col gap-1">
        <label class="text-xs font-medium text-gray-600">Language</label>
        <select v-model="explainLang" class="rounded-md border border-gray-300 p-2 text-sm">
          <option value="ts">TypeScript</option>
          <option value="js">JavaScript</option>
          <option value="vue">Vue</option>
          <option value="css">CSS</option>
          <option value="python">Python</option>
        </select>
      </div>
      <div class="flex flex-col gap-1">
        <label class="text-xs font-medium text-gray-600">Code snippet</label>
        <textarea
          v-model="explainCode"
          placeholder="Paste the code you want explained…"
          rows="8"
          class="rounded-md border border-gray-300 bg-gray-900 p-3 font-mono text-xs text-green-400 focus:border-indigo-400 focus:outline-none resize-none"
          spellcheck="false"
        />
      </div>
    </div>

    <!-- Generated prompt -->
    <div class="border-t border-gray-200 p-4 space-y-2">
      <label class="text-xs font-medium text-gray-500 uppercase tracking-wide">Generated Prompt</label>
      <pre class="max-h-40 overflow-y-auto rounded-md bg-gray-50 p-3 text-xs text-gray-700 whitespace-pre-wrap break-words">{{ generatedPrompt }}</pre>
      <button
        class="w-full rounded-md bg-indigo-600 py-2 text-sm font-medium text-white hover:bg-indigo-700 transition-colors"
        @click="copyPrompt"
      >
        {{ copied ? '✓ Copied!' : '📋 Copy Prompt' }}
      </button>
    </div>
  </aside>
</template>

<script lang="ts">
import { ref, computed, reactive } from 'vue'

const ASK_FIELDS = [
  { key: 'context', label: 'Context — what are you building?', placeholder: 'I am building a Vue 3 dashboard with Pinia…' },
  { key: 'problem', label: 'Problem', placeholder: 'Describe what is going wrong or what you need…' },
  { key: 'expected', label: 'Expected behavior', placeholder: 'What you expected to happen…' },
  { key: 'tried', label: 'What have you tried?', placeholder: 'What you already tried and why it didn't work…' },
  { key: 'question', label: 'Specific question', placeholder: 'The exact thing you need help with…' },
] as const

type AskKey = typeof ASK_FIELDS[number]['key']

export default {
  props: {
    open: { type: Boolean, default: false },
  },
  emits: ['close'],
  setup() {
    const mode = ref<'ask' | 'explain'>('ask')
    const modes = [
      { id: 'ask', label: 'Ask a Question' },
      { id: 'explain', label: 'Explain Code' },
    ]

    // Ask form state
    const askForm = reactive<Record<AskKey, string>>({
      context: '',
      problem: '',
      expected: '',
      tried: '',
      question: '',
    })
    const askFields = ASK_FIELDS

    // Explain form state
    const explainCode = ref('')
    const explainLang = ref('ts')

    // Prompt generation
    const generatedPrompt = computed(() => {
      if (mode.value === 'ask') {
        const parts: string[] = []
        if (askForm.context)  parts.push(`Context: ${askForm.context}`)
        if (askForm.problem)  parts.push(`Problem: ${askForm.problem}`)
        if (askForm.expected) parts.push(`Expected: ${askForm.expected}`)
        if (askForm.tried)    parts.push(`Tried: ${askForm.tried}`)
        if (askForm.question) parts.push(`Question: ${askForm.question}`)
        return parts.length ? parts.join('\n\n') : 'Fill in the fields above to generate a prompt.'
      }

      if (!explainCode.value.trim()) return 'Paste a code snippet above to generate an explanation prompt.'

      return `Please explain the following code:

\`\`\`${explainLang.value}
${explainCode.value.trim()}
\`\`\`

Cover:
1. What this code does (high-level summary)
2. How it works step by step
3. Why each key decision was made (patterns, tradeoffs)
4. Edge cases or potential issues to watch out for
5. How I could improve or extend it`
    })

    // Copy to clipboard
    const copied = ref(false)
    async function copyPrompt() {
      await navigator.clipboard.writeText(generatedPrompt.value)
      copied.value = true
      setTimeout(() => (copied.value = false), 2000)
    }

    return {
      mode, modes,
      askForm, askFields,
      explainCode, explainLang,
      generatedPrompt,
      copied, copyPrompt,
    }
  },
}
</script>
```

## Composable — `usePairProgrammer.ts`

```ts
import { ref } from 'vue'

export function usePairProgrammer() {
  const panelOpen = ref(false)

  function openPanel() { panelOpen.value = true }
  function closePanel() { panelOpen.value = false }
  function togglePanel() { panelOpen.value = !panelOpen.value }

  return { panelOpen, openPanel, closePanel, togglePanel }
}
```

## Wiring It Together

In `App.vue` or any layout component:

```vue
<template>
  <div>
    <!-- Trigger button -->
    <button
      class="fixed bottom-4 right-4 z-40 rounded-full bg-indigo-600 px-4 py-2 text-sm font-medium text-white shadow-lg hover:bg-indigo-700"
      @click="togglePanel"
    >
      🤝 Pair Programmer
    </button>

    <RouterView />

    <PairProgrammerPanel :open="panelOpen" @close="closePanel" />
  </div>
</template>

<script lang="ts">
import { usePairProgrammer } from '@/composables/usePairProgrammer'
import PairProgrammerPanel from '@/components/PairProgrammerPanel.vue'

export default {
  components: { PairProgrammerPanel },
  setup() {
    return usePairProgrammer()
  },
}
</script>
```

## Asking Good Questions — Rules

| Rule | Why |
|---|---|
| Always include **context** (what you're building) | AI responses without context are generic and miss the real problem |
| State **what you expected** vs **what happened** | Narrows the problem space immediately |
| Include **what you tried** | Prevents suggestions you already ruled out |
| Ask **one specific question** | Vague questions get vague answers |
| Include **relevant code** in a code block | AI cannot guess your implementation details |

## Explaining Code — Rules

| Rule | Why |
|---|---|
| Specify the **language** | Explanation style and terminology differ by language |
| Include **enough context** — don't trim imports | Missing pieces cause incorrect explanations |
| Ask for **tradeoffs** explicitly | Ensures you learn the "why", not just the "what" |
| Request **edge cases** | Surfaces bugs and assumptions you might miss |
| Ask for **improvement suggestions** | Turns an explanation into a learning opportunity |

## Keyboard Shortcut

```ts
// In App.vue or a global keydown handler
function onKeyDown(e: KeyboardEvent) {
  if (e.ctrlKey && e.shiftKey && e.key === 'P') {
    e.preventDefault()
    togglePanel()
  }
}
document.addEventListener('keydown', onKeyDown)
```

## Checklist

- [ ] Panel fixed to right side, full height, `z-50`
- [ ] Ask mode has all five fields: context, problem, expected, tried, question
- [ ] Explain mode has language selector + monospace code textarea
- [ ] Generated prompt updates reactively as fields are filled in
- [ ] Copy button uses `navigator.clipboard.writeText` and shows confirmation
- [ ] Toggle button is always visible (fixed, high z-index)
- [ ] `Ctrl+Shift+P` keyboard shortcut wired up
- [ ] Panel close button emits `close` event — parent controls open state
