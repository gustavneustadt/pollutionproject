<script>
	import * as d3 from 'd3'
	import chroma from "chroma-js"
	import { onMount, setContext } from 'svelte'
	import PollutantBubble from './PollutantBubble.svelte'
	
	import { tweened } from 'svelte/motion';
	import { cubicOut } from 'svelte/easing';
	
	let data = []
	
	let pollutantsData = []
	

		
	const width = 800
	const height = 1000
	
	var currentYear = 4
	
	const pollutants = 		
	["BC",		"CO", 		"NH3", 		"NMVOC", 	"NOx",		"PM10", 	"PM2.5",	"SO2",		"TSP"]
	const pollutantColors = 
	["#509CF7", "#FFCC00", 	"#5856D6", 	"#34C759", 	"#FF2D55",	"#AF52DE", 	"#FF9500",	"#37D2AC",	"#83DCFF"]
	const years = [1990, 1991, 1992, 1993, 1994, 1995, 1996, 1997, 1998, 1999, 2000, 2001, 2002, 2003, 2004, 2005, 2006, 2007, 2008, 2009, 2010, 2011, 2012, 2013, 2014, 2015, 2016, 2017, 2018, 2019]
	const sources = ["Energy", "Industry", "Agriculture", "Waste"]
	
	let trendLineValues = pollutants.map(pollutant => {
		return 0;
	})
	
	let pollutantPositions = pollutants.map(pollutant => {
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

	onMount(async () => {
		data = await d3.csv("./SummedData.csv").then(d => {
			return d
		})

		pollutantsData = pollutants.map(name => {
			
			let pollutantData = data.filter(d => {
				return d[""] == name
			})[0]
			
			let sorted = years.map(year => pollutantData[year])
			
			return sorted
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
	
	$: pollutantValues = pollutants.map((name, i) => {
		if(pollutantsData.length > 0) {
			return years.map((yearNumber, y) => {
				return Number(pollutantsData[i][y])
			})				
		}
	})
	
	
	$: xScale = d3.scaleLinear().domain([0, 100]).range([0, width - 210])
	$: yScale = d3.scaleLinear().domain([-0.63, 0.1]).range([500, 0])
	$: sizeScale = d3.scaleSqrt().domain([getMinValue(), getMaxValue()]).range([5, 80])
	
	$: console.log(getMinValue(), getMaxValue())
	
	const axisLabels = {
		x: [0, 10, 20, 30, 40, 50, 60, 70, 80, 90, 100],
		y: [0.1, 0, -0.1, -0.2, -0.3, -0.4, -0.5]
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
.axis-grid--horizontal, .axis-grid--vertical {
	stroke: #f4f4f4;
}

text {
	fill: #d0d0d0;
	font-family: "Merriweather";
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

.pollutant-information {
	font-family: "Merriweather";
	font-size: .9rem;
	line-height: 1.6;
	width: 25rem;
	background: #F6F6F6;
	padding: 1rem;
}

.pollutant-information p {
	margin: 1rem 0 2.5rem;
}

.pollutant-information h1 {
	font-size: 1.5rem;
}

.pollutant-information h2 {
	font-size: 1.1rem;
	margin: 1rem 0 .5rem;
}

</style>

<h1>DEMO</h1>


<input type="range" min="0" max="{years.length-1}" bind:value="{currentYear}">

<pre>
	{years[currentYear]}
</pre>

<div class="flex">
	<div class="graph-wrapper">
		
		{#if pollutantsData.length > 0}
			<svg {width} {height}>
				<g transform="translate(100, 40)">
					{#each sources as source, i}
						{#each pollutantPositions as connection, j}
							<path class="connection" d={getConnection(connection, getPollutantConnectionPoint(i, j))}></path>
						{/each}
						<g transform="translate({getSourcePosition(i).x}, {getSourcePosition(i).y})">
							<rect class="source__block" x="5" width="{xScale(maxLabel.xMax / sources.length) - 10}" height="20" rx="2" />
							<text class="source__name" y="10" x="{xScale((maxLabel.xMax / sources.length) / 2)}" alignment-baseline="central" text-anchor="middle">
								{source}
							</text>
						</g>
					{/each}
					
					<text transform="rotate(-90) translate({yScale(-0.2) * -1}, -70)" text-anchor="middle">Change per year</text>
					<text transform="translate({xScale(50)}, {yScale(maxLabel.yMin) + 60})" text-anchor="middle">Change to base year</text>
					
					<text class="trend" x="{xScale(maxLabel.xMax) + 10}" y="{yScale($meanTrend)}" alignment-baseline="central" text-anchor="left">Trend per year</text>
					
					<line class="axis-grid--horizontal trend" x1="{xScale(maxLabel.xMin)}" y1="{yScale($meanTrend)}" x2="{xScale(maxLabel.xMax)}" y2="{yScale($meanTrend)}"></line>
				{#each axisLabels.y as label}
					<line class="axis-grid--horizontal {label == 0 ? 'zero' : ''}" x1="{xScale(0)}" y1="{yScale(label)}" x2="{xScale(maxLabel.xMax)}" y2="{yScale(label)}"></line>
					<text class="axis-label--horizontal" x="-10" y="{yScale(label)}" text-anchor="end" alignment-baseline="central">{niceNumbers.format(label)} %</text>
				{/each}
				{#each axisLabels.x as label}
					<line class="axis-grid--vertical" x1="{xScale(label)}" y1="{yScale(maxLabel.yMax)}" x2="{xScale(label)}" y2="{yScale(maxLabel.yMin)}"></line>
					<text class="axis-label--vertical" x="{xScale(label)}" y="{yScale(maxLabel.yMin) + 20}" text-anchor="middle">{niceNumbers.format(label)} %</text>
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

<div class="pollutant-information">
	<h1>PM 2.5</h1>
	<p>
		It can be caused by human activity: Particulate matter can originate from energy supply and industrial processes, metal and steel production as well as the reloading of bulk material or it can be of natural origin, for example as a result of soil erosion. In agglomerations road traffic is the dominant source.
	</p>
	<h2>
		Health risk
	</h2>
	<p>
		Research studies of the World Health Organisation (WHO) have shown increased occurrence of respiratory and cardiovascular diseases at high particulate matter concentrations. Persons with pre-existing disease are especially vulnerable. Studies have shown a measurable reduction in life expectancy with increasing particulate matter concentrations.
	</p>
	<h2>
		Limit values
	</h2>
	<p>
		Since 1 January 2005 limit values for particulate matter (PM10) for protection of human health are put into force. The daily limit value is 50 µg/m3, not to be exceeded more than 35 times per calendar year. The permitted annual limit value is 40 µg/m3. Information on ambient concentrations of particulate matter and exceedances must be made available to the public as promptly as possible.
	</p>
	
	<a class="information-source">German Federal Environmental Agency →</a>
</div>
</div>
