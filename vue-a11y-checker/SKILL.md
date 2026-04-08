---
name: vue-a11y-checker
description: "Build a live web accessibility checker in a Vue 3 app. Use when: creating an accessibility audit tool, highlighting missing labels, detecting contrast issues, checking ARIA usage, inspecting focusability, reporting WCAG violations, building an a11y linter UI, visualizing accessibility problems on a live page."
argument-hint: "Description of the page or component to audit"
---

# Live Accessibility Checker — a11y Audit with Visual Highlights

## When to Use

- Building an in-app accessibility panel that scans the live DOM for WCAG violations
- Highlighting elements with missing labels, low contrast, or broken ARIA
- Teaching WCAG 2.1 / 2.2 best practices with interactive feedback
- Generating a structured report of issues grouped by severity and rule

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│  A11yCheckerView (page-level view)                                      │
│                                                                         │
│  ┌────────────────────────────────────┐  ┌────────────────────────────┐ │
│  │  AuditTarget                       │  │  ReportPanel               │ │
│  │  (the page region being scanned)   │  │                            │ │
│  │                                    │  │  ┌──────────────────────┐  │ │
│  │  ┌────────────────────────────┐    │  │  │ RuleSummaryBar       │  │ │
│  │  │ rendered HTML content      │    │  │  │ ✕ 3 errors           │  │ │
│  │  │                            │    │  │  │ ⚠ 5 warnings         │  │ │
│  │  │ ┌──────────┐               │    │  │  │ ℹ 2 notices          │  │ │
│  │  │ │highlighted│◀─────────────│────│──│  └──────────────────────┘  │ │
│  │  │ │  element  │  overlay     │    │  │                            │ │
│  │  │ └──────────┘               │    │  │  IssueList                 │ │
│  │  │                            │    │  │  ┌──────────────────────┐  │ │
│  │  └────────────────────────────┘    │  │  │ [contrast] #btn-buy  │  │ │
│  │                                    │  │  │ ratio 2.1:1 < 4.5:1  │  │ │
│  │  [Scan Page]  [Clear Highlights]   │  │  ├──────────────────────┤  │ │
│  └────────────────────────────────────┘  │  │ [label] <input #q>   │  │ │
│                                          │  │ no associated label  │  │ │
│  ┌─────────────────────────────────────┐ │  ├──────────────────────┤  │ │
│  │  RuleFilterBar                      │ │  │ [aria] role=button   │  │ │
│  │  [All] [Errors] [Warnings] [Labels] │ │  │ missing aria-label   │  │ │
│  │  [Contrast] [ARIA] [Focus] [Images] │ │  └──────────────────────┘  │ │
│  └─────────────────────────────────────┘ └────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
```

Six pieces:

1. **useA11yChecker composable** — scans a DOM root, runs all rules, returns structured issues
2. **AuditTarget** — the region being scanned; wraps a slot so any content can be audited
3. **A11yOverlay** — positions highlight boxes over offending elements (like the DOM inspector)
4. **ReportPanel** — displays issues grouped by rule with severity badges
5. **RuleFilterBar** — filter buttons to narrow by severity or rule category
6. **RuleSummaryBar** — counts of errors / warnings / notices at a glance

## Data Model

Create `src/types/a11y-checker.ts`:

```ts
export type IssueSeverity = "error" | "warning" | "notice";

export type RuleCategory =
  | "contrast"
  | "label"
  | "aria"
  | "focus"
  | "image"
  | "heading"
  | "landmark"
  | "language";

export interface A11yIssue {
  id: string;
  ruleId: string;
  category: RuleCategory;
  severity: IssueSeverity;
  element: HTMLElement;
  /** Short description shown in the issue list */
  message: string;
  /** WCAG success criterion reference, e.g. "1.4.3" */
  wcag: string;
  /** Actionable fix hint */
  fix: string;
}

export interface A11yRule {
  id: string;
  category: RuleCategory;
  severity: IssueSeverity;
  description: string;
  wcag: string;
  /** Pure function: receives the element under test; returns a failure message or null if passing */
  test: (el: HTMLElement, root: HTMLElement) => string | null;
  /** Selector for elements this rule applies to */
  selector: string;
}

export interface AuditResult {
  issues: A11yIssue[];
  scannedAt: Date;
  elementCount: number;
}
```

**Key conventions:**

- `A11yIssue.element` holds the live `HTMLElement` reference — used to position highlights and focus the element
- `A11yRule.test()` is a pure function — takes the element plus the document root for context queries (e.g., checking if a `<label for>` exists elsewhere)
- `ruleId` on `A11yIssue` links back to the rule definition for grouping and filtering

## Rule Definitions

Create `src/data/a11y-rules.ts`:

```ts
import type { A11yRule } from "@/types/a11y-checker";

// ─── Contrast helpers ───────────────────────────────────────────────────────

