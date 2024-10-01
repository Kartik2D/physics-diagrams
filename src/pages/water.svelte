<script lang="ts">
	import Viewer from '../viewer.svelte'
	import paper from 'paper'
	import { onMount } from 'svelte'

	let canvas: HTMLCanvasElement
	let voltage = 5
	let resistance = 2.75
	let current = 1
	let dashOffset = 0
	let particles: paper.Path.Circle[] = []

	$: {
		// Calculate current based on voltage and resistance
		current = voltage / resistance
		if (paper.project) {
			drawCircuit()
		}
	}

	const waterColor = new paper.Color('rgba(52, 152, 219, 0.7)')
	const outlineColor = new paper.Color('rgba(41, 128, 185, 0.7)')
	const voltageColor = new paper.Color('#3498db')
	const resistanceColor = new paper.Color('#e74c3c')
	const currentColor = new paper.Color('#2ecc71')
	const tankOutlineColor = new paper.Color('rgba(128, 128, 128, 0.5)')  // transparent grey

	interface Particle extends paper.Path.Circle {
		data: {
			velocity: number;
			lifespan: number;
		};
	}

	function createParticle(x: number, y: number, scale: number): Particle {
		const particle = new paper.Path.Circle({
			center: [x, y],
			radius: 2 * scale,
			fillColor: new paper.Color(1, 1, 1, 0.5)
		}) as Particle;
		particle.data = {
			velocity: Math.random() * 1 + 0.5, // Decreased speed
			lifespan: Math.random() * 0.5 + 0.5
		};
		return particle;
	}

	function updateParticles(scale: number): void {
		const tankWidth = 100 * scale;
		const tankHeight = 200 * scale;
		const tankLeft = 50 * scale;
		const tankTop = 50 * scale;

		// Remove dead particles
		particles = particles.filter(p => p.data.lifespan > 0);

		// Update existing particles
		particles.forEach(p => {
			p.position.y += p.data.velocity * (voltage / 5);
			p.data.lifespan -= 0.016 * (voltage / 5);
			if (p.fillColor) {
				p.fillColor.alpha = Math.min(p.data.lifespan, 1);
			}

			if (p.position.y > tankTop + tankHeight) {
				p.position.y = tankTop;
			}
		});

		// Add new particles
		while (particles.length < 50) {
			const x = Math.random() * tankWidth + tankLeft;
			const y = Math.random() * tankHeight + tankTop;
			particles.push(createParticle(x, y, scale));
		}
	}

	function drawCircuit(): void {
		if (!paper.project) return

		paper.project.activeLayer.removeChildren()

		// Increase overall size
		const scale = 1.5

		// Draw tank (voltage) without a roof
		const tank = new paper.Path()
		tank.add(new paper.Point(50 * scale, 50 * scale))
		tank.lineTo(new paper.Point(50 * scale, 250 * scale))
		tank.lineTo(new paper.Point(150 * scale, 250 * scale))
		tank.lineTo(new paper.Point(150 * scale, 50 * scale))
		tank.strokeColor = tankOutlineColor
		tank.strokeWidth = 6

		// Draw static water level with gradient
		const waterHeight = 220 * scale
		const waterTop = 30 * scale
		const waterGradient = {
			gradient: {
				stops: [
					{ offset: 0, color: new paper.Color('rgba(52, 152, 219, 0.7)') },
					{ offset: 1, color: new paper.Color('rgba(52, 152, 219, 0)') }
				]
			},
			origin: new paper.Point(50 * scale, waterTop + waterHeight),
			destination: new paper.Point(50 * scale, waterTop)
		}
		const water = new paper.Path.Rectangle({
			point: [50 * scale, waterTop],
			size: [100 * scale, waterHeight],
			fillColor: waterGradient
		})

		// Update and draw particles
		updateParticles(scale);
		particles.forEach(p => paper.project.activeLayer.addChild(p));

		// Draw resistance (gap between curves at exit)
		const resistanceGap = 50 * scale * ((resistance - 0.5) / 4.5)
		const startY = (250 * scale) - resistanceGap
		const endY = 250 * scale

		// Calculate curve distance based on current
		const curveDistance = 200 * scale * (current / 5)
		const topCurveExtension = 20 * scale * current

		// Draw water flow (two Bézier curves)
		const topCurve = new paper.Path()
		topCurve.add(new paper.Point(150 * scale, startY))
		topCurve.cubicCurveTo(
			new paper.Point((150 * scale) + (curveDistance / 2), startY),
			new paper.Point((150 * scale) + curveDistance + topCurveExtension, startY),
			new paper.Point((150 * scale) + curveDistance + topCurveExtension, paper.view.size.height + (50 * scale))
		)

		const bottomCurve = new paper.Path()
		bottomCurve.add(new paper.Point(150 * scale, endY))
		bottomCurve.cubicCurveTo(
			new paper.Point((150 * scale) + (curveDistance / 2), endY),
			new paper.Point((150 * scale) + curveDistance, endY),
			new paper.Point((150 * scale) + curveDistance, paper.view.size.height + (50 * scale))
		)

		const flowOutline = new paper.Group([topCurve, bottomCurve])
		flowOutline.strokeColor = outlineColor
		flowOutline.strokeWidth = 6
		flowOutline.dashArray = [8, 8]
		flowOutline.dashOffset = -dashOffset

		// Fill water area
		const waterFlow = new paper.Path()
		waterFlow.add(new paper.Point(150 * scale, startY))
		waterFlow.cubicCurveTo(
			new paper.Point((150 * scale) + (curveDistance / 2), startY),
			new paper.Point((150 * scale) + curveDistance + topCurveExtension, startY),
			new paper.Point((150 * scale) + curveDistance + topCurveExtension, paper.view.size.height + (50 * scale))
		)
		waterFlow.lineTo(new paper.Point((150 * scale) + curveDistance, paper.view.size.height + (50 * scale)))
		waterFlow.cubicCurveTo(
			new paper.Point((150 * scale) + curveDistance, endY),
			new paper.Point((150 * scale) + (curveDistance / 2), endY),
			new paper.Point(150 * scale, endY)
		)
		waterFlow.closePath()
		waterFlow.fillColor = waterColor

		// Add labels
		const labels = new paper.Group([
			new paper.PointText({
				point: [20 * scale, 150 * scale],
				content: 'V',
				fillColor: voltageColor,
				fontFamily: 'Segoe UI, sans-serif',
				fontSize: 24,
				fontWeight: 'bold'
			}),
			new paper.PointText({
				point: [160 * scale, 250 * scale],
				content: 'Ω',
				fillColor: resistanceColor,
				fontFamily: 'Segoe UI, sans-serif',
				fontSize: 24,
				fontWeight: 'bold'
			}),
			new paper.PointText({
				point: [(150 * scale) + curveDistance, 320 * scale],
				content: 'A',
				fillColor: currentColor,
				fontFamily: 'Segoe UI, sans-serif',
				fontSize: 24,
				fontWeight: 'bold'
			})
		])

		// Update dash offset for animation
		dashOffset += 2 * (current / 5) // Increased speed
		if (dashOffset > 16) {
			dashOffset = 0
		}

		paper.view.update()
	}

	onMount(() => {
		paper.setup(canvas)

		// Set the canvas to be resizable
		paper.view.autoUpdate = true
		const resizeCanvas = () => {
			paper.view.viewSize = new paper.Size(canvas.offsetWidth, canvas.offsetHeight)
			drawCircuit()
		}
		window.addEventListener('resize', resizeCanvas)
		resizeCanvas()

		// Start animation loop
		paper.view.onFrame = () => {
			drawCircuit()
		}
	})
