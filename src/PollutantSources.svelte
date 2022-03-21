<script>
	import * as d3 from "d3"
	import { getContext } from "svelte"
	import { sumArrayOfObjects } from "./functions.svelte"
	import { tweened } from 'svelte/motion';
	import { cubicOut } from 'svelte/easing';
	export let year
	// export let data
	export let xScale
	
	let store = getContext("store")
	
	$: sourceGroups = $store.sourceGroups
	$: sourceGroupsCurrentYear = $store.getSourceGroupsOfYear(year)

	$: totalTons = sourceGroupsCurrentYear.reduce((acc, curr) => acc + curr[2], 0)
	$: percentagePerSourceGroup = sourceGroupsCurrentYear.map(([a, b, amount]) => (amount / totalTons) * 100)
	
	$: totalTonsBase = $store.getSourceGroupsOfYear().reduce((acc, curr) => acc + curr[2], 0)
	
	$: totalTonsPercentageToBaseTweened.set(totalTons / totalTonsBase * 100)
	
	$: percentagePerSourceGroupTweened.set(percentagePerSourceGroup)
	
	let percentagePerSourceGroupTweened = tweened(null, {
		easing: cubicOut
	})
	
	let totalTonsPercentageToBaseTweened = tweened(100, {
		easing: cubicOut
	})
	
	$: getPercentageOffsetSourceGroup = (i) => {
		if($percentagePerSourceGroupTweened.length < 1) {
			return 0
		}
		return $percentagePerSourceGroupTweened.slice(0, i).reduce((acc, curr) => acc + curr, 0)
	}
</script>

<style>

</style>

{#each sourceGroupsCurrentYear as sourceGroup, i}
<g transform="translate({xScale(getPercentageOffsetSourceGroup(i))},0)">
	<rect x="0" width="{xScale($percentagePerSourceGroupTweened[i])}" height="30" rx="5" />
	<text y="-10">
		{sourceGroup[0]}
	</text>
</g>
{/each}

<g transform="translate(0, 200)">
	<rect x="0" width="{xScale($totalTonsPercentageToBaseTweened)}" height="30" rx="5" />
</g>