/** Parse rgb/rgba string → [r, g, b] */
function parseRgb(color: string): [number, number, number] | null {
  const m = color.match(/rgba?\((\d+),\s*(\d+),\s*(\d+)/);
  if (!m) return null;
  return [Number(m[1]), Number(m[2]), Number(m[3])];
}

/** sRGB → linear */
function toLinear(c: number): number {
  const s = c / 255;
  return s <= 0.04045 ? s / 12.92 : ((s + 0.055) / 1.055) ** 2.4;
}

/** Relative luminance per WCAG 2.x */
function luminance(r: number, g: number, b: number): number {
  return 0.2126 * toLinear(r) + 0.7152 * toLinear(g) + 0.0722 * toLinear(b);
}

/** Contrast ratio between two luminance values */
function contrastRatio(l1: number, l2: number): number {
  const [lighter, darker] = l1 > l2 ? [l1, l2] : [l2, l1];
  return (lighter + 0.05) / (darker + 0.05);
}

function getContrastRatio(el: HTMLElement): number | null {
  const style = window.getComputedStyle(el);
  const fg = parseRgb(style.color);
  const bg = parseRgb(style.backgroundColor);
  if (!fg || !bg) return null;
  // Skip fully transparent backgrounds
  if (style.backgroundColor === "rgba(0, 0, 0, 0)") return null;
  return contrastRatio(luminance(...fg), luminance(...bg));
}

// ─── Rules ───────────────────────────────────────────────────────────────────

export const a11yRules: A11yRule[] = [
  // ── Contrast ──────────────────────────────────────────────────────────────
  {
    id: "contrast-normal-text",
    category: "contrast",
    severity: "error",
    description: "Normal text must have a contrast ratio of at least 4.5:1",
    wcag: "1.4.3",
    selector:
      "p, span, li, td, th, label, a, h1, h2, h3, h4, h5, h6, button, legend",
    test(el) {
      const style = window.getComputedStyle(el);
      const fontSize = parseFloat(style.fontSize);
      const fontWeight = style.fontWeight;
      const isLargeText =
        fontSize >= 24 ||
        (fontSize >= 18.67 &&
          (fontWeight === "bold" || Number(fontWeight) >= 700));
      if (isLargeText) return null; // handled by large-text rule
      const ratio = getContrastRatio(el);
      if (ratio === null) return null;
      if (ratio < 4.5) {
        return `Contrast ratio ${ratio.toFixed(2)}:1 is below the 4.5:1 minimum for normal text`;
      }
      return null;
    },
  },
  {
    id: "contrast-large-text",
    category: "contrast",
    severity: "error",
    description: "Large text must have a contrast ratio of at least 3:1",
    wcag: "1.4.3",
    selector: "p, span, li, td, th, label, a, h1, h2, h3, h4, h5, h6, button",
    test(el) {
      const style = window.getComputedStyle(el);
      const fontSize = parseFloat(style.fontSize);
      const fontWeight = style.fontWeight;
      const isLargeText =
        fontSize >= 24 ||
        (fontSize >= 18.67 &&
          (fontWeight === "bold" || Number(fontWeight) >= 700));
      if (!isLargeText) return null;
      const ratio = getContrastRatio(el);
      if (ratio === null) return null;
      if (ratio < 3) {
        return `Contrast ratio ${ratio.toFixed(2)}:1 is below the 3:1 minimum for large text`;
      }
      return null;
    },
  },

  // ── Labels ────────────────────────────────────────────────────────────────
  {
    id: "input-missing-label",
    category: "label",
    severity: "error",
    description: "Form inputs must have an accessible label",
    wcag: "1.3.1",
    selector:
      'input:not([type="hidden"]):not([type="submit"]):not([type="button"]):not([type="reset"]), textarea, select',
    test(el, root) {
      const id = el.getAttribute("id");
      const ariaLabel = el.getAttribute("aria-label");
      const ariaLabelledBy = el.getAttribute("aria-labelledby");
      const title = el.getAttribute("title");
      if (ariaLabel || ariaLabelledBy || title) return null;
      if (id && root.querySelector(`label[for="${id}"]`)) return null;
      // Check if wrapped in a label
      if (el.closest("label")) return null;
      return "Input has no associated label (id+for, aria-label, aria-labelledby, or wrapping label)";
    },
  },
  {
    id: "label-empty",
    category: "label",
    severity: "warning",
    description: "Label elements must have non-empty text",
    wcag: "1.3.1",
    selector: "label",
    test(el) {
      if (!el.textContent?.trim()) {
        return "Label element is empty — screen readers will announce nothing";
      }
      return null;
    },
  },

  // ── Images ────────────────────────────────────────────────────────────────
  {
    id: "img-missing-alt",
    category: "image",
    severity: "error",
    description: "Images must have an alt attribute",
    wcag: "1.1.1",
    selector: "img",
    test(el) {
      if (!el.hasAttribute("alt")) {
        return "<img> is missing an alt attribute entirely";
      }
      return null;
    },
  },
  {
    id: "img-alt-filename",
    category: "image",
    severity: "warning",
    description: "Alt text should not be a filename",
    wcag: "1.1.1",
    selector: "img[alt]",
    test(el) {
      const alt = el.getAttribute("alt") ?? "";
      if (/\.(png|jpe?g|gif|svg|webp|bmp)$/i.test(alt)) {
        return `Alt text "${alt}" looks like a filename — use a meaningful description`;
      }
      return null;
    },
  },

  // ── ARIA ──────────────────────────────────────────────────────────────────
  {
    id: "aria-role-button-label",
    category: "aria",
    severity: "error",
    description: 'Elements with role="button" must have an accessible name',
    wcag: "4.1.2",
    selector: '[role="button"]',
    test(el) {
      const ariaLabel = el.getAttribute("aria-label");
      const ariaLabelledBy = el.getAttribute("aria-labelledby");
      const text = el.textContent?.trim();
      const title = el.getAttribute("title");
      if (!text && !ariaLabel && !ariaLabelledBy && !title) {
        return 'Element with role="button" has no accessible name';
      }
      return null;
    },
  },
  {
    id: "aria-hidden-focusable",
    category: "aria",
    severity: "error",
    description: 'Elements with aria-hidden="true" must not be focusable',
    wcag: "4.1.2",
    selector: '[aria-hidden="true"]',
    test(el) {
      const focusable = el.querySelectorAll(
        'a[href], button:not([disabled]), input:not([disabled]), select:not([disabled]), textarea:not([disabled]), [tabindex]:not([tabindex="-1"])',
      );
      if (focusable.length > 0) {
        return `aria-hidden="true" element contains ${focusable.length} focusable child(ren) — they will be unreachable by keyboard`;
      }
      return null;
    },
  },
  {
    id: "aria-required-attr",
    category: "aria",
    severity: "error",
    description: "ARIA roles must have their required attributes",
    wcag: "4.1.2",
    selector: "[role]",
    test(el) {
      const role = el.getAttribute("role");
      const requiredAttrs: Record<string, string[]> = {
        checkbox: ["aria-checked"],
        combobox: ["aria-expanded"],
        slider: ["aria-valuenow", "aria-valuemin", "aria-valuemax"],
        scrollbar: [
          "aria-valuenow",
          "aria-valuemin",
          "aria-valuemax",
          "aria-controls",
        ],
        progressbar: ["aria-valuenow"],
      };
      if (!role || !requiredAttrs[role]) return null;
      const missing = requiredAttrs[role].filter(
        (attr) => !el.hasAttribute(attr),
      );
      if (missing.length) {
        return `role="${role}" is missing required attribute(s): ${missing.join(", ")}`;
      }
      return null;
    },
  },

  // ── Focus ─────────────────────────────────────────────────────────────────
  {
    id: "tabindex-positive",
    category: "focus",
    severity: "warning",
    description:
      "Avoid positive tabindex values — they disrupt natural focus order",
    wcag: "2.4.3",
    selector: "[tabindex]",
    test(el) {
      const val = Number(el.getAttribute("tabindex"));
      if (val > 0) {
        return `tabindex="${val}" disrupts the natural tab order — use 0 or -1 instead`;
      }
      return null;
    },
  },
  {
    id: "interactive-disabled-focus",
    category: "focus",
    severity: "notice",
    description: "Disabled interactive elements should not be focusable",
    wcag: "2.1.1",
    selector: "[disabled][tabindex]",
    test(el) {
      if (el.getAttribute("tabindex") !== "-1") {
        return 'Disabled element is still in the tab order — add tabindex="-1" to remove it';
      }
      return null;
    },
  },

  // ── Headings ──────────────────────────────────────────────────────────────
  {
    id: "heading-empty",
    category: "heading",
    severity: "error",
    description: "Heading elements must not be empty",
    wcag: "2.4.6",
    selector: "h1, h2, h3, h4, h5, h6",
    test(el) {
      if (!el.textContent?.trim()) {
        return `<${el.tagName.toLowerCase()}> is empty — remove it or add meaningful text`;
      }
      return null;
    },
  },
  {
    id: "heading-skip-level",
    category: "heading",
    severity: "warning",
    description: "Heading levels should not be skipped",
    wcag: "1.3.1",
    selector: "h1, h2, h3, h4, h5, h6",
    test(el, root) {
      const level = Number(el.tagName[1]);
      const allHeadings = Array.from(
        root.querySelectorAll("h1,h2,h3,h4,h5,h6"),
      );
      const idx = allHeadings.indexOf(el);
      if (idx === 0) return null;
      const prevLevel = Number(allHeadings[idx - 1].tagName[1]);
      if (level > prevLevel + 1) {
        return `<h${level}> skips from h${prevLevel} — use h${prevLevel + 1} to maintain hierarchy`;
      }
      return null;
    },
  },

  // ── Landmarks ─────────────────────────────────────────────────────────────
  {
    id: "landmark-main-missing",
    category: "landmark",
    severity: "warning",
    description: "Page should contain a <main> landmark",
    wcag: "1.3.6",
    selector: "body",
    test(el) {
      if (!el.querySelector('main, [role="main"]')) {
        return "No <main> landmark found — screen reader users cannot jump to main content";
      }
      return null;
    },
  },
  {
    id: "landmark-duplicate-main",
    category: "landmark",
    severity: "error",
    description: "Page must not have more than one <main> landmark",
    wcag: "1.3.6",
    selector: "body",
    test(el) {
      const mains = el.querySelectorAll('main, [role="main"]');
      if (mains.length > 1) {
        return `Found ${mains.length} <main> landmarks — only one is allowed per page`;
      }
      return null;
    },
  },
  {
    id: "nav-missing-label",
    category: "landmark",
    severity: "warning",
    description:
      "Multiple <nav> elements should be distinguished with aria-label",
    wcag: "2.4.1",
    selector: "nav",
    test(el, root) {
      const navCount = root.querySelectorAll('nav, [role="navigation"]').length;
      if (navCount < 2) return null;
      if (
        !el.getAttribute("aria-label") &&
        !el.getAttribute("aria-labelledby")
      ) {
        return "Page has multiple <nav> elements — add aria-label to distinguish them";
      }
      return null;
    },
  },

  // ── Language ──────────────────────────────────────────────────────────────
  {
    id: "html-lang-missing",
    category: "language",
    severity: "error",
    description: "The <html> element must have a lang attribute",
    wcag: "3.1.1",
    selector: "html",
    test(el) {
      if (!el.getAttribute("lang")) {
        return "<html> is missing a lang attribute — screen readers need this to select the correct voice";
      }
      return null;
    },
  },
  {
    id: "html-lang-valid",
    category: "language",
    severity: "error",
    description: "The lang attribute must be a valid BCP 47 language tag",
    wcag: "3.1.1",
    selector: "html[lang]",
    test(el) {
      const lang = el.getAttribute("lang") ?? "";
      if (!/^[a-zA-Z]{2,3}(-[a-zA-Z0-9]{2,8})*$/.test(lang)) {
        return `lang="${lang}" is not a valid BCP 47 tag (examples: "en", "en-US", "fr-FR")`;
      }
      return null;
    },
  },
];
```

## Composable — `useA11yChecker`

Create `src/composables/useA11yChecker.ts`:

```ts
import { ref, readonly, computed } from "vue";
import { a11yRules } from "@/data/a11y-rules";
import type {
  A11yIssue,
  AuditResult,
  RuleCategory,
  IssueSeverity,
} from "@/types/a11y-checker";

export type FilterMode = IssueSeverity | RuleCategory | "all";

export function useA11yChecker() {
  const result = ref<AuditResult | null>(null);
  const isScanning = ref(false);
  const activeFilter = ref<FilterMode>("all");
  const highlightedElementId = ref<string | null>(null);

  // --- Filtered issues based on activeFilter ---
  const filteredIssues = computed<A11yIssue[]>(() => {
    if (!result.value) return [];
    const issues = result.value.issues;
    if (activeFilter.value === "all") return issues;
    return issues.filter(
      (i) =>
        i.severity === activeFilter.value || i.category === activeFilter.value,
    );
  });

  // --- Summary counts ---
  const summary = computed(() => ({
    total: result.value?.issues.length ?? 0,
    errors:
      result.value?.issues.filter((i) => i.severity === "error").length ?? 0,
    warnings:
      result.value?.issues.filter((i) => i.severity === "warning").length ?? 0,
    notices:
      result.value?.issues.filter((i) => i.severity === "notice").length ?? 0,
  }));

  // --- Run the audit against a DOM root ---
  function scan(root: HTMLElement) {
    isScanning.value = true;
    const issues: A11yIssue[] = [];
    let elementCount = 0;

    for (const rule of a11yRules) {
      const elements = Array.from(
        root.querySelectorAll<HTMLElement>(rule.selector),
      );
      elementCount += elements.length;

      for (const el of elements) {
        const message = rule.test(el, root);
        if (message) {
          issues.push({
            id: crypto.randomUUID().slice(0, 8),
            ruleId: rule.id,
            category: rule.category,
            severity: rule.severity,
            element: el,
            message,
            wcag: rule.wcag,
            fix: getFixForRule(rule.id),
          });
        }
      }
    }

    result.value = {
      issues,
      scannedAt: new Date(),
      elementCount,
    };
    isScanning.value = false;
  }

  // --- Fix hints are centralized here (outside the rule test fns for separation) ---
  function getFixForRule(ruleId: string): string {
    const fixes: Record<string, string> = {
      "contrast-normal-text":
        "Increase the color contrast between text and background. Tools: https://webaim.org/resources/contrastchecker/",
      "contrast-large-text":
        "Large text needs a 3:1 ratio minimum. Darken text or lighten background.",
      "input-missing-label":
        'Wrap the input in a <label>, or add a <label for="inputId">, or add aria-label.',
      "label-empty": "Add visible text inside the <label> element.",
      "img-missing-alt":
        'Add alt="" for decorative images, or a descriptive alt for meaningful ones.',
      "img-alt-filename":
        "Replace the filename with a short description of what the image shows.",
      "aria-role-button-label":
        'Add visible text content inside the element, or add aria-label="...".',
      "aria-hidden-focusable":
        'Remove aria-hidden from containers that include focusable children, or add tabindex="-1" to each focusable descendant.',
      "aria-required-attr":
        "Add the missing required ARIA attribute(s) to satisfy the role contract.",
      "tabindex-positive":
        'Use tabindex="0" to include in natural order, or tabindex="-1" to exclude.',
      "interactive-disabled-focus":
        'Add tabindex="-1" to disabled interactive elements.',
      "heading-empty": "Remove empty headings or add meaningful text content.",
      "heading-skip-level":
        "Use headings in order: h1 → h2 → h3. Do not skip levels.",
      "landmark-main-missing":
        "Wrap the primary page content in a <main> element.",
      "landmark-duplicate-main":
        'Ensure only one <main> or role="main" exists per page.',
      "nav-missing-label":
        'Add aria-label="Primary navigation" (or similar) to each <nav>.',
      "html-lang-missing":
        'Add a lang attribute to <html>, e.g. <html lang="en">.',
      "html-lang-valid":
        "Use a valid BCP 47 language tag: https://www.iana.org/assignments/language-subtag-registry",
    };
    return fixes[ruleId] ?? "Review the WCAG documentation for this criterion.";
  }

  // --- Clear results ---
  function clearResults() {
    result.value = null;
    highlightedElementId.value = null;
    activeFilter.value = "all";
  }

  // --- Set filter ---
  function setFilter(filter: FilterMode) {
    activeFilter.value = filter;
  }

  // --- Set highlighted element ---
  function setHighlight(issueId: string | null) {
    highlightedElementId.value = issueId;
  }

  // --- Scroll to and focus a specific element ---
  function focusElement(el: HTMLElement) {
    el.scrollIntoView({ behavior: "smooth", block: "center" });
    el.focus({ preventScroll: true });
  }

  return {
    result: readonly(result),
    isScanning: readonly(isScanning),
    activeFilter: readonly(activeFilter),
    highlightedElementId: readonly(highlightedElementId),
    filteredIssues,
    summary,
    scan,
    clearResults,
    setFilter,
    setHighlight,
    focusElement,
  };
}
```

**Key conventions:**

- `scan(root)` accepts any `HTMLElement` — pass a `ref` to a container to scope the audit
- `a11yRules` is declarative — adding a new rule never requires touching the composable
- `filteredIssues` is a computed that reacts to `activeFilter` — no manual re-filter calls
- `getFixForRule()` is separated from the rule `test()` for clean separation of concerns

## Overlay Component — `A11yOverlay.vue`

Create `src/components/A11yOverlay.vue`:

```vue
<template>
  <div
    v-if="rect"
    data-a11y-ignore
    class="pointer-events-none fixed z-[9998] transition-all duration-100"
    :style="overlayStyle"
  >
    <!-- Highlight border with severity color -->
    <div
      class="absolute inset-0 border-2"
      :class="{
        'border-red-500 bg-red-500/10': severity === 'error',
        'border-amber-400 bg-amber-400/10': severity === 'warning',
        'border-blue-400 bg-blue-400/10': severity === 'notice',
      }"
    />
    <!-- Badge label -->
    <span
      class="absolute -top-6 left-0 rounded px-1.5 py-0.5 text-xs font-semibold text-white whitespace-nowrap"
      :class="{
        'bg-red-500': severity === 'error',
        'bg-amber-400': severity === 'warning',
        'bg-blue-400': severity === 'notice',
      }"
    >
      {{ label }}
    </span>
  </div>
