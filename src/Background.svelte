<script>
	import * as THREE from 'three';
	import {onMount} from "svelte"
	
	
	class Particle {
		constructor(position = null) {
			this.position = position ?? {
				x: 0,
				y: 0,
				z: 0
			}
			this.direction = {
				x: .00032,
				y: .00061,
				z: .00021,
			}
			
			this.frequency = Math.random() * 10
		}
		
		reset() {
			let bounding = {
				x: {
					min: -1,
					max: 1
				},
				y: {
					min: -1,
					max: 1
				},
				z: {
					min: -.1,
					max: .1
				}
			}
			
			let p = this.position
			this.position = {
				x: p.x < bounding.x.min ? bounding.x.max : p.x > bounding.x.max ? bounding.x.min : p.x,
				y: p.y < bounding.y.min ? bounding.y.max : p.y > bounding.y.max ? bounding.y.min : p.y,
				z: p.z < bounding.z.min ? bounding.z.max : p.z > bounding.z.max ? bounding.z.min : p.z
			}
			
			return this
		}
		
		move() {
			let p = this.position
			let d = this.direction
			this.position = {
				x: p.x + d.x,
				y: p.y + d.y,
				z: p.z + d.z
			}
			
			// this.position.x = Math.sin(this.position.y) / this.frequency + this.frequency / 100
			
			this.reset()
			
			return this
		}
		
		getPositionArray() {
			return [this.position.x, this.position.y, this.position.y]
		}
	}
	
	
	class ParticleBig {
		constructor() {
			
			let materialRandom = Math.random()
			
			this.geometry = materialRandom > 0.8 ? new THREE.DodecahedronGeometry(.015) : materialRandom > 0.5 ? new THREE.ConeGeometry(.015, 0.03) : new THREE.OctahedronGeometry(0.015)
			this.material = new THREE.MeshNormalMaterial()
			this.mesh = new THREE.Mesh( this.geometry, this.material)
			
			this.mesh.position.x = Math.random() * 2 - 1
			this.mesh.position.y = Math.random() * 2 - 1
			
			this.speed = {
				x: Math.random() * 0.0025 - 0.0012,
				y: Math.random() * 0.0015 + 0.0003,
				z: Math.random() * 0.002 - 0.0005,
				rx: Math.random() * 0.025,
				ry: Math.random() * 0.021,
				rz: Math.random() * 0.018,
			}
		}
		
		rotate() {
			this.mesh.rotation.x += this.speed.rx
			this.mesh.rotation.y += this.speed.ry
			this.mesh.rotation.z += this.speed.ry
			return this
		}
		
		move() {
			// console.log(this.mesh.position)
			this.mesh.position.x += this.speed.x
			this.mesh.position.y += this.speed.y
			// this.mesh.position.z += this.speed.z
			
			
			let bounding = {
				x: {
					min: -1.1,
					max: 1.1
				},
				y: {
					min: -1.1,
					max: 1.1
				},
				z: {
					min: 0,
					max: 0
				}
			}
			
			let p = this.mesh.position

			this.mesh.position.x = p.x < bounding.x.min ? bounding.x.max : p.x > bounding.x.max ? bounding.x.min : p.x
			this.mesh.position.y = p.y < bounding.y.min ? bounding.y.max : p.y > bounding.y.max ? bounding.y.min : p.y
			// this.mesh.position.z = p.z < bounding.z.min ? bounding.z.max : p.z > bounding.z.max ? bounding.z.min : p.z
			
			
			return this
		}
	}
	
	
	let element = null
	
	let size = {
		h: 0,
		w: 0
	}
	
	const scene = new THREE.Scene();
	
	
	let bigParticles = []
	
	for(let i = 0; i < 50; i++) {
		let particle = new ParticleBig()
		bigParticles.push(particle)
		scene.add(particle.mesh)
	}
	
	

	
	// const cubeGeometry = new THREE.DodecahedronGeometry(.01)
	// const cubeMaterial = new THREE.MeshNormalMaterial();
	// 
	// const mesh = new THREE.Mesh( cubeGeometry, cubeMaterial );
	// scene.add( mesh );
	
	const renderer = new THREE.WebGLRenderer( { antialias: true, alpha: true} );
	renderer.setPixelRatio( window.devicePixelRatio );
	
	let camera = null
	
	$: {
		if(size.h > 0) {
			// camera = new THREE.PerspectiveCamera( 70, size.w / size.h, 0.01, 10 );
			if(size.h > size.w) {
				camera = new THREE.OrthographicCamera(size.w / size.h * -1, size.w / size.h, 1, -1, 0, 10)
			} else {
				camera = new THREE.OrthographicCamera(-1, 1, size.h / size.w, size.h / size.w * -1, 0, 10)
			}
			camera.position.z = 1
			camera.position.x = 0
			
		}
	}
	$: renderer.setSize( size.w, size.h );
	
	onMount(() => {
		renderer.setAnimationLoop( animation );
		element.appendChild( renderer.domElement );
	})
	
	
	// animation
	
	function animation( time ) {
		// if(time < 200) {
			bigParticles.forEach(particle => particle.rotate().move() )
		// }
		renderer.render( scene, camera );
	
	}
	
</script>

<style>
	.wrapper {
		height: 100%;
		width: 100%;
		position: absolute;
		top: 0;
		left: 0;
		opacity: .35;
		z-index: -1;
		/* filter: grayscale(.9); */
		/* mix-blend-mode: hard-light; */
	}
	.canvas-wrapper {
		filter: blur(1.5px) hue-rotate(240deg) brightness(1.4) saturate(.4);
	}
</style>

<div class="wrapper" bind:clientHeight={size.h} bind:clientWidth={size.w}>
	<div class="canvas-wrapper" bind:this={element}>
		
	</div>
</div>