<script>
	import chroma from "chroma-js"
	import * as d3 from "d3"
	import { tweened } from 'svelte/motion';
	import { cubicOut } from 'svelte/easing';
	
	import { createEventDispatcher } from "svelte";
	
	
	export let pollutant 
	export let year
	let scales = pollutant.scale
	export let activePollutant
	export let trend
	export let xScale
	export let pollutantPosition
	
	
	let element 
	const dispatch = createEventDispatcher();
	
	const clickEvent = () => {
		dispatch("click", pollutant.id);
	};
	
	$: pollutantPosition = {
		x: $pos.x,
		y: $pos.y
	}
	let pos = tweened(calcPosition(year), {
		duration: 600,
		easing: cubicOut
	})
	
	let size = tweened(null, {
		duration: 400,
		easing: cubicOut
	})
	
	let opacity = tweened(1, {
		duration: 400,
		easing: cubicOut
	})
	
	let lineGenerator = d3.line().curve(d3.curveCardinal.tension(0.5))
	let lastPos = null
	$: positionsArray = pollutant.values.map(([mapYear, _], i, array) => {
		if(year < mapYear) {
			return [mapYear, [lastPos.x, lastPos.y]]
		}
		let pos = calcPosition(mapYear)
		lastPos = pos
		return [mapYear, [pos.x, pos.y]]
	})

	// $: console.log(positionsArray)
	
	$: linePathTweened.set(positionsArray)
	let linePathTweened = tweened(null, {
		easing: cubicOut
	})
	$: linePath = lineGenerator($linePathTweened.filter(([filterYear, _]) => filterYear <= year + 1).map(p => p[1]))
	// $: console.log($linePathTweened)
	
	$: trend = pollutant.values.filter(([filterYear, _]) => filterYear <= year).map(([mapYear, value], _, values) => {
			return [mapYear, getPercentageChange(values, mapYear, value)]
		})
	
	
	$: {
		xScale
		pos.set(calcPosition(year))
		size.set(scales.size(currentValue))
		opacity.set(currentValue > 0 ? 1 : 0)
	}
	
	function getPos(x, y, xScale, yScale) {
		x = x || 0.00
		y = y || 0.00
		
		return {
			x: xScale(x),
			y: yScale(y)
		}
	}
	
	function getFirstYearIndex(values) {
		let index = values.findIndex(([_, amount]) => amount > 0)
		
		return index ?? 0
	}
	
	function getFirstYear(values) {
		let year = values.find(([_, amount]) => amount > 0)
		
		return year ? year[0] : null
	}
	
	function getBaseValue(values) {
		let value = values.find(([_, amount]) => amount > 0)
		return value ? value[1] : null
	}
	
	function getPrevValue(values, needleYear) {	
		
		let value = values.find(([year, amount]) => year == needleYear - 1 && amount > 0)
		return value ? value[1] : getBaseValue(values)
	}
	
	function getCurrentValue(needleYear) {
		let value = pollutant.values.find(([year, _]) => year == needleYear)
		let returnVal = value ? value[1] : null
		return returnVal
	}
	
	function getPercentageChange(values, currentYear, currentValue) {
		let prev = getPrevValue(values, currentYear)
		
		let val = ((prev / currentValue) - 1) * -100
		
		if(Math.abs(val) == Infinity || val == NaN) {
			return 0
		}
		return val
	}
	
	function sliceArray(array, start, number) {
		return array.filter((item, i) => i >= start && i < number + start)
	}
	
	function getPercantageString(percentage) {
		return (percentage * 100).toFixed(0)
	}
	
	let color = scales.color(pollutant.name)
	$: bgColor = chroma(color).luminance(0.86)
	$: bgColorTransparent = bgColor.alpha(0.5)
	$: textColor = chroma(color).luminance(0.5).desaturate(0.9)
	$: borderColor = chroma(color).luminance(0.79)
	$: baseSizeStrokeColor = borderColor.luminance(0.60)
	
	$: currentValue = getCurrentValue(year)
	
	
	
	function calcPosition(year = -1) {
		// if the year is 0 or start of the pollutant => change is 0
		let firstYear = getFirstYear(pollutant.values)
		
		if(firstYear >= year) {
			return {
				x: xScale(100),
				y: scales.y(0.00)
			}
		}
		
		
		let currentValue = getCurrentValue(year) ?? 0.0
		let percentageChange = getPercentageChange(pollutant.values, year, currentValue)
		
		let baseValue = getCurrentValue(firstYear)
		
		let y = percentageChange
		let x = (currentValue / baseValue) * 100
		
		return {
			x: xScale(x) || 0.00,
			y: scales.y(y) || 0.00
		}
	}
	
	$: baseValue = getBaseValue(pollutant.values)
	$: prevValue = getPrevValue(pollutant.values, year)
	
	$: percentageChange = getPercentageChange(pollutant.values, year, currentValue)
	$: trend = trend.map(i => {
		
		return i
	})
	
	// $: console.log(trend)
	
	$: percentageChangeToBase = currentValue / baseValue
	
	// $: console.log(percentageChangeToBase)
	
	$: textPos = {
		x: 0,
		y: $size < 30 ? -$size - 10 : 6
	}


	function activate() {
		if(isActive) {
			activePollutant = null
			return
		}
		activePollutant = pollutant.name
		d3.select(element).raise()
	}
	$: isActive = activePollutant == pollutant.name
	$: someActive = activePollutant != null
	
	
	let linePathLabels = {
		"BC": [2003, 2009, 2018, 2014],
		"CO": [1994, 1998, 2003, 2009, 2019],
		"NH3": [1991, 2019],
		"NMVOC": [1991, 1994, 2000, 2003, 2009, 2019],
		"NOx": [1991, 1994, 2000, 2009, 2019],
		"PM10": [1996, 2001, 2005, 2009],
		"PM2.5": [1996, 2001, 2005, 2009, 2016],
		"SO2": [1991, 1995, 1998, 2009, 2019],
		"TSP": [1991, 1995, 2001, 2009]
	}
