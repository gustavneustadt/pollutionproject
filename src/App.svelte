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
	
	import PollutantGraph from "./PollutantGraph.svelte"
	
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
		pollutants: null,
		years: [1990, 1991, 1992, 1993, 1994, 1995, 1996, 1997, 1998, 1999, 2000, 2001, 2002, 2003, 2004, 2005, 2006, 2007, 2008, 2009, 2010, 2011, 2012, 2013, 2014, 2015, 2016, 2017, 2018, 2019],
		getMinYear: function() {
			return Math.min(...this.years)	
		},
		getMaxYear: function() {
			return Math.max(...this.years)	
		},
		pollutantNames: ["BC",		"CO", 		"NH3", 		"NMVOC", 	"NOx",		"PM10", 	"PM2.5",	"SO2",		"TSP"],
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
	
	let initialized = false
	
	var currentYear = 0

	async function init() {
		let reshapeGroupedItems = (groupedItemsArray) => {
			return groupedItemsArray.map(([sourceGroupName, [...sourceGroupYears]]) => {
				let yearData = sourceGroupYears.map(([year, data]) => {
					let reshapedData = Object.fromEntries(
						Object.entries(data[0]).filter(
							([key, _]) => $store.pollutantNames.includes(key)
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
		
		
		$store.pollutants = await d3.csv("./pollutantsSum.csv").then(d => {
			let grouped = [...d3.group(d, d => d.pollutant)]
			
			let reshape = grouped.map(([pollutantName, years]) => {
				return [pollutantName, years.map(year => [year.year, parseFloat(year.amount)])]
			})
			return reshape
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
		
		return true
	}

	onMount(async () => {
		await init().then(() => {
			initialized = true
		})
	})
	
</script>

<style>
.flex {
	display: flex
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
	{#if initialized}
		<PollutantGraph currentYearIndex={currentYear}/>
	{/if}
</div>
