<script>
	import * as d3 from 'd3'
	import { onMount } from 'svelte'
	
	let data = []
	let labels = []
	
	let pollutantsData = []
	
	let currentTemporalContextKey = 0
	let currentTemporalContext
		
	const width = 900
	const height = 700
	
	var currentYear = 0
	let pollutantPadding = 10
	
	const pollutants = ["BC", "CO", "NH3", "NMVOC", "NOx", "PM 10", "PM 2.5", "SO2", "TSP"]
	const years = [1990, 1995, 2000, 2005, 2010, 2011, 2012, 2013, 2014, 2015, 2016, 2017, 2018, 2019]
	const sources = ["Energy", "Industry", "Agriculture", "Waste"]
	


	onMount(async () => {
		data = await d3.csv("./summedDataFrame.csv").then(d => {
			return d
		})

		pollutantsData = pollutants.map(name => {
			let pollutantData = data.filter(d => name == Object.values(d)[0])
			return years.map(year => {
				let yearData = pollutantData.find(d => d.valueColumn == year)
				return sources.map(s => parseFloat(yearData[s]))
			})
		})
	
	})
	
	$: getPollutantSizes = () => {
		return pollutants.map((pollutant, i) => {
			let sum = getArraySum(pollutantsData[i][currentYear])
			return Math.sqrt(sum / Math.PI)
		})
	}
	
	let getArraySum = (array = []) => {
		return array.reduce((a, b) => a + b, 0.0)
	}
	
	let sliceOfArray = (start, number, array) => {
		return array.filter((item, index) => index >= start && index < number + start)
	}
	
	let getCummulativeSize = (array) => {
		return array.reduce((acc, curr, i) => {
			let cummulativeValue = acc[acc.length-1] ?? 0.0
			
			acc.push(cummulativeValue + curr + array[i+1])
			return acc
		}, [0.0])
	}
	
	$: getPollutantPosition = () => {
		let sizes = getPollutantSizes()
		let cummulativeSizes = getCummulativeSize(sizes)
		console.log(sizes, cummulativeSizes)
		return pollutants.map((pollutant, i) => {
			let sum = getArraySum(pollutantsData[i][currentYear])
			return {
				x: cummulativeSizes[i] + pollutantPadding * i,
				y: 0
			}
		})
	}
	
	
	// $: console.log(pollutantsData)acc
	// $: currentTemporalContext = data[currentTemporalContextKey]
	// $: console.log(currentTemporalContext)
	
	const radius = 200
	
	const ribbonGenerator = d3.ribbon().radius(radius);
	
	// $: chords = d3.chord()
	// 	.sortSubgroups(d3.descending)
	// 	.sortChords(d3.descending)
	// 	.padAngle(0.05)(data)
		
</script>

<style>
circle {
	opacity: 0.3;
}
</style>

<h1>DEMO</h1>


<input type="range" min="0" max="{years.length-1}" bind:value="{currentYear}">

<pre>
	{years[currentYear]}
<!-- textMargin: {textMargin}
rotateX: {rotateX}
rotateY: {rotateY}
rotate: {rotate} rad
	 -->
</pre>


{#if pollutantsData.length > 0}
	<svg {width} {height}>
		<g transform="translate(100, 100)">
			{#each pollutants as name, i}
				<g transform="translate({getPollutantPosition()[i].x}, {getPollutantPosition()[i].y})">
					<circle cx="0" cy="0" r="{getPollutantSizes()[i]}" fill="grey"></circle>
					<text>{name}</text>
				</g>
			{/each}
		</g>
	</svg>
{/if}