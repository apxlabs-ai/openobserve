<!-- Copyright 2026 OpenObserve Inc.

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU Affero General Public License as published by
the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.

This program is distributed in the hope that it will be useful
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU Affero General Public License for more details.

You should have received a copy of the GNU Affero General Public License
along with this program.  If not, see <http://www.gnu.org/licenses/>.
-->

<!-- eslint-disable vue/v-on-event-hyphenation -->
<!-- eslint-disable vue/attribute-hyphenation -->
<template>
  <div
    data-test="chart-renderer"
    ref="chartRef"
    id="chart"
    @mouseover="handleMouseOver"
    @mouseleave="handleMouseLeave"
    style="height: 100%; width: 100%"
  ></div>
</template>

<script>
import {
  defineComponent,
  ref,
  onMounted,
  watch,
  onUnmounted,
  nextTick,
  inject,
} from "vue";
import * as echarts from "echarts";

// Import all components and renderers once for generic usage
import {
  TitleComponent,
  TooltipComponent,
  GridComponent,
  ToolboxComponent,
  LegendComponent,
  DataZoomComponent,
} from "echarts/components";
import { CanvasRenderer, SVGRenderer } from "echarts/renderers";

import DOMPurify from "dompurify";

// Register necessary components
echarts.use([
  TitleComponent,
  TooltipComponent,
  GridComponent,
  ToolboxComponent,
  LegendComponent,
  DataZoomComponent,
  CanvasRenderer,
  SVGRenderer,
]);

export default defineComponent({
  name: "CustomChartRenderer",
  emits: ["error", "mousemove", "mouseout", "click"],
  props: {
    data: {
      required: true,
      type: Object,
      default: () => ({}),
    },
    renderType: {
      type: String,
      default: "canvas",
    },
    height: {
      type: String,
      default: "100%",
    },
  },
  setup(props, { emit }) {
    const chartRef = ref(null);
    let chart = null;

    const hoveredSeriesState = inject("hoveredSeriesState", null);
    const looksLikeFunctionSource = (value) =>
      /^(?:async\s+)?function\b|^(?:async\s*)?(?:\([^)]*\)|[A-Za-z_$][\w$]*)\s*=>/.test(
        value.trim(),
      );

    const stripExecutableValues = (value) => {
      if (
        typeof value === "function" ||
        (typeof value === "string" && looksLikeFunctionSource(value))
      ) {
        return undefined;
      }
      if (Array.isArray(value)) {
        return value
          .map(stripExecutableValues)
          .filter((item) => item !== undefined);
      }
      if (typeof value === "object" && value !== null) {
        return Object.fromEntries(
          Object.entries(value)
            .map(([key, item]) => [key, stripExecutableValues(item)])
            .filter(([, item]) => item !== undefined),
        );
      }
      return value;
    };
    function deepSanitize(obj) {
      if (typeof obj === "string") {
        return DOMPurify.sanitize(obj);
      } else if (Array.isArray(obj)) {
        return obj.map(deepSanitize);
      } else if (typeof obj === "object" && obj !== null) {
        return Object.fromEntries(
          Object.entries(obj).map(([key, value]) => [key, deepSanitize(value)]),
        );
      }
      return obj;
    }

    const initChart = async () => {
      if (!chartRef.value) return;

      const echartsGL = await import("echarts-gl");

      // Initialize chart if it doesn't exist, otherwise reuse existing instance
      if (!chart) {
        chart = echarts.init(chartRef.value, undefined, {
          renderer: "canvas",
        });
      } else {
        // Clear previous chart data to prevent overlay of old and new charts
        chart.clear();
      }

      try {
        const dataOnlyOptions = stripExecutableValues(props.data);
        const safeChartOptions = deepSanitize(dataOnlyOptions);
        chart.setOption(safeChartOptions);
      } catch (e) {
        emit({
          message: "Error while executing the code",
          code: "",
        });
      }
    };

    const handleResize = async () => {
      await nextTick();
      chart?.resize();
    };

    const handleMouseOver = () => {
      if (hoveredSeriesState)
        hoveredSeriesState.panelId = props.data.extras?.panelId;
    };

    const handleMouseLeave = () => {
      // if (hoveredSeriesState) hoveredSeriesState.setIndex(-1, -1, -1, null);
    };

    onMounted(() => {
      initChart();

      // Watch for window resize
      window.addEventListener("resize", handleResize);
    });
    watch(
      () => props.data,
      async () => {
        await initChart(); // Re-initialize chart when option change
      },
      { deep: true },
    );

    onUnmounted(() => {
      chart?.dispose();
      window.removeEventListener("resize", handleResize);
    });

    return { chartRef, handleMouseOver, handleMouseLeave };
  },
});
</script>
