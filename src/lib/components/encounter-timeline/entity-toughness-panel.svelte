<script lang="ts">
  import BossVitalsChart from "$lib/components/encounter-timeline/boss-vitals-chart.svelte";
  import type {
    EncounterCurveData,
    EncounterDetailData,
  } from "$lib/bindings";
  import { commands } from "$lib/bindings";
  import { entityCurvesToSeries } from "$lib/history-derived";
  import { t } from "$lib/i18n/index.svelte";

  type Props = {
    detail: EncounterDetailData;
  };

  let { detail }: Props = $props();

  let selectedIds = $state<string[]>([]);
  let curves = $state<Record<string, EncounterCurveData>>({});
  let loadingIds = $state<string[]>([]);
  let errorMessage = $state<string | null>(null);
  let showHp = $state(false);

  const index = $derived(detail.curveIndex ?? []);
  const series = $derived(
    entityCurvesToSeries(
      selectedIds
        .map((entityId) => curves[entityId])
        .filter((curve): curve is EncounterCurveData => Boolean(curve)),
    ),
  );
  const allSelected = $derived(
    index.length > 0 && selectedIds.length === index.length,
  );

  function label(row: (typeof index)[number]): string {
    const suffix = row.entityId.slice(-6);
    return row.monsterId == null
      ? `#${suffix}`
      : `#${row.monsterId} · ${suffix}`;
  }

  async function loadCurves(entityIds: string[]) {
    const missing = entityIds.filter(
      (entityId) =>
        !curves[entityId] && !loadingIds.includes(entityId),
    );
    if (missing.length === 0) return;
    loadingIds = [...loadingIds, ...missing];
    errorMessage = null;
    try {
      const result = await commands.getEncounterCurves(
        detail.encounterId,
        missing,
      );
      if (result.status === "error") {
        errorMessage = String(result.error);
        return;
      }
      const next = { ...curves };
      for (const curve of result.data) {
        next[curve.entityId] = curve;
      }
      curves = next;
    } catch (error) {
      errorMessage = String(error);
    } finally {
      loadingIds = loadingIds.filter(
        (entityId) => !missing.includes(entityId),
      );
    }
  }

  function toggle(entityId: string, checked: boolean) {
    if (checked) {
      if (!selectedIds.includes(entityId)) {
        selectedIds = [...selectedIds, entityId];
      }
      void loadCurves([entityId]);
    } else {
      selectedIds = selectedIds.filter((id) => id !== entityId);
    }
  }

  function selectAll() {
    selectedIds = index.map((row) => row.entityId);
    void loadCurves(selectedIds);
  }

  function clearAll() {
    selectedIds = [];
  }
</script>

<div class="grid gap-4 lg:grid-cols-[240px_1fr]">
  <aside class="border-border rounded-lg border p-3">
    <div class="mb-2 flex gap-2">
      <button
        type="button"
        class="border-border hover:bg-muted/50 rounded border px-2 py-1 text-xs disabled:opacity-40"
        disabled={allSelected || index.length === 0}
        onclick={selectAll}
      >
        {t("history.detail.bossVitals.selectAll")}
      </button>
      <button
        type="button"
        class="border-border hover:bg-muted/50 rounded border px-2 py-1 text-xs disabled:opacity-40"
        disabled={selectedIds.length === 0}
        onclick={clearAll}
      >
        {t("history.detail.bossVitals.clearAll")}
      </button>
    </div>

    {#if index.length === 0}
      <p class="text-muted-foreground text-xs">
        {t("history.detail.bossVitals.empty")}
      </p>
    {:else}
      <ul class="max-h-[320px] space-y-1 overflow-auto pr-1">
        {#each index as row (row.entityId)}
          <li>
            <label
              class="hover:bg-muted/40 flex cursor-pointer items-center gap-2 rounded px-1 py-1 text-xs"
            >
              <input
                type="checkbox"
                class="accent-primary"
                checked={selectedIds.includes(row.entityId)}
                onchange={(event) =>
                  toggle(
                    row.entityId,
                    (event.currentTarget as HTMLInputElement).checked,
                  )}
              />
              <span class="text-foreground flex-1 truncate">{label(row)}</span>
              <span class="text-muted-foreground tabular-nums">
                {row.pointCount}
              </span>
            </label>
          </li>
        {/each}
      </ul>
    {/if}
  </aside>

  <section class="min-w-0">
    <div class="mb-2 flex flex-wrap items-center justify-between gap-2">
      <h2 class="text-foreground text-sm font-medium">
        {t("history.detail.bossVitals.title")}
      </h2>
      <label class="text-muted-foreground flex items-center gap-1 text-xs">
        <input type="checkbox" class="accent-primary" bind:checked={showHp} />
        {t("history.detail.bossVitals.showHp")}
      </label>
    </div>

    {#if errorMessage}
      <p class="text-destructive mb-2 text-xs">{errorMessage}</p>
    {/if}

    {#if index.length === 0}
      <p class="text-muted-foreground text-sm">
        {t("history.detail.bossVitals.empty")}
      </p>
    {:else if selectedIds.length === 0}
      <p class="text-muted-foreground text-sm">
        {t("history.detail.bossVitals.selectPrompt")}
      </p>
    {:else if loadingIds.length > 0 && series.length === 0}
      <p class="text-muted-foreground text-sm">
        {t("history.detail.bossVitals.loading")}
      </p>
    {:else if series.length > 0}
      <BossVitalsChart
        {series}
        startMs={detail.startMs}
        endMs={detail.endMsExclusive}
        {showHp}
      />
    {:else}
      <p class="text-muted-foreground text-sm">
        {t("history.detail.bossVitals.empty")}
      </p>
    {/if}
  </section>
</div>
