<template>
  <Suspense @resolve="onResolve">
    <ErrorComponent v-if="error" :error="error" />
    <IslandRenderer v-else-if="islandContext" :context="islandContext" />
    <component :is="SingleRenderer" v-else-if="SingleRenderer" />
    <AppComponent v-else />
  </Suspense>
</template>

<script setup>
import { defineAsyncComponent, onErrorCaptured, onServerPrefetch, provide } from 'vue'
import { useNuxtApp } from '../nuxt'
import { isNuxtError, showError, useError } from '../composables/error'
import { useRoute } from '../composables/router'
import { PageRouteSymbol } from '../components/injections'
import AppComponent from '#build/app-component.mjs'
import ErrorComponent from '#build/error-component.mjs'

const IslandRenderer = import.meta.server
  ? defineAsyncComponent(() => import('./island-renderer').then(r => r.default || r))
  : () => null

const nuxtApp = useNuxtApp()
const onResolve = nuxtApp.deferHydration()

const url = import.meta.server ? nuxtApp.ssrContext.url : window.location.pathname
const SingleRenderer = import.meta.test && import.meta.dev && import.meta.server && url.startsWith('/__nuxt_component_test__/') && defineAsyncComponent(() => import('#build/test-component-wrapper.mjs')
  .then(r => r.default(import.meta.server ? url : window.location.href)))

// Inject default route (outside of pages) as active route
provide(PageRouteSymbol, useRoute())

// vue:setup hook
const results = nuxtApp.hooks.callHookWith(hooks => hooks.map(hook => hook()), 'vue:setup')
if (import.meta.dev && results && results.some(i => i && 'then' in i)) {
  console.error('[nuxt] Error in `vue:setup`. Callbacks must be synchronous.')
}

// error handling
const error = useError()
onErrorCaptured((err, target, info) => {
  nuxtApp.hooks.callHook('vue:error', err, target, info).catch(hookError => console.error('[nuxt] Error in `vue:error` hook', hookError))
  if (import.meta.server || (isNuxtError(err) && (err.fatal || err.unhandled))) {
    const p = nuxtApp.runWithContext(() => showError(err))
    onServerPrefetch(() => p)
    return false // suppress error from breaking render
  }
})

// Component islands context
const islandContext = import.meta.server && nuxtApp.ssrContext.islandContext
</script>
