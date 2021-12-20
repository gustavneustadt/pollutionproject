<script>
	import * as d3 from 'd3'
	import { onMount } from 'svelte'
	import { tweened } from 'svelte/motion'
	import { cubicOut } from 'svelte/easing'
	
	let data = null
	let labelsX = []
	let labelsY = []
	let dataGroupedByValueColumn = []
	const width = 900
	const height = 700
	let temporalSlider = 0
	let temporalData = []
	let temporalDataKey
	let tween = tweened()

	onMount(async () => {
		data = await d3.csv("./summedDataFrame.csv").then(d => {
			return d
		})
		dataGroupedByValueColumn = data.reduce(
			(prevVal, currVal) => {	
			if(!(currVal.valueColumn in prevVal)) {
				prevVal[currVal.valueColumn] = []
			}
				prevVal[currVal.valueColumn].push(currVal)
		
			return prevVal
		}, {})	
				
		
		labelsY = Object.values(dataGroupedByValueColumn)[0].map(row => {
			return row[""]
		}).sort()
		
		labelsX = Object.values(dataGroupedByValueColumn)[0].map(row => {
			return Object.keys(row).filter(word => word !== "" && word !== "valueColumn").sort()
		})[0]
		
		console.log(labelsX, labelsY)
		
		Object.entries(dataGroupedByValueColumn).forEach(([key, values]) => {
			
			values = values.map(value => {
				console.log(value)
				return value
			})
			dataGroupedByValueColumn[key] = values.map(value => {
				return labelsX.map(label => {
					return parseFloat(value[label])
				})
			})
			
			// if(!(currVal.valueColumn in prevVal)) {
			// 	prevVal[currVal.valueColumn] = []
			// }
			// prevVal[currVal.valueColumn].push(labels.map(label => parseFloat(currVal[label])))
			// 
			
		})	
		
		console.log(dataGroupedByValueColumn)	
		
		// temporalData = Object.values(data)[temporalSlider]
		// temporalDataKey = Object.keys(data)[temporalSlider]
		
		// tween = tweened(temporalData, {
		// 	duration: 5000,
		// 	easing: cubicOut
		// })
		
	})

    $: temporalData = Object.values(data ?? {})[temporalSlider]
	$: tween.set(temporalData)
	$: temporalDataKey = Object.keys(data ?? {})[temporalSlider]

</script>

<style>
	
</style>

<h1>DEMO</h1>
<pre>
	Year: {temporalDataKey} {temporalSlider}
	<input type="range" bind:value="{temporalSlider}" step="1" min="0" max="{Object.keys(data ?? {}).length - 1}">
</pre>

<!-- <input type="range" bind:value="{textMargin}" min="-500" max="500">
<input type="range" bind:value="{rotateY}" min="-500" max="500"> 
<input type="range" bind:value="{rotate}" step="0.01" min="0" max="10"> -->
<pre>
<!-- textMargin: {textMargin}
rotateX: {rotateX}
rotateY: {rotateY}
rotate: {rotate} rad -->

<!-- {$tween} -->
	
{data}
</pre>

{#if data}

<svg {width} {height}>
	<g>
			{#each $tween as row, x}
				{#each row as val, y}
					<text x="{x*70}" y="{y*30+20}">
						{parseFloat(val).toFixed(1)}
					</text>
				{/each}
			{/each}
		
	</g>
</svg>
{/if}