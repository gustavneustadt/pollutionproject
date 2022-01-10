<script>
	export let pollutant = {
		pos: {
			x: 0,
			y: 0
		},
		name: "JUST",
		percentageChange: "100 %",
		size: 20
	}
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

<g transform="translate({pollutant.pos.x}, {pollutant.pos.y})" style="
	--borderColor: grey;
	--textColor: darkgrey;
	--bgColor: lightgrey;
">
	<circle cx="0" cy="0" r="{pollutant.size}" fill="grey"></circle>
	<text y={pollutant.text.pos.y} x={pollutant.text.pos.x} class="pollutant--name" text-anchor="middle">{pollutant.name}</text>
	<text y="{pollutant.size + 20}" x="" class="pollutant--percentage" text-anchor="middle">
		{
		pollutant.percentageChange
		}
	</text>
</g>