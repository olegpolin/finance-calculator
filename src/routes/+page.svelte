<script lang="ts">
  import { PersistedState } from 'runed';
  import { Button } from '$lib/components/ui/button';
  import { Input } from '$lib/components/ui/input';
  import { Label } from '$lib/components/ui/label';
  import * as Select from '$lib/components/ui/select';
  import * as Item from '$lib/components/ui/item';
  import * as AlertDialog from '$lib/components/ui/alert-dialog';
  import * as Chart from '$lib/components/ui/chart';
  import { buttonVariants } from '$lib/components/ui/button';
  import { Progress } from '$lib/components/ui/progress';
  import { Switch } from '$lib/components/ui/switch';
  import { PieChart } from 'layerchart';

  type Occurrence = 'daily' | 'weekly' | 'monthly' | 'yearly';

  type SpendingItem = {
    id: string;
    name: string;
    amount: number;
    occurrence: Occurrence;
  };

  type FormErrors = {
    name: string | null;
    amount: string | null;
    occurrence: string | null;
  };

  const STORAGE_KEY = 'finance-calculator:items';

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

  const validOccurrences = occurrenceOptions.map((o) => o.value);

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

  const yearlyFor = (item: SpendingItem) => item.amount * occurrenceMultiplier[item.occurrence];

  const currency = (n: number) =>
    n.toLocaleString('en-US', { style: 'currency', currency: 'USD' });

  function validateForm(
    submitted: boolean,
    name: string,
    amount: number | null,
    occurrence: Occurrence | ''
  ): FormErrors {
    if (!submitted) return { name: null, amount: null, occurrence: null };
    return {
      name: !name.trim() ? 'Enter an item name.' : null,
      amount:
        amount == null
          ? 'Enter an amount.'
          : amount <= 0
            ? 'Amount must be greater than 0.'
            : null,
      occurrence: !occurrence ? 'Choose how often you pay.' : null
    };
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

  // Period view (yearly vs monthly) — data stays normalized to yearly internally.
  let monthly = $state(false);
  const periodLabel = $derived(monthly ? 'month' : 'year');
  const periodTitle = $derived(monthly ? 'monthly' : 'yearly');
  const toPeriod = (yearly: number) => (monthly ? yearly / 12 : yearly);

  // Add-form state (top of page)
  let name = $state('');
  let amount = $state<number | null>(null);
  let occurrence = $state<Occurrence | ''>('');
  let addSubmitted = $state(false);

  const addErrors = $derived(validateForm(addSubmitted, name, amount, occurrence));

  // Inline-edit state (one row at a time)
  let editingId = $state<string | null>(null);
  let editName = $state('');
  let editAmount = $state<number | null>(null);
  let editOccurrence = $state<Occurrence | ''>('');
  let editSubmitted = $state(false);
  let editNameInputEl = $state<HTMLInputElement | null>(null);

  const editErrors = $derived(
    validateForm(editSubmitted, editName, editAmount, editOccurrence)
  );
  const editOccurrenceLabel = $derived(
    editOccurrence
      ? occurrenceOptions.find((o) => o.value === editOccurrence)?.label
      : 'Occurrence'
  );

  const items = $derived(persistedItems.current);

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
      label: item.name,
      value: item.yearly,
      color: `var(--color-${item.id})`
    }))
  );

  const chartConfig = $derived(
    Object.fromEntries(
      sortedItems.map((item) => [item.id, { label: item.name, color: item.color }])
    ) as Chart.ChartConfig
  );

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

  function clearAll() {
    persistedItems.current = [];
    resetEdit();
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
            aria-invalid={addErrors.name ? 'true' : undefined}
            aria-describedby={addErrors.name ? 'item-name-error' : undefined}
          />
          {#if addErrors.name}
            <p id="item-name-error" class="text-destructive text-sm font-medium">
              {addErrors.name}
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
            aria-invalid={addErrors.amount ? 'true' : undefined}
            aria-describedby={addErrors.amount ? 'item-amount-error' : undefined}
          />
          {#if addErrors.amount}
            <p id="item-amount-error" class="text-destructive text-sm font-medium">
              {addErrors.amount}
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
              aria-invalid={addErrors.occurrence ? 'true' : undefined}
              aria-describedby={addErrors.occurrence ? 'item-occurrence-error' : undefined}
            >
              {occurrenceLabel}
            </Select.Trigger>
            <Select.Content>
              {#each occurrenceOptions as opt (opt.value)}
                <Select.Item value={opt.value}>{opt.label}</Select.Item>
              {/each}
            </Select.Content>
          </Select.Root>
          {#if addErrors.occurrence}
            <p id="item-occurrence-error" class="text-destructive text-sm font-medium">
              {addErrors.occurrence}
            </p>
          {/if}
        </div>

        <Button type="submit" class="w-full">Add item</Button>
      </form>

      <div class="space-y-3">
        <div class="flex items-center justify-between gap-2">
          <h2 class="text-sm font-black uppercase tracking-wide">Added items</h2>
          {#if items.length > 0}
            <AlertDialog.Root>
              <AlertDialog.Trigger
                class={buttonVariants({ variant: 'outline', size: 'sm' })}
              >
                Clear all
              </AlertDialog.Trigger>
              <AlertDialog.Content>
                <AlertDialog.Header>
                  <AlertDialog.Title class="font-black uppercase tracking-tight">
                    Clear all items?
                  </AlertDialog.Title>
                  <AlertDialog.Description>
                    This removes all {items.length}
                    {items.length === 1 ? 'item' : 'items'} from your list. This action cannot
                    be undone.
                  </AlertDialog.Description>
                </AlertDialog.Header>
                <AlertDialog.Footer>
                  <AlertDialog.Cancel>Cancel</AlertDialog.Cancel>
                  <AlertDialog.Action variant="destructive" onclick={clearAll}>
                    Clear all
                  </AlertDialog.Action>
                </AlertDialog.Footer>
              </AlertDialog.Content>
            </AlertDialog.Root>
          {/if}
        </div>
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
                      aria-invalid={editErrors.name ? 'true' : undefined}
                      aria-describedby={editErrors.name ? `edit-name-${item.id}-error` : undefined}
                    />
                    {#if editErrors.name}
                      <p
                        id={`edit-name-${item.id}-error`}
                        class="text-destructive text-xs font-medium"
                      >
                        {editErrors.name}
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
                        aria-invalid={editErrors.amount ? 'true' : undefined}
                        aria-describedby={editErrors.amount
                          ? `edit-amount-${item.id}-error`
                          : undefined}
                      />
                      {#if editErrors.amount}
                        <p
                          id={`edit-amount-${item.id}-error`}
                          class="text-destructive text-xs font-medium"
                        >
                          {editErrors.amount}
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
                          aria-invalid={editErrors.occurrence ? 'true' : undefined}
                          aria-describedby={editErrors.occurrence
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
                      {#if editErrors.occurrence}
                        <p
                          id={`edit-occurrence-${item.id}-error`}
                          class="text-destructive text-xs font-medium"
                        >
                          {editErrors.occurrence}
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
                      {currency(item.amount)} / {item.occurrence} · {currency(toPeriod(yearlyFor(item)))} per {periodLabel}
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
        <div class="flex items-center justify-between gap-3">
          <div class="text-sm font-black uppercase tracking-wide">
            Total {periodTitle} spending
          </div>
          <div class="flex items-center gap-2 text-xs font-bold uppercase">
            <span class={monthly ? 'text-muted-foreground' : ''}>Yearly</span>
            <Switch bind:checked={monthly} aria-label="Toggle monthly view" />
            <span class={monthly ? '' : 'text-muted-foreground'}>Monthly</span>
          </div>
        </div>

        <div class="mt-4">
          {#if chartData.length === 0}
            <div
              class="border-2 border-dashed border-border rounded-md text-muted-foreground flex aspect-square flex-col items-center justify-center gap-2 px-4 text-center text-sm"
            >
              <div class="text-foreground text-4xl font-black tabular-nums">
                {currency(0)}
              </div>
              <p>Add an item to see the breakdown.</p>
            </div>
          {:else}
            <div class="relative mx-auto aspect-square max-h-75">
              <Chart.Container config={chartConfig} class="aspect-square h-full w-full">
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
                    <Chart.Tooltip hideLabel>
                      {#snippet formatter({ value, item })}
                        {@const num = Number(value)}
                        {@const pct = totalYearly > 0 ? (num / totalYearly) * 100 : 0}
                        {@const lookupKey = String(item.key ?? item.label ?? '')}
                        {@const displayLabel = chartConfig[lookupKey]?.label ?? item.label}
                        <span
                          style="--color-bg: {item.color}; --color-border: {item.color};"
                          class="border-(--color-border) bg-(--color-bg) size-2.5 shrink-0 rounded-xs"
                        ></span>
                        <div
                          class="flex flex-1 items-center justify-between gap-3 leading-none"
                        >
                          <span class="text-muted-foreground">{displayLabel}</span>
                          <span class="text-foreground font-mono font-medium tabular-nums">
                            {currency(toPeriod(num))}
                            <span class="text-muted-foreground ml-1 font-normal">
                              ({pct.toFixed(1)}%)
                            </span>
                          </span>
                        </div>
                      {/snippet}
                    </Chart.Tooltip>
                  {/snippet}
                </PieChart>
              </Chart.Container>
              <div
                class="pointer-events-none absolute inset-0 flex flex-col items-center justify-center text-center"
              >
                <div
                  class="text-muted-foreground text-[0.625rem] font-black uppercase tracking-widest"
                >
                  Per {periodLabel}
                </div>
                <div class="text-3xl font-black tabular-nums leading-tight">
                  {currency(toPeriod(totalYearly))}
                </div>
                <div class="text-muted-foreground text-xs font-medium">
                  {items.length} {items.length === 1 ? 'item' : 'items'}
                </div>
              </div>
            </div>
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
                    {currency(toPeriod(entry.yearly))} · Per {periodLabel}
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
