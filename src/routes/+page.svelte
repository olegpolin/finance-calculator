<script lang="ts">
  import { Button } from '$lib/components/ui/button';
  import { Input } from '$lib/components/ui/input';
  import { Label } from '$lib/components/ui/label';
  import * as Select from '$lib/components/ui/select';
  import * as Item from '$lib/components/ui/item';
  import * as Chart from '$lib/components/ui/chart';
  import { Progress } from '$lib/components/ui/progress';
  import { PieChart } from 'layerchart';

  type Occurrence = 'daily' | 'weekly' | 'monthly' | 'yearly';

  type SpendingItem = {
    id: string;
    name: string;
    amount: number;
    occurrence: Occurrence;
  };

  const occurrenceOptions: { value: Occurrence; label: string }[] = [
    { value: 'daily', label: 'Daily' },
    { value: 'weekly', label: 'Weekly' },
    { value: 'monthly', label: 'Monthly' },
    { value: 'yearly', label: 'Yearly' }
  ];

  const occurrenceMultiplier: Record<Occurrence, number> = {
    daily: 365,
    weekly: 52,
    monthly: 12,
    yearly: 1
  };

  let name = $state('');
  let amount = $state<number | null>(null);
  let occurrence = $state<Occurrence | ''>('');
  let items = $state<SpendingItem[]>([]);

  const yearlyFor = (item: SpendingItem) => item.amount * occurrenceMultiplier[item.occurrence];

  const totalYearly = $derived(items.reduce((sum, i) => sum + yearlyFor(i), 0));

  const sortedItems = $derived(
    items
      .map((item, index) => ({
        ...item,
        yearly: yearlyFor(item),
        color: `var(--chart-${(index % 10) + 1})`
      }))
      .sort((a, b) => b.yearly - a.yearly)
  );

  const chartData = $derived(
    sortedItems.map((item) => ({
      key: item.id,
      value: item.yearly,
      color: `var(--color-${item.id})`
    }))
  );

  const chartConfig = $derived(
    Object.fromEntries(
      sortedItems.map((item) => [item.id, { label: item.name, color: item.color }])
    ) as Chart.ChartConfig
  );

  const currency = (n: number) =>
    n.toLocaleString('en-US', { style: 'currency', currency: 'USD' });

  const occurrenceLabel = $derived(
    occurrence ? occurrenceOptions.find((o) => o.value === occurrence)?.label : 'Occurrence'
  );

  function addItem(e: Event) {
    e.preventDefault();
    if (!name.trim() || amount == null || amount <= 0 || !occurrence) return;
    items.push({
      id: crypto.randomUUID(),
      name: name.trim(),
      amount,
      occurrence
    });
    name = '';
    amount = null;
    occurrence = '';
  }

  function removeItem(id: string) {
    items = items.filter((i) => i.id !== id);
  }
</script>

<svelte:head>
  <title>Finance Calculator</title>
  <meta name="description" content="Calculate how much you are spending on everyday items." />
</svelte:head>

<div class="mx-auto w-full max-w-5xl px-4 pb-12 lg:px-8">
  <div
    class="border-2 border-border bg-primary text-primary-foreground rounded-md shadow-md mb-10 p-6 text-center"
  >
    <h1 class="text-4xl font-black tracking-tight uppercase">Finance Calculator</h1>
    <p class="mt-2 font-medium">
      Calculate how much you are spending on everyday items.
    </p>
  </div>

  <div class="grid gap-8 lg:grid-cols-2">
    <section class="space-y-6">
      <form
        class="border-2 border-border bg-card rounded-md shadow-md p-6 space-y-5"
        onsubmit={addItem}
      >
        <div class="space-y-2">
          <Label for="item-name" class="font-bold">Enter an item</Label>
          <Input id="item-name" placeholder="Item name" bind:value={name} />
        </div>

        <div class="space-y-2">
          <Label for="item-amount" class="font-bold">Enter your current spending in $</Label>
          <Input
            id="item-amount"
            type="number"
            min="0"
            step="0.01"
            placeholder="0"
            bind:value={amount}
          />
        </div>

        <div class="space-y-2">
          <Label for="item-occurrence" class="font-bold">
            Enter how often you pay for that item
          </Label>
          <Select.Root type="single" bind:value={occurrence}>
            <Select.Trigger id="item-occurrence" class="w-full">
              {occurrenceLabel}
            </Select.Trigger>
            <Select.Content>
              {#each occurrenceOptions as opt (opt.value)}
                <Select.Item value={opt.value}>{opt.label}</Select.Item>
              {/each}
            </Select.Content>
          </Select.Root>
        </div>

        <Button type="submit" class="w-full">Add item</Button>
      </form>

      <div class="space-y-3">
        <h2 class="text-sm font-black uppercase tracking-wide">Added items</h2>
        {#if items.length === 0}
          <p class="text-muted-foreground text-sm">No items added yet.</p>
        {:else}
          <Item.Group class="gap-3">
            {#each items as item (item.id)}
              <Item.Root variant="outline">
                <Item.Content>
                  <Item.Title class="font-bold">{item.name}</Item.Title>
                  <Item.Description>
                    {currency(item.amount)} / {item.occurrence} · {currency(yearlyFor(item))} per year
                  </Item.Description>
                </Item.Content>
                <Item.Actions>
                  <Button variant="outline" size="sm" onclick={() => removeItem(item.id)}>
                    Remove
                  </Button>
                </Item.Actions>
              </Item.Root>
            {/each}
          </Item.Group>
        {/if}
      </div>
    </section>

    <section class="space-y-6">
      <div class="border-2 border-border bg-card rounded-md shadow-md p-6">
        <div class="text-sm font-black uppercase tracking-wide">Total yearly spending</div>
        <div class="mt-1 text-5xl font-black tabular-nums">
          {currency(totalYearly)}
        </div>

        <div class="mt-6">
          {#if chartData.length === 0}
            <div
              class="border-2 border-dashed border-border rounded-md text-muted-foreground flex aspect-square items-center justify-center text-sm"
            >
              Add an item to see the breakdown.
            </div>
          {:else}
            <Chart.Container config={chartConfig} class="mx-auto aspect-square max-h-75">
              <PieChart
                data={chartData}
                key="key"
                value="value"
                c="color"
                innerRadius={90}
                padding={29}
                props={{ pie: { motion: 'tween' } }}
              >
                {#snippet tooltip()}
                  <Chart.Tooltip hideLabel />
                {/snippet}
              </PieChart>
            </Chart.Container>
          {/if}
        </div>
      </div>

      {#if sortedItems.length > 0}
        <div class="space-y-3">
          <h2 class="text-sm font-black uppercase tracking-wide">Top spending contributors</h2>
          <Item.Group class="gap-3">
            {#each sortedItems.slice(0, 3) as entry (entry.id)}
              {@const pct = totalYearly > 0 ? (entry.yearly / totalYearly) * 100 : 0}
              <Item.Root variant="outline">
                <Item.Media>
                  <span
                    class="inline-block size-4 rounded-sm border-2 border-border"
                    style="background-color: {entry.color};"
                  ></span>
                </Item.Media>
                <Item.Content>
                  <Item.Title class="font-bold">{entry.name}</Item.Title>
                  <Item.Description>
                    {currency(entry.yearly)} · Per year
                  </Item.Description>
                  <Progress value={pct} class="mt-2" style="--primary: {entry.color};" />
                </Item.Content>
                <Item.Actions>
                  <span class="text-sm font-bold tabular-nums">
                    {pct.toFixed(1)}%
                  </span>
                </Item.Actions>
              </Item.Root>
            {/each}
          </Item.Group>
        </div>
      {/if}
    </section>
  </div>
</div>
