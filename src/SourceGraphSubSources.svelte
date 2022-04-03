<script>
	console.log("jes")
	import * as d3 from 'd3'
	import * as math from "mathjs"
	import {getContext} from 'svelte'
	import {tweened} from 'svelte/motion'
	import {cubicOut} from 'svelte/easing'
	export let xScale
	
	
	export let activeSourceGroup
	export let currentYearBased
	export let currentActivePollutant
	
	function handleMouse(e) {
		pointer = d3.pointer(e)
	}
	let group
	
	let store = getContext("store")
	
	$: groupRectWidth = group ? group.getBBox().width : null
	$: progress = pointer[0] / xScale(100)
	
	
	let pointer = [0, 0]
	$: {
		activeSourceGroup
		pointer = [0, 0]
		progressTweened = tweened(0, {
			easing: cubicOut
		})
	}
	let progressTweened = tweened(0, {
		easing: cubicOut
	})
	$: progressTweened.set(progress)
	
	$: positionScale = d3.scaleBand()
	    .domain(d3.range(subSources.length))
		.range([xScale(0), xScale(100)])
	
	const bellScale = d3.interpolate(0, 10)
	
	
	$: positionScaleCust = function(i, domain = [0, subSources.length - 1], range = [xScale(0), xScale(100)]) {
		
		let stack = d3.stack()
			.keys([...Array(subSources.length)])
			.values()
		
	}
	
	function bell(x, spread = null, peak = null) {
	
		x = x 
		let width = spread //wideness
		let max = peak //highest point of bell
		
		let first = 1 / (width * Math.sqrt(Math.PI * 2))
		let third = -(1/2)*Math.pow(((x - max) / width), 2)
		let second = Math.pow(Math.E, third)
		
		return 2.5 * width * (first * second)
	}
	
	
	$: stackValues = subSources.map((_, i) => bell(i, 0.5, (subSources.length - 1) * $progressTweened))
	$: stackKeys = subSources.map((_, i) => i)
	// $: console.log(stackValues, stackKeys)
	
	
	$: stack = d3.stack()
		.keys(stackKeys)
		.value((data, i) => {
			return data[i]
		})

	
	$: stacked = stack([stackValues]).flat()
	$: stackedScaled = stacked.map(([xFrom, xTo]) => [xScaleStacked(xFrom), xScaleStacked(xTo)])
	
	$: subSourcesLabels = subSourcesData.map((source, i) => {
		let progress = bell(i, 0.65, (subSources.length - 1) * $progressTweened)
		return {
			x: math.round(stackedScaled[i][0], 3),
			y: 0,
			// size: bell(i, 1, (subSources.length ) * $progressTweened) * 20,
			size: 20,
			progress: bell(i, 0.65, (subSources.length - 1) * $progressTweened),
			width: stackedScaled[i][1] - stackedScaled[i][0]
		}
		
		// return [
		// 	positionScale(i) + positionScale.bandwidth() / 2 + 
		// 	$progressTweened * xScale(-100) + xScale(50) + width * 100,
		// 	60,
		// 	bell(i) * 100
		// ]
	})
	
	
	
	$: subSources = $store.getSubSourcesOfSourceGroup(activeSourceGroup)
	
	$: subSourcesOfYear = $store.getSourcesOfYear(subSources, currentYearBased).map(([name, pollutants, totalAmount]) => {
		
		totalAmount = currentActivePollutant ? pollutants[currentActivePollutant] : totalAmount
		
		return [name, pollutants, totalAmount]
	})
	
	$: totalSubSourceTons = subSourcesOfYear.reduce((acc, curr) => acc + curr[2], 0)
	$: percentagePerSubSource = subSourcesOfYear.map(([name, _, amount]) => [name, ((amount / totalSubSourceTons) || 0) * 100, amount]
	)
	
	$: percentagePerSubSourceCummulative = percentagePerSubSource.map(([name, percentage], i, array) => {
		let cummulated = array.slice(0, i).reduce((acc, [_, percentage]) => acc + percentage, 0)
		return [name, cummulated]
	})
	
	$: subSourceColorScale = d3.scaleLinear()
		.domain([0, subSourcesOfYear.length / 2, subSourcesOfYear.length - 1])
		.range(["#577f8c", "#56baa0", "#FFE591"])
	
	let subSourcesDataTweened = null
	
	$: {
		activeSourceGroup
		subSourcesDataTweened = tweened(null, {
			easing: cubicOut
		})
	}
	
	
	
	$: subSourcesDataTweened.set(subSourcesData)
		
	$: subSourcesData = percentagePerSubSource.map(([name, percentage, amount], i) => {
		let width = i != 0 ? xScale(percentage) > 1 ? xScale(percentage) - 1 : 0 : xScale(percentage)
		let leftX = i != 0 ? xScale(percentagePerSubSourceCummulative[i][1]) + 1 : xScale(percentagePerSubSourceCummulative[i][1])
		let middleX = leftX + width / 2
		
		return {
			name: name,
			percentage: percentage,
			amount: amount,
			position: {
				width: width,
				x: leftX,
				y: 0,
				middle: middleX
			}
		}
	})
	
	$: xScaleStacked = d3.scaleLinear()
		.domain([stacked.at(0)[0], stacked.at(-1)[1]])
		.range([xScale(0), xScale(100)])
		
	const link = d3.linkVertical()
	
</script>
<style>
	.bar {
		fill: var(--color);
	}
</style>
<g bind:this={group} on:mousemove={handleMouse} transform="translate(0, 80)">
	
	{#each $subSourcesDataTweened as subSource, i}
		<g 
		transform="translate({subSource.position.x}, {subSource.position.y})"
		style="
			--color: {subSourceColorScale(i)}
		">
			<rect class="bar" x="0" y="0" width={subSource.position.width} height="40" />
				
		</g>
	{/each}
	
	<g transform="translate(0, 100)">
		
	{#each subSourcesLabels as source, i}
		
		<!-- {console.log(source.x + source.width / 2)} -->
		{#if $subSourcesDataTweened[i].amount > 0}
			<path d={
				link({
					source: [
						math.round(source.x + source.width / 2, 5),
						0
					],
					target: [
						$subSourcesDataTweened[i].position.x + $subSourcesDataTweened[i].position.width / 2,
						-80
					]
				})
			}
			stroke-width="2"
			stroke={subSourceColorScale(i)}
			fill="none"/>
		{/if}
		<g
		opacity={source.progress}
		transform="translate({source.x + source.width / 2}, 20)"
		style="	--color: {subSourceColorScale(i)}">
			<text fill="black" text-anchor="middle"
			style="font-size: 15px"
			>
				{subSourcesData[i].name}
			</text>
			
			<text 
			fill="black" text-anchor="middle"
			y="30"
			style="font-size: 15px"
			>
				{$store.subSourceDescriptions[subSourcesData[i].name]}
			</text>
			
		</g>
	{/each}
	{#each stacked as item, i}
		<!-- <rect x={stackedScaled[i][0]} width={stackedScaled[i][1] - stackedScaled[i][0]} height="20" y="40" fill="red" opacity="0.2"/> -->
		<!-- <line x1={stackedScaled[i][0]} y1="40" x2={stackedScaled[i][0]} y2="70" stroke="red"/>
		<line x1={stackedScaled[i][1]} y1="40" x2={stackedScaled[i][0]} y2="70" stroke="blue"/> -->
	{/each}
	</g>
</g>

