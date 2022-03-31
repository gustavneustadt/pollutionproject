<script>
	import * as d3 from 'd3'
	import {bboxCollide} from "d3-bboxCollide"
	import forceBoundary from 'd3-force-boundary'
	import chroma from "chroma-js"
	import { onMount, getContext } from 'svelte'
	
	import { tweened } from 'svelte/motion';
	import { cubicOut } from 'svelte/easing';

	let store = getContext("store")
	let width = 800
	let height = 600
	
	export let currentYear
	
	
	$: sourceGroups = $store.sourceGroups
	$: sourceGroupsCurrentYear = $store.getSourceGroupsOfYear(currentYear)
	
	$: totalTons = sourceGroupsCurrentYear.reduce((acc, curr) => acc + curr[2], 0)
	$: percentagePerSourceGroup = sourceGroupsCurrentYear.map(([name, _, amount]) => [name, (amount / totalTons) * 100]).filter(([_, val]) => val > 0)
	$: percentagePerSourceGroupCumulative = percentagePerSourceGroup.map(([name, percentage], i, array) => {
		let cummulated = array.slice(0, i).reduce((acc, [_, percentage]) => acc + percentage, 0)
		return [name, cummulated]
	})
	
	$: totalTonsBase = $store.getSourceGroupsOfYear().reduce((acc, curr) => acc + curr[2], 0)
	
	
	// $: console.log({percentagePerSourceGroup, percentagePerSourceGroupCumulative})
	
	$: xScale = d3.scaleLinear().domain([0, 100]).range([0, width - 120])
	let colorScale = d3.scaleLinear()
	.domain([0, 5, 13])
	.range(["#3E2D8B", "#FFEE7D", "#830F0F"])
	
	
	
	const axisLabels = {
		x: [
			0, 10, 20, 30, 40, 50, 60, 70, 80, 90, 100
		]
	}

	let sourceLabelPositions = []
	let sourceGroupsData = []
	$: subSources = $store.getSubSourcesOfSourceGroup(activeSourceGroup)
	$: subSourcesOfYear = $store.getSourcesOfYear(subSources, currentYear).filter(([a,b,val]) => val > 0)
	$: totalSubSourceTons = subSourcesOfYear.reduce((acc, curr) => acc + curr[2], 0)
	$: percentagePerSubSource = subSourcesOfYear.map(([name, _, amount]) => [name, amount / totalSubSourceTons * 100])
	$: percentagePerSubSourceCummulative = percentagePerSubSource.map(([name, percentage], i, array) => {
		let cummulated = array.slice(0, i).reduce((acc, [_, percentage]) => acc + percentage, 0)
		return [name, cummulated]
	})
	
	$: subSourceColorScale = d3.scaleLinear()
	.domain([0, subSourcesOfYear.length])
	.range(["#e05353", "#f3e472"])
	
	$: console.log({subSources, percentagePerSubSource, totalSubSourceTons, percentagePerSubSourceCummulative})
	
	
	$: subSourcesData = percentagePerSubSource.map(([name, percentage], i) => {
		let width = xScale(percentage)
		let leftX = xScale(percentagePerSubSourceCummulative[i][1])
		let middleX = leftX + width / 2
		
		return {
			name: name,
			percentage: percentage,
			position: {
				width: width,
				x: leftX,
				y: 100,
				middle: middleX
			}
		}
	})
	
	$: {
		anySourceGroupActive
		sourceGroupsData = percentagePerSourceGroup.map(([name, percentage], i) => {
			let width = xScale(percentage)
			let leftX = xScale(percentagePerSourceGroupCumulative[i][1])
			let middleX = leftX + width / 2
			
			
			let bottom = i % 2 > 0 && false
			let forceY = bottom ? 200 : 0
			
			let oldPosition = sourceLabelPositions[i] ? sourceLabelPositions[i] : {
				x: middleX,
				y: 50
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
				position: {
					width: width,
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
	
	$: labelWidths = labelElements.map(el => el.clientWidth + 10)
	
	let labelPositions = []
	let renderedLabels = []
	
	let tickAnimation = null
	
	$: {
		percentagePerSourceGroup
		cancelAnimation()
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

		if(labelSimulation.alpha() < 0.2) {
			cancelAnimation()
			return
		}
		tickAnimation = window.requestAnimationFrame(tickSimulation)
	}
	
	
	
	
	$: labelSimulation = d3.forceSimulation()
		.force("boundary", forceBoundary(-30, -40, xScale(100), 80).strength(0.03))
		.force("forceX", d3.forceX().x(d => d.forceX).strength(0.2))
		.force("forceY", d3.forceY().y(d => d.forceY).strength(0.05))
		// .force("center", d3.forceCenter().x(50).y(100))
		// .force("charge", d3.forceManyBody().strength(-100))
		.force("collision", bboxCollide(d => {
			let hw = d.width > 0 ? d.width / 2 : 0
			return [[(hw + 3) * -1, -14], [hw + 3, 14]]
		}).strength(.5))
		.nodes(sourceLabelPositions)
	
	
	const labelRenames = {
		"AgriLivestock": "Agricultural Livestock",
		"AgriOther": "Agriculture",
		"Aviation": "Aviation",
		"Fugitive": "Fugitive Emissions",
		"Industry": "Industry",
		"Offroad": "Non-Road Traffic",
		"OtherStationaryComb": "Small Combustion Sectors",
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


	let activeSourceGroup = null
	$: anySourceGroupActive = activeSourceGroup != null
	function activateSourceGroup(name) {
		if(activeSourceGroup == name) {
			activeSourceGroup = null
			return
		}
		activeSourceGroup = name
	}
	
	let sourceGroupElementGroup = []
	
	function raise(i) {
		d3.select(sourceGroupElementGroup[i]).raise()
	}
	
	
</script>

<style>

	.axis-grid {
		stroke-width: 1px;
		stroke: var(--colorBorder);
	}
	
	.bar {
		fill: var(--color);
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
		stroke-width: 1px;
		transition: .2s stroke ease, .2s stroke-width ease;
	}
	.source-group {
		cursor: pointer;
	}
	
	.source-group.not-active .label-line, .source-group.not-active .label-bg {
		opacity: .2;
	}
	
	.source-group.not-active .label-text {
		opacity: .7;
	}
	
	.source-group:hover .label-line, .source-group.active .label-line {
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
</style>

<div class="wrapper">
	<svg class="graph" viewBox="0 0 {width} {height}" preserveAspectRatio="xMidYMid meet" on:click={() => activateSourceGroup(null)}>
		<g transform="translate(100, 40)">
			{#each sourceGroupsData as sourceGroup, i}
				<g class="source-group"
				class:active={activeSourceGroup == sourceGroup.name}
				class:not-active={anySourceGroupActive && activeSourceGroup != sourceGroup.name}
				bind:this={sourceGroupElementGroup[i]}
				on:mouseenter={() => {raise(i)}}
				on:click|stopPropagation={() => {activateSourceGroup(sourceGroup.name)}}
				style="
				--color: {colorScale(i)};
				--color-label-text: {getLabelColor(colorScale(i))};
				--color-label-bg: {getLabelBgColor(colorScale(i))};
				">
					<g 
					transform="translate({sourceGroup.position.x}, {sourceGroup.position.y})">
						<rect class="bar" x="0" y="0" width={sourceGroup.position.width} height="20" />
					</g>
					<line 
						class="label-line" 
						x1="{renderedLabels[i] ? renderedLabels[i].x : 0}" 
						y1="{renderedLabels[i] ? sourceGroup.labelLine.bottom ? renderedLabels[i].y : renderedLabels[i].y + 24 : 0}" 
						x2="{sourceGroup.labelLine ? sourceGroup.labelLine.x : 0}" 
						y2="{sourceGroup.labelLine ? sourceGroup.labelLine.y : 0}"
					/>
					
					
					<g 
						transform="translate({renderedLabels[i] ? renderedLabels[i].x : 0}, {renderedLabels[i] ? renderedLabels[i].y : 0})"
					>
						<rect class="label-bg" x="{(labelWidths[i] ? labelWidths[i] + 1 : 0) / -2}" y="0" width={labelWidths[i]} height="24" rx="5" ry="5"/>
						<text class="label-text" x="" y="5" text-anchor="middle" bind:this={labelElements[i]} dominant-baseline="hanging">{labelRenames[sourceGroup.name]}</text>
					</g>
				</g>
			{/each}
			{#each axisLabels.x as label}
				<!-- <line class="axis-grid vertical" x1="{xScale(label)}" y1="0" x2="{xScale(label)}" y2="30"></line> -->
			{/each}
			
			{#if anySourceGroupActive} 
				<g transform="translate(0, 300)">
					{#each subSourcesData as subSource, i}
						<g 
						transform="translate({subSource.position.x}, {subSource.position.y})"
						style="
							--color: {subSourceColorScale(i)}
						">
							<rect class="bar" x="0" y="0" width={subSource.position.width} height="20" />
								
							<text>{subSource.name}</text>
						</g>
					{/each}
				</g>
			{/if}
			
		</g>
	</svg>
</div>