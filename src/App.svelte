<script>
	import * as d3 from 'd3'
	import { onMount } from 'svelte'
	
	let data = []
	let labels = []
	
	const width = 900
	const height = 700
	
	onMount(async () => {
		data = await d3.csv("./DataCorrelation.csv").then(d => {
			return d
		})
		
		labels = data.map(obj => Object.values(obj)).map(arr => arr[0])
		
		data = data.map(obj => Object.values(obj)).map(arr => arr.splice(1)).map(arr => arr.map(val => parseFloat(val) || 0))
	})


	let rotateX = 0
	let rotateY = 0
	let rotate = Math.PI * 1.5
	
	let textReverseAngle = Math.PI/2
	
	const radius = 200

	let textLabels = []
	
	let textMargin = 10
	
	const ribbonGenerator = d3.ribbon().radius(radius);
	
	$: chords = d3.chord()
		.sortSubgroups(d3.descending)
		.sortChords(d3.descending)
		.padAngle(0.05)(data)
		
	$: groups = chords.groups.map((group, index) => {
		group.label = labels[group.index]
		group.startAngle = group.startAngle
		group.endAngle = group.endAngle
		group.labelPosition = (calcLabelPos(group.startAngle, group.endAngle) + rotate)  % (Math.PI * 2)

		
		group.textLabelReverse = group.labelPosition > Math.PI ? true : false
		
				
		group.textLabel = textLabels[index]
		group.textLabelLength = 0
		
		if(group.textLabel) {
			group.textLabelLength = Math.max(...[...group.textLabel.children].map(el => el.textLength.baseVal.value))
		}
		
		group.valueTextLabel = new Intl.NumberFormat("en-US", {
			maximumSignificantDigits: 2
		}).format(group.value)+" kt"
		
		// console.log(group.textLabel ? [group.textLabel.textLength, group.textLabel] : null)
		
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
	
	function Rad2Deg(radians) {
	  var pi = Math.PI;
	  return radians * (180/pi);
	}
			  
</script>

<style>
	path {
		fill: black;
		opacity: 0.3
	}
	
	path:hover {
		opacity: 1
	}
	
	text {
		font-family: "SF Mono"
	}
	
	.title {
		font-weight: bold;
	}
	.value {
		fill: #aaa;
	}
</style>

<h1>DEMO</h1>

<input type="range" bind:value="{textMargin}" min="-500" max="500">
<input type="range" bind:value="{rotateY}" min="-500" max="500"> 
<input type="range" bind:value="{rotate}" step="0.01" min="0" max="10">
<pre>
textMargin: {textMargin}
rotateX: {rotateX}
rotateY: {rotateY}
rotate: {rotate} rad
	
</pre>

<svg {width} {height}>

		<g transform="translate({width/2} {height/2}) rotate({-90} {rotateX} {rotateY})">
			{#each groups as group, i}
<!-- 			rotate({Rad2Deg(group.labelPosition)} -{radius} 0)  -->
				<text 
					bind:this="{textLabels[i]}"
					text-anchor="{group.textLabelReverse ? 'end' : ''}" 
					transform="
						rotate({Rad2Deg(group.labelPosition)} 0 0) 
						translate({textMargin + radius})
						{group.textLabelReverse ? 'scale(-1 -1)' : ''}
					"
					>
					<tspan class="title" x="0" y="0">{group.label}</tspan><tspan class="value" x="0" y="19">{group.valueTextLabel}</tspan>
				</text>
			{/each}
		</g>
		
		<g transform="translate({width/2} {height/2}) rotate({Rad2Deg(rotate)} 0 0)" >
			{#each chords as entry, i}
				<path d="{ribbonGenerator(entry)}">
					
				</path>
			{/each}
		</g>
</svg>