</template>

<script lang="ts">
import { computed, defineComponent, type PropType } from "vue";
import type { IssueSeverity } from "@/types/a11y-checker";

export default defineComponent({
  props: {
    rect: { type: Object as PropType<DOMRect | null>, default: null },
    severity: { type: String as PropType<IssueSeverity>, default: "error" },
    label: { type: String, default: "" },
  },
  setup(props) {
    const overlayStyle = computed(() => {
      if (!props.rect) return {};
      return {
        top: `${props.rect.top + window.scrollY}px`,
        left: `${props.rect.left + window.scrollX}px`,
        width: `${props.rect.width}px`,
        height: `${props.rect.height}px`,
      };
    });
    return { overlayStyle };
  },
});
</script>
```

**Key conventions:**

- `pointer-events-none` prevents the overlay from blocking mouse events
- `position: fixed` + scroll adjustment (`window.scrollY`) places the box correctly even on scrolled pages
- `z-[9998]` keeps it below the inspector overlay (z-[9999]) if both are active simultaneously
- Color-coded by severity: red = error, amber = warning, blue = notice

## Rule Filter Bar Component — `RuleFilterBar.vue`

Create `src/components/RuleFilterBar.vue`:

```vue
<template>
  <div class="flex flex-wrap gap-2" data-a11y-ignore>
    <button
      v-for="option in filterOptions"
      :key="option.value"
      type="button"
      class="rounded-full px-3 py-1 text-xs font-medium transition-colors"
      :class="
        activeFilter === option.value
          ? 'bg-indigo-600 text-white'
          : 'bg-white border border-gray-300 text-gray-600 hover:border-indigo-400'
      "
      @click="$emit('filter', option.value)"
    >
      {{ option.label }}
      <span v-if="option.count !== undefined" class="ml-1 opacity-75"
        >({{ option.count }})</span
      >
    </button>
  </div>
