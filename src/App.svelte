<script>
	import * as d3 from 'd3'
	import chroma from "chroma-js"
	import { onMount, setContext } from 'svelte'
	import PollutantBubble from './PollutantBubble.svelte'
	
	let data = []
	
	let pollutantsData = []
	

		
	const width = 1000
	const height = 1000
	
	var currentYear = 4
	
	const pollutants = 		
	["BC",		"CO", 		"NH3", 		"NMVOC", 	"NOx",		"PM 10", 	"PM 2.5",	"SO2",		"TSP"]
	const pollutantColors = 
	["#509CF7", "#FFCC00", 	"#5856D6", 	"#34C759", 	"#FF2D55",	"#AF52DE", 	"#FF9500",	"#37D2AC",	"#83DCFF"]
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
	$: pollutantValues = pollutants.map((name, i) => {
		if(pollutantsData.length > 0) {
			return years.map((yearNumber, y) => {
				return getArraySum(pollutantsData[i][y])
			})				
		}
	})
	
	
	$: xScale = d3.scaleLog().domain([getMinValue(), getMaxValue()]).range([20, 700])
	$: yScale = d3.scaleLinear().domain([2, 1]).range([0, 100])
	$: sizeScale = d3.scaleSqrt().domain([0, 1]).range([5, 80])
	
	$: console.log(getMinValue(), getMaxValue())
	
	const axisLabels = {
		x: [11, 100, 1000, 10000, 100000],
		y: [-1, -0.5, 0, 0.5]
	}
	
	$: scales = {
		x: xScale,
		y: yScale,
		size: sizeScale
	}
	
	function getArraySum(array = []) {
		return array.reduce((a, b) => parseFloat(a) + parseFloat(b), 0.0)
	}
	
	const niceNumbers = new Intl.NumberFormat("en-US")

	
	$: getOrderedPollutants = () => {
		let mapped = pollutants.map((name, i) => {
			return {
				name: name,
				i: i
			}
		})
		return mapped
		// let sizes = mapped.map(pollutant => getPollutantValue(pollutant.i))
		// return mapped.sort((a, b) => {
		// 	return sizes[a.i] > sizes[b.i]
		// })
	}
	
	const radius = 200
	
	const ribbonGenerator = d3.ribbon().radius(radius);

	$: pollutantsComponentData = pollutants.map((pollutant, i) => {
		let values = pollutantValues[i]

		return {
			scale: {
				x: xScale,
				y: yScale,
				size: sizeScale
			},
			year: currentYear,
			values: values,
			name: pollutant,
			color: pollutantColors[i],
			textColor: "#202020",
			borderColor: "#101010"
		}
	})
</script>

<style>
.axis-grid--horizontal, .axis-grid--vertical {
	stroke: #f4f4f4;
}

.axis-label--horizontal, .axis-label--vertical {
	fill: #d0d0d0;
	font-family: "Merriweather";
	font-weight: 200;
	font-size: 0.8rem;
}



</style>

<h1>DEMO</h1>


<input type="range" min="0" max="{years.length-1}" bind:value="{currentYear}">

<pre>
	{years[currentYear]}
<!-- 	{getOrderedPollutants()} -->
<!-- textMargin: {textMargin}
rotateX: {rotateX}
rotateY: {rotateY}
rotate: {rotate} rad
	 -->
</pre>


{#if pollutantsData.length > 0}
	<svg {width} {height}>
		<g transform="translate(100, 400)">
		{#each axisLabels.y as label}
			<line class="axis-grid--horizontal" x1="{xScale(10)}" y1="{yScale(label)}" x2="{xScale(100000)}" y2="{yScale(label)}"></line>
			<text class="axis-label--horizontal" x="-10" y="{yScale(label)}" text-anchor="end" alignment-baseline="central">{label} %</text>
		{/each}
		{#each axisLabels.x as label}
			<line class="axis-grid--vertical" x1="{xScale(label)}" y1="{yScale(0)}" x2="{xScale(label)}" y2="{yScale(-1)}"></line>
			<text class="axis-label--vertical" x="{xScale(label)}" y="{yScale(2)}" text-anchor="middle">{niceNumbers.format(label)} kt</text>
		{/each}
		
		{#each getOrderedPollutants() as pollutant, i}
				<PollutantBubble 
					scales={scales}
					year={currentYear} 
					pollutant={pollutantsComponentData[i]}
				></PollutantBubble>
		{/each}
	</g>
</svg>
{/if}
