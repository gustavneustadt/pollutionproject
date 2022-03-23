<script>
	import * as d3 from 'd3'
	import chroma from "chroma-js"
	import { onMount, setContext, getContext } from 'svelte'
	import { writable } from 'svelte/store'
	import PollutantBubble from './PollutantBubble.svelte'
	import PollutantSourceGroup from './PollutantSourceGroup.svelte'
	import PollutantSources from './PollutantSources.svelte'
	import Header from './Header.svelte'
	
	
	import { tweened } from 'svelte/motion';
	import { cubicOut } from 'svelte/easing';
	
	let data = []
	
	let pollutantsData = []
	
	let sourcesData
	
	let sourceGroupsData
	let subSourcesData
	let subSourcesPerSourceGroupData
	let descriptionPerSubSourcesData
	
	

	const store = writable({
		sourceGroups: null,
		subSources: null,
		subSourceDescriptions: null,
		subSourcesPerSourceGroup: null,
		years: [1990, 1991, 1992, 1993, 1994, 1995, 1996, 1997, 1998, 1999, 2000, 2001, 2002, 2003, 2004, 2005, 2006, 2007, 2008, 2009, 2010, 2011, 2012, 2013, 2014, 2015, 2016, 2017, 2018, 2019],
		getMinYear: function() {
			return Math.min(...this.years)	
		},
		getMaxYear: function() {
			return Math.max(...this.years)	
		},
		pollutants: ["BC",		"CO", 		"NH3", 		"NMVOC", 	"NOx",		"PM10", 	"PM2.5",	"SO2",		"TSP"],
		getYear: function(yearIndex) {
			return this.years[yearIndex]	
		},
		niceNumbers: function(number) {
			let string = new Intl.NumberFormat("en-US").format(number)
			
			return string.replace("-", "–")
		},
		getSubSourceDescription: function(subSourceCode) {
			return this.subSourceDescriptions[subSourceCode]
		},
		getSourceGroupsOfYear: function(year = null) {
			year = year ?? this.getYear(0)
			return this.sourceGroups.map(([sourceGroupName, years]) => {
				let pollutants = years.filter(([groupsYear, _]) => groupsYear == year)[0][1]
				return [
					sourceGroupName,
					pollutants,
					Object.values(pollutants).reduce((acc, curr) => acc + curr, 0)
				]
			})
		},
		getSubSourcesOfYear: function(year) {
			
		},
		getSubSourcesOfSourceGroup: function(sourceGroupName) {
			let subSourceCodes = this.subSourcesPerSourceGroup[sourceGroupName]
			return [...this.subSources].filter(([key, _]) => subSourceCodes.includes(key))
		},
		getValuesOfYear: function(map, year) {
			return map.find(([needleYear, _]) => needleYear == year)[1]
		}
	})
	
	setContext("store", store)
	
	const width = 800
	const height = 1000
	
	let initialized = false
	
	var currentYear = 0
	
	const pollutants = $store.pollutants
	const pollutantColors = 
	["#509CF7", "#FFCC00", 	"#5856D6", 	"#34C759", 	"#FF2D55",	"#AF52DE", 	"#FF9500",	"#37D2AC",	"#83DCFF"]
	const years = $store.years
	const sources = ["Energy", "Industry", "Agriculture", "Waste"]
	
	let trendLineValues = pollutants.map(_ => {
		return 0;
	})
	
	let pollutantPositions = pollutants.map(_ => {
		return {
			x: 0,
			y: 0
		}
	})
	
	$: cleanedTrendValues = trendLineValues.filter(val => Math.abs(val) != Infinity)
	let meanTrend = tweened(0, {
		duration: 600,
		easing: cubicOut
	})
	
	$: {
		meanTrend.set(cleanedTrendValues.reduce((acc, curr) => acc + curr)/cleanedTrendValues.length)
	}

	async function init() {
		let reshapeGroupedItems = (groupedItemsArray) => {
			return groupedItemsArray.map(([sourceGroupName, [...sourceGroupYears]]) => {
				let yearData = sourceGroupYears.map(([year, data]) => {
					let reshapedData = Object.fromEntries(
						Object.entries(data[0]).filter(
							([key, _]) => pollutants.includes(key)
						).map(
							([key, value]) => {
							return [key, parseFloat(value)]
						})
					)
					return [year, reshapedData]
				})
				
				return [sourceGroupName, yearData]
			})
		}
		
		
		data = await d3.csv("./pollutantsSum.csv")
		
		pollutantsData = pollutants.map(name => {
			
			let pollutantData = data.filter(d => {
				return d[""] == name
			})[0]
			
			let sorted = years.map(year => pollutantData[year])
			
			return sorted
		})
		
		const fetchSourceGroupsSum = await d3.csv("./sourceGroupsSum.csv")
		
		const groupedSourceGroupsSum = [...d3.group(fetchSourceGroupsSum, d => d.index, d => d.year)]
		
		$store.sourceGroups = reshapeGroupedItems(groupedSourceGroupsSum)
		
		
		const fetchSubSources = await d3.csv("./subSourcesSum.csv")
		
		const groupedSubSources = [...d3.group(fetchSubSources, d => d.code, d => d.year)]
		
		$store.subSources = reshapeGroupedItems(groupedSubSources)
		
		
		const fetchSubSourceDescriptions = await d3.json("./subSourcesDescriptions.json").then(d => {
			return Object.fromEntries(
				Object.entries(d).map(([key, value]) => {
					return [key, value[0]]
				})
			)
		})
		
		$store.subSourceDescriptions = fetchSubSourceDescriptions
		
		
		const fetchSubSourcesPerSourceGroup = await d3.json("./subSourcesPerSourceGroup.json")
		
		$store.subSourcesPerSourceGroup = fetchSubSourcesPerSourceGroup
		
		return 1
	}

	onMount(async () => {
		await init().then(() => {
			initialized = true
		})
	})
	
	$: getMaxValue = () => {
		if(pollutantsData.length < 1) {
			return 0
		}
		
		let pollutantMaxes = pollutantsData.map(pollutant => Math.max(...pollutant))
		
		return Math.max(...pollutantMaxes)
	}
	
	$: getMinValue = () => {
		if(pollutantsData.length < 1) {
			return 0
		}
		
		let pollutantMaxes = pollutantsData.map(pollutant => Math.min(...pollutant.filter(x => x > 0)))
				
		return Math.min(...pollutantMaxes)
	}
	
	$: pollutantValues = pollutants.map((_, i) => {
		if(pollutantsData.length > 0) {
			return years.map((_, y) => {
				return Number(pollutantsData[i][y])
			})				
		}
	})

	$: xScale = d3.scaleLinear().domain([100, 0]).range([0, width - 210])
	$: yScale = d3.scaleLinear().domain([-63, 10]).range([500, 0])
	$: sizeScale = d3.scaleSqrt().domain([getMinValue(), getMaxValue()]).range([5, 80])
	
	const axisLabels = {
		x: [0, 10, 20, 30, 40, 50, 60, 70, 80, 90, 100],
		y: [10, 0, -10, -20, -30, -40, -50]
	}
	
	$: scales = {
		x: xScale,
		y: yScale,
		size: sizeScale
	}
	
	let maxLabel = {
		xMin: Math.min(...axisLabels.x),
		xMax: Math.max(...axisLabels.x),
		yMin: Math.min(...axisLabels.y),
		yMax: Math.max(...axisLabels.y),
	}
	
	function getArraySum(array = []) {
		return array.reduce((a, b) => parseFloat(a) + parseFloat(b), 0.0)
	}
	
   	const niceNumbers = $store.niceNumbers

	
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
	
	const linkGenerator = d3.linkVertical()
		.x(function(d) {
			return d.x;
		})
		.y(function(d) {
			return d.y;
		});
	
	$: links = sources.map((source, i) => {
		let target = {
			x: i * 100,
			y: 0
		}
		return pollutantPositions.map(position => {
			let data = {
				target: target,
				source: position
			}
			return linkGenerator(data)
		})
	})
	  
	function newLink(d) {
		let [source, target] = [d.source, d.target]
		
		let cu = .2
		
		let yi = d3.interpolateNumber(d.target.y, d.source.y)
		
		return	"M" + source.x + "," + source.y + " " +
				"C" + source.x + "," + (source.y - yi(cu)) + " " +
					  target.x + "," + (target.y + yi(cu)) + " " +
					  target.x + "," + target.y + " " +
				"L" + (target.x + target.width) + "," + target.y + " " +
				"C" + (target.x + target.width) + "," + (target.y + yi(cu)) + " " +
			  		  (source.x + source.width) + "," + (source.y - yi(cu)) + " " +
			  		  (source.x + source.width) + "," + source.y + " " +
				"L" + source.x + "," + source.y + " "
					  
	}
	
	function getConnection (pollutantPosition, sourcePosition) {

		let newLinkData = {
			source: {
				x: sourcePosition.x,
				y: sourcePosition.y,
				width: 20,
			},
			target: {
				x: pollutantPosition.x,
				y: pollutantPosition.y,
				width: 1
			}
		}
		
		return newLink(newLinkData)
	}
	
	function getSourcePosition(i) {
		return {
			x: xScale((maxLabel.xMax / sources.length) * i),
			y: yScale(maxLabel.yMin) + 100
		}
	}
	
	function getPollutantConnectionPoint(i, j) {
		let sourcePosition = getSourcePosition(i)
		return {
			x: sourcePosition.x + j * 20,
			y: sourcePosition.y
		}
	}
