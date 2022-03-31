<script>
	import * as d3 from 'd3'
	import chroma from "chroma-js"
	import { onMount, setContext, getContext } from 'svelte'
	import { writable } from 'svelte/store'
	import PollutantBubble from './PollutantBubble.svelte'
	import PollutantSourceGroup from './PollutantSourceGroup.svelte'
	import PollutantSources from './PollutantSources.svelte'
	import Header from './Header.svelte'
	import Background from './Background.svelte'
	
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
	
	let introHeight
	
	

	const store = writable({
		sourceGroups: null,
		subSources: null,
		subSourceDescriptions: null,
		subSourcesPerSourceGroup: null,
		pollutants: null,
		years: [1990, 1991, 1992, 1993, 1994, 1995, 1996, 1997, 1998, 1999, 2000, 2001, 2002, 2003, 2004, 2005, 2006, 2007, 2008, 2009, 2010, 2011, 2012, 2013, 2014, 2015, 2016, 2017, 2018, 2019],
		getFirstYearIndex: function(values) {
			let index = values.findIndex(([_, amount]) => amount > 0)
			
			return index ?? 0
		},
		getFirstYear: function(values) {
			let year = values.find(([_, amount]) => amount > 0)
			
			return year ? year[0] : null
		},
		getBaseValue: function(values) {
			let value = values.find(([_, amount]) => amount > 0)
			return value ? value[1] : null
		},
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
			return this.getSourcesOfYear(this.sourceGroups, year)
		},
		getSourcesOfYear: function(collection, year) {
			return collection.map(([itemName, years]) => {
				let pollutants = years.filter(([groupYear, _]) => groupYear == year)[0][1]
				
				return [
					itemName,
					pollutants,
					Object.values(pollutants).reduce((acc, curr) => acc + curr, 0)
				]
				
			})
		},
		getSubSourcesOfYear: function(year) {
			
		},
		getSubSourcesOfSourceGroup: function(sourceGroupName) {
			let subSourceCodes = this.subSourcesPerSourceGroup[sourceGroupName]
			if(!subSourceCodes) {
				return []
			}
			return [...this.subSources].filter(([key, _]) => subSourceCodes.includes(key))
		},
		getValuesOfYear: function(map, year) {
			return map.find(([needleYear, _]) => needleYear == year)[1]
		}
	})
	
	setContext("store", store)
	
	let initialized = false
	
	var currentYear = 15

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
	display: flex;
	/* flex-direction: column; */
	/* max-width: 40rem; */
	margin: 0 2rem 0 min(8.125rem, calc(70% * .125));
	/* align-items: center; */
	align-items: baseline;
	position: relative;
	justify-content: space-between;
	gap: 2rem;
}


p {
	line-height: 1.7;
	max-width: 28rem;
	display: block;
	box-sizing: border-box;
	/* margin-left: 4rem; */
	font-variant-numeric: oldstyle-nums;
	margin: .5rem 0 1rem 0;
}

p a {
	color: var(--colorTextEm);
	text-decoration: underline;
	text-underline-offset: 2px;
	text-decoration-color:  var(--colorBorder);
	position: relative;
}

p a:hover {
	color: var(--colorPrimary);
}

.source {
	font-style: italic;
}
.info-text, .source {
	font-size: .8rem;
	color: var(--colorTextMuted);
}
.graphs-wrapper {
	background: var(--colorBackgroundLight);
	/* height: 200vh; */
}

h1 .subtitle {
	color: var(--colorPrimary);
	font-style: italic;
	/* font-variant-numeric: oldstyle-nums; */
	/* font-weight: 800; */
}
h1 {
	margin-top: 10rem;
	display: flex;
	flex-direction: column;
	line-height: 1.2;
}
.intro-wrapper {
	z-index: 10;
	position: sticky;
	top: calc(var(--intro-height) + 6rem);
	/* padding-top: 100%; */
	/* transform: translateY(calc(-100% + 6rem)); */
	/* outline: 100px solid red; */
}

.flex > .right, .flex > .left {
	display: flex;
	flex-direction: column;
}

.right {
	flex: 10 4;
	align-items: flex-end;
}
.left {
	flex: 16 10;
}

.credits {
	font-size: .8rem;
	color: var(--colorTextMuted);
	text-align: right;
	font-style: italic;
	margin-left: 0;
}
.seperator {
	height: 1px;
	width: 30%;
	/* margin-left: 4rem; */
	margin: 1rem 0;
	background: var(--colorBorder);
}

.title-subline {
	font-size: 1.1rem;
}
</style>
<div class="page">
	
	<div class="intro-wrapper" bind:clientHeight={introHeight} style="--intro-height: {introHeight*-1}px">
		<Background></Background>
		<div class="flex">
			<div class="left">
				<h1>
					<span class="title">
						Visualizing Air Emissions Data
					</span>
					<span class="subtitle">
						1990–2019 Germany
					</span>
				</h1>
				<p class="title-subline">
					Vizualizing Air Emissinos <a href="https://cdr.eionet.europa.eu/de/un/clrtap/inventories/envyb590q/overview" target="_blank">Data reported to the European Environment Agency (EEA)</a> by the German Environment Agency (<a href="https://www.umweltbundesamt.de" target="_blank">Umweltbundesamt</a>) on February 2021.
				</p>
				<div class="seperator"></div>
				<p>
					Most Europeans live in areas, especially cities, where air pollution can reach high levels. Both short- and long-term exposure to air pollution can lead to a wide range of diseases, including stroke, chronic obstructive pulmonary disease, trachea, bronchus and lung cancers, aggravated asthma and lower respiratory infections. The <a href="https://www.euro.who.int/en/health-topics/environment-and-health/air-quality/publications/2016/who-expert-consultation-available-evidence-for-the-future-update-of-the-who-global-air-quality-guidelines-aqgs-2016" target="_blank">World Health Organization</a> (WHO) provides evidence of links between exposure to air pollution and type 2 diabetes, obesity, systemic inflammation, Alzheimer’s disease and dementia. The <a href="https://www.iarc.fr/news-events/iarc-outdoor-air-pollution-a-leading-environmental-cause-of-cancer-deaths/" target="_blank">International Agency for Research on Cancer</a> has classified air pollution, in particular PM2.5, as a leading cause of cancer. A recent <a href="https://www.ncbi.nlm.nih.gov/pmc/articles/PMC6904854/" target="_blank">global review</a> found that <em>chronic exposure can affect every organ in the body</em>, complicating and exacerbating existing health conditions.
				</p>
				
				<p>
					The <a href="https://www.eea.europa.eu/publications/health-risks-of-air-pollution/health-impacts-of-air-pollution" target="_blank">EEA estimates that</a>, in 2019, approximately 307,000 premature deaths were attributable to PM2.5 in the 27 EU Member States. Nitrogen dioxide (NO2) was linked to 40,400 premature deaths, and ground-level ozone was linked to 16,800 premature deaths. 
				</p>
				<p>
					In 2021, the World Health Organization (WHO) published <a href="https://apps.who.int/iris/handle/10665/345329" target="_blank">new air quality guidelines</a> to protect human health, updating the <a href="https://www.who.int/publications/i/item/WHO-SDE-PHE-OEH-06.02" target="_blank">2005 air quality guidelines</a> on the basis of a systematic review of the latest scientific evidence of how air pollution damages human health.
				</p>
				<p>
					The European Union (EU) also set <a href="https://ec.europa.eu/environment/air/quality/standards.htm" target="_blank">standards</a> for key air pollutants in the <a href="https://ec.europa.eu/environment/air/quality/existing_leg.htm" target="_blank">ambient air quality directives</a>. Although these values were based on the 2005 WHO air quality guidelines, they also reflect the technical and economic feasibility of their attainment across EU Member States. The EU air quality standards are therefore less demanding than the WHO air quality guidelines.
				</p>
				<p class="source">Text Source: <a href="https://www.eea.europa.eu/themes/air/health-impacts-of-air-pollution" target="_blank">Air pollution: how it affects our health</a></p>
				<p class="info-text">
					With the slider below you can change the following visualizations and it shows the annual total emissions in the years from 1990 to 2019.
				</p>
				
			</div>
			<div class="right">
				<p class="credits">
					Created by Gustav Neustadt <br>
					»Advances in Data Visualization: Network & Hierarchies«<br>
					by Mark-Jan Bludau<br>
					Data from <a href="https://cdr.eionet.europa.eu/de/un/clrtap/inventories/envyb590q/overview" target="_blank">European Environment Agency (EEA)</a><br>
					Font used <a href="https://www.lucasfonts.com/fonts/the-mix" target="_blank">»TheMix« by Luc(as) de Groot</a> <br>
				</p>
				
				<div class="seperator"/>
					
				<p class="credits">
					Other Sources<br>
					<a href="https://projects.au.dk/mapeire/spatial-modelling/requirements/" target="_blank">GNFR Definiton Table</a> <br>
					<a href="https://unfccc.int/sites/default/files/resource/AGI%202020_final.pdf" target="_blank">Aggregate information on greenhouse gas emissions […]</a><br>
					<a href="https://www.eea.europa.eu/sandbox/laura/air-quality-status-briefing-2021" target="_blank">Air Quality Status (EEA)</a><br>
					<a href="https://www.eea.europa.eu/publications/air-quality-in-europe-2021/health-impacts-of-air-pollution" target="_blank">Health Impacts of Air Pollution (EEA)</a><br>
				</p>
				
			</div>
		</div>
		{#if initialized}
			<Header bind:currentYearIndex={currentYear} />
		{/if}
	</div>
	
	
	<div class="graphs-wrapper">
		{#if initialized}
			<PollutantGraph currentYearIndex={currentYear}/>
		{/if}
	</div>
</div>
