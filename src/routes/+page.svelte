<script lang="ts">
	import { browser } from "$app/environment";
	import { onMount } from "svelte";
	import { Spring } from "svelte/motion";
	import * as Tone from "tone";
	import { Volume, Synth, PolySynth } from "tone";

	let viewPosition = new Spring({ x: 0, y: 0 });

	const scaleX = 100;
	const scaleY = 50;

	let screenW = $state(browser ? window.innerWidth : 1000);
	let screenH = $state(browser ? window.innerHeight : 1000);

	let mapW = $derived(Math.ceil(screenW / scaleX) + 1);
	let mapH = $derived(Math.ceil(screenH / scaleY) + 1);

	let map = $derived([...Array(mapW * mapH)].map((_, i) => {
		let x = i % mapW + Math.floor(viewPosition.current.x / scaleX);
		let y = Math.floor(i / mapW) + Math.floor(viewPosition.current.y / scaleY);
		let note = x - (y * 5);
		let noteMod = ((note % 12) + 12) % 12;

		return {
			x,
			y,
			note,
			noteMod,
		};
	}));

	const labels = [
		"R",
		"2m",
		"2M",
		"3m",
		"3M",
		"4",
		"TT",
		"5",
		"6m",
		"6M",
		"7m",
		"7M",
	];

	const whiteKeys = [0, 2, 4, 5, 7, 9, 11];

	let dragging = $state(false);

	let mainVolume: Volume | null = null;
	let synth1: PolySynth | null = null;
	let synth2: PolySynth | null = null;

	function makeSynth() {
		return new PolySynth(Synth, {
			oscillator: {
				type: "square",
			},
			envelope: {
				attack: 0.02,
				decay: 0.3,
				sustain: 0,
				release: 0.02,
			},
		});
	}

	function playNote(note: number) {
		if (!mainVolume) {
			mainVolume = new Volume(-18).toDestination();
		}
		if (!synth1) {
			synth1 = makeSynth().connect(mainVolume);
		}
		if (!synth2) {
			synth2 = makeSynth().connect(mainVolume);
		}

		note = ((note % 12) + 12) % 12;

		const now = Tone.getContext().currentTime;

		synth1.triggerAttackRelease(128 * Math.pow(2, note / 12), "8n", now, note / 12);
		synth2.triggerAttackRelease(128 * Math.pow(2, (note + 12) / 12), "8n", now, 1 - (note / 12));
	}

	const keyMap = [
		"Digit1", "Digit2", "Digit3", "Digit4", "Digit5", "Digit6", "Digit7", "Digit8", "Digit9", "Digit0",
		"KeyQ", "KeyW", "KeyE", "KeyR", "KeyT", "KeyY", "KeyU", "KeyI", "KeyO", "KeyP",
		"KeyA", "KeyS", "KeyD", "KeyF", "KeyG", "KeyH", "KeyJ", "KeyK", "KeyL", "Semicolon",
		"KeyZ", "KeyX", "KeyC", "KeyV", "KeyB", "KeyN", "KeyM", "Comma", "Period", "Slash",
	];

	const keyMapW = 10;

	let pointerDown = false;
	let pointerLastX = 0;
	let pointerLastY = 0;

	function boardOnWheel(e: WheelEvent) {
		viewPosition.target.x += !e.shiftKey ? e.deltaX : e.deltaY;
		viewPosition.target.y += !e.shiftKey ? e.deltaY : e.deltaX;
		viewPosition.target = viewPosition.target;
	};

	function boardOnPointerdown(e: PointerEvent) {
		pointerDown = true;

		pointerLastX = e.clientX;
		pointerLastY = e.clientY;
	};

	function boardOnPointerup() {
		pointerDown = false;
	};

	function boardOnPointermove(e: PointerEvent) {
		if (pointerDown && dragging) {
			let offsetX = e.clientX - pointerLastX;
			let offsetY = e.clientY - pointerLastY;

			viewPosition.target.x -= offsetX;
			viewPosition.target.y -= offsetY;
			viewPosition.target = viewPosition.target;

			pointerLastX = e.clientX;
			pointerLastY = e.clientY;
		}
	};

	onMount(() => {
		const resizeListener = () => {
			screenW = window.innerWidth;
			screenH = window.innerHeight;
		};

		const keydownListener = (e: KeyboardEvent) => {
			if (e.code === "ArrowLeft") {
				viewPosition.target.x = Math.ceil((viewPosition.target.x - scaleX) / scaleX) * scaleX;
				viewPosition.target = viewPosition.target;
				return;
			}

			if (e.code === "ArrowRight") {
				viewPosition.target.x = Math.floor((viewPosition.target.x + scaleX) / scaleX) * scaleX;
				viewPosition.target = viewPosition.target;
				return;
			}

			if (e.code === "ArrowUp") {
				viewPosition.target.y = Math.ceil((viewPosition.target.y - scaleY) / scaleY) * scaleY;
				viewPosition.target = viewPosition.target;
				return;
			}

			if (e.code === "ArrowDown") {
				viewPosition.target.y = Math.floor((viewPosition.target.y + scaleY) / scaleY) * scaleY;
				viewPosition.target = viewPosition.target;
				return;
			}

			let index = keyMap.findIndex((v) => v === e.code);

			if (index === -1) return;

			const x = index % keyMapW;
			const y = index / keyMapW | 0;

			playNote(map[x + (y * mapW)].note);
		};

		addEventListener("resize", resizeListener);
		addEventListener("keydown", keydownListener);

		return () => {
			removeEventListener("resize", resizeListener);
			removeEventListener("keydown", keydownListener);
		};
	});
</script>

<main class="absolute inset-0 bg-zinc-800 text-white select-none touch-none">
	<div
		class="absolute inset-0 overflow-hidden"
		onpointerdown={boardOnPointerdown}
		onpointermove={boardOnPointermove}
		onpointerup={boardOnPointerup}
		onwheel={boardOnWheel}
	>
		<div
			style:left={`${-viewPosition.current.x}px`}
			style:top={`${-viewPosition.current.y}px`}
			class="absolute"
		>
			{#each map as { x, y, note, noteMod } (`${x}-${y}`)}
				<div
					style:width={`${scaleX}px`}
					style:height={`${scaleY}px`}
					style:left={`${x * scaleX}px`}
					style:top={`${y * scaleY}px`}
					class={[
						"absolute border border-zinc-200/20 flex justify-center items-center hover:bg-zinc-700",
						{
							"bg-zinc-700/30": noteMod === 0,
							"text-white/50": !whiteKeys.includes(noteMod),
						},
					]}
					onpointerdown={() => { if (!dragging) playNote(note); }}
				>
					{labels[noteMod]}
				</div>
			{/each}
		</div>
	</div>
	<div class="absolute left-2 top-2 flex flex-col items-start p-1 bg-black/60 rounded gap-1">
		<button
			class="p-1 cursor-pointer"
			aria-label="toggle dragging"
			onpointerdown={() => dragging = !dragging}
		>
			<svg
				opacity={dragging ? "1" : "0.4"}
				viewBox="0 0 32 32"
				width="24"
			>
				<path
					d="m 16,0 -5,7 h 3 v 7 H 7 v -3 l -7,5 7,5 v -3 h 7 v 7 h -3 l 5,7 5,-7 h -3 v -7 h 7 v 3 l 7,-5 -7,-5 v 3 H 18 V 7 h 3 z"
					fill="#ffffff"
				/>
			</svg>
		</button>
	</div>
</main>
