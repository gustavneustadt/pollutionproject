<script>
	export let pollutant 
	export let year
	export let scales
	import chroma from "chroma-js"
	
	import { tweened } from 'svelte/motion';
	import { cubicOut } from 'svelte/easing';
	
	import { createEventDispatcher } from "svelte";
	
	
	export let trend
	
	export let pollutantPosition
	
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
	
	let size = tweened(scales.size(0), {
		duration: 400,
		easing: cubicOut
	})
	
	let opacity = tweened(0.0, {
		duration: 400,
		easing: cubicOut
	})
	
	
	$: {
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
	
	function getFirstYearIndex() {
		return pollutant.values.findIndex(x => x > 0)
	}
	
	function getBaseValue(values) {
		return values.find(x => x > 0)
	}
	
	function getPrevValue(values, year) {
		let baseValue = getBaseValue(values)
		return values[year-1]? values[year-1] : baseValue
	}
	
	function getCurrentValue(year) {
		return pollutant.values[year]
	}
	
	function getPercentageChange(values, currentYear, currentValue) {
		let prev = getPrevValue(values, currentYear)
		return ((prev / currentValue) - 1) * -100
	}
	
	function sliceArray(array, start, number) {
		return array.filter((item, i) => i >= start && i < number + start)
	}
	
	function getPercantageString(percentage) {
		return (percentage * 100).toFixed(0)
	}
	
	$: bgColor = chroma(pollutant.color).luminance(0.86)
	$: textColor = chroma(pollutant.color).luminance(0.5).desaturate(0.9)
	$: borderColor = chroma(pollutant.color).luminance(0.79)
	$: baseSizeStrokeColor = borderColor.luminance(0.60)
	
	$: currentValue = getCurrentValue(year)
	
	function calcPosition(year = -1) {
		// if the year is 0 or start of the pollutant => change is 0
		
		let firstYear = getFirstYearIndex()
		
		if(firstYear >= year) {
			return {
				x: scales.x(100),
				y: scales.y(0.00)
			}
		}
		
		
		let currentValue = getCurrentValue(year) ?? 0.0
		let percentageChange = getPercentageChange(pollutant.values, year, currentValue)
		
		let baseValue = getCurrentValue(firstYear)
		
		let y = percentageChange
		let x = (currentValue / baseValue) * 100
		
		return {
			x: scales.x(x) || 0.00,
			y: scales.y(y) || 0.00
		}
	}
	
	$: baseValue = getBaseValue(pollutant.values)
	$: prevValue = getPrevValue(pollutant.values, year)
	
	$: percentageChange = getPercentageChange(pollutant.values, year, currentValue)
	$: trend = getPercentageChange(pollutant.values, year, currentValue)
	
	$: percentageChangeToBase = currentValue / baseValue
	
	// $: console.log(percentageChangeToBase)
	
	$: textPos = {
		x: 0,
		y: $size < 30 ? -$size - 10 : 8
	}
	
	$: valuesScaled = sliceArray(pollutant.values, 0, year).map((value, i) => {
		let pos = getPos(
			value,
		   	getPercentageChange(pollutant.values, i, value),
			scales.x,
			scales.y
		   )
		   
		return pos
	})
	
	// console.log(valuesScaled)
</script>

<style>
	circle.size {
		fill: var(--bgColor);
		stroke: var(--borderColor);
		stroke-width: 4px;
		opacity: 0.5;
/* 		mix-blend-mode: multiply; */
	}
	
	.base-size {
		stroke: var(--baseSizeStrokeColor);
		stroke-width: 1px;
		fill: none;
	}
	
	text {
		font-family: "Fira Sans";
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
		font-variant-numeric: oldstyle-nums;
	}
</style>


<g transform="translate({$pos.x}, {$pos.y})" opacity={$opacity} style="
	--borderColor: {borderColor};
	--baseSizeStrokeColor: {baseSizeStrokeColor};
	--textColor: {textColor};
	--bgColor: {bgColor};
">
	<text y={textPos.y} x={textPos.x} class="pollutant--name" text-anchor="middle">{pollutant.name}</text>
	<circle cx="0" cy="0" r="{$size}" class="size"></circle>
	<circle cx="0" cy="0" r="{scales.size(baseValue)}" stroke-dasharray="5,10" on:click={clickEvent} class="base-size"></circle>
	<text y="{$size + 20}" x="" class="pollutant--percentage" text-anchor="middle">
		{getPercantageString(percentageChangeToBase)} %
	</text>
</g>
