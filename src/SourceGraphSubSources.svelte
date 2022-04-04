<script>
	console.log("jes")
	import * as d3 from 'd3'
	import * as math from "mathjs"
	import {getContext} from 'svelte'
	import { pan } from 'svelte-gestures';
	import {tweened} from 'svelte/motion'
	import {cubicOut} from 'svelte/easing'
	import chroma from "chroma-js"
	import { unit } from "mathjs"
	export let xScale
	
	
	export let activeSourceGroup
	export let currentYearBased
	export let currentActivePollutant
	
	export const scrolled = function(e) {
		if(mouseIsOver) {
			// e.deltaY
			setScrollDelta(e.deltaY / subSourcesData.length)
			e.preventDefault()
			
		}
	}
	
	function setScrollDelta(deltaY) {
		let newScroll = scrollDelta + deltaY
		scrollDelta = newScroll > groupRectWidth ? groupRectWidth : newScroll < 0 ? 0 : newScroll
	}
	
	let group
	
	// $: d3.select(group).call(d3.zoom().on("zoom", zoomed))
	let store = getContext("store")
	
	$: groupRectWidth = group ? group.getBBox().width : null
	$: scrollDelta = 0
	$: progress = scrollDelta / xScale(100)
	
	let pointer = [0, 0]
	$: {
		activeSourceGroup
		pointer = [0, 0]
		scrollDelta = 0
		progressTweened = tweened(0, {
			easing: cubicOut,
			duration: 1000
		})
	}
	let progressTweened = tweened(0, {
		easing: cubicOut,
		duration: 1000
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
			label: $store.subSourceDescriptions[source.name],
			// size: bell(i, 1, (subSources.length ) * $progressTweened) * 20,
			size: 20,
			progress: math.round(bell(i, 0.65, (subSources.length - 1) * $progressTweened), 10),
			width: math.round(stackedScaled[i][1] - stackedScaled[i][0], 5)
		}
		
		// return [
		// 	positionScale(i) + positionScale.bandwidth() / 2 + 
		// 	$progressTweened * xScale(-100) + xScale(50) + width * 100,
		// 	60,
		// 	bell(i) * 100
		// ]
	})
	
	$: subSourceLabelsSplitted = subSourcesData.map(source => {
		let label = $store.subSourceDescriptions[source.name]
		return convertSourceDescription(label)
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
		.range(["#577f8c", "#56baa0", "#FFC06F"])
		// .range(["#77ACBD", "#7BDFC5", "#FFE88F"])
		
	$: backgroundColorScale = (i) => {
		return chroma(subSourceColorScale(i)).luminance(.7)
	}
	
	$: textColorScale = (i) => {
		return chroma(subSourceColorScale(i)).luminance(.1)
	}
	
	$: textMutedColorScale = (i) => {
		return chroma(subSourceColorScale(i)).luminance(.35)
	}
	
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
	
	
	function convertSourceDescription(description) {
		const regex = /((?<group01a>.*?): (?<group02a>.*);(?<group03a>.*))|((?<group01b>.*?): (?<group02b>.*))|((?<group02c>.*);(?<group03c>.*))|(?<group02d>.*)/g;
		
		let [all, ga, g01a, g02a, g03a, gb, g01b, g02b, gc, g02c, g03c, g02d] = regex.exec(description).map(match => match ?? "")		
		
		let groupsMerged = [
			g01a + g01b,
			g02a + g02b + g02c + g02d,
			g03a + g03c
		]
		
		let splittedGroups = groupsMerged.map(group => group.split("%n"))
		
		let lineHeight = [17, 20, 20]
		let groupSpace = [25, 5, 0]
		let groupClasses = ["source-above-title", "source-title", "source-sub-title"]
		
		let tSpanned = splittedGroups.map((group, i) => {	
			let height = lineHeight[i]
			let lines = group.reduce((acc, lineText, j) => {
				if(j > 0) {
					return acc + `<tspan dy="${height}" x="0">${lineText}</tspan>`
				}
					return acc + `<tspan x="0">${lineText}</tspan>`
			}, "")
		
			return {
				lines: lines,
				y: group.length * height
			}
		})
		
		let groupHeight = 0
		let groups = splittedGroups.map((group, i) => {
			
			
			let height = tSpanned[i].y
			let y = groupHeight + groupSpace[i]
			
			groupHeight = height + y
			
			return {
				class: groupClasses[i],
				y: y,
				tspans: tSpanned[i].lines
			}
		})
		return groups
	}
	function showSourceInfo(i) {
		if(grabbing) {
			return
		}
		scrollDelta = (xScale(100) * i) / (subSourcesData.length - 1)
	}
	
	function mouseOver(state) {
		mouseIsOver = state
	}
	let mouseIsOver = false
	
	let lastPanPos = [null, null]
	let grabbing = false
	function handlePanDown(e) {
		grabbing = true
	}
	
	function handlePanUp(e) {
		e.preventDefault()
		lastPanPos = [null, null]
		grabbing = false
	}
	
	function handlePan(e) {
		let pos = [e.detail.x, e.detail.y]
		
		let delta = [lastPanPos[0] - pos[0], lastPanPos[1] - pos[1]]
		if(lastPanPos[0] != null && Math.abs(delta[0]) < 100) {
			setScrollDelta(delta[0] / subSourcesData.length * 5)
		}
		lastPanPos = pos
	}
	function handleBarClick(e, i) {
		showSourceInfo(i)
		
	}
	
	function formatAmount(mathJsUnit) {
		let number = mathJsUnit.toJSON()
		return $store.niceNumbers(number.value.toFixed(0)) + " " + number.unit
	}
</script>
<style>
	.bar {
		fill: var(--color);
		cursor: pointer;
	}
	.wrapper-group {
		cursor: grab;
	}
	
	.grabbing {
		cursor: grabbing;
	}
	.drag-hint {
		font-feature-settings: "smcp";
		letter-spacing: .04rem;
		fill: var(--colorTextMuted);
		font-size: .9rem;
	}
	.source-identifier {
		font-variant: all-small-caps;
		font-feature-settings: "smcp";
		font-size: .8rem;
		fill: var(--colorTextMuted);
		letter-spacing: .1rem;
	}
	.source-above-title {
		fill: var(--colorTextMuted);
		font-size: .8rem;
	}
	.source-title {
		
		fill: var(--colorText);
		font-size: 1rem;
	}
	.source-sub-title {
		fill: var(--colorTextMuted);
		font-size: .8rem;
		font-style: italic;
	}
	
	.source-amount {
		font-size: 1.2rem;
		font-weight: 700;
		fill: var(--colorText);
		font-variant-numeric: tabular-nums;
	}
	.source-percentage {
		fill: var(--colorTextMuted);
		font-variant-numeric: tabular-nums;
	}
</style>
<g class="wrapper-group" transform="translate(0, 300)" 
on:mouseenter={() => mouseOver(true)} on:mouseleave={() => mouseOver(false)}>
<text x={xScale(50)} y="-10" text-anchor="middle" class="drag-hint">← drag, scroll or click☟ →</text>
	<g bind:this={group}>
		{#each $subSourcesDataTweened as subSource, i}
			<g 
			transform="translate({subSource.position.x}, {subSource.position.y})"
			style="
				--color: {subSourceColorScale(i)}
			">
				<rect class="bar" x="0" y="0" width={subSource.position.width} height="40" 
				on:click={(e) => handleBarClick(e, i)}
				/>
					
			</g>
		{/each}
	</g>
	
	<g transform="translate(0, 40)"
	class:grabbing={grabbing}
	on:panup={handlePanUp}
	on:pandown={handlePanDown}
	on:pan={handlePan}
	use:pan="{{delay:80}}">
		
		<rect width={xScale(100)} height="300" fill="transparent"/>
		{#each subSourcesLabels as source, i}
			
			<!-- {console.log(source.x + source.width / 2)} -->
			{#if $subSourcesDataTweened[i].amount > 0}
				<path d={
					link({
						source: [
							math.round(source.x + source.width / 2, 5),
							100
						],
						target: [
							$subSourcesDataTweened[i].position.x + $subSourcesDataTweened[i].position.width / 2,
							0
						]
					})
				}
				stroke-width="{source.progress * 1 + 1}"
				opacity="{source.progress * 0.8 + .2}"
				stroke={subSourceColorScale(i)}
				fill="none"/>
			{/if}
			<g
			
			transform="translate({(source.x + source.width / 2)}, 100) scale({source.progress}, {source.progress})"
			style="	--color: {subSourceColorScale(i)}; --colorTextMuted: {textMutedColorScale(i)}; --colorText: {textColorScale(i)}">
				
				<rect width={320} height={155} x="-160" rx="5" ry="5" fill={backgroundColorScale(i)}/>
				
				<g transform="translate(-150, 20)">
					
					<text 
					fill="black" text-anchor="left"
					y="0"
					class="source-identifier"
					>
						{subSourcesData[i].name}
					</text>
					{#each subSourceLabelsSplitted[i] as group}
						<text 
						fill="black" text-anchor="left"
						dy={group.y}
						class="{group.class}"
						>
							{@html group.tspans}
						</text>
					{/each}
					
					
					<text 
					y="120"
					class="source-percentage"
					>
						{$subSourcesDataTweened[i].percentage.toFixed(1)} %
					</text>
					<text 
					text-anchor="end"
					class="source-amount"
					y="120"
					x="300"
					>	
					
						{#if $subSourcesDataTweened[i].amount < 1}
							{formatAmount(unit($subSourcesDataTweened[i].amount, 'kt').to('t'))}
						{:else if $subSourcesDataTweened[i].amount >= 1}
							{formatAmount(unit($subSourcesDataTweened[i].amount, 'kt').to('kt'))}
						{/if}
	
					</text>
				</g>
				
				
				
				
			</g>
		{/each}
	</g>
</g>

<!-- ((.*)(: | - )(.+))|(.+) -->