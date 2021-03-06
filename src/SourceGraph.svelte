<script>
	import * as d3 from 'd3'
	import {bboxCollide} from "d3-bboxCollide"
	import forceBoundary from 'd3-force-boundary'
	import chroma from "chroma-js"
	import { onMount, getContext } from 'svelte'
	import { unit, round } from "mathjs"
	
	import { tweened } from 'svelte/motion';
	import { cubicOut } from 'svelte/easing';
	import SourceGraphSubSources from './SourceGraphSubSources.svelte'

	let store = getContext("store")
	export let width
	let height = 700
	
	export let currentYear
	export let currentActivePollutant
	export let pollutantColorScale
	
	$: sourceGroups = $store.sourceGroups
	let firstYear = 0
	$: {
		if(currentActivePollutant) {
			let pollutant = $store.pollutants.find(([name, _]) => currentActivePollutant == name)[1]
			firstYear = $store.getFirstYear(pollutant)
		} else {
			firstYear = 0
		}
	}
	$: pollutantDataForCurrentYear = currentYear >= firstYear
	
	$: currentYearBased = pollutantDataForCurrentYear ? currentYear : firstYear


	$: sourceGroupsCurrentYear = $store.getSourceGroupsOfYear(currentYearBased).map(([name, pollutants, totalAmount]) => {
		
		totalAmount = currentActivePollutant ? pollutants[currentActivePollutant] : totalAmount
		
		return [name, pollutants, totalAmount]
	})
	
	// $: console.log(percentagePerSourceGroup.length, percentagePerSourceGroup)
	
	
	$: totalTons = sourceGroupsCurrentYear.reduce((acc, curr) => acc + curr[2], 0)
	$: percentagePerSourceGroup = sourceGroupsCurrentYear.map(([name, _, amount]) => [name, (amount / totalTons) * 100, amount]).filter(([name, a, b]) => !(["Other"]).includes(name))
	$: percentagePerSourceGroupCumulative = percentagePerSourceGroup.map(([name, percentage], i, array) => {
		let cummulated = array.slice(0, i).reduce((acc, [_, percentage]) => acc + percentage, 0)
		return [name, cummulated]
	})
	
	$: totalTonsBase = $store.getSourceGroupsOfYear().reduce((acc, curr) => acc + curr[2], 0)
	
	$: sourceGroupCount = percentagePerSourceGroup.length ?? 0
	// $: console.log({percentagePerSourceGroup, percentagePerSourceGroupCumulative})
	// let xScale = () => 0
	$: xScale = d3.scaleLinear().domain([0, 100]).range([0, width - 120])
	$: colorScale = d3.scaleLinear()
	.domain([0, sourceGroupCount / 2, sourceGroupCount - 1])
	.range(["#A5CBDD", "#E096C2", "#FFE591"])
	
	
	
	const axisLabels = {
		x: [
			0, 10, 20, 30, 40, 50, 60, 70, 80, 90, 100
		]
	}

	let sourceLabelPositions = []
	let sourceGroupsData = []

	

	let sourceGroupDataTweened = tweened(null, {
		easing: cubicOut,
		duration: 500
	})
	
	
	
	$: sourceGroupDataTweened.set(sourceGroupsData)
	

	$: {
		sourceGroupsData = percentagePerSourceGroup.map(([name, percentage, amount], i) => {
			let barWidth = i != 0 ? xScale(percentage) > 1 ? xScale(percentage) - 1 : 1 : xScale(percentage)
			let leftX = i != 0 ? xScale(percentagePerSourceGroupCumulative[i][1]) + 1 : xScale(percentagePerSourceGroupCumulative[i][1])
			let middleX = leftX + barWidth / 2
			
			
			let bottom = i % 2 > 0 && false
			let forceY = bottom ? 200 : 0
			
			let oldPosition = sourceLabelPositions[i] ? sourceLabelPositions[i] : {
				x: middleX,
				y: forceY
			}
			
			sourceLabelPositions[i] = {
				correction: {
					x: 0,
					y: 0	
				},
				width: labelWidths[i],
				forceX: middleX,
				forceY: forceY,
				radius: labelWidths[i],
				x: oldPosition.x,
				y: oldPosition.y
			}
			
			return {
				name: name,
				percentage: percentage,
				amount: amount,
				index: i,
				position: {
					width: barWidth,
					x: leftX,
					y: 100,
					middle: middleX
				},
				labelLine: {
					x: middleX,
					y: (bottom ? 120 : 100),
					bottom: bottom
				},
				x: middleX,
				y: 0
			}
		})
	}

	let labelElements = []

	$: labelWidths = labelElements.map(el => {
		if(el == null) {
			return 0
		}
		return el.getBBox().width + 10
	})
	
	let labelPositions = []
	let renderedLabels = []
	
	let tickAnimation = null
	
	$: {
		sourceGroupsData
		width
		activeSourceGroup
		
		cancelAnimation()
		labelSimulation.alpha(1)
		startAnimation()
		
	}
	function startAnimation() {
		tickAnimation = window.requestAnimationFrame(tickSimulation)
	}
	
	function cancelAnimation() {
		window.cancelAnimationFrame(tickAnimation)
	}
	
	function tickSimulation() {
		labelSimulation.tick()
		renderedLabels = [ ... sourceLabelPositions]

		if(labelSimulation.alpha() < 0.01) {
			cancelAnimation()
			return
		}
		tickAnimation = window.requestAnimationFrame(tickSimulation)
	}
	
	
	function getBorderForce(x, width = 10, middle = 0, steepness = 10) {

		let result = Math.pow(((x-middle)/(width/2)), steepness)
		return round(result, 3)
	}
	


	function getForceSimulation(xScale) {
		return d3.forceSimulation()
		.force("boundary", forceBoundary(-30, -40, xScale(100), 70).strength(0.08))
		.force("forceX", d3.forceX().x(d => d.forceX).strength(0.1))
		.force("forceY", d3.forceY().y(d => d.forceY).strength(0.008))
		.force("collision", bboxCollide(d => {
			let hw = d.width > 0 ? d.width / 2 : 0
			return [[(hw) * -1, -14], [hw, 14]]
		}).strength(.1))
		.velocityDecay(0.65)
		.nodes(sourceLabelPositions)
	}
	
	let labelSimulation
	$: {
		sourceLabelPositions
		width
		labelSimulation = getForceSimulation(xScale)
	}
	
	const labelRenames = {
		"AgriLivestock": "Agricultural Livestock",
		"AgriOther": "Agriculture",
		"Aviation": "Aviation",
		"Fugitive": "Fugitive Emissions",
		"Industry": "Industry",
		"Offroad": "Non-Road Transport",
		"OtherStationaryComb": "Small Stationary Combustion",
		"PublicPower": "Energy Production",
		"RoadTransport": "Road Traffic",
		"Shipping": "Shipping",
		"Solvents": "Solvents",
		"Waste": "Waste"
	}
	
	function getLabelColor(color) {
		return chroma(color).luminance(.15)
	}
	function getLabelBgColor(color) {
		return chroma(color).luminance(.85)
	}
	
	function getLabelBgNotActiveColor(color) {
		return chroma(color).luminance(.9)
	}
	
	let userActiveSourceGroup = null

	$: activeSourceGroup = userActiveSourceGroup

	$: activeSourceGroupData = sourceGroupsData.find(group => group.name == activeSourceGroup)
	
	
	let activeSourceGroupDataTweened = null
	$: {
		activeSourceGroup
		activeSourceGroupDataTweened = tweened(null, {
			easing: cubicOut,
			duration: 500
		})
	}
	
	$: {
		if(activeSourceGroupData) {
			activeSourceGroupDataTweened.set(activeSourceGroupData)
		}
	}
	

	
	$: anySourceGroupActive = activeSourceGroupData != null
	function activateSourceGroup(name) {
		
		if(userActiveSourceGroup == name) {
			userActiveSourceGroup = null
			return
		}
		userActiveSourceGroup = name
	

	}
	
	$: {
		currentActivePollutant
		// let activeLabelLines = graph.querySelectorAll(".label-line.active")
		if(graph) {
			graph.querySelectorAll(".label-line.active").forEach(el => el.classList.remove("active"))
		}
		
		
		if(activeSourceGroupData) {
			let index = activeSourceGroupData.index
			graph.querySelectorAll(".label-line:not(.label-line"+index+")").forEach(el => el.classList.add("not-active"))
			let labelLine = graph.querySelector(".label-line"+index)
			
			if(labelLine) {
				labelLine.classList.add("active")
			}
		}
		
	}
	
	let sourceGroupElementGroup = []
	
	let graph
	
	function raise(i) {
		let labelLine = graph.querySelector(".label-line"+i)
		if(labelLine) {
			labelLine.classList.add("hovered")
		}
		
		d3.select(sourceGroupElementGroup[i]).raise()
	}
	
	function lower(i) {
		let labelLine = graph.querySelector(".label-line"+i)
		if(labelLine) {
			labelLine.classList.remove("hovered")
		}
	}
	
	function newLink(d) {
		let [source, target] = [d.source, d.target]
		
		let cu = 0.5
		
		let yi = d3.interpolateNumber(d.target.y, d.source.y)
		
		let interpolatedY = {
			source: yi(1 - cu),
			target: yi(cu)
		}

		
		return	"M" + source.x + "," + source.y + " " +
				"C" + source.x + "," + (interpolatedY.source) + " " +
					  target.x + "," + (interpolatedY.target) + " " +
					  target.x + "," + target.y + " " +
				"L" + (target.x + target.width) + "," + target.y + " " +
				"C" + (target.x + target.width) + "," + (interpolatedY.target) + " " +
						(source.x + source.width) + "," + (interpolatedY.source) + " " +
						(source.x + source.width) + "," + source.y + " " +
				"L" + source.x + "," + source.y + " "
					  
	}
	
	function getConnection (pollutantPosition, sourcePosition) {
	
		let newLinkData = {
			source: {
				x: sourcePosition.x,
				y: sourcePosition.y,
				width: sourcePosition.width,
			},
			target: {
				x: pollutantPosition.x,
				y: pollutantPosition.y,
				width: pollutantPosition.width
			}
		}
		
		return newLink(newLinkData)
	}
	
	function getSubSourcesConnectionArc() {
		if(!anySourceGroupActive || !activeSourceGroupData) {
			return ""
		}
		
		let	sourceGroup = $activeSourceGroupDataTweened
		let sourceGroupPositions = {
			x: sourceGroup.position.x,
			y: sourceGroup.position.y + 40,
			width: sourceGroup.position.width
		}
		
		let subSourcePosition = {
			x: xScale(50 - $subGroupTotalWidth * 50),
			y: sourceGroup.position.y + 200,
			width: xScale($subGroupTotalWidth * 100)
		}
		
		return newLink({
			target: subSourcePosition,
			source: sourceGroupPositions
		})
		
	}
	
	let subSourcePath = ""
	
	$: {
		currentActivePollutant
		currentYear
		$activeSourceGroupDataTweened
		$subGroupTotalWidth
		subSourcePath = getSubSourcesConnectionArc()
	}
	
	const link = d3.linkVertical()
	
	function formatAmount(mathJsUnit) {
		let number = mathJsUnit.toJSON()
		return $store.niceNumbers(number.value.toFixed(0)) + " " + number.unit
	}
	
	let subGroupTotalWidth = null
	
	$: {
		activeSourceGroup
		subGroupTotalWidth = tweened(0, {
			duration: 600,
			easing: cubicOut
		})
		
		subGroupTotalWidth.set(1)
	}
	$: {
		if(activeSourceGroup == null) {			
			subGroupTotalWidth.set(0)
		}
	}
	

	let subSourcesGraph
	function handleWheel(e) {
		if(subSourcesGraph) {
			subSourcesGraph.scrolled(e)
		}
	}
