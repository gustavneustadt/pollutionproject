<script>
	export let pollutant 
	export let year
	export let scales
	import chroma from "chroma-js"
	
	import { tweened } from 'svelte/motion';
	import { cubicOut } from 'svelte/easing';
	
	
	
	
	let pos = tweened({
		x: 0.00,
		y: 0.00
		}, {
		duration: 600,
		easing: cubicOut
	})
	
	let size = tweened(0, {
		duration: 400,
		easing: cubicOut
	})
	

	$: {
		pos.set(getPos(currentValue, percentageChange, scales.x, scales.y))
		if(pollutant.name == "BC") {
			// console.log($pos)
		}
	}
	$: {
		size.set(scales.size(percentageChangeToBase))
	}
	
	function getPos(x, y, xScale, yScale) {
		return {
			x: xScale(currentValue) || 0.00,
			y: yScale(percentageChange) || 0.00
		}
	}
	
	function getBaseValue(values) {
		return values.find(x => x > 0)
	}
	
	function getPrevValue(values, year) {
		let baseValue = getBaseValue(values)
		return values[year-1]? values[year-1] : baseValue
	}
	
	function getPercentageChange(values, currentYear, currentValue) {
		let prev = getPrevValue(values, currentYear)
		return (prev / currentValue) * -1 - 1
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
	
	$: currentValue = pollutant.values[year]
	
	
	$: baseValue = getBaseValue(pollutant.values)
	$: prevValue = getPrevValue(pollutant.values, year)
	
	$: percentageChange = getPercentageChange(pollutant.values, year, currentValue)
	$: percentageChangeToBase = currentValue / baseValue
	
	// $: console.log(percentageChangeToBase)
	
	$: textPos = {
		x: 0,
		y: size < 35 ? -size - 10 : 8
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
		mix-blend-mode: multiply;
	}
	
	.base-size {
		stroke: var(--baseSizeStrokeColor);
		stroke-width: 1px;
		fill: none;
	}
	
	text {
		font-family: "Merriweather Sans";
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





<g transform="translate({$pos.x}, {$pos.y})" style="
	--borderColor: {borderColor};
	--baseSizeStrokeColor: {baseSizeStrokeColor};
	--textColor: {textColor};
	--bgColor: {bgColor};
	display: {currentValue > 0 ? "block" : "none"};
">
	<circle cx="0" cy="0" r="{$size}" class="size"></circle>
	<circle cx="0" cy="0" r="{scales.size(1)}" stroke-dasharray="5,10" class="base-size"></circle>
	<text y={textPos.y} x={textPos.x} class="pollutant--name" text-anchor="middle">{pollutant.name}</text>
	<text y="{$size + 20}" x="" class="pollutant--percentage" text-anchor="middle">
		{getPercantageString(percentageChangeToBase)} %
	</text>
</g>
