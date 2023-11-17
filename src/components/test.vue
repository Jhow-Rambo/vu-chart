<template>
  <div>
    <svg :width="width" :height="height" @mousemove="handleMouseMove" @mouseleave="handleMouseLeave">
      <!-- Linhas do gráfico -->
      <g v-for="line in lines" :key="line.id">
        <!-- Adicione a área preenchida com gradiente -->
        <defs v-if="hasGradient">
          <linearGradient :id="'gradient-' + line.id" x1="0" x2="0" y1="0" y2="1">
            <stop :offset="0" :stop-color="line.color" :stop-opacity="0.2" />
            <stop offset="50%" :stop-color="line.color" stop-opacity="0.1" />
            <stop offset="100%" stop-color="white" stop-opacity="0" />
          </linearGradient>
        </defs>
        <path :d="line.path" :stroke="line.color" :stroke-width="line.width" fill="none" />
        <g>
          <!-- Adicione pontos de dados com eventos de mouse -->
          <circle v-for="(point, index) in line.data" :key="index" :cx="xScale(point.x)" :cy="yScale(point.y)" :r="4"
            :fill="line.color" @mouseover="pointHovered = { ...point, id: line.id }" @mouseleave="pointHovered = null" />
          <!-- Adicione linhas pontilhadas quando o mouse está sobre o ponto -->
          <line v-if="pointHovered && pointHovered.id === line.id" :x1="xScale(pointHovered.x)"
            :y1="yScale(pointHovered.y)" :x2="xScale(pointHovered.x)" :y2="height - margin.bottom" stroke-dasharray="4"
            stroke="#ccc" />
          <line v-if="pointHovered && pointHovered.id === line.id" :x1="xScale(pointHovered.x)"
            :y1="yScale(pointHovered.y)" :x2="margin.left" :y2="yScale(pointHovered.y)" stroke-dasharray="4"
            stroke="#ccc" />
          <!-- Adicione a área preenchida com gradiente -->
          <path v-if="hasGradient" :d="getAreaPath(line.data, line.id)" :fill="`url(#gradient-${line.id})`" />
        </g>
      </g>

      <!-- Eixos -->
      <g class="x-axis" :transform="`translate(0, ${height - margin.bottom})`"></g>
      <g class="y-axis" :transform="`translate(${margin.left}, 0)`"></g>

      <!-- Rótulos dos Eixos -->
      <text :x="width / 2" :y="height - margin.bottom / 2" text-anchor="middle">{{ xLabel }}</text>
      <text :x="-(height / 2)" :y="margin.left / 2" text-anchor="middle" transform="rotate(-90)">{{ yLabel }}</text>

      <!-- Grid -->
      <g v-if="grid">
        <g class="x-grid" :transform="`translate(0, ${height - margin.bottom})`"></g>
        <g class="y-grid" :transform="`translate(${margin.left}, 0)`"></g>
      </g>

      <!-- Tooltip -->
      <g v-if="pointHovered">
        <rect :x="xScale(pointHovered.x) + margin.left - 40" :y="yScale(pointHovered.y) + margin.top - 40" width="80"
          height="30" rx="5" ry="5" fill="#f9f9f9" stroke="#333" stroke-width="1" />
        <text :x="xScale(pointHovered.x) + margin.left" :y="yScale(pointHovered.y) + margin.top - 23" text-anchor="middle"
          dy="0.3em">{{ `(${pointHovered.x}, ${pointHovered.y})` }}</text>
      </g>
    </svg>
  </div>
</template>

<script>
import * as d3 from 'd3';

export default {
  props: {
    width: {
      type: Number,
      default: 400,
    },
    height: {
      type: Number,
      default: 300,
    },
    margin: {
      type: Object,
      default: () => ({ top: 20, right: 20, bottom: 30, left: 40 }),
    },
    series: {
      type: Array,
      default: () => [
        {
          data: [],
          color: '',
          width: 2,
        },
        // Adicione mais séries conforme necessário
      ],
    },
    xLabel: {
      type: String,
      default: 'X-Axis',
    },
    yLabel: {
      type: String,
      default: 'Y-Axis',
    },
    grid: {
      type: Boolean,
      default: false,
    },
    hasGradient: {
      type: Boolean,
      default: true,
    },
  },
  data() {
    return {
      pointHovered: null,
    };
  },
  computed: {
    xScale() {
      const allData = this.series.flatMap((serie) => serie.data);
      return d3
        .scaleLinear()
        .domain([d3.min(allData, (d) => d.x), d3.max(allData, (d) => d.x)])
        .range([this.margin.left, this.width - this.margin.right])
        .nice();
    },
    yScale() {
      const allData = this.series.flatMap((serie) => serie.data);
      return d3
        .scaleLinear()
        .domain([d3.min(allData, (d) => d.y), d3.max(allData, (d) => d.y)])
        .range([this.height - this.margin.bottom, this.margin.top]);
    },
    lines() {
      const lineGenerator = d3.line().x((d) => this.xScale(d.x)).y((d) => this.yScale(d.y));

      return this.series.map((serie, index) => ({
        id: `line${index + 1}`,
        path: lineGenerator(serie.data),
        color: serie.color || 'blue',
        width: serie.width || 2,
        data: serie.data,
      }));
    },
  },
  methods: {
    renderAxes() {
      const xAxis = d3.axisBottom(this.xScale).ticks(d3.max(this.series.flatMap((serie) => serie.data), (d) => d.x));
      const yAxis = d3.axisLeft(this.yScale);

      d3.select('.x-axis').call(xAxis);
      d3.select('.y-axis').call(yAxis);

      // Adiciona grid se a propriedade grid for verdadeira
      if (this.grid) {
        d3.select('.x-grid').call(d3.axisBottom(this.xScale).tickSize(-this.height).tickFormat(''));
        d3.select('.y-grid').call(d3.axisLeft(this.yScale).tickSize(-this.width).tickFormat(''));
      }
    },
    handleMouseMove(event) {
      const mouseX = event.pageX - this.$el.getBoundingClientRect().left;
      const mouseY = event.pageY - this.$el.getBoundingClientRect().top;

      // Encontrar o ponto mais próximo ao mouse
      let closestPoint = null;
      let closestDistance = Number.MAX_SAFE_INTEGER;

      for (const line of this.lines) {
        for (const point of line.data) {
          const distance = Math.sqrt((this.xScale(point.x) - mouseX) ** 2 + (this.yScale(point.y) - mouseY) ** 2);

          if (distance < closestDistance) {
            closestDistance = distance;
            closestPoint = { ...point, id: line.id };
          }
        }
      }

      this.pointHovered = closestPoint;
    },
    handleMouseLeave() {
      this.pointHovered = null;
    },
    getAreaPath(data, id) {
      const areaGenerator = d3
        .area()
        .x((d) => this.xScale(d.x))
        .y1((d) => this.yScale(d.y))
        .y0(this.height - this.margin.bottom);
      return areaGenerator(data);
    },
  },
  mounted() {
    this.renderAxes();
  },
};
</script>

<style scoped>
/* Adicione estilos conforme necessário */
</style>