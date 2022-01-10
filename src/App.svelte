<script>
	import * as d3 from 'd3'
	import chroma from "chroma-js"
	import { onMount } from 'svelte'
	import PollutantBubble from './PollutantBubble.svelte'
	
	let data = []
	let labels = []
	
	let pollutantsData = []
	
	let currentTemporalContextKey = 0
	let currentTemporalContext
		
	const width = 1000
	const height = 700
	
	var currentYear = 0
	let pollutantPadding = 10
	
	const maxPollutantSize = 6000
	const minPollutantSize = 100
	
	const pollutants = 		
	["BC",		"CO", 		"NH3", 		"NMVOC", 	"NOx",		"PM 10", 	"PM 2.5",	"SO2",		"TSP"]
	const pollutantColors = 
	["#3B90ED", "#FFE789", 	"#37D2AC", 	"#E8574F", 	"#34B1EA",	"#6C6BCC", 	"#66C27D",	"#AA64CE",	"#E19F41"]
	const years = [1990, 1995, 2000, 2005, 2010, 2011, 2012, 2013, 2014, 2015, 2016, 2017, 2018, 2019]
	const sources = ["Energy", "Industry", "Agriculture", "Waste"]
	
	
	function getPollutantTextColor(pollutantIndex) {
		let color = chroma(pollutantColors[pollutantIndex])
		return color.desaturate(1).luminance(0.3)
	}
	
	function getPollutantBorderColor(pollutantIndex) {
		let color = chroma(pollutantColors[pollutantIndex])
		return color.desaturate(0.3).luminance(0.7)
	}
	
	function getPollutantBgColor(pollutantIndex) {
		let color = chroma(pollutantColors[pollutantIndex])
		return color.desaturate(0.3).luminance(0.8)
	}


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
	
	$: getMaxValue = () => {
		if(pollutantsData.length < 1) {
			return 0
		}
		
		return Math.max(...pollutants.map(
			(pollutant, i) => 
				Math.max(...pollutantsData[i].map(year => getArraySum(year))
				
			)
		))
	}
	
	$: getMinValue = () => {
		if(pollutantsData.length < 1) {
			return 0
		}
		
		return Math.min(...pollutants.map(
			(pollutant, i) => 
				Math.min(
					...pollutantsData[i].map(year => 
						getArraySum(year)
					).filter(x => x > 0)
				
				)
			)
		)
	}
	
	$: getPollutantValue = (pollutantIndex, year = null, yearShift = null) => {
		if(pollutantsData.length < 1) {
			return 0.0
		}
		
		year = year ?? currentYear
		let value = getArraySum(pollutantsData[pollutantIndex][year+yearShift])
		
		return value
	}
	
	
	
	function getPollutantSize(pollutantIndex) {
		// let pollutantValue = getPollutantValue(pollutantIndex)
		let sizeValue = getPollutantPercentageChangeOverAll(pollutantIndex)* (maxPollutantSize - minPollutantSize) + minPollutantSize
		
		return Math.sqrt(sizeValue / Math.PI)
	}

	function getFirstPollutantValue(pollutantIndex) {
		for(let i = 0; i < years.length -1; i++) {
			let value = getPollutantValue(pollutantIndex, i)
			if(value > 0) {
				return value
			}
		}
	}
	
	function getPollutantPercentageChangeOverAll(pollutantIndex) {
		let firstPollutantValue = getFirstPollutantValue(pollutantIndex)
		let currentPollutantValue = getPollutantValue(pollutantIndex)
		
		return currentPollutantValue/firstPollutantValue
	}
	
	function getPollutantPercentageChange(pollutantIndex) {
		let prevPollutantValue = getPollutantValue(pollutantIndex, null, -1)
		let currentPollutantValue = getPollutantValue(pollutantIndex)
		let value = (currentPollutantValue/prevPollutantValue)
		
		return isFinite(value) ? value : 1.0
	}
	
	// let scaleY = d3.scaleLog().
	
	$: xScale = d3.scaleLog().domain([getMinValue(), getMaxValue()]).range([10, 700])
	$: yScale = d3.scaleLog().domain([1, 1.2]).range([0.0, 100.0])
	
	function getPollutantPosition(pollutantIndex, xScale, yScale) {
		let x = getPollutantValue(pollutantIndex)
		let y = getPollutantPercentageChange(pollutantIndex)
		return {
			x: xScale(x),
			// y: yScale(getPollutantPercentageChange(pollutantIndex))
			y: yScale(y)* -1
		}
	}
	
	
	function getArraySum(array = []) {
		return array.reduce((a, b) => a + b, 0.0)
	}
	
	function sliceOfArray(start, number, array) {
		return array.filter((item, index) => index >= start && index < number + start)
	}
	
	function getPercantageString(percentage) {
		return ((percentage - 1.0) * 100.0).toFixed(0) + " %"
	}
	
	function getPollutantNamePosition(pollutantIndex) {
		let size = getPollutantSize(pollutantIndex)
		let yTextOffset = 6
		if(size < 35) {
			return {
				x: 0,
				y: -size - 10
			}
		}
		return {
			x: 0,
			y: yTextOffset
		}
	}
	
	$: getOrderedPollutants = () => {
		let mapped = pollutants.map((name, i) => {
			return {
				name: name,
				i: i
			}
		})
		let sizes = mapped.map(pollutant => getPollutantValue(pollutant.i))
		return mapped.sort((a, b) => {
			return sizes[a.i] > sizes[b.i]
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
	fill: var(--bgColor);
	stroke: var(--borderColor);
	stroke-width: 4px;
}
text {
	font-family: "Merriweather Sans";
	font-size: 1rem;
	letter-spacing: 0.1rem;
	fill: var(--textColor);
}

.pollutant--name {
	font-weight: 400;
}
.pollutant--percentage {
	font-weight: 300;
}
</style>

<h1>DEMO</h1>


<input type="range" min="0" max="{years.length-1}" bind:value="{currentYear}">

<pre>
	{years[currentYear]}
	{getOrderedPollutants()}
<!-- textMargin: {textMargin}
rotateX: {rotateX}
rotateY: {rotateY}
rotate: {rotate} rad
	 -->
</pre>


{#if pollutantsData.length > 0}
	<svg {width} {height}>
		<g transform="translate(100, 100)">
		{#each getOrderedPollutants() as pollutant, i}
			{#if getPollutantValue(pollutant.i) > 0}
				<g transform="translate({getPollutantPosition(pollutant.i, xScale, yScale).x}, {getPollutantPosition(pollutant.i, xScale, yScale).y})" style="
					--borderColor: {getPollutantBorderColor(pollutant.i)};
					--textColor: {getPollutantTextColor(pollutant.i)};
					--bgColor: {getPollutantBgColor(pollutant.i)};
				">
					<circle cx="0" cy="0" r="{getPollutantSize(pollutant.i)}" fill="grey"></circle>
					<text y={getPollutantNamePosition(pollutant.i).y} x={getPollutantNamePosition(pollutant.i).x} class="pollutant--name" text-anchor="middle">{pollutant.name}</text>
					<text y="{getPollutantSize(pollutant.i) + 20}" x="" class="pollutant--percentage" text-anchor="middle">{
						getPercantageString(getPollutantPercentageChange(pollutant.i))
						
						}</text>
				</g>
			{/if}
		{/each}
	</g>
</svg>
{/if}