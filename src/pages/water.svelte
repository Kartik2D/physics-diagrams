<script lang="ts">
	import Viewer from '../viewer.svelte'
	import paper from 'paper'
	import { onMount } from 'svelte'

	// Tweakable properties
	const PROPERTIES = {
		SCALE: 1.5,
		TANK_WIDTH: 100,
		TANK_HEIGHT: 200,
		TANK_LEFT: 50,
		TANK_TOP: 50,
		WATER_HEIGHT: 220,
		WATER_TOP: 30,
		TANK_PARTICLE_COUNT: 50,
		TANK_PARTICLE_MIN_VELOCITY: 0.5,
		TANK_PARTICLE_MAX_VELOCITY: 1.5,
		TANK_PARTICLE_MIN_LIFESPAN: 0.5,
		TANK_PARTICLE_MAX_LIFESPAN: 1,
		FLOW_PARTICLE_COUNT: 100,
		FLOW_PARTICLE_MIN_VELOCITY: 2,
		FLOW_PARTICLE_MAX_VELOCITY: 4,
		FLOW_PARTICLE_MIN_LIFESPAN: 0.3,
		FLOW_PARTICLE_MAX_LIFESPAN: 0.8,
		DASH_ARRAY: [8, 8],
		DASH_OFFSET_SPEED: 3,
		DASH_OFFSET_MAX: 16,
		CURVE_DISTANCE_FACTOR: 200,
		TOP_CURVE_EXTENSION_FACTOR: 20,
		RESISTANCE_GAP_FACTOR: 50,
	}

	const COLORS = {
		WATER: new paper.Color('rgba(52, 152, 219, 0.7)'),
		OUTLINE: new paper.Color('rgba(41, 128, 185, 0.7)'),
		VOLTAGE: new paper.Color('#3498db'),
		RESISTANCE: new paper.Color('#e74c3c'),
		CURRENT: new paper.Color('#2ecc71'),
		TANK_OUTLINE: new paper.Color('rgba(128, 128, 128, 0.5)'),
		PARTICLE: new paper.Color(1, 1, 1, 0.5),
	}

	let canvas: HTMLCanvasElement
	let voltage = 5
	let resistance = 2.75
	let current = 1
	let dashOffset = 0
	let tankParticles: paper.Path.Circle[] = []
	let flowParticles: paper.Path.Circle[] = []

	$: {
		// Calculate current based on voltage and resistance
		current = voltage / resistance
		if (paper.project) {
			drawCircuit()
		}
	}

	interface Particle extends paper.Path.Circle {
		data: {
			velocity: number;
			lifespan: number;
			direction?: paper.Point;
		};
	}

	function createParticle(x: number, y: number, scale: number, isFlow: boolean = false): Particle {
		const particle = new paper.Path.Circle({
			center: [x, y],
			radius: 2 * scale,
			fillColor: COLORS.PARTICLE
		}) as Particle;
		
		if (isFlow) {
			particle.data = {
				velocity: Math.random() * (PROPERTIES.FLOW_PARTICLE_MAX_VELOCITY - PROPERTIES.FLOW_PARTICLE_MIN_VELOCITY) + PROPERTIES.FLOW_PARTICLE_MIN_VELOCITY,
				lifespan: Math.random() * (PROPERTIES.FLOW_PARTICLE_MAX_LIFESPAN - PROPERTIES.FLOW_PARTICLE_MIN_LIFESPAN) + PROPERTIES.FLOW_PARTICLE_MIN_LIFESPAN,
				direction: new paper.Point(0, 0)
			};
		} else {
			particle.data = {
				velocity: Math.random() * (PROPERTIES.TANK_PARTICLE_MAX_VELOCITY - PROPERTIES.TANK_PARTICLE_MIN_VELOCITY) + PROPERTIES.TANK_PARTICLE_MIN_VELOCITY,
				lifespan: Math.random() * (PROPERTIES.TANK_PARTICLE_MAX_LIFESPAN - PROPERTIES.TANK_PARTICLE_MIN_LIFESPAN) + PROPERTIES.TANK_PARTICLE_MIN_LIFESPAN
			};
		}
		return particle;
	}

	function updateTankParticles(scale: number): void {
		const tankWidth = PROPERTIES.TANK_WIDTH * scale;
		const tankHeight = PROPERTIES.TANK_HEIGHT * scale;
		const tankLeft = PROPERTIES.TANK_LEFT * scale;
		const tankTop = PROPERTIES.TANK_TOP * scale;

		// Remove dead particles
		tankParticles = tankParticles.filter(p => p.data.lifespan > 0);

		// Update existing particles
		tankParticles.forEach(p => {
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
		while (tankParticles.length < PROPERTIES.TANK_PARTICLE_COUNT) {
			const x = Math.random() * tankWidth + tankLeft;
			const y = Math.random() * tankHeight + tankTop;
			tankParticles.push(createParticle(x, y, scale));
		}
	}

	function updateFlowParticles(scale: number, topCurve: paper.Path, bottomCurve: paper.Path): void {
		// Remove dead particles
		flowParticles = flowParticles.filter(p => p.data.lifespan > 0);

		// Update existing particles
		flowParticles.forEach(p => {
			if (p.data.direction) {
				p.position = p.position.add(p.data.direction.multiply(p.data.velocity * (current / 5)));
			}
			p.data.lifespan -= 0.016 * (current / 5);
			if (p.fillColor) {
				p.fillColor.alpha = Math.min(p.data.lifespan, 1);
			}
		});

		// Add new particles
		while (flowParticles.length < PROPERTIES.FLOW_PARTICLE_COUNT) {
			const t = Math.random();
			const topPoint = topCurve.getPointAt(topCurve.length * t);
			const bottomPoint = bottomCurve.getPointAt(bottomCurve.length * t);
			const x = topPoint.x + Math.random() * (bottomPoint.x - topPoint.x);
			const y = topPoint.y + Math.random() * (bottomPoint.y - topPoint.y);
			const particle = createParticle(x, y, scale, true);
			
			// Calculate direction based on the curve
			const tangent = topCurve.getTangentAt(topCurve.length * t);
			particle.data.direction = tangent.normalize();
			
			flowParticles.push(particle);
		}
	}

	function drawCircuit(): void {
		if (!paper.project) return

		paper.project.activeLayer.removeChildren()

		const scale = PROPERTIES.SCALE

		// Draw tank (voltage) without a roof
		const tank = new paper.Path()
		tank.add(new paper.Point(PROPERTIES.TANK_LEFT * scale, PROPERTIES.TANK_TOP * scale))
		tank.lineTo(new paper.Point(PROPERTIES.TANK_LEFT * scale, (PROPERTIES.TANK_TOP + PROPERTIES.TANK_HEIGHT) * scale))
		tank.lineTo(new paper.Point((PROPERTIES.TANK_LEFT + PROPERTIES.TANK_WIDTH) * scale, (PROPERTIES.TANK_TOP + PROPERTIES.TANK_HEIGHT) * scale))
		tank.lineTo(new paper.Point((PROPERTIES.TANK_LEFT + PROPERTIES.TANK_WIDTH) * scale, PROPERTIES.TANK_TOP * scale))
		tank.strokeColor = COLORS.TANK_OUTLINE
		tank.strokeWidth = 6

		// Draw static water level with gradient
		const waterGradient = {
			gradient: {
				stops: [
					{ offset: 0, color: COLORS.WATER },
					{ offset: 1, color: new paper.Color('rgba(52, 152, 219, 0)') }
				]
			},
			origin: new paper.Point(PROPERTIES.TANK_LEFT * scale, (PROPERTIES.WATER_TOP + PROPERTIES.WATER_HEIGHT) * scale),
			destination: new paper.Point(PROPERTIES.TANK_LEFT * scale, PROPERTIES.WATER_TOP * scale)
		}
		const water = new paper.Path.Rectangle({
			point: [PROPERTIES.TANK_LEFT * scale, PROPERTIES.WATER_TOP * scale],
			size: [PROPERTIES.TANK_WIDTH * scale, PROPERTIES.WATER_HEIGHT * scale],
			fillColor: waterGradient
		})

		// Update and draw tank particles
		updateTankParticles(scale);
		tankParticles.forEach(p => paper.project.activeLayer.addChild(p));

		// Draw resistance (gap between curves at exit)
		const resistanceGap = PROPERTIES.RESISTANCE_GAP_FACTOR * scale * ((resistance - 0.5) / 4.5)
		const startY = ((PROPERTIES.TANK_TOP + PROPERTIES.TANK_HEIGHT) * scale) - resistanceGap
		const endY = (PROPERTIES.TANK_TOP + PROPERTIES.TANK_HEIGHT) * scale

		// Calculate curve distance based on current
		const curveDistance = PROPERTIES.CURVE_DISTANCE_FACTOR * scale * (current / 5)
		const topCurveExtension = PROPERTIES.TOP_CURVE_EXTENSION_FACTOR * scale * current

		// Draw water flow (two Bézier curves)
		const topCurve = new paper.Path()
		topCurve.add(new paper.Point((PROPERTIES.TANK_LEFT + PROPERTIES.TANK_WIDTH) * scale, startY))
		topCurve.cubicCurveTo(
			new paper.Point(((PROPERTIES.TANK_LEFT + PROPERTIES.TANK_WIDTH) * scale) + (curveDistance / 2), startY),
			new paper.Point(((PROPERTIES.TANK_LEFT + PROPERTIES.TANK_WIDTH) * scale) + curveDistance + topCurveExtension, startY),
			new paper.Point(((PROPERTIES.TANK_LEFT + PROPERTIES.TANK_WIDTH) * scale) + curveDistance + topCurveExtension, paper.view.size.height + (50 * scale))
		)

		const bottomCurve = new paper.Path()
		bottomCurve.add(new paper.Point((PROPERTIES.TANK_LEFT + PROPERTIES.TANK_WIDTH) * scale, endY))
		bottomCurve.cubicCurveTo(
			new paper.Point(((PROPERTIES.TANK_LEFT + PROPERTIES.TANK_WIDTH) * scale) + (curveDistance / 2), endY),
			new paper.Point(((PROPERTIES.TANK_LEFT + PROPERTIES.TANK_WIDTH) * scale) + curveDistance, endY),
			new paper.Point(((PROPERTIES.TANK_LEFT + PROPERTIES.TANK_WIDTH) * scale) + curveDistance, paper.view.size.height + (50 * scale))
		)

		const flowOutline = new paper.Group([topCurve, bottomCurve])
		flowOutline.strokeColor = COLORS.OUTLINE
		flowOutline.strokeWidth = 6
		flowOutline.dashArray = PROPERTIES.DASH_ARRAY
		flowOutline.dashOffset = -dashOffset

		// Fill water area
		const waterFlow = new paper.Path()
		waterFlow.add(new paper.Point((PROPERTIES.TANK_LEFT + PROPERTIES.TANK_WIDTH) * scale, startY))
		waterFlow.cubicCurveTo(
			new paper.Point(((PROPERTIES.TANK_LEFT + PROPERTIES.TANK_WIDTH) * scale) + (curveDistance / 2), startY),
			new paper.Point(((PROPERTIES.TANK_LEFT + PROPERTIES.TANK_WIDTH) * scale) + curveDistance + topCurveExtension, startY),
			new paper.Point(((PROPERTIES.TANK_LEFT + PROPERTIES.TANK_WIDTH) * scale) + curveDistance + topCurveExtension, paper.view.size.height + (50 * scale))
		)
		waterFlow.lineTo(new paper.Point(((PROPERTIES.TANK_LEFT + PROPERTIES.TANK_WIDTH) * scale) + curveDistance, paper.view.size.height + (50 * scale)))
		waterFlow.cubicCurveTo(
			new paper.Point(((PROPERTIES.TANK_LEFT + PROPERTIES.TANK_WIDTH) * scale) + curveDistance, endY),
			new paper.Point(((PROPERTIES.TANK_LEFT + PROPERTIES.TANK_WIDTH) * scale) + (curveDistance / 2), endY),
			new paper.Point((PROPERTIES.TANK_LEFT + PROPERTIES.TANK_WIDTH) * scale, endY)
		)
		waterFlow.closePath()
		waterFlow.fillColor = COLORS.WATER

		// Update and draw flow particles
		updateFlowParticles(scale, topCurve, bottomCurve);
		flowParticles.forEach(p => paper.project.activeLayer.addChild(p));

		// Add labels
		const labels = new paper.Group([
			new paper.PointText({
				point: [20 * scale, 150 * scale],
				content: 'V',
				fillColor: COLORS.VOLTAGE,
				fontFamily: 'Segoe UI, sans-serif',
				fontSize: 24,
				fontWeight: 'bold'
			}),
			new paper.PointText({
				point: [160 * scale, 250 * scale],
				content: 'Ω',
				fillColor: COLORS.RESISTANCE,
				fontFamily: 'Segoe UI, sans-serif',
				fontSize: 24,
				fontWeight: 'bold'
			}),
			new paper.PointText({
				point: [((PROPERTIES.TANK_LEFT + PROPERTIES.TANK_WIDTH) * scale) + curveDistance, 320 * scale],
				content: 'A',
				fillColor: COLORS.CURRENT,
				fontFamily: 'Segoe UI, sans-serif',
				fontSize: 24,
				fontWeight: 'bold'
			})
		])

		// Update dash offset for animation
		dashOffset += PROPERTIES.DASH_OFFSET_SPEED * (current / 5)
		if (dashOffset > PROPERTIES.DASH_OFFSET_MAX) {
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