</template>

<script lang="ts">
import { defineComponent, type PropType } from "vue";
import type { FilterMode } from "@/composables/useA11yChecker";

export interface FilterOption {
  value: FilterMode;
  label: string;
  count?: number;
}

export default defineComponent({
  props: {
    activeFilter: { type: String as PropType<FilterMode>, required: true },
    filterOptions: { type: Array as PropType<FilterOption[]>, required: true },
  },
  emits: ["filter"],
  setup() {
    return {};
  },
});
</script>
```

## Rule Summary Bar Component — `RuleSummaryBar.vue`

Create `src/components/RuleSummaryBar.vue`:

```vue
<template>
  <div
    class="flex items-center gap-4 rounded-lg border border-gray-200 bg-white px-4 py-3 shadow-sm"
    data-a11y-ignore
  >
    <div
      v-if="summary.total === 0"
      class="flex items-center gap-2 text-green-600"
    >
      <span class="text-lg">✓</span>
      <span class="text-sm font-medium">No issues found</span>
    </div>
    <template v-else>
      <div class="flex items-center gap-1.5 text-red-600">
        <span class="text-base font-bold">{{ summary.errors }}</span>
        <span class="text-xs">error{{ summary.errors !== 1 ? "s" : "" }}</span>
      </div>
      <div class="h-4 w-px bg-gray-200" />
      <div class="flex items-center gap-1.5 text-amber-500">
        <span class="text-base font-bold">{{ summary.warnings }}</span>
        <span class="text-xs"
          >warning{{ summary.warnings !== 1 ? "s" : "" }}</span
        >
      </div>
      <div class="h-4 w-px bg-gray-200" />
      <div class="flex items-center gap-1.5 text-blue-500">
        <span class="text-base font-bold">{{ summary.notices }}</span>
        <span class="text-xs"
          >notice{{ summary.notices !== 1 ? "s" : "" }}</span
        >
      </div>
      <div class="ml-auto text-xs text-gray-400">
        {{ summary.total }} issue{{ summary.total !== 1 ? "s" : "" }} found
      </div>
    </template>
  </div>
