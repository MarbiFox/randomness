<script lang="ts">
	import { onMount } from 'svelte';
	import { fly, fade } from 'svelte/transition';

	// ~300 stars with subtle color variation (mostly white, some cool tones)
	const STAR_COUNT = 300;
	const COLORS = ['#ffffff', '#ffffff', '#ffffff', '#ffffff', '#a0b8ff', '#c8b8ff'];

	const DAMPING = 0.92;
	const SCROLL_SENS = 0.0003;
	const Z_VELOCITY_CLAMP = 0.04;
	const STREAK_VELOCITY_THRESHOLD = 0.0005;
	const Z_RECYCLE_FRONT = 0.01;
	const Z_MIN_SPAWN = 0.05;
	const Z_MAX_SPAWN = 0.15;

	interface Star {
		/** -1..1 relative to center */
		x: number;
		/** -1..1 relative to center */
		y: number;
		/** Depth 0..1 (closer = smaller z visually in projection = star.z in denominator) */
		z: number;
		prevScreenX: number;
		prevScreenY: number;
		color: string;
		phase: number;
		speed: number;
	}

	let canvas: HTMLCanvasElement;
	let animationId: number;
	let stars: Star[] = [];
	let zVelocity = 0;

	function randomStarPosition(): { x: number; y: number } {
		return {
			x: Math.random() * 2 - 1,
			y: Math.random() * 2 - 1
		};
	}

	/** Each star: centered x/y, depth z, previous screen pos for motion streaks. */
	function createStars(): Star[] {
		return Array.from({ length: STAR_COUNT }, () => {
			const { x, y } = randomStarPosition();
			return {
				x,
				y,
				z: 0.05 + Math.random() * 0.95,
				prevScreenX: Number.NaN,
				prevScreenY: Number.NaN,
				color: COLORS[Math.floor(Math.random() * COLORS.length)],
				phase: Math.random() * Math.PI * 2,
				speed: 0.4 + Math.random() * 1.6
			};
		});
	}

	function handleWheel(e: WheelEvent) {
		zVelocity += e.deltaY * SCROLL_SENS;
		zVelocity = Math.max(-Z_VELOCITY_CLAMP, Math.min(Z_VELOCITY_CLAMP, zVelocity));
	}

	/** Match canvas resolution to viewport, accounting for device pixel ratio. */
	function resizeCanvas() {
		if (!canvas) return;

		const dpr = window.devicePixelRatio || 1;
		const { innerWidth: w, innerHeight: h } = window;

		canvas.width = w * dpr;
		canvas.height = h * dpr;
		canvas.style.width = `${w}px`;
		canvas.style.height = `${h}px`;

		const ctx = canvas.getContext('2d');
		if (ctx) ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
	}

	/** Perspective projection from center; rAF loop with warp + optional streaks. */
	function drawStars(timestamp: number) {
		const ctx = canvas.getContext('2d');
		if (!ctx) return;

		const w = window.innerWidth;
		const h = window.innerHeight;
		const cx = w / 2;
		const cy = h / 2;
		const focal = Math.min(w, h) * 0.45;

		ctx.clearRect(0, 0, w, h);

		zVelocity *= DAMPING;

		const t = timestamp * 0.001;
		const absVel = Math.abs(zVelocity);

		for (const star of stars) {
			star.z -= zVelocity;

			if (star.z <= Z_RECYCLE_FRONT) {
				const pos = randomStarPosition();
				star.x = pos.x;
				star.y = pos.y;
				star.z = 1;
				star.prevScreenX = Number.NaN;
				star.prevScreenY = Number.NaN;
			} else if (star.z > 1) {
				const pos = randomStarPosition();
				star.x = pos.x;
				star.y = pos.y;
				star.z = Z_MIN_SPAWN + Math.random() * (Z_MAX_SPAWN - Z_MIN_SPAWN);
				star.prevScreenX = Number.NaN;
				star.prevScreenY = Number.NaN;
			}

			const safeZ = Math.max(star.z, 0.02);
			const screenX = (star.x / safeZ) * focal + cx;
			const screenY = (star.y / safeZ) * focal + cy;

			const depthFactor = 1 - star.z;
			const twinkle = (Math.sin(t * star.speed + star.phase) + 1) / 2;
			const baseOpacity = depthFactor * 0.9;
			const opacity = baseOpacity * (0.75 + 0.25 * twinkle);
			const radius = Math.max(0.3, 2.5 * depthFactor);

			if (!Number.isNaN(star.prevScreenX) && absVel > STREAK_VELOCITY_THRESHOLD) {
				const streakAlpha = Math.min(1, absVel * 80) * depthFactor * 0.85;
				const lineW = Math.max(0.5, 2.2 * depthFactor * Math.min(1, absVel * 50));
				ctx.beginPath();
				ctx.moveTo(star.prevScreenX, star.prevScreenY);
				ctx.lineTo(screenX, screenY);
				ctx.strokeStyle = star.color;
				ctx.globalAlpha = streakAlpha;
				ctx.lineWidth = lineW;
				ctx.lineCap = 'round';
				ctx.stroke();
			}

			ctx.beginPath();
			ctx.arc(screenX, screenY, radius, 0, Math.PI * 2);
			ctx.fillStyle = star.color;
			ctx.globalAlpha = opacity;
			ctx.fill();

			star.prevScreenX = screenX;
			star.prevScreenY = screenY;
		}

		ctx.globalAlpha = 1;
		animationId = requestAnimationFrame(drawStars);
	}

	function handleResize() {
		resizeCanvas();
	}

	onMount(() => {
		stars = createStars();
		resizeCanvas();
		animationId = requestAnimationFrame(drawStars);
		window.addEventListener('resize', handleResize);
		window.addEventListener('wheel', handleWheel, { passive: true });

		return () => {
			cancelAnimationFrame(animationId);
			window.removeEventListener('resize', handleResize);
			window.removeEventListener('wheel', handleWheel);
		};
	});
