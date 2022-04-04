<script>
let element
let size = {
	h: 0,
	w: 0
}
	
</script>

<style>
	.wrapper {
		height: 100%;
		width: 100%;
		position: absolute;
		top: 0;
		left: 0;
		/* opacity: .35; */
		/* z-index: -1; */
		/* filter: grayscale(.9); */
		/* mix-blend-mode: hard-light; */
	}
	video {
		height: 100%;
		width: 100%;
		position: absolute;
		top: 0;
		object-fit: cover;
		filter: blur(1.5px);
	}
	/* .canvas-wrapper {
		filter: blur(1.5px) hue-rotate(240deg) brightness(1.4) saturate(.4);
	} */
</style>

<div class="wrapper" bind:clientHeight={size.h} bind:clientWidth={size.w}>

	<video src="pollution.mp4" autoplay loop>
			
	</video>
	
</div>