</template>

<script lang="ts">
import { defineComponent, type PropType } from "vue";

export default defineComponent({
  props: {
    summary: {
      type: Object as PropType<{
        total: number;
        errors: number;
        warnings: number;
        notices: number;
      }>,
      required: true,
    },
  },
  setup() {
    return {};
  },
});
</script>
```

## Issue List Component — `IssueList.vue`

Create `src/components/IssueList.vue`:

```vue
<template>
  <div
    class="flex flex-col divide-y divide-gray-100 overflow-y-auto rounded-lg border border-gray-200 bg-white shadow-sm"
    data-a11y-ignore
  >
    <div
      v-if="!issues.length"
      class="px-4 py-8 text-center text-sm text-gray-400"
    >
      {{
        scanned
          ? "No issues match the current filter."
          : "Run a scan to see results."
      }}
    </div>

    <div
      v-for="issue in issues"
      :key="issue.id"
      class="cursor-pointer px-4 py-3 hover:bg-gray-50 transition-colors"
      :class="{ 'bg-indigo-50': highlightedId === issue.id }"
      @mouseenter="$emit('highlight', issue)"
      @mouseleave="$emit('highlight', null)"
      @click="$emit('focus-element', issue)"
    >
      <div class="flex items-start gap-3">
        <!-- Severity icon -->
        <span
          class="mt-0.5 shrink-0 text-sm font-bold"
          :class="{
            'text-red-500': issue.severity === 'error',
            'text-amber-400': issue.severity === 'warning',
            'text-blue-400': issue.severity === 'notice',
          }"
        >
          {{
            issue.severity === "error"
              ? "✕"
              : issue.severity === "warning"
                ? "⚠"
                : "ℹ"
          }}
        </span>

        <div class="min-w-0 flex-1">
          <!-- Rule + WCAG badge -->
          <div class="flex flex-wrap items-center gap-2 mb-1">
            <span
              class="rounded bg-gray-100 px-1.5 py-0.5 text-xs font-mono text-gray-600"
            >
              {{ issue.category }}
            </span>
            <span
              class="rounded bg-indigo-100 px-1.5 py-0.5 text-xs text-indigo-600"
            >
              WCAG {{ issue.wcag }}
            </span>
          </div>

          <!-- Message -->
          <p class="text-xs font-medium text-gray-800">{{ issue.message }}</p>

          <!-- Fix hint -->
          <p class="mt-1 text-xs text-gray-500">{{ issue.fix }}</p>
        </div>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent, type PropType } from "vue";
import type { A11yIssue } from "@/types/a11y-checker";

export default defineComponent({
  props: {
    issues: { type: Array as PropType<A11yIssue[]>, required: true },
    highlightedId: { type: String as PropType<string | null>, default: null },
    scanned: { type: Boolean, default: false },
  },
  emits: ["highlight", "focus-element"],
  setup() {
    return {};
  },
});
</script>
```

**Key conventions:**

- `@mouseenter` emits the full issue object to drive overlay positioning in the parent
- `@click` emits `focus-element` — parent calls `focusElement(issue.element)` to scroll to and focus the violating element
- `highlightedId` drives the active row highlight so the overlay and list stay in sync

## Audit Target Component — `AuditTarget.vue`

Create `src/components/AuditTarget.vue`:

```vue
<template>
  <div ref="containerRef" class="relative">
    <slot />
  </div>
</template>

<script lang="ts">
import { defineComponent, ref } from "vue";

export default defineComponent({
  emits: ["ready"],
  setup(_, { emit }) {
    const containerRef = ref<HTMLElement | null>(null);

    function getRoot(): HTMLElement | null {
      return containerRef.value;
    }

    return { containerRef, getRoot };
  },
});
</script>
```

**Key conventions:**

- Wrap any HTML content in `<AuditTarget>` to scope the audit — no full-document scanning required
- Expose `getRoot()` via template ref so the parent view can pass the element to `scan()`
- The slot lets you place real, editable HTML content inside whilst the composable treats it as the target DOM

## Page View — `A11yCheckerView.vue`

Create `src/views/A11yCheckerView.vue`:

```vue
<template>
  <div class="mx-auto max-w-screen-xl space-y-4">
    <div class="flex items-center justify-between">
      <h1 class="text-2xl font-bold text-gray-800">Accessibility Checker</h1>
      <div class="flex gap-2" data-a11y-ignore>
        <button
          type="button"
          class="rounded-md bg-indigo-600 px-4 py-2 text-sm font-medium text-white hover:bg-indigo-700 disabled:opacity-50"
          :disabled="isScanning"
          @click="runScan"
        >
          {{ isScanning ? "Scanning…" : "Scan Page" }}
        </button>
        <button
          v-if="result"
          type="button"
          class="rounded-md border border-gray-300 px-4 py-2 text-sm text-gray-600 hover:bg-gray-100"
          @click="clearResults"
        >
          Clear
        </button>
      </div>
    </div>

    <!-- Summary + filters -->
    <template v-if="result">
      <RuleSummaryBar :summary="summary" />
      <RuleFilterBar
        :activeFilter="activeFilter"
        :filterOptions="filterOptions"
        @filter="setFilter"
      />
    </template>

    <div class="grid grid-cols-1 gap-6 lg:grid-cols-2">
      <!-- Left: target content -->
      <div class="flex flex-col gap-4">
        <h2 class="text-sm font-semibold text-gray-600">Audit Target</h2>
        <AuditTarget ref="auditTargetRef">
          <!-- Demo content with intentional violations for demonstration -->
          <div class="space-y-4 rounded-lg border border-gray-200 bg-white p-6">
            <h1 style="color: #aaa; background: #fff;">
              Page Title (low contrast)
            </h1>
            <h3>Skipped to h3 (heading skip)</h3>
            <img src="https://picsum.photos/200/100" />
            <form>
              <input type="text" placeholder="Search…" />
              <button type="submit" style="background:#e53e3e; color: #fff;">
                Go
              </button>
            </form>
            <nav>Primary nav</nav>
            <nav>Secondary nav</nav>
            <div role="button">Click me</div>
            <div aria-hidden="true"><button>Hidden but focusable</button></div>
          </div>
        </AuditTarget>
      </div>

      <!-- Right: issue list -->
      <div class="flex flex-col gap-4">
        <h2 class="text-sm font-semibold text-gray-600">
          Issues
          <span v-if="result" class="ml-1 text-gray-400 font-normal">
            ({{ filteredIssues.length }} shown)
          </span>
        </h2>
        <IssueList
          :issues="filteredIssues"
          :highlightedId="highlightedElementId"
          :scanned="!!result"
          @highlight="onHighlight"
          @focus-element="onFocusElement"
        />
      </div>
    </div>

    <!-- Overlay rendered at view level so it sits above all content -->
    <A11yOverlay
      :rect="overlayRect"
      :severity="overlayIssue?.severity ?? 'error'"
      :label="
        overlayIssue
          ? `${overlayIssue.category} · WCAG ${overlayIssue.wcag}`
          : ''
      "
    />
  </div>
