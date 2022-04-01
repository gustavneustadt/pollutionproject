<script>
	import * as d3 from 'd3'
	import chroma from "chroma-js"
	import { onMount, getContext } from 'svelte'
	
	import SourceGraph from './SourceGraph.svelte'
	import PollutantBubble from './PollutantBubble.svelte'
	import PollutantSourceGroup from './PollutantSourceGroup.svelte'
	import PollutantSources from './PollutantSources.svelte'
	import PollutantInfoPanel from './PollutantInfoPanel.svelte'

	import { tweened } from 'svelte/motion';
	import { cubicOut } from 'svelte/easing';
	
	export let currentYearIndex
	
	const store = getContext("store")
	

	
	let pollutantsData = $store.pollutants
	
	let sourcesData
	
	let sourceGroupsData = $store.sourceGroups
	let subSourcesData = $store.subSources

	
	const width = 800
	const height = 600
	
	let initialized = false
	
	$: currentYear = $store.getYear(currentYearIndex)
	
	const pollutants = $store.pollutants
	const pollutantColors = 
	["#509CF7", "#FFCC00", 	"#5856D6", 	"#34C759", 	"#FF2D55",	"#AF52DE", 	"#FF9500",	"#37D2AC",	"#83DCFF"]
	const years = $store.years
	
	let activePollutant = null
	
	let trendLineValues = pollutants.map(_ => {
		return []
	})
	
	let pollutantPositions = pollutants.map(_ => {
		return {
			x: 0,
			y: 0
		}
	})
	
	let meanTrend = tweened(0, {
		duration: 600,
		easing: cubicOut
	})
	
	$: totalPollutionAmountBaseYear = $store.getSourceGroupsOfYear().reduce((acc, curr) => acc + curr[2], 0)
	// $: totalPollutionAmount = $store.getSourceGroupsOfYear(currentYear).reduce((acc, curr) => acc + curr[2], 0)
	$: maxYearPollutionAmount = $store.getSourceGroupsOfYear($store.getMaxYear()).reduce((acc, curr) => acc + curr[2], 0)
	
	let lineGenerator = d3.line().curve(d3.curveCardinal.tension(0.5))
	let linePath = ""
	let trendCoords = []
	$: {
		let trendY = trendLineValues[0].map((_, i) => {
			return trendLineValues.reduce((acc, curr) => {
				
				if(curr[i]) {
					return acc + (isNaN(curr[i][1]) ? 0 : curr[i][1])
				} else {
					return null
				}
			}, null) / trendLineValues.length
		})

		
		let trendX = trendLineValues[0].map(d => {
			if(d) {
				let year = d[0]
				return $store.getSourceGroupsOfYear(year).reduce((acc, curr) => acc + curr[2], 0) / totalPollutionAmountBaseYear * 100
			}
		})
		trendCoords = trendY.map((value, i) => {
			
			return [xScale(trendX[i]), yScale(value)]
		})
		
		let trendsCoordsTweening = $store.years.map((_, i) => {
			return trendCoords[i] ? trendCoords[i] : trendCoords.length == 0 ? [0, 0] : trendCoords.at(-1)
		})
		
		tweenedTrendValues.set(trendsCoordsTweening)
	}
	$: trendLabelsCoords = $tweenedTrendValues
		.map((value, i) => [$store.getYear(i), value])
		.slice(0, currentYearIndex + 1)
		.filter(([year, _]) => [1991, 1994, 2000, 2009, 2019].includes(year))
	
	let tweenedTrendValues = tweened(null, {
		easing: cubicOut
	})
	$: linePath = lineGenerator($tweenedTrendValues.slice(0, currentYearIndex + 2))
	
	// $: console.log($tweenedTrendValues)
	
	$: getMaxValue = () => {
		if(pollutants.length < 1) {
			return 1
		}
		return Math.max(...pollutants.map(([_, years]) => years.map(([year, value]) => value)).flat())
	}
	
	$: getMinValue = () => {
		if(pollutantsData.length < 1) {
			return 0
		}
		return Math.min(...pollutants.map(([_, years]) => years.map(([year, value]) => value)).flat().filter(x => x > 0))
	}
	
	$: pollutantValues = pollutants.map((_, i) => {
		if(pollutantsData.length > 0) {
			return years.map((_, y) => {
				return Number(pollutantsData[i][y])
			})				
		}
	})
	
	$: xScale = d3.scaleLinear().domain([100, 0]).range([0, width - 120])
	$: yScale = d3.scaleLinear().domain([-63, 10]).range([500, 0])
	$: sizeScale = d3.scaleSqrt().domain([getMinValue(), getMaxValue()]).range([5, 80])
	$: colorScale = d3.scaleOrdinal()
		.domain($store.pollutants.map(d => d[0]))
		.range(pollutantColors)
	
	const axisLabels = {
		x: [0, 10, 20, 30, 40, 50, 60, 70, 80, 90, 100],
		y: [10, 0, -10, -20, -30, -40, -50]
	}
	
	$: scales = {
		x: xScale,
		y: yScale,
		size: sizeScale
	}
	
	let maxLabel = {
		xMin: Math.min(...axisLabels.x),
		xMax: Math.max(...axisLabels.x),
		yMin: Math.min(...axisLabels.y),
		yMax: Math.max(...axisLabels.y),
	}
	
	function getArraySum(array = []) {
		return array.reduce((a, b) => parseFloat(a) + parseFloat(b), 0.0)
	}
	
	const niceNumbers = $store.niceNumbers
	
	const radius = 200
	
	const ribbonGenerator = d3.ribbon().radius(radius);
	
	$: pollutantsComponentData = pollutants.map(([pollutantName, pollutantData], i) => {
	
		return {
			scale: {
				x: xScale,
				y: yScale,
				size: sizeScale,
				color: colorScale
			},
			year: currentYear,
			values: pollutantData,
			name: pollutantName
		}
	})