</script>

<main class="hero">
	<canvas bind:this={canvas} class="starfield" aria-hidden="true"></canvas>

	<div class="vignette" aria-hidden="true"></div>

	<div class="content">
		<h1 in:fly={{ y: 40, duration: 900 }}>COSMOS</h1>

		<p in:fade={{ duration: 800, delay: 800 }}>
			The universe is not only stranger than we imagine — it is stranger than we can imagine.
		</p>

		<div class="scroll-hint">
			<div class="scroll-line" aria-hidden="true"></div>
			<span class="scroll-label">SCROLL</span>
		</div>
	</div>
</main>

<style>
	:global(*) {
		margin: 0;
		padding: 0;
		box-sizing: border-box;
	}

	:global(html),
	:global(body) {
		overflow: hidden;
		background: #000008;
		font-family: 'Orbitron', sans-serif;
	}

	.hero {
		position: relative;
		display: flex;
		align-items: center;
		justify-content: center;
		width: 100vw;
		height: 100svh;
		background: #000008;
		overflow: hidden;
	}

	/* Full-viewport star canvas behind all content */
	.starfield {
		position: absolute;
		inset: 0;
		z-index: 0;
		display: block;
	}

	/* Subtle edge darkening for depth */
	.vignette {
		position: absolute;
		inset: 0;
		z-index: 0;
		pointer-events: none;
		background: radial-gradient(ellipse at center, transparent 40%, rgba(0, 0, 4, 0.7) 100%);
	}

	.content {
		position: relative;
		z-index: 1;
		display: flex;
		flex-direction: column;
		align-items: center;
		text-align: center;
		padding: 0 1.5rem;
	}

	h1 {
		font-family: 'Orbitron', sans-serif;
		font-weight: 900;
		font-size: clamp(3rem, 10vw, 8rem);
		letter-spacing: 0.2em;
		background: linear-gradient(45deg, #ae9bfb, #2575fc);
		-webkit-background-clip: text;
		background-clip: text;
		-webkit-text-fill-color: transparent;
		animation: float 6s ease-in-out infinite;
	}

	@keyframes float {
		0%,
		100% {
			transform: translateY(-14px);
		}
		50% {
			transform: translateY(14px);
		}
	}

	p {
		margin-top: 1.75rem;
		max-width: 36rem;
		font-family: 'Orbitron', sans-serif;
		font-weight: 400;
		font-size: clamp(0.65rem, 1.5vw, 0.9rem);
		letter-spacing: 0.25em;
		line-height: 1.8;
		color: #4d6080;
	}

	.scroll-hint {
		display: flex;
		flex-direction: column;
		align-items: center;
		margin-top: 2.5rem;
	}

	.scroll-line {
		width: 1px;
		height: 28px;
		background: linear-gradient(to bottom, #ae9bfb, transparent);
		animation: pulse 2.4s ease-in-out infinite;
	}

	@keyframes pulse {
		0%,
		100% {
			opacity: 0.2;
		}
		50% {
			opacity: 1;
		}
	}

	.scroll-label {
		margin-top: 0.6rem;
		font-family: 'Orbitron', sans-serif;
		font-size: 9px;
		font-weight: 400;
		letter-spacing: 0.4em;
		color: #ae9bfb33;
	}
</style>