</script>

<style>
* {
	font-family: "Fira Sans";
}
.axis-grid--horizontal, .axis-grid--vertical {
	stroke: #f4f4f4;
}

text {
/* 	fill: #d0d0d0; */
/* 	font-family: "Merriweather"; */
	font-weight: 200;
	font-size: 0.8rem;
}

.axis-grid--horizontal.zero {
	stroke: #d9d9d9;
}

.axis-grid--horizontal.trend {
	stroke: red;
	stroke-width: 3px;
	opacity: .1;
}

.source__name {
	font-weight: 500;
	fill: white;
}

.source__block {
	fill: #d9d9d9;
}
.connection {
	fill: black;
	stroke: none;
/* 	fill: none; */
/* 	stroke: #f4f4f4; */
/* 	stroke-width: 2; */
	opacity: 0.03;
}

.flex {
	display: flex
}

svg {
	width: 100%;
	position: absolute;
	top: 0;
	left: 0;
	
}
.graph-wrapper {
	width: 100%;
	height: 100rem;
	position: relative;
}
header {
	position: fixed;
	top: 0;
	width: 100%;
	left: 0;
	z-index: 100;
	background: white;
}


</style>

<h1>
	Air Pollution in Germany 1990 – 2019
</h1>
<div class="title-subline">
	Vizualizing Air Pollution Data reported from the European Environment Agency.
	Report
	Data from 1990 to 2019