</script>

<style>
	circle.size {
		fill: var(--bgColorTransparent);
		stroke: var(--borderColor);
		stroke-width: 4px;
		transition: .3s fill ease;
		/* opacity: 0.5; */
/* 		mix-blend-mode: multiply; */
	}
	
	.base-size {
		stroke: var(--baseSizeStrokeColor);
		stroke-width: 1px;
		fill: none;
	}
	
	text {
		font-size: 1rem;
		fill: var(--textColor);
	}
	
	.pollutant--name {
		font-weight: 400;
		letter-spacing: 0.1rem;
	}
	.pollutant--percentage {
		font-size: .8rem;
		font-weight: 300;
		font-variant-numeric: oldstyle-nums tabular-nums;
	}
	.bubble {
		cursor: pointer;
		transition: .3s opacity ease;
	}
	.bubble:hover circle.size  {
		fill: var(--bgColor);
	}
	
	.bubble.hide {
		opacity: 0;
		pointer-events: none;
	}
	
	.line-path {
		fill: none;
		stroke: var(--borderColor);
		stroke-width: 4px;
	}
	.line-label {
		font-size: .9rem;
		color: var(--textColor);
		font-weight: 500;
	}
</style>

<g
	style="
	--borderColor: {borderColor};
	--baseSizeStrokeColor: {baseSizeStrokeColor};
	--textColor: {textColor};
	--bgColor: {bgColor};
	--bgColorTransparent: {bgColorTransparent}"
>

{#if isActive}

<path d={linePath} class="line-path" />
{#each $linePathTweened.filter(([filterYear, _]) => filterYear <= year) as point}
	{#if linePathLabels[pollutant.name].includes(parseInt(point[0]))}
		<text class="line-label" x={point[1][0]} y={point[1][1]} transform="translate(0, 20)" text-anchor="middle">{point[0]}</text>
		
	{/if}
{/each}

{/if}

<g 
class="bubble" 
class:active={isActive}
class:hide={!isActive && someActive}
transform="translate({$pos.x}, {$pos.y})" opacity={$opacity}
bind:this={element}
on:click|stopPropagation={() => activate()}
>

	<circle cx="0" cy="0" r="{$size}" class="size"></circle>
	<circle cx="0" cy="0" r="{scales.size(baseValue)}" stroke-dasharray="5,10" on:click={clickEvent} class="base-size"></circle>
	<text y={textPos.y} x={textPos.x} class="pollutant--name" text-anchor="middle">{pollutant.name}</text>
	<text y="{$size + 20}" x="" class="pollutant--percentage" text-anchor="middle">
		{getPercantageString(percentageChangeToBase)} %
	</text>
</g>
</g>