</template>

<script lang="ts">
import { ref, computed } from "vue";
import { useA11yChecker } from "@/composables/useA11yChecker";
import type { A11yIssue } from "@/types/a11y-checker";
import type { FilterOption } from "@/components/RuleFilterBar.vue";
import AuditTarget from "@/components/AuditTarget.vue";
import A11yOverlay from "@/components/A11yOverlay.vue";
import RuleSummaryBar from "@/components/RuleSummaryBar.vue";
import RuleFilterBar from "@/components/RuleFilterBar.vue";
import IssueList from "@/components/IssueList.vue";

export default {
  components: {
    AuditTarget,
    A11yOverlay,
    RuleSummaryBar,
    RuleFilterBar,
    IssueList,
  },
  setup() {
    const auditTargetRef = ref<InstanceType<typeof AuditTarget> | null>(null);
    const overlayIssue = ref<A11yIssue | null>(null);
    const overlayRect = ref<DOMRect | null>(null);

    const {
      result,
      isScanning,
      activeFilter,
      highlightedElementId,
      filteredIssues,
      summary,
      scan,
      clearResults,
      setFilter,
      setHighlight,
      focusElement,
    } = useA11yChecker();

    function runScan() {
      const root = auditTargetRef.value?.getRoot();
      if (root) scan(root);
    }

    function onHighlight(issue: A11yIssue | null) {
      overlayIssue.value = issue;
      overlayRect.value = issue ? issue.element.getBoundingClientRect() : null;
      setHighlight(issue?.id ?? null);
    }

    function onFocusElement(issue: A11yIssue) {
      focusElement(issue.element);
    }

    const filterOptions = computed<FilterOption[]>(() => [
      { value: "all", label: "All", count: result.value?.issues.length },
      { value: "error", label: "Errors", count: summary.value.errors },
      { value: "warning", label: "Warnings", count: summary.value.warnings },
      { value: "notice", label: "Notices", count: summary.value.notices },
      { value: "contrast", label: "Contrast" },
      { value: "label", label: "Labels" },
      { value: "image", label: "Images" },
      { value: "aria", label: "ARIA" },
      { value: "focus", label: "Focus" },
      { value: "heading", label: "Headings" },
      { value: "landmark", label: "Landmarks" },
    ]);

    return {
      auditTargetRef,
      overlayIssue,
      overlayRect,
      result,
      isScanning,
      activeFilter,
      highlightedElementId,
      filteredIssues,
      summary,
      runScan,
      clearResults,
      setFilter,
      filterOptions,
      onHighlight,
      onFocusElement,
    };
  },
};
</script>
```

## Routing

Add to `src/router/index.ts`:

```ts
{
  path: '/a11y-checker',
  name: 'a11y-checker',
  component: () => import('../views/A11yCheckerView.vue'),
},
```

Add a nav link in `App.vue`:

```vue
<RouterLink
  class="rounded-md px-3 py-2 text-sm font-medium text-white"
  :to="{ name: 'a11y-checker' }"
>A11y Checker</RouterLink>
```

## Testing Pattern

Tests in `src/components/__tests__/A11yChecker.test.ts`:

```ts
import { describe, it, expect, beforeEach } from "vitest";
import { useA11yChecker } from "@/composables/useA11yChecker";

function makeRoot(html: string): HTMLElement {
  const div = document.createElement("div");
  div.innerHTML = html;
  document.body.appendChild(div);
  return div;
}