</script>

<style>
	.axis-grid--horizontal, .axis-grid--vertical {
		stroke: var(--colorPrimaryGreyish);
	}
	
	.axis-label {
		font-variant-numeric: tabular-nums;
		font-weight: 400;
		fill: var(--colorTextMuted);
		font-size: .8rem;
	}
	
	.axis-title {
		font-size: 1rem;
		font-weight: 400;
		font-feature-settings: "smcp";
		letter-spacing: .04rem;
		fill: var(--colorTextMuted);
	}
	
	.graph text {
	/* 	font-family: "Merriweather"; */
		font-weight: 400;
		font-size: 0.8rem;
	}
	
	.axis-grid--horizontal.zero {
		stroke: var(--colorTextMuted);
		
	}
	
	.axis-grid--horizontal.trend {
		stroke: red;
		stroke-width: 3px;
		opacity: .1;
	}
	
	.source__name {
		font-weight: 500;
		fill: white;
	}
	
	.source__block {
		fill: #d9d9d9;
	}
	.connection {
		fill: black;
		stroke: none;
	/* 	fill: none; */
	/* 	stroke: #f4f4f4; */
	/* 	stroke-width: 2; */
		opacity: 0.03;
	}
	
	.flex {
		display: flex;
	}
	
	svg {
		width: 100%;
		top: 0;
		left: 0;
		
	}
	.graph-wrapper {
		width: 70%;
		max-width: 65rem;
		height: 100%;
		position: relative;
	}
	.trend-line {
		stroke: var(--colorTextMutedLight);
		fill: none;
		stroke-width: 3px;
	}
	.trend-label {
		font-weight: 700;
		fill: var(--colorTextMuted);
		font-size: .8rem;
	}
	
	.trend-label-title {
		font-weight: 400;
		fill: var(--colorTextMuted);
		font-size: .8rem;
	}

	.wrapper {
		display: flex;
		align-content: stretch;
		justify-content: space-between;
	}
	
	.legend {
		width: 30%;
		margin: 1rem 0 0 10%;
	}
	h2 {
		margin: 0rem 0 0rem 12.5%;
		color: var(--colorTextMuted);
	}
	
	h2:first-of-type {
		margin: 4rem 0 0rem 12.5%;
	}
	.hint {
		font-size: .9rem;
		padding-left: 12.5%;
		color: var(--colorTextMuted);
	}
</style>


<div class="wrapper">
	<div class="graph-wrapper">
		<h2>Annual Air Emissions of Germany</h2>
		<span class="hint">
			{#if !activePollutant}
				Click on a pollutant bubble to get more information
			{:else}
				Click anywhere on the graph to see all the bubbles again
			{/if}
		</span>
		
			<svg class="graph" viewBox="0 0 {width} {height}" 
			on:click={() => {activePollutant = null}}
			preserveAspectRatio="xMidYMid meet">
				<g transform="translate(100, 40)">
					
					<path class="trend-line" d="{linePath}" />
					<!-- {console.log(trendLabelsCoords)} -->
					
					<text class="axis-title" transform="rotate(-90) translate({yScale(40)}, -70)" text-anchor="middle">change to previous year</text>
					<text class="axis-title" transform="translate({xScale(50)}, {yScale(maxLabel.yMin) + 60})" text-anchor="middle">compared to base year</text>
	
				{#each axisLabels.x as label}
					<line class="axis-grid--vertical" x1="{xScale(label)}" y1="{yScale(maxLabel.yMax)}" x2="{xScale(label)}" y2="{yScale(maxLabel.yMin)}"></line>
					<text class="axis-label axis-label--vertical" x="{xScale(label)}" y="{yScale(maxLabel.yMin) + 20}" text-anchor="middle">{niceNumbers(label)} %</text>
				{/each}
				{#each axisLabels.y as label}
					<line class="axis-grid--horizontal {label == 0 ? 'zero' : ''}" x1="{xScale(0)}" y1="{yScale(label)}" x2="{xScale(maxLabel.xMax)}" y2="{yScale(label)}"></line>
					<text class="axis-label axis-label--horizontal" x="-10" y="{yScale(label)}" text-anchor="end" alignment-baseline="central">{niceNumbers(label)} %</text>
				{/each}
				
				
				{#each trendLabelsCoords as year, i}
				
					<g transform="translate({year[1][0]},{year[1][1]})" >
	
						{#if i == 0}
							<text class="trend-label-title" transform="translate(0, 40)" text-anchor="middle">Average trend</text>
						{/if}
						
						<text class="trend-label" transform="translate(0, 20)" text-anchor="middle">{year[0]}</text>
					</g>
				{/each}
				{#each pollutantsComponentData as pollutant, i}
						<PollutantBubble
							bind:activePollutant="{activePollutant}"
							year={currentYear} 
							bind:pollutantPosition={pollutantPositions[i]}
							bind:trend={trendLineValues[i]}
							pollutant={pollutant}
						></PollutantBubble>
				{/each}
			</g>
		</svg>
		<h2>
			Sources Composition of annual emissions
		</h2>
		<SourceGraph bind:currentYear={currentYear} bind:currentActivePollutant={activePollutant} pollutantColorScale={colorScale}/>
	</div>
	
	<PollutantInfoPanel currentYear={currentYear} currentActivePollutant={activePollutant} colorScale={colorScale}/>
</div>
