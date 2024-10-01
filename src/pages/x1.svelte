<script lang="ts">
	import Viewer from '../viewer.svelte'
	import paper from 'paper'
	import { onMount } from 'svelte'

	let canvas: HTMLCanvasElement
	let radius = 50 // Initial radius value
	let circle: paper.Path.Circle

	$: if (circle && paper.view) {
		circle.scale(radius / circle.bounds.width * 2)
		paper.view.update()
	}

	onMount(() => {
		paper.setup(canvas)
		circle = new paper.Path.Circle({
			center: paper.view.center,
			radius: radius,
			fillColor: new paper.Color('red')
		})
	})
</script>

<Viewer>
	<canvas bind:this={canvas} slot="canvas"></canvas>
	<slot name="title" slot="title">Circle Radius Control</slot>
	<slot name="body" slot="body">
		<input type="range" bind:value={radius} min="10" max="100" step="1">
		<p>Radius: {radius}px</p>
	</slot>
</Viewer>