<script lang="ts">
  import './layout.css';
  import favicon from '$lib/assets/favicon.svg';
  import { ModeWatcher } from 'mode-watcher';
  import Header from '$lib/components/header.svelte';

  let { children } = $props();
</script>

<svelte:head><link rel="icon" href={favicon} /></svelte:head>

<ModeWatcher defaultMode="light" />

<div class="flex min-h-svh flex-col">
  <Header />

  <main class="flex flex-1 flex-col">
    {@render children()}
  </main>
</div>
