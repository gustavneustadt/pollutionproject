<script>
	import {getContext} from "svelte"
	
	export let currentYearIndex
	
	let store = getContext("store")
	$: currentYear = $store.getYear(currentYearIndex)
	$: totalPollutionAmountBaseYear = $store.getSourceGroupsOfYear().reduce((acc, curr) => acc + curr[2], 0)
	$: totalPollutionAmount = $store.getSourceGroupsOfYear(currentYear).reduce((acc, curr) => acc + curr[2], 0)
	$: maxYearPollutionAmount = $store.getSourceGroupsOfYear($store.getMaxYear()).reduce((acc, curr) => acc + curr[2], 0)
	
	$: maxYearIndex = $store.years.length -1
	$: sliderProgress = (currentYearIndex / maxYearIndex) * 100
</script>

<style>
	header {
		/* position:sticky; */
		width: 100%;
		top: 0;
		left: 0;
		z-index: 100;
		background: var(--colorBackground);
		border-bottom: 2px solid var(--colorBorder);
		height: 6rem;
		display: flex;
		align-items: flex-end;
		justify-content: center;
	}
	.total-tons-base, .total-tons, .current-year {
		font-variant-numeric: tabular-nums;
		text-align: right;
	}
	.total-tons-base, .total-tons {
		/* font-weight: 700; */
		color: var(--colorTextMuted);
	}
	.total-tons.base {
		color: var(--colorTextMuted);
		/* height: .9rem; */
		font-size: .9rem;
		position: relative;
		bottom: 2px;
	}
	
	.year-slider {
		height: .75rem;
		
		margin: 0;
		padding: 0;
		width: 100%; 	
	}
	
	.wrapper {
		display: flex;
		align-items: baseline;
		justify-content: center;
		gap: 1rem;
		padding: 0 0 .5rem;
	}
	.slider-wrapper {
		width: 30rem;
		/* max-width: 30rem; */
		position: relative;
	}
	
	.slider-wrapper .total-tons {
		position: absolute;
		pointer-events: none;
		white-space: nowrap;
		transform: translateX(-50%);
		bottom: 2rem;
		display: flex;
		flex-direction: column;
		align-items: center;
		font-size: .9rem;	
		left: var(--leftMargin);
	}
	
	.slider-wrapper .year {
		font-size: 1.2rem;
		color: var(--colorPrimary);
	}
	
	input[type="range"]::-webkit-slider-thumb, input::-moz-range-thumb, input::-ms-thumb {
		/* -webkit-appearance: none; */
		/* background-color: #666; */
		background: var(--colorPrimary);
		/* width: 10px; */
		/* height: 20px; */
	 }
	 
	span.year {
		/* color: var(--colorPrimary); */
		/* font-size: 1.2rem; */
		font-weight: 700;
		/* color: var(--colorTextMuted); */
	}
	
	.total-tons.hide {
		color: transparent;
	}
	.progress-label-wrapper {
		width: calc(100% - 1rem);
		position: relative;
		margin: .5rem;
		box-sizing: border-box;
		bottom: -1.5rem;
	}
	
</style>
<header>
	<div class="wrapper">
		<div class="total-tons base">

				{$store.niceNumbers(totalPollutionAmountBaseYear.toFixed(0))} kt
				<span class="year">
					{$store.getYear(0)}
				</span>
		</div>
		<div class="slider-wrapper">
			<div class="progress-label-wrapper">
				<div class="total-tons" 
					style="--leftMargin: {sliderProgress}%"
				>
					<span class="year">
						← {currentYear} →
					</span>
					{$store.niceNumbers(totalPollutionAmount.toFixed(0))} kt
				</div>
				
			</div>
			<input class="year-slider" type="range" bind:value={currentYearIndex} min="0" max="{maxYearIndex}"/>
		</div>
		<div class="total-tons base"

		>
				<span class="year">
					{$store.getMaxYear()}
				</span>
				{$store.niceNumbers(maxYearPollutionAmount.toFixed(0))} kt
		</div>
	</div>
</header>