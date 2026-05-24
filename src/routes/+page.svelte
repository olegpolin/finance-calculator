<script lang="ts">
  import { Button } from '$lib/components/ui/button';
  import { Input } from '$lib/components/ui/input';
  import { Label } from '$lib/components/ui/label';
  import * as Select from '$lib/components/ui/select';
  import * as Chart from '$lib/components/ui/chart';
  import { ArcChart } from 'layerchart';
</script>

<svelte:head>
  <title>Finance Calculator</title>
  <meta name="description" content="Calculate how much you are spending on everyday items." />
</svelte:head>

<div class="mx-auto max-w-350 px-4 pb-6 lg:px-8">
  <h1 class="text-center">Finance Calculator</h1>
  <p class="text-center">Calculate how much you are spending on everyday items.</p>
</div>
