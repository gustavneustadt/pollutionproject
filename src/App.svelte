<script>
	import * as d3 from 'd3'
	import { onMount } from 'svelte'
	
	let data = []
	let labels = []
	
	const width = 900
	const height = 500
	
	onMount(async () => {
		data = await d3.csv("./DataCorrelation.csv").then(d => {
			return d
		})
		
		labels = data.map(obj => Object.values(obj)).map(arr => arr[0])
		
		data = data.map(obj => Object.values(obj)).map(arr => arr.splice(1)).map(arr => arr.map(val => parseFloat(val) || 0))
	})


	const radius = 150
	let rotateX = 0
	let rotateY = 0
	let rotate = 0
	
	let textMargin = 10
	
	const ribbonGenerator = d3.ribbon().radius(radius);
	
	$: chords = d3.chord()
		.sortSubgroups(d3.descending)
		.sortChords(d3.descending)
		.padAngle(0.05)(data)

	$: groups = chords.groups.map(group => {
		group.label = labels[group.index]
		group.startAngle = Rad2Deg(group.startAngle)
		group.endAngle = Rad2Deg(group.endAngle)
		group.labelPosition = calcLabelPos(group.startAngle, group.endAngle)
		group.labelRotation = group.labelPosition > 180 ? 180 : 0
		
		return group
	})
	
	$: yScale = d3.scaleBand()
		.domain(data.map(d => d["Source"]))
		.range([0, height])
		
	$: xScale = d3.scaleLinear()
		.domain([0, d3.max(data, d => parseFloat(d["2019"]))])
		.range([0, width])
	
	function calcLabelPos(startAngle, endAngle) {
		let sumAngle = endAngle - startAngle
		return startAngle + sumAngle / 2
	}
	
	function Rad2Deg(radians)
	{
	  var pi = Math.PI;
	  return radians * (180/pi);
	}
			  
</script>

<style>
	path {
		fill: grey;
		opacity: 0.3
	}
	
	path:hover {
		opacity: 1
	}
	
	text {
		font-family: "SF Mono"
	}
</style>

<h1>DEMO</h1>

<!-- <input type="range" bind:value="{rotateX}" min="-200" max="200">
<input type="range" bind:value="{rotateY}" min="-200" max="200"> -->
<input type="range" bind:value="{textMargin}" min="-200" max="200">

<svg {width} {height}>
	{console.log(groups)}
	
	<g transform="translate({width/2} {height/2}) rotate(270 -{radius/2} -{radius/2})">
		{#each groups as group, i}
			<text transform="rotate({group.labelPosition} -{radius} 0) translate({textMargin} 0)">
				{group.label}
			</text>
		{/each}
	</g>
	
	<g transform="translate({width/2} {height/2})">
		{#each chords as entry, i}
			<path d="{ribbonGenerator(entry)}">
				
			</path>
		{/each}
	</g>
</svg>