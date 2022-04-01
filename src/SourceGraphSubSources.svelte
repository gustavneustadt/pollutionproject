<script>
	console.log("jes")
	import * as d3 from 'd3'
	import {tweened} from 'svelte'
	export let subSources = []
	export let xScale
	
	function handleMouse(e) {
		pointer = d3.pointer(e)
	}
	let group
	$: groupRectWidth = group ? group.getBBox().width : null
	$: progress = pointer[0] / groupRectWidth ?? 1
	
	
	let pointer = []
	let 
	
	$: positionScale = d3.scaleBand()
	    .domain(d3.range(subSources.length))
		.range([xScale(0), xScale(100)])
	   
	// $: subSources.forEach((source, i) => console.log(positionScale(i)))
</script>

<style>

</style>
<g bind:this={group} on:mousemove={handleMouse}>
	<rect fill="grey" width={xScale(100)} height="40" y="20">
	</rect>
	<g transform="translate({progress * 100})">
	{#each subSources as source, i}
		<text fill="black" x={positionScale(i)}>
			Hello
		</text>
	{/each}
	</g>
</g>

