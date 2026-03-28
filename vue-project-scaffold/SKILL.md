---
name: vue-project-scaffold
description: "Scaffold a new Vue 3 + TypeScript + Vite project. Use when: creating a new Vue project, initializing a Vue app, setting up Vue with Vite, bootstrapping a Vue SPA, configuring Tailwind with Vue, setting up gh-pages deployment."
argument-hint: "Project name and optional description of the app"
---

# Vue 3 + TypeScript + Vite Project Scaffold

## When to Use

- Starting a new Vue 3 project from scratch
- Setting up a Vue SPA with TypeScript, Vite, Tailwind CSS, and Vue Router
- Need a deployable Vue project with GitHub Pages support

## Tech Stack

| Tool | Version/Details |
|------|----------------|
| Vue | 3.5+ |
| TypeScript | Strict mode |
| Vite | 5+ with `@vitejs/plugin-vue`, `vite-plugin-vue-devtools` |
| Vue Router | 4+ with `createWebHistory` |
| Tailwind CSS | Via CDN script in `index.html` |
| Testing | Vitest + jsdom + @vue/test-utils |
| Linting | ESLint flat config with vue + typescript + prettier |
| Formatting | Prettier |
| Deployment | gh-pages to GitHub Pages |

## Project Structure

```
├── index.html
├── package.json
├── vite.config.ts
├── tsconfig.json
├── tsconfig.app.json
├── tsconfig.node.json
├── tsconfig.vitest.json
├── vitest.config.ts
├── eslint.config.js
├── env.d.ts
├── .gitignore
├── src/
│   ├── main.ts
│   ├── App.vue
│   ├── router/
│   │   └── index.ts
│   ├── views/
│   │   └── (page-level .vue files)
│   ├── components/
│   │   └── __tests__/
│   │       └── *.test.ts
│   └── (data files like .json)
└── public/
```

## Procedure

### 1. Create package.json

Use these scripts and dependencies (adjust `name`, `homepage`):

```json
{
  "name": "PROJECT_NAME",
  "version": "0.0.0",
  "private": true,
  "scripts": {
    "dev": "vite",
    "build": "run-p \"build-only {@}\" --",
    "build-only": "vite build",
    "preview": "vite preview",
    "type-check": "vue-tsc --build --force",
    "test:unit": "vitest",
    "lint": "eslint . --ext .vue,.js,.jsx,.cjs,.mjs,.ts,.tsx,.cts,.mts --fix --ignore-path .gitignore",
    "format": "prettier --write src/",
    "deploy": "gh-pages -d dist"
  },
  "dependencies": {
    "vue": "^3.5.12",
    "vue-router": "^4.3.3"
  },
  "devDependencies": {
    "@rushstack/eslint-patch": "^1.8.0",
    "@tsconfig/node20": "^20.1.4",
    "@types/jsdom": "^21.1.7",
    "@types/node": "^20.14.5",
    "@vitejs/plugin-vue": "^5.0.5",
    "@vitejs/plugin-vue-jsx": "^4.0.0",
    "@vue/eslint-config-prettier": "^9.0.0",
    "@vue/eslint-config-typescript": "^13.0.0",
    "@vue/test-utils": "^2.4.6",
    "@vue/tsconfig": "^0.5.1",
    "eslint": "^8.57.0",
    "eslint-plugin-vue": "^9.23.0",
    "gh-pages": "^6.3.0",
    "jsdom": "^24.1.0",
    "npm-run-all2": "^6.2.0",
    "prettier": "^3.2.5",
    "vite": "^5.3.1",
    "vite-plugin-vue-devtools": "^7.5.4",
    "vitest": "^1.6.0",
    "vue-tsc": "^2.0.0"
  }
}
```

### 2. Create vite.config.ts

```ts
import { fileURLToPath, URL } from 'node:url'
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import vueJsx from '@vitejs/plugin-vue-jsx'
import vueDevTools from 'vite-plugin-vue-devtools'

export default defineConfig({
  base: './',
  plugins: [vue(), vueJsx(), vueDevTools()],
  resolve: {
    alias: {
      '@': fileURLToPath(new URL('./src', import.meta.url)),
    },
  },
})
```

Key: `base: './'` enables relative paths for GitHub Pages hosting.

### 3. Create TypeScript configs

**tsconfig.json** (root):
```json
{
  "compilerOptions": {
    "target": "esnext",
    "module": "esnext",
    "moduleResolution": "bundler",
    "lib": ["esnext", "dom"],
    "allowJs": true,
    "jsx": "preserve",
    "esModuleInterop": true,
    "skipLibCheck": true,
    "strict": true,
    "types": ["node", "vue"]
  },
  "include": ["env.d.ts", "src/**/*.ts", "src/**/*.d.ts", "src/**/*.tsx", "src/**/*.vue", "src/main.ts"],
  "exclude": ["node_modules"]
}
```

**tsconfig.app.json**:
```json
{
  "extends": "@vue/tsconfig/tsconfig.dom.json",
  "include": ["env.d.ts", "src/**/*.vue"],
  "exclude": ["src/**/__tests__/*"],
  "compilerOptions": {
    "composite": true,
    "tsBuildInfoFile": "./node_modules/.tmp/tsconfig.app.tsbuildinfo",
    "baseUrl": ".",
    "paths": { "@/*": ["./src/*"] }
  }
}
```

**tsconfig.node.json**:
```json
{
  "compilerOptions": {
    "composite": true,
    "noEmit": true,
    "tsBuildInfoFile": "./node_modules/.tmp/tsconfig.node.tsbuildinfo",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "types": ["node"]
  },
  "include": ["vite.config.ts", "vitest.config.ts"]
}
```

**tsconfig.vitest.json**:
```json
{
  "extends": "./tsconfig.app.json",
  "exclude": [],
  "compilerOptions": {
    "composite": true,
    "tsBuildInfoFile": "./node_modules/.tmp/tsconfig.vitest.tsbuildinfo",
    "lib": [],
    "types": ["node", "jsdom"]
  }
}
```

### 4. Create vitest.config.ts

```ts
import { fileURLToPath } from 'node:url'
import { mergeConfig, defineConfig, configDefaults } from 'vitest/config'
import viteConfig from './vite.config'

export default mergeConfig(
  viteConfig,
  defineConfig({
    test: {
      environment: 'jsdom',
      exclude: [...configDefaults.exclude, 'e2e/**'],
      root: fileURLToPath(new URL('./', import.meta.url))
    }
  })
)
```

### 5. Create eslint.config.js

```js
import pluginVue from 'eslint-plugin-vue'
import vueTsEslintConfig from '@vue/eslint-config-typescript'
import pluginVitest from '@vitest/eslint-plugin'
import skipFormatting from '@vue/eslint-config-prettier/skip-formatting'

export default [
  {
    name: 'app/files-to-lint',
    files: ['**/*.{ts,mts,tsx,vue}'],
  },
  {
    name: 'app/files-to-ignore',
    ignores: ['**/dist/**', '**/dist-ssr/**', '**/coverage/**'],
  },
  ...pluginVue.configs['flat/essential'],
  ...vueTsEslintConfig(),
  {
    ...pluginVitest.configs.recommended,
    files: ['src/**/__tests__/*'],
  },
  skipFormatting,
]
```

### 6. Create index.html

```html
<!doctype html>
<html lang="">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <script src="https://cdn.tailwindcss.com?plugins=forms,typography,aspect-ratio,line-clamp,container-queries"></script>
    <title>APP_TITLE</title>
  </head>
  <body>
    <div id="app"></div>
    <script type="module" src="./src/main.ts"></script>
  </body>
</html>
```

### 7. Create env.d.ts

```ts
/// <reference types="vite/client" />
```

### 8. Create src/main.ts

```ts
import { createApp } from 'vue'
import App from './App.vue'
import router from './router'

const app = createApp(App)
app.use(router)
app.mount('#app')
```

### 9. Create src/App.vue

See the [vue-component skill](../vue-component/SKILL.md) for the App.vue layout pattern with nav + RouterView.

### 10. Create src/router/index.ts

See the [vue-routing skill](../vue-routing/SKILL.md) for the routing setup pattern.

### 11. Create .gitignore

```
# Logs
logs
*.log
npm-debug.log*
yarn-debug.log*
yarn-error.log*
pnpm-debug.log*
lerna-debug.log*

node_modules
.DS_Store
coverage
*.local
/cypress/videos/
/cypress/screenshots/

# Editor directories and files
.vscode/
.idea
*.suo
*.ntvs*
*.njsproj
*.sln
*.sw?

*.tsbuildinfo
vite/
dist/
```

### 12. Install and Run

```bash
npm install
npm run dev
```

### 13. Deploy to GitHub Pages

```bash
npm run build
npm run deploy
```

## Conventions

- Use `@/` import alias for anything under `src/`
- Views (pages) go in `src/views/`, reusable components in `src/components/`
- Tests go in `src/components/__tests__/` using `*.test.ts` naming