describe("useA11yChecker", () => {
  it("flags an img missing alt", () => {
    const root = makeRoot('<img src="photo.jpg">');
    const { scan, result } = useA11yChecker();
    scan(root);
    expect(
      result.value?.issues.some((i) => i.ruleId === "img-missing-alt"),
    ).toBe(true);
  });

  it('passes an img with alt=""', () => {
    const root = makeRoot('<img src="photo.jpg" alt="">');
    const { scan, result } = useA11yChecker();
    scan(root);
    expect(
      result.value?.issues.some((i) => i.ruleId === "img-missing-alt"),
    ).toBe(false);
  });

  it("flags an input with no label", () => {
    const root = makeRoot('<input type="text">');
    const { scan, result } = useA11yChecker();
    scan(root);
    expect(
      result.value?.issues.some((i) => i.ruleId === "input-missing-label"),
    ).toBe(true);
  });

  it("passes an input with aria-label", () => {
    const root = makeRoot('<input type="text" aria-label="Search">');
    const { scan, result } = useA11yChecker();
    scan(root);
    expect(
      result.value?.issues.some((i) => i.ruleId === "input-missing-label"),
    ).toBe(false);
  });

  it("flags a heading skip", () => {
    const root = makeRoot("<h1>Title</h1><h3>Sub</h3>");
    const { scan, result } = useA11yChecker();
    scan(root);
    expect(
      result.value?.issues.some((i) => i.ruleId === "heading-skip-level"),
    ).toBe(true);
  });

  it("filters issues by severity", () => {
    const root = makeRoot('<img src="x.jpg">');
    const { scan, filteredIssues, setFilter } = useA11yChecker();
    scan(root);
    setFilter("error");
    expect(filteredIssues.value.every((i) => i.severity === "error")).toBe(
      true,
    );
  });

  it("summary counts match issue list", () => {
    const root = makeRoot('<img src="x.jpg"><input type="text">');
    const { scan, summary, result } = useA11yChecker();
    scan(root);
    const total = result.value?.issues.length ?? 0;
    expect(
      summary.value.errors + summary.value.warnings + summary.value.notices,
    ).toBe(total);
  });
});
```

## Extending with New Rules

To add a new rule, append an `A11yRule` object to `src/data/a11y-rules.ts` and add a fix string in `getFixForRule()`:

```ts
// Example: flag inline event handlers (onclick, onmousedown, etc.)
{
  id: 'no-inline-event-handlers',
  category: 'aria',
  severity: 'notice',
  description: 'Prefer addEventListener over inline event attributes',
  wcag: '4.1.1',
  selector: '[onclick], [onmousedown], [onkeydown]',
  test(el) {
    const attrs = ['onclick', 'onmousedown', 'onkeydown'].filter((a) => el.hasAttribute(a))
    if (attrs.length) {
      return `Element uses inline event handler(s): ${attrs.join(', ')}`
    }
    return null
  },
},
```

No other files need to change — the composable picks up new rules automatically.

## File Checklist

| File                                | Purpose                                                                                 |
| ----------------------------------- | --------------------------------------------------------------------------------------- |
| `src/types/a11y-checker.ts`         | `A11yIssue`, `A11yRule`, `AuditResult`, severity + category types                       |
| `src/data/a11y-rules.ts`            | All rule definitions — contrast, label, image, ARIA, focus, heading, landmark, language |
| `src/composables/useA11yChecker.ts` | Scan engine, filtering, summary counts, highlight + focus helpers                       |
| `src/components/A11yOverlay.vue`    | Fixed-position highlight box over the offending element                                 |
| `src/components/AuditTarget.vue`    | Scoped scan container (slot wrapper with ref)                                           |
| `src/components/RuleSummaryBar.vue` | Error / warning / notice count bar                                                      |
| `src/components/RuleFilterBar.vue`  | Filter buttons by severity or rule category                                             |
| `src/components/IssueList.vue`      | Scrollable issue list with hover-to-highlight and click-to-focus                        |
| `src/views/A11yCheckerView.vue`     | Page view wiring all components and the overlay                                         |
| `src/router/index.ts`               | Add `/a11y-checker` route                                                               |
| `App.vue`                           | Add nav link                                                                            |

## a11y Best Practices Reference (WCAG 2.1 / 2.2)

| Rule ID                      | WCAG  | Principle      | What to check                                         |
| ---------------------------- | ----- | -------------- | ----------------------------------------------------- |
| `contrast-normal-text`       | 1.4.3 | Perceivable    | 4.5:1 minimum for normal text                         |
| `contrast-large-text`        | 1.4.3 | Perceivable    | 3:1 minimum for large/bold text                       |
| `input-missing-label`        | 1.3.1 | Perceivable    | Every form control needs an accessible name           |
| `label-empty`                | 1.3.1 | Perceivable    | Labels must have non-empty text                       |
| `img-missing-alt`            | 1.1.1 | Perceivable    | All `<img>` need an `alt` attribute                   |
| `img-alt-filename`           | 1.1.1 | Perceivable    | Alt text must describe content, not be a filename     |
| `aria-role-button-label`     | 4.1.2 | Robust         | role="button" needs an accessible name                |
| `aria-hidden-focusable`      | 4.1.2 | Robust         | aria-hidden must not hide focusable elements          |
| `aria-required-attr`         | 4.1.2 | Robust         | ARIA roles need their required attributes             |
| `tabindex-positive`          | 2.4.3 | Operable       | Positive tabindex breaks natural focus order          |
| `interactive-disabled-focus` | 2.1.1 | Operable       | Disabled elements should not be reachable by keyboard |
| `heading-empty`              | 2.4.6 | Operable       | Headings must have text                               |
| `heading-skip-level`         | 1.3.1 | Perceivable    | Heading levels must be sequential                     |
| `landmark-main-missing`      | 1.3.6 | Perceivable    | One `<main>` landmark per page                        |
| `landmark-duplicate-main`    | 1.3.6 | Perceivable    | Only one `<main>` allowed                             |
| `nav-missing-label`          | 2.4.1 | Operable       | Multiple `<nav>` need distinct labels                 |
| `html-lang-missing`          | 3.1.1 | Understandable | `<html>` must declare its language                    |
| `html-lang-valid`            | 3.1.1 | Understandable | `lang` must be a valid BCP 47 tag                     |

## Security Notes

- `scan()` reads the DOM using `querySelectorAll` and `getComputedStyle` — it never writes to the DOM or injects HTML
- `data-a11y-ignore` attribute on all checker UI elements prevents the audit from flagging its own interface
- `focusElement()` calls `el.focus()` which is browser-controlled — no script injection risk
- `navigator.clipboard` is only called from user-initiated events (button clicks) — never automatically
