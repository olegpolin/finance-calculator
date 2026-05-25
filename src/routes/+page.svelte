<script lang="ts">
  import { PersistedState } from 'runed';
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

  const STORAGE_KEY = 'finance-calculator:items';
  const validOccurrences: Occurrence[] = ['daily', 'weekly', 'monthly', 'yearly'];

  function isSpendingItem(x: unknown): x is SpendingItem {
    if (typeof x !== 'object' || x === null) return false;
    const v = x as Record<string, unknown>;
    return (
      typeof v.id === 'string' &&
      typeof v.name === 'string' &&
      typeof v.amount === 'number' &&
      Number.isFinite(v.amount) &&
      typeof v.occurrence === 'string' &&
      validOccurrences.includes(v.occurrence as Occurrence)
    );
  }

  const persistedItems = new PersistedState<SpendingItem[]>(STORAGE_KEY, [], {
    serializer: {
      serialize: JSON.stringify,
      deserialize: (raw) => {
        try {
          const parsed: unknown = JSON.parse(raw);
          if (Array.isArray(parsed)) return parsed.filter(isSpendingItem);
        } catch {
          // fall through to default
        }
        return [];
      }
    }
  });

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

  // Add-form state (top of page)
  let name = $state('');
  let amount = $state<number | null>(null);
  let occurrence = $state<Occurrence | ''>('');
  let addSubmitted = $state(false);

  const nameError = $derived(addSubmitted && !name.trim() ? 'Enter an item name.' : null);
  const amountError = $derived(
    addSubmitted
      ? amount == null
        ? 'Enter an amount.'
        : amount <= 0
          ? 'Amount must be greater than 0.'
          : null
      : null
  );
  const occurrenceError = $derived(
    addSubmitted && !occurrence ? 'Choose how often you pay.' : null
  );

  // Inline-edit state (one row at a time)
  let editingId = $state<string | null>(null);
  let editName = $state('');
  let editAmount = $state<number | null>(null);
  let editOccurrence = $state<Occurrence | ''>('');
  let editSubmitted = $state(false);
  let editNameInputEl = $state<HTMLInputElement | null>(null);

  const editNameError = $derived(
    editSubmitted && !editName.trim() ? 'Enter an item name.' : null
  );
  const editAmountError = $derived(
    editSubmitted
      ? editAmount == null
        ? 'Enter an amount.'
        : editAmount <= 0
          ? 'Amount must be greater than 0.'
          : null
      : null
  );
  const editOccurrenceError = $derived(
    editSubmitted && !editOccurrence ? 'Choose how often you pay.' : null
  );
  const editOccurrenceLabel = $derived(
    editOccurrence
      ? occurrenceOptions.find((o) => o.value === editOccurrence)?.label
      : 'Occurrence'
  );

  const items = $derived(persistedItems.current);

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

  function resetAddForm() {
    name = '';
    amount = null;
    occurrence = '';
    addSubmitted = false;
  }

  function resetEdit() {
    editingId = null;
    editName = '';
    editAmount = null;
    editOccurrence = '';
    editSubmitted = false;
  }

  function startEdit(item: SpendingItem) {
    editingId = item.id;
    editName = item.name;
    editAmount = item.amount;
    editOccurrence = item.occurrence;
    editSubmitted = false;
    queueMicrotask(() => editNameInputEl?.focus());
  }

  function cancelEdit() {
    resetEdit();
  }

  function addItem(e: Event) {
    e.preventDefault();
    addSubmitted = true;
    if (!name.trim() || amount == null || amount <= 0 || !occurrence) return;
    persistedItems.current = [
      ...persistedItems.current,
      {
        id: crypto.randomUUID(),
        name: name.trim(),
        amount,
        occurrence
      }
    ];
    resetAddForm();
  }

  function saveEdit(e: Event) {
    e.preventDefault();
    editSubmitted = true;
    if (!editName.trim() || editAmount == null || editAmount <= 0 || !editOccurrence) return;
    const id = editingId;
    if (!id) return;
    const trimmed = editName.trim();
    const newAmount = editAmount;
    const newOccurrence = editOccurrence;
    persistedItems.current = persistedItems.current.map((i) =>
      i.id === id
        ? { ...i, name: trimmed, amount: newAmount, occurrence: newOccurrence }
        : i
    );
    resetEdit();
  }

  function removeItem(id: string) {
    persistedItems.current = persistedItems.current.filter((i) => i.id !== id);
    if (editingId === id) resetEdit();
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
          <Input
            id="item-name"
            placeholder="Item name"
            bind:value={name}
            required
            aria-invalid={nameError ? 'true' : undefined}
            aria-describedby={nameError ? 'item-name-error' : undefined}
          />
          {#if nameError}
            <p id="item-name-error" class="text-destructive text-sm font-medium">
              {nameError}
            </p>
          {/if}
        </div>

        <div class="space-y-2">
          <Label for="item-amount" class="font-bold">Enter your current spending in $</Label>
          <Input
            id="item-amount"
            type="number"
            min="0.01"
            step="0.01"
            placeholder="0"
            bind:value={amount}
            required
            aria-invalid={amountError ? 'true' : undefined}
            aria-describedby={amountError ? 'item-amount-error' : undefined}
          />
          {#if amountError}
            <p id="item-amount-error" class="text-destructive text-sm font-medium">
              {amountError}
            </p>
          {/if}
        </div>

        <div class="space-y-2">
          <Label for="item-occurrence" class="font-bold">
            Enter how often you pay for that item
          </Label>
          <Select.Root type="single" bind:value={occurrence}>
            <Select.Trigger
              id="item-occurrence"
              class="w-full"
              aria-invalid={occurrenceError ? 'true' : undefined}
              aria-describedby={occurrenceError ? 'item-occurrence-error' : undefined}
            >
              {occurrenceLabel}
            </Select.Trigger>
            <Select.Content>
              {#each occurrenceOptions as opt (opt.value)}
                <Select.Item value={opt.value}>{opt.label}</Select.Item>
              {/each}
            </Select.Content>
          </Select.Root>
          {#if occurrenceError}
            <p id="item-occurrence-error" class="text-destructive text-sm font-medium">
              {occurrenceError}
            </p>
          {/if}
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
              {#if editingId === item.id}
                <form
                  class="border-2 border-border bg-card rounded-md shadow-md p-4 space-y-3"
                  onsubmit={saveEdit}
                >
                  <div class="space-y-1">
                    <Label for={`edit-name-${item.id}`} class="text-xs font-bold uppercase">
                      Name
                    </Label>
                    <Input
                      id={`edit-name-${item.id}`}
                      bind:ref={editNameInputEl}
                      bind:value={editName}
                      required
                      aria-invalid={editNameError ? 'true' : undefined}
                      aria-describedby={editNameError ? `edit-name-${item.id}-error` : undefined}
                    />
                    {#if editNameError}
                      <p
                        id={`edit-name-${item.id}-error`}
                        class="text-destructive text-xs font-medium"
                      >
                        {editNameError}
                      </p>
                    {/if}
                  </div>

                  <div class="grid gap-3 sm:grid-cols-2">
                    <div class="space-y-1">
                      <Label
                        for={`edit-amount-${item.id}`}
                        class="text-xs font-bold uppercase"
                      >
                        Amount ($)
                      </Label>
                      <Input
                        id={`edit-amount-${item.id}`}
                        type="number"
                        min="0.01"
                        step="0.01"
                        bind:value={editAmount}
                        required
                        aria-invalid={editAmountError ? 'true' : undefined}
                        aria-describedby={editAmountError
                          ? `edit-amount-${item.id}-error`
                          : undefined}
                      />
                      {#if editAmountError}
                        <p
                          id={`edit-amount-${item.id}-error`}
                          class="text-destructive text-xs font-medium"
                        >
                          {editAmountError}
                        </p>
                      {/if}
                    </div>
                    <div class="space-y-1">
                      <Label
                        for={`edit-occurrence-${item.id}`}
                        class="text-xs font-bold uppercase"
                      >
                        Frequency
                      </Label>
                      <Select.Root type="single" bind:value={editOccurrence}>
                        <Select.Trigger
                          id={`edit-occurrence-${item.id}`}
                          class="w-full"
                          aria-invalid={editOccurrenceError ? 'true' : undefined}
                          aria-describedby={editOccurrenceError
                            ? `edit-occurrence-${item.id}-error`
                            : undefined}
                        >
                          {editOccurrenceLabel}
                        </Select.Trigger>
                        <Select.Content>
                          {#each occurrenceOptions as opt (opt.value)}
                            <Select.Item value={opt.value}>{opt.label}</Select.Item>
                          {/each}
                        </Select.Content>
                      </Select.Root>
                      {#if editOccurrenceError}
                        <p
                          id={`edit-occurrence-${item.id}-error`}
                          class="text-destructive text-xs font-medium"
                        >
                          {editOccurrenceError}
                        </p>
                      {/if}
                    </div>
                  </div>

                  <div class="flex justify-end gap-2">
                    <Button type="button" variant="outline" size="sm" onclick={cancelEdit}>
                      Cancel
                    </Button>
                    <Button type="submit" size="sm">Save</Button>
                  </div>
                </form>
              {:else}
                <Item.Root variant="outline">
                  <Item.Content>
                    <Item.Title class="font-bold">{item.name}</Item.Title>
                    <Item.Description>
                      {currency(item.amount)} / {item.occurrence} · {currency(yearlyFor(item))} per year
                    </Item.Description>
                  </Item.Content>
                  <Item.Actions class="gap-2">
                    <Button
                      variant="outline"
                      size="sm"
                      aria-label={`Edit ${item.name}`}
                      onclick={() => startEdit(item)}
                    >
                      Edit
                    </Button>
                    <Button
                      variant="outline"
                      size="sm"
                      aria-label={`Remove ${item.name}`}
                      onclick={() => removeItem(item.id)}
                    >
                      Remove
                    </Button>
                  </Item.Actions>
                </Item.Root>
              {/if}
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
                props={{
                  pie: { motion: 'tween' },
                  arc: { class: 'stroke-2 stroke-border' }
                }}
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
            {#each sortedItems as entry (entry.id)}
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
