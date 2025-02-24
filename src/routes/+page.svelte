<script lang="ts">
	import { browser } from "$app/environment";
	import { onMount } from "svelte";

	let wheelX = $state(0);
	let wheelY = $state(0);

	const scaleX = 100;
	const scaleY = 50;

	let screenW = $state(browser ? window.innerWidth : 1000);
	let screenH = $state(browser ? window.innerHeight : 1000);

	let mapW = $derived(Math.ceil(screenW / scaleX) + 1);
	let mapH = $derived(Math.ceil(screenH / scaleY) + 1);

	let map = $derived([...Array(mapW * mapH)].map((_, i) => {
		let x = i % mapW + Math.floor(wheelX / scaleX);
		let y = Math.floor(i / mapW) + Math.floor(wheelY / scaleY);
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

	let audioCtx: AudioContext | null = null;

	function playNote(note: number) {
		if (!audioCtx) {
			audioCtx = new AudioContext();
		}

		const shepardPhase = ((note % 12) + 12) % 12;

		const osc1 = audioCtx.createOscillator();
		const osc2 = audioCtx.createOscillator();
		const gain1 = audioCtx.createGain();
		const gain2 = audioCtx.createGain();

		osc1.type = "square";
		osc2.type = "square";
		osc1.frequency.setValueAtTime(256 * Math.pow(2, (((note % 12) + 12) % 12) / 12), audioCtx.currentTime);
		osc2.frequency.setValueAtTime(512 * Math.pow(2, (((note % 12) + 12) % 12) / 12), audioCtx.currentTime);
		gain1.gain.value = shepardPhase / 12;
		gain2.gain.value = 1 - (shepardPhase / 12);

		osc1.connect(gain1);
		osc2.connect(gain2);
		gain1.connect(audioCtx.destination);
		gain2.connect(audioCtx.destination);

		osc1.start();
		osc2.start();

		setTimeout(() => {
			osc1.disconnect();
			osc2.disconnect();
		}, 200);
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
		wheelX += !e.shiftKey ? e.deltaX : e.deltaY;
		wheelY += !e.shiftKey ? e.deltaY : e.deltaX;
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

			wheelX -= offsetX;
			wheelY -= offsetY;

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
				wheelX = Math.ceil((wheelX - scaleX) / scaleX) * scaleX;
				return;
			}

			if (e.code === "ArrowRight") {
				wheelX = Math.floor((wheelX + scaleX) / scaleX) * scaleX;
				return;
			}

			if (e.code === "ArrowUp") {
				wheelY = Math.ceil((wheelY - scaleY) / scaleY) * scaleY;
				return;
			}

			if (e.code === "ArrowDown") {
				wheelY = Math.floor((wheelY + scaleY) / scaleY) * scaleY;
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
			style:left={`${-wheelX}px`}
			style:top={`${-wheelY}px`}
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
