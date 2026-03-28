---
name: vue-routing
description: "Set up Vue Router with views, named routes, and route params. Use when: adding routing to a Vue app, creating new pages/views, setting up navigation, configuring Vue Router with history mode, adding dynamic route parameters, deploying Vue Router app to GitHub Pages."
argument-hint: "Description of the routes/pages needed"
---

# Vue Router Setup

## When to Use

- Adding client-side routing to a Vue 3 app
- Creating new pages/views with navigation
- Setting up dynamic routes with parameters
- Configuring for GitHub Pages deployment

## Router Configuration

Create `src/router/index.ts`:

```ts
import { createRouter, createWebHistory } from 'vue-router'
import HomeView from '../views/HomeView.vue'

const router = createRouter({
  history: createWebHistory('/REPO_NAME/'),
  routes: [
    {
      path: '/',
      name: 'home',
      component: HomeView,
    },
    // Add more routes as needed
  ],
})

export default router
```

## Key Conventions

### History Mode Base Path

- Use `createWebHistory('/REPO_NAME/')` where `REPO_NAME` matches your GitHub repo name
- This pairs with `base: './'` in `vite.config.ts` for GitHub Pages deployment
- For local dev, routes work relative to this base

### Route Types

| Pattern | Example | Use Case |
|---------|---------|----------|
| Static | `path: '/'` | Home/landing pages |
| Static named | `path: '/add'` | Form pages |
| Dynamic param | `path: '/details/:id'` | Detail/edit views |

### Named Routes

Always give routes a `name` for programmatic navigation:

```ts
// In templates
<RouterLink :to="{ name: 'details', params: { id: item.id } }">View</RouterLink>

// In setup()
import { useRouter } from 'vue-router'
const router = useRouter()
router.push({ name: 'home' })
router.push('/')  // string path also works
```

### Accessing Route Params

```ts
import { useRoute } from 'vue-router'
import { computed } from 'vue'

const route = useRoute()
const itemId = computed(() => route.params.id)
```

## Registration in main.ts

```ts
import { createApp } from 'vue'
import App from './App.vue'
import router from './router'

const app = createApp(App)
app.use(router)
app.mount('#app')
```

## Navigation in App.vue

Use `RouterLink` for nav links and `RouterView` for the page outlet:

```vue
<template>
  <nav>
    <RouterLink :to="{ name: 'home' }">Home</RouterLink>
    <RouterLink to="/add">Add</RouterLink>
  </nav>
  <RouterView></RouterView>
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

## Adding a New Route

1. Create the view component in `src/views/NewView.vue`
2. Import it in `src/router/index.ts`
3. Add the route object to the `routes` array
4. Add a `RouterLink` in `App.vue` navigation (if needed)

## Deployment Checklist

- `vite.config.ts` has `base: './'`
- Router uses `createWebHistory('/REPO_NAME/')`
- `package.json` has `"deploy": "gh-pages -d dist"` script
- Run `npm run build && npm run deploy`