</script>

<Viewer>
	<canvas bind:this={canvas} slot="canvas" data-paper-resize></canvas>
	<slot name="title" slot="title">Voltage, Resistance, and Current</slot>
	<slot name="body" slot="body">
		<div class="explanation">
			<p>This simulator uses a water analogy to explain electrical concepts:</p>
			<ul>
				<li><strong style="color: #3498db;">Voltage (V)</strong> is like the water pressure in the tank. More voltage means more pressure pushing the water.</li>
				<li><strong style="color: #e74c3c;">Resistance (Ω)</strong> is like the size of the hole the water flows through. Higher resistance means a smaller hole, restricting the flow.</li>
				<li><strong style="color: #2ecc71;">Current (A)</strong> is like the amount of water flowing. It depends on both the pressure (voltage) and the hole size (resistance).</li>
			</ul>
			<p>Adjust the sliders to see how changing voltage and resistance affects the current flow!</p>
		</div>
		<div class="controls">
			<div class="control-group voltage">
				<label for="voltage">
					Voltage (V): <span>{voltage.toFixed(1)}</span>
				</label>
				<input type="range" id="voltage" bind:value={voltage} min="0" max="10" step="0.1">
			</div>
			<div class="control-group resistance">
				<label for="resistance">
					Resistance (Ω): <span>{resistance.toFixed(1)}</span>
				</label>
				<input type="range" id="resistance" bind:value={resistance} min="0.5" max="5" step="0.1">
			</div>
			<div class="control-group current">
				<label for="current">
					Current (A): <span>{current.toFixed(2)}</span>
				</label>
				<div id="currentBar" style="width: {current * 10}%;"></div>
			</div>
		</div>
	</slot>
</Viewer>

<style>
	canvas[data-paper-resize] {
		width: 100%;
		height: 100%;
	}

	.explanation {
		margin-bottom: 20px;
		padding: 15px;
		background-color: rgba(255, 255, 255, 0.1);
		border-radius: 5px;
	}

	.explanation p {
		margin-bottom: 10px;
	}

	.explanation ul {
		padding-left: 20px;
	}

	.explanation li {
		margin-bottom: 5px;
	}

	.controls {
		display: flex;
		flex-direction: column;
		gap: 20px;
	}

	.control-group {
		display: flex;
		flex-direction: column;
		gap: 5px;
	}

	.control-group label {
		font-weight: bold;
	}

	.control-group input[type="range"] {
		width: 100%;
	}

	.voltage label {
		color: #3498db;
	}

	.resistance label {
		color: #e74c3c;
	}

	.current label {
		color: #2ecc71;
	}

	#currentBar {
		height: 20px;
		background-color: #2ecc71;
		border-radius: 10px;
		transition: width 0.3s ease-in-out;
	}
</style>