</script>

<style>

	.axis-grid {
		stroke-width: 1px;
		stroke: var(--colorBorder);
	}
	
	.bar {
		fill: var(--color);
		transition: .1s opacity ease, .1s fill ease;
	}
	
	.source-group:hover .bar, .source-group.active .bar {
		fill: var(--colorPrimary);
	}
	
	.source-group.not-active:hover .bar {
		opacity: .2;
	}
	
	.label-bg {
		fill: var(--color-label-bg);
		transition: .2s fill ease;
	}
	.label-text {
		fill: var(--color-label-text);
		transition: .2s fill ease;
	}

	.label-line {
		stroke: var(--color);
		stroke-width: 2px;
		transition: .2s stroke ease, .2s stroke-width ease;
		fill: none;
	}
	.source-group {
		cursor: pointer;
	}
	
	.label-line.not-active {
		opacity: .2;
	}
	
	.source-group.not-active .label-bg {
		fill: var(--color-label-bg-not-active);
	}
	
	.source-group.not-active .label-text {
		opacity: .6;
	}
	
	.label-line.hovered, .label-line.active {
		stroke: var(--colorBorder);
		stroke-width: 2px;
		opacity: 1;
	}
	.source-group:hover .label-bg, .source-group.active .label-bg{
		fill: var(--colorBackgroundDark);
		opacity: 1;
	}
	.source-group:hover .label-text, .source-group.active .label-text{
		fill: var(--colorTextEm);
		opacity: 1;
	}
	.sub-source-connection {
		fill: var(--colorBackgroundDark);
	}
	.source-group-amount {
		font-size: 1.2rem;
		font-variant-numeric: tabular-nums;
		font-weight: 700;
		fill: var(--colorTextEm);
	}
	.source-group-amount.small-number {
		font-size: 1rem;
		font-weight: 600;
	}
	.source-group-percentage {
		font-variant-numeric: tabular-nums;
		font-weight: 600;
		fill: var(--colorTextMuted);
	}
	
	.wrapper.deactivated .graph {
		pointer-events: none;
		opacity: .8;
		filter: saturate(0) blur(3px);
	}
	
	.graph {
		transition: .2s opacity ease, .2s filter ease;
	}
	
	.no-data-hint {
		color: var(--colorTextMuted);
		margin: 1rem 0;
		height: 0;
		font-style: italic;
		font-variant-numeric: tabular-nums;
	}
	
	.no-data-hint.hide {
		opacity: 0;
	}
	
	.left-padding {
		margin-left: 100px;
	}
	
	h3 .year, h3 .amount {
		font-variant-numeric: tabular-nums;
		font-weight: 500;
	}
	h3 .year {
		color: var(--colorTextMuted);
	}
	h3 {
		color: var(--colorTextEm);
		margin-top: .5rem;
		margin-bottom: .5rem;
	}
	
	.pollutant-select-wrapper {
		display: flex;
		gap: .5rem;
		margin-left: 100px;
		margin-top: 1rem;
		z-index: 0;
		position: relative;
		flex-wrap: wrap;
		/* overflow: hidden; */
	}
	
	.pollutant-select-item {
		color: var(--colorTextMuted);
		font-size: .9rem;
		padding: .25rem .5rem;
		border-radius: .2rem;
		background: var(--colorBackground);
		cursor: pointer;
		box-shadow: 0 0 0 1px var(--colorBackgroundDark);
	}
	.pollutant-select-item:hover, .pollutant-select-item.active {
		color: var(--colorTextEm);
		background: var(--colorBackgroundDark);
	}
	.pollutant-wrapper {
		display: flex;
		gap: 1rem;
	}
	.pollutant-wrapper .amount {
		text-align: right;
		width: 5rem;
	}
	.pollutant-wrapper .pollutant {
		width: 7.5rem;
	}
	
	.click-hint {
		font-feature-settings: "smcp";
		letter-spacing: .04rem;
		fill: var(--colorTextMuted);
		font-size: .9rem;
	}
	
	@media screen and (max-width: 680px) {
		h3, .pollutant-select-wrapper  {
			margin-left: 1rem !important;
		}
	}
</style>

<div class="wrapper" id="sources"
	class:deactivated={!pollutantDataForCurrentYear}
>
	<h3 class="left-padding">
		<span class="year">
			{currentYear}
		</span><br />
		<div class="pollutant-wrapper">
			<span class="pollutant">
			{#if currentActivePollutant}
				{currentActivePollutant} 
			{:else}
				All Pollutants
			{/if}
			</span>
			<span class="amount">
				{formatAmount(unit(totalTons, 'kt'))}
			</span>
		</div>
	</h3>
	
	<div class="pollutant-select-wrapper">
		<div class="pollutant-select-item"
			class:active={currentActivePollutant == null}
			on:click={() => {
				currentActivePollutant = null
			}}
		>All</div>
		{#each $store.pollutants as pollutant}
			<div class="pollutant-select-item"
				class:active={currentActivePollutant == pollutant[0]}
				on:click={() => {
					currentActivePollutant = pollutant[0]
				}}>
				{pollutant[0]}
			</div>
		{/each}
	</div>
	
	<div class="no-data-hint left-padding" class:hide={pollutantDataForCurrentYear}>
		No data is available for the selected year {currentYear}. Go to <b>{firstYear}</b>
	</div>
	<svg bind:this={graph} use:startAnimation on:wheel={handleWheel} class="graph" viewBox="0 0 {width} {height}" preserveAspectRatio="xMidYMid meet">
		<g transform="translate(100, 80)">		
			{#if activeSourceGroupData}
					{#if activeSourceGroupData.amount > 0}
						<path class="sub-source-connection" d={subSourcePath} fill="black"/>			
					{/if}
					<g
						class="active-source-wrapper"
						style="opacity: {$subGroupTotalWidth}"
						transform="translate(
						{xScale(50) + (activeSourceGroupData.amount > 0 ?($activeSourceGroupDataTweened.position.middle - xScale(50)) / 1.08 : 0)},
						{$activeSourceGroupDataTweened.position.y + 100}
						)"
					>
						<text
							class="source-group-percentage"
							text-anchor="middle"
							y="-20"
						>	
						<!-- {unit(activeSourceGroupData.percentage, "percentage")} -->
							{$activeSourceGroupDataTweened.percentage.toFixed(1)} %
						</text>
						<text 
							class="source-group-amount"
							class:small-number={activeSourceGroupData.amount < 1}
							text-anchor="middle"
						>
							{#if $activeSourceGroupDataTweened.amount < 1}
								{formatAmount(unit($activeSourceGroupDataTweened.amount, 'kt').to('t'))}
							{:else if $activeSourceGroupDataTweened.amount >= 1}
								{formatAmount(unit($activeSourceGroupDataTweened.amount, 'kt').to('kt'))}
							{/if}
							
						</text>
					</g>

			{/if}
			{#each $sourceGroupDataTweened as sourceGroup, i}
				{#if sourceGroup.amount > 0}
					<path 
					style="
					--color: {colorScale(i)};
					"
					class="label-line label-line{sourceGroup.name} label-line{i}"
					d={
						link({
							source: [
								renderedLabels[i] ? renderedLabels[i].x : 0,
								renderedLabels[i] ? sourceGroup.labelLine.bottom ? renderedLabels[i].y : renderedLabels[i].y + 24 : 0
							],
							target: [
								sourceGroup.labelLine ? sourceGroup.labelLine.x : 0,
								sourceGroup.labelLine ? sourceGroup.labelLine.y : 0
							]
						})
					}/>
				{/if}
			{/each}
			{#each $sourceGroupDataTweened as sourceGroup, i}
					<g class="source-group"
					class:active={activeSourceGroup == sourceGroup.name}
					class:not-active={anySourceGroupActive && activeSourceGroup != sourceGroup.name}
					bind:this={sourceGroupElementGroup[i]}
					on:mouseenter={() => {raise(i)}}
					on:mouseleave={() => {lower(i)}}
					on:click|stopPropagation={() => {activateSourceGroup(sourceGroup.name)}}
					style="
					--color: {colorScale(i)};
					--color-label-text: {getLabelColor(colorScale(i))};
					--color-label-bg: {getLabelBgColor(colorScale(i))};
					--color-label-bg-not-active: {getLabelBgNotActiveColor(colorScale(i))};
					">
						{#if sourceGroup.amount > 0}
							<g 
							transform="translate({sourceGroup.position.x}, {sourceGroup.position.y})">
								<rect class="bar" x="0" y="0" width={sourceGroup.position.width} height="40"/>
							</g>
						{/if}
					
						
						<g 
							transform="translate({renderedLabels[i] ? renderedLabels[i].x : 0}, {renderedLabels[i] ? renderedLabels[i].y : 0})"
						>
							<rect class="label-bg" x="{(labelWidths[i] ? labelWidths[i] + 1 : 0) / -2}" y="0" width={labelWidths[i]} height="24" rx="5" ry="5"/>
							<text class="label-text" x="" y="5" text-anchor="middle" bind:this={labelElements[i]} dominant-baseline="hanging">		{labelRenames[sourceGroup.name]}
							</text>
						</g>
					</g>
			{/each}
			
			{#if !anySourceGroupActive}
				<text class="click-hint" x={xScale(50)} y={160} text-anchor="middle">select ⤴</text>
			{/if}
			
			{#if anySourceGroupActive} 
				<g transform="translate({xScale(50 - ($subGroupTotalWidth * 50))}, 0) scale({$subGroupTotalWidth}, 1)">
				
					<SourceGraphSubSources 
						bind:this={subSourcesGraph}
						xScale={xScale} 
						activeSourceGroup={activeSourceGroup}
						currentYearBased={currentYearBased}	
						currentActivePollutant={currentActivePollutant}
					/>
				</g>
			{/if}
		</g>
			
	</svg>

</div>