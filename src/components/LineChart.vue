<template>
	<div ref="chartRef">
		<svg :width="width" :height="height" @mousemove="handleMouseMove" @mouseleave="handleMouseLeave">
			<!-- Graph Lines -->
			<g v-for="line in lines" :key="line.id">
				<!-- Gradient Area -->
				<defs v-if="hasGradient">
					<linearGradient :id="'gradient-' + line.id" x1="0" x2="0" y1="0" y2="1">
						<stop :offset="0" :stop-color="line.color" :stop-opacity="0.2" />
						<stop offset="50%" :stop-color="line.color" stop-opacity="0.1" />
						<stop offset="100%" stop-color="white" stop-opacity="0" />
					</linearGradient>
				</defs>

				<path :d="line.path" :stroke="line.color" :stroke-width="line.width" fill="none" />

				<!-- Area Fill -->
				<g v-if="enableArea">
					<path
						:d="line.path + ` L${width - margin.right},${height - margin.bottom} L${margin.left},${height - margin.bottom} Z`"
						:fill="line.color"
						:opacity="areaOpacity || 0.2"
					/>
				</g>

				<g>
					<g v-if="enablePoint">
						<circle 
							v-for="(point, index) in line.data" 
							:key="index" 
							:cx="xScale(point.x)" 
							:cy="yScale(point.y)"
							:r="pointRadius" 
							:fill="pointColor"
							:stroke="pointBorderColor"
							:stroke-width="pointBorderWidth"
							@mouseover="pointHovered = { ...point, id: line.id }"
							@mouseleave="pointHovered = null" 
						/>
					</g>

					<g v-if="enableCrosshair && pointHovered && pointHovered.id === line.id" >
						<line
							v-for="(crosshairLine, index) in getCrosshairLines(crosshairType, pointHovered, xScale, yScale)"
							:key="index"
							:x1="crosshairLine.x1"
							:y1="crosshairLine.y1"
							:x2="crosshairLine.x2"
							:y2="crosshairLine.y2"
							stroke-dasharray="4" stroke="#ccc"
						/>
					</g>

					<path v-if="hasGradient" :d="getAreaPath(line.data, line.id)" :fill="`url(#gradient-${line.id})`" />
				</g>
			</g>

			<!-- Axis -->
			<g class="x-axis" :transform="`translate(0, ${height - margin.bottom})`"></g>
			<g class="y-axis" :transform="`translate(${margin.left}, 0)`"></g>

			<!-- Axis Labels -->
			<text :x="width / 2" :y="height - margin.bottom / 4" text-anchor="middle">{{ xLabel }}</text>
			<text :x="-(height / 2)" :y="margin.left / 2" text-anchor="middle" transform="rotate(-90)">{{ yLabel }}</text>

			<!-- Grid -->
			<g v-if="grid">
				<g class="x-grid" :transform="`translate(0, ${height - margin.bottom})`"></g>
				<g class="y-grid" :transform="`translate(${margin.left}, 0)`"></g>
			</g>

			<defs>
				<!-- Define the shadow filter -->
				<filter id="shadow" x="-50%" y="-50%" width="200%" height="200%">
					<feDropShadow dx="0.5" dy="0.5" stdDeviation="0.5" flood-color="#333" />
				</filter>
			</defs>

			<!-- Tooltip -->
			<g v-if="pointHovered">
				<rect
					:x="xScale(pointHovered.x) + margin.left - 40"
					:y="yScale(pointHovered.y) + margin.top - 40"
					width="140"
					height="30"
					rx="5"
					ry="5"
					fill="#fff"
					stroke="#333"
					stroke-width="0"
					:filter="'url(#shadow)'"
				/>
				<!-- Color Square -->
				<rect
					:x="xScale(pointHovered.x) + margin.left - 35"
					:y="yScale(pointHovered.y) + margin.top - 31"
					width="12"
					height="12"
					:fill="lineColor(pointHovered.id)"
				/>
				<!-- Coordinates Text -->
				<text
					:x="xScale(pointHovered.x) + margin.left - 15"
					:y="yScale(pointHovered.y) + margin.top - 25.5"
					text-anchor="start"
					dy="0.3em"

				>
					<tspan>x:  </tspan>
					<tspan stroke="#000" stroke-width="1" font-size="small">{{ pointHovered.x.toFixed(2) }}</tspan>
					<tspan>, y:  </tspan>
					<tspan stroke="#000" stroke-width="1" font-size="small">{{ pointHovered.y.toFixed(2) }}</tspan>
				</text>
			</g>
		</svg>
	</div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted } from 'vue';
import * as d3 from 'd3';

interface ChartProps {
	width?: number;
	height?: number;
	margin?: { top: number; right: number; bottom: number; left: number };
	series?: { data: { x: number; y: number }[]; color?: string; width?: number }[];
	xLabel?: string;
	yLabel?: string;
	grid?: boolean;
	hasGradient?: boolean;
	curveType?: 'linear' | 'basis' | 'step' | 'cardinal' | 'catmullRom' | 'monotoneX' | 'monotoneY' | 'natural' | 'stepAfter' | 'stepBefore';
	enableCrosshair?: boolean;
	axisColor?: string;
	gridColor?: string;
	crosshairType?: string;
	enablePoint?: boolean;
	pointRadius?: number;
	pointColor?: string;
	pointBorderColor?: string;
	pointBorderWidth?: number;
	enableArea?: boolean;
	areaOpacity?: number;
}

interface GeneratorFactory {
	createLine(): any;
	createArea(): any;
}

// Destructure props directly in the setup function
const { width, height, margin, series, xLabel, yLabel, grid, hasGradient, curveType, enableCrosshair, axisColor, gridColor, crosshairType, enablePoint, pointRadius, pointColor, pointBorderColor, pointBorderWidth, enableArea, areaOpacity } = defineProps<ChartProps>();

const generatorFactories: Record<'linear' | 'basis' | 'step' | 'cardinal' | 'catmullRom' | 'monotoneX' | 'monotoneY' | 'natural' | 'stepAfter' | 'stepBefore', GeneratorFactory> = {
	linear: { createLine: d3.line(), createArea: d3.area() },
	basis: { createLine: d3.line().curve(d3.curveBasis), createArea: d3.area().curve(d3.curveBasis) },
	step: { createLine: d3.line().curve(d3.curveStep), createArea: d3.area().curve(d3.curveStep) },
	cardinal: { createLine: d3.line().curve(d3.curveCardinal), createArea: d3.area().curve(d3.curveCardinal) },
	catmullRom: { createLine: d3.line().curve(d3.curveCatmullRom), createArea: d3.area().curve(d3.curveCatmullRom) },
	monotoneX: { createLine: d3.line().curve(d3.curveMonotoneX), createArea: d3.area().curve(d3.curveMonotoneX) },
	monotoneY: { createLine: d3.line().curve(d3.curveMonotoneY), createArea: d3.area().curve(d3.curveMonotoneY) },
	natural: { createLine: d3.line().curve(d3.curveNatural), createArea: d3.area().curve(d3.curveNatural) },
	stepAfter: { createLine: d3.line().curve(d3.curveStepAfter), createArea: d3.area().curve(d3.curveStepAfter) },
	stepBefore: { createLine: d3.line().curve(d3.curveStepBefore), createArea: d3.area().curve(d3.curveStepBefore) },
};

const { createLine, createArea } = generatorFactories[curveType];

const chartRef = ref<HTMLDivElement | null>(null);
const pointHovered = ref<{ x: number; y: number; id: string } | null>(null);

const xScale = computed(() => {
	const allData = series.flatMap((serie) => serie.data);
	return d3
		.scaleLinear()
		.domain([d3.min(allData, (d) => d.x) ?? 0, d3.max(allData, (d) => d.x) ?? 1])
		.range([margin.left, width - margin.right])
		.nice();
});

const yScale = computed(() => {
	const allData = series.flatMap((serie) => serie.data);
	return d3
		.scaleLinear()
		.domain([d3.min(allData, (d) => d.y) ?? 0, d3.max(allData, (d) => d.y) ?? 1])
		.range([height - margin.bottom, margin.top]);
});

const lineColor = (lineId) => {
    const line = lines.value.find((l) => l.id === lineId);
    return line ? line.color : 'black';
};

const lines = computed(() =>
	series.map((serie, index) => ({
		id: `line${index + 1}`,
		path: createLine.x(({ x }) => xScale.value(x)).y(({ y }) => yScale.value(y))(serie.data),
		color: serie.color,
		width: serie.width,
		data: serie.data,
	}))
);

const getCrosshairLines = (crosshairType: string, point: { x: number; y: number }, xScale, yScale) => {
    const crosshairMappings = {
        'cross': [
            { x1: xScale(point.x), y1: margin.top, x2: xScale(point.x), y2: height - margin.bottom },
            { x1: margin.left, y1: yScale(point.y), x2: width - margin.right, y2: yScale(point.y) }
        ],
		'bottom-left': [
			{ x1: xScale(point.x), y1: yScale(point.y), x2: xScale(point.x), y2: height - margin.bottom },
			{ x1: xScale(point.x), y1: yScale(point.y), x2: margin.left, y2: yScale(point.y) }
		],
		'bottom-right': [
			{ x1: xScale(point.x), y1: yScale(point.y), x2: xScale(point.x), y2: height - margin.bottom },
			{ x1: xScale(point.x), y1: yScale(point.y), x2: width - margin.right, y2: yScale(point.y) }
		],
		'top-left': [
			{ x1: xScale(point.x), y1: margin.top, x2: xScale(point.x), y2: yScale(point.y) },
			{ x1: xScale(point.x), y1: yScale(point.y), x2: margin.left, y2: yScale(point.y) }
		],
		'top-right': [
			{ x1: xScale(point.x), y1: margin.top, x2: xScale(point.x), y2: yScale(point.y) },
			{ x1: xScale(point.x), y1: yScale(point.y), x2: width - margin.right, y2: yScale(point.y) }
		],
		'right': [
			{ x1: xScale(point.x), y1: yScale(point.y), x2: width - margin.right, y2: yScale(point.y) }
		],
		'left': [
			{ x1: xScale(point.x), y1: yScale(point.y), x2: margin.left, y2: yScale(point.y) }
		],
		'top': [
			{ x1: xScale(point.x), y1: margin.top, x2: xScale(point.x), y2: yScale(point.y) }
		],
		'bottom': [
			{ x1: xScale(point.x), y1: yScale(point.y), x2: xScale(point.x), y2: height - margin.bottom }
		],
		'x': [
			{ x1: xScale(point.x), y1: margin.top, x2: xScale(point.x), y2: height - margin.bottom }
		],
		'y': [
			{ x1: margin.left, y1: yScale(point.y), x2: width - margin.right, y2: yScale(point.y) }
		],
	};

    return crosshairMappings[crosshairType] || [];
};

const renderGrid = () => {
	// Add grid if the grid property is true
	if (grid) {
		const xGrid = d3.select('.x-grid').call(d3.axisBottom(xScale.value).tickSize(-height).tickFormat(''));
		const yGrid = d3.select('.y-grid').call(d3.axisLeft(yScale.value).tickSize(-width).tickFormat(''));

		if (gridColor) {
			xGrid.selectAll('line').style('stroke', gridColor);
			yGrid.selectAll('line').style('stroke', gridColor);
		}
	}
};

const renderAxes = () => {
	const xAxis = d3.axisBottom(xScale.value).ticks(d3.max(series.flatMap((serie) => serie.data), (d) => d.x));
	const yAxis = d3.axisLeft(yScale.value);

	const xAxisSelection = d3.select('.x-axis').call(xAxis);
	const yAxisSelection = d3.select('.y-axis').call(yAxis);

	if (axisColor) {
		xAxisSelection.selectAll('line, path, text').style('stroke', axisColor);
		yAxisSelection.selectAll('line, path, text').style('stroke', axisColor);
	}

};

const handleMouseMove = (event: MouseEvent) => {
	if (chartRef.value) {
		const mouseX = event.pageX - chartRef.value.getBoundingClientRect().left;
		const mouseY = event.pageY - chartRef.value.getBoundingClientRect().top;

		// Find the point closest to the mouse
		let closestPoint = null;
		let closestDistance = Number.MAX_SAFE_INTEGER;

		for (const line of lines.value) {
			for (const point of line.data) {
				const distance = Math.sqrt((xScale.value(point.x) - mouseX) ** 2 + (yScale.value(point.y) - mouseY) ** 2);

				if (distance < closestDistance) {
					closestDistance = distance;
					closestPoint = { ...point, id: line.id };
				}
			}
		}

		pointHovered.value = closestPoint;
	}
};

const handleMouseLeave = () => {
	pointHovered.value = null;
};

const getAreaPath = (data: { x: number; y: number }[], _id: string) =>
	createArea.x(({ x }) => xScale.value(x)).y1(({ y }) => yScale.value(y)).y0(height - margin.bottom)(data);

onMounted(() => {
	renderAxes();
	renderGrid();
});

</script>

<style scoped>

</style>