</div>

<p>
	
</p>

{#if initialized}
	<Header bind:currentYearIndex={currentYear} />
{/if}
<div class="flex">
	<div class="graph-wrapper">
		
		{#if initialized}
			<svg viewBox="0 0 {width} {height}" preserveAspectRatio="xMidYMid meet">
				<g transform="translate(100, 40)">
					<g transform="translate(0, {yScale(-50) + 100})">
						<PollutantSources year={years[currentYear]} xScale={xScale}/>			
							
						{#each axisLabels.x as label}
							<line class="axis-grid--vertical" x1="{xScale(label)}" y1="-70" x2="{xScale(label)}" y2="0"></line>
						{/each}		
					</g>
					
					<text transform="rotate(-90) translate({yScale(40)}, -70)" text-anchor="middle">Change to previous year</text>
					<text transform="translate({xScale(50)}, {yScale(maxLabel.yMin) + 60})" text-anchor="middle">Compared to base year</text>
					
					<text class="trend" x="{xScale(maxLabel.xMax) + 10}" y="{yScale($meanTrend)}" alignment-baseline="central" text-anchor="left">Trend per year</text>
					
					<line class="axis-grid--horizontal trend" x1="{xScale(maxLabel.xMin)}" y1="{yScale($meanTrend)}" x2="{xScale(maxLabel.xMax)}" y2="{yScale($meanTrend)}"></line>
				{#each axisLabels.y as label}
					<line class="axis-grid--horizontal {label == 0 ? 'zero' : ''}" x1="{xScale(0)}" y1="{yScale(label)}" x2="{xScale(maxLabel.xMax)}" y2="{yScale(label)}"></line>
					<text class="axis-label--horizontal" x="-10" y="{yScale(label)}" text-anchor="end" alignment-baseline="central">{niceNumbers(label)} %</text>
				{/each}
				{#each axisLabels.x as label}
					<line class="axis-grid--vertical" x1="{xScale(label)}" y1="{yScale(maxLabel.yMax)}" x2="{xScale(label)}" y2="{yScale(maxLabel.yMin)}"></line>
					<text class="axis-label--vertical" x="{xScale(label)}" y="{yScale(maxLabel.yMin) + 20}" text-anchor="middle">{niceNumbers(label)} %</text>
				{/each}
				
				{#each getOrderedPollutants() as pollutant, i}
						<PollutantBubble
							on:click={() => alert(1)}
							scales={scales}
							year={currentYear} 
							bind:trend={trendLineValues[i]}
							bind:pollutantPosition={pollutantPositions[i]}
							pollutant={pollutantsComponentData[i]}
						></PollutantBubble>
				{/each}
			</g>
		</svg>
		{/if}
	</div>
</div>
