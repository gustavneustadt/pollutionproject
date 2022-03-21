<script>
	export let data
	export let order
	
	$: position = order * 100
</script>

<style>
	.background {
		fill: blue;
	}
</style>

<g transform="translate({position}, 0)">
<!-- 	<rect class="background" x="5" width="100" height="20" rx="2" /> -->
	<text>
		{data[0]}
	</text>
</g>