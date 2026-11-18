<script lang="ts">
  // Entity vitals panel: one percentage line per selected entity (HP solid,
  // toughness dashed). It shares the history timeline palette and time
  // formatter, but keeps its own ECharts tooltip/legend because it is a
  // standalone tab rather than a gesture layer.
  import * as echarts from "echarts/core";
  import { LineChart } from "echarts/charts";
  import {
    GridComponent,
    LegendComponent,
    TooltipComponent,
  } from "echarts/components";
  import { CanvasRenderer } from "echarts/renderers";
  import { t } from "$lib/i18n/index.svelte";
  import type { EntityVitalsSeries } from "$lib/history-derived";
  import { formatTimeMs } from "./timeline-format";
  import { TIMELINE_PALETTE } from "./timeline-palette";

  echarts.use([
    LineChart,
    GridComponent,
    LegendComponent,
    TooltipComponent,
    CanvasRenderer,
  ]);

  type Props = {
    series: EntityVitalsSeries[];
    startMs: number;
    endMs: number;
    height?: number;
    /** Disabled by default: a trash pull can contain many mobs. */
    showHp?: boolean;
  };

  let { series, startMs, endMs, height = 300, showHp = true }: Props =
    $props();

  // One colour per entity; HP uses a solid line, toughness a dashed line of
  // the same colour so dense pulls stay readable.
  const SERIES_COLORS = [
    "#f87171",
    "#fbbf24",
    "#60a5fa",
    "#a78bfa",
    "#34d399",
    "#fb923c",
    "#22d3ee",
    "#f472b6",
    "#c084fc",
    "#4ade80",
  ];

  const chartOption = $derived.by((): echarts.EChartsCoreOption => {
    const lines: Record<string, unknown>[] = [];
    series.forEach((entity, index) => {
      const color = SERIES_COLORS[index % SERIES_COLORS.length];
      const suffix = entity.entityId.slice(-4);
      const label =
        entity.monsterId == null
          ? `#${suffix}`
          : `#${entity.monsterId}-${suffix}`;
      if (showHp && entity.hp.length > 0) {
        lines.push({
          name: `${label} · ${t("history.detail.bossVitals.hp")}`,
          type: "line",
          data: entity.hp,
          showSymbol: false,
          smooth: false,
          lineStyle: { width: 1.8, color },
          itemStyle: { color },
        });
      }
      if (entity.toughness.length > 0) {
        lines.push({
          name: `${label} · ${t("history.detail.bossVitals.toughness")}`,
          type: "line",
          data: entity.toughness,
          showSymbol: false,
          smooth: false,
          lineStyle: {
            width: 1.6,
            type: "dashed",
            color,
          },
          itemStyle: { color },
        });
      }
    });

    return {
      animation: false,
      backgroundColor: "transparent",
      grid: { left: 42, right: 14, top: 30, bottom: 24 },
      legend: {
        top: 0,
        type: "scroll",
        width: "82%",
        textStyle: { color: TIMELINE_PALETTE.fgMuted, fontSize: 11 },
      },
      tooltip: {
        trigger: "axis",
        backgroundColor: TIMELINE_PALETTE.tooltipBg,
        borderColor: TIMELINE_PALETTE.tooltipBorder,
        textStyle: { color: TIMELINE_PALETTE.fg, fontSize: 11 },
        valueFormatter: (value: unknown) =>
          `${Number(value).toFixed(1)}%`,
      },
      xAxis: {
        type: "value",
        min: startMs,
        max: Math.max(startMs + 1, endMs),
        axisLabel: {
          color: TIMELINE_PALETTE.fgMuted,
          fontSize: 10,
          formatter: (value: number) => formatTimeMs(value),
        },
        splitLine: { show: false },
        axisLine: { lineStyle: { color: TIMELINE_PALETTE.axis } },
        axisTick: { lineStyle: { color: TIMELINE_PALETTE.axis } },
      },
      yAxis: {
        type: "value",
        min: 0,
        max: 100,
        splitNumber: 4,
        axisLabel: {
          color: TIMELINE_PALETTE.fgMuted,
          fontSize: 10,
          formatter: "{value}%",
        },
        splitLine: { lineStyle: { color: TIMELINE_PALETTE.grid } },
      },
      series: lines,
    };
  });

  function chartAttachment(node: HTMLDivElement) {
    const chart = echarts.init(node, null, { renderer: "canvas" });
    const resizeObserver = new ResizeObserver(() => chart.resize());
    resizeObserver.observe(node);

    const effectCleanup = $effect.root(() => {
      $effect(() => {
        chart.setOption(chartOption, { notMerge: true });
      });
    });

    return () => {
      effectCleanup();
      resizeObserver.disconnect();
      chart.dispose();
    };
  }
</script>

<div
  class="w-full"
  style:height={`${height}px`}
  {@attach chartAttachment}
></div>
