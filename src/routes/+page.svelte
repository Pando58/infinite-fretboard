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

	let audioCtx: AudioContext | null = null;

	function playNote(note: number) {
		if (!audioCtx) {
			audioCtx = new AudioContext();
		}

		const osc = audioCtx.createOscillator();

		osc.type = "square";
		osc.frequency.setValueAtTime(512 * Math.pow(2, note / 12), audioCtx.currentTime);

		osc.connect(audioCtx.destination);
		osc.start();

		setTimeout(() => osc.disconnect(), 200);

	}

	const keyMap = [
		"Digit1", "Digit2", "Digit3", "Digit4", "Digit5", "Digit6", "Digit7", "Digit8", "Digit9", "Digit0",
		"KeyQ", "KeyW", "KeyE", "KeyR", "KeyT", "KeyY", "KeyU", "KeyI", "KeyO", "KeyP",
		"KeyA", "KeyS", "KeyD", "KeyF", "KeyG", "KeyH", "KeyJ", "KeyK", "KeyL", "Semicolon",
		"KeyZ", "KeyX", "KeyC", "KeyV", "KeyB", "KeyN", "KeyM", "Comma", "Period", "Slash",
	]

	const keyMapW = 10;

	let pointerDown = false;
	let pointerLastX = 0;
	let pointerLastY = 0;

	onMount(() => {

		//
		const wheelListener = (e: WheelEvent) => {
			wheelX += e.deltaX;
			wheelY += e.deltaY;
		}

		const pointerdownListener = (e: PointerEvent) => {
			pointerDown = true;

			pointerLastX = e.clientX;
			pointerLastY = e.clientY;
		}

		const pointerupListener = () => {
			pointerDown = false;
		}

		const pointermoveListener = (e: PointerEvent) => {
			console.log(e.clientX);

			if (pointerDown) {
				let offsetX = e.clientX - pointerLastX;
				let offsetY = e.clientY - pointerLastY;

				wheelX -= offsetX;
				wheelY -= offsetY;

				pointerLastX = e.clientX;
				pointerLastY = e.clientY;
			}
		}

		const resizeListener = () => {
			screenW = window.innerWidth;
			screenH = window.innerHeight;
		}

		const keydownListener = (e: KeyboardEvent) => {
			console.log(e.code)

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

			let index = keyMap.findIndex(v => v === e.code);

			if (index === -1) return;


			const x = index % keyMapW;
			const y = index / keyMapW | 0;

			playNote(map[x + (y * mapW)].note);
		}

		addEventListener("wheel", wheelListener);
		addEventListener("pointerdown", pointerdownListener);
		addEventListener("pointerup", pointerupListener);
		addEventListener("pointermove", pointermoveListener);
		addEventListener("resize", resizeListener);
		addEventListener("keydown", keydownListener);

		return () => {
			removeEventListener("wheel", wheelListener);
			removeEventListener("pointerdown", pointerdownListener);
			removeEventListener("pointerup", pointerupListener);
			removeEventListener("pointermove", pointermoveListener);
			removeEventListener("resize", resizeListener);
			removeEventListener("keydown", keydownListener);
		}
	});
</script>

<main
	class="absolute inset-0 bg-zinc-800 text-white select-none touch-none"
>
	<!-- <div
		class="absolute w-8 h-8 bg-red-400"
		style:left={`${wheelX}px`}
		style:top={`${wheelY}px`}
	></div> -->
	<div class="absolute inset-0 overflow-hidden">
		<div
			class="absolute"
			style:left={`${-wheelX}px`}
			style:top={`${-wheelY}px`}
		>
			{#each map as {x, y, note, noteMod} (`${x}-${y}`)}
				<div
					class={[
						"absolute border border-zinc-200/20 flex justify-center items-center hover:bg-zinc-700",
						{
							"bg-zinc-700/30": noteMod === 0,
							"text-white/50": !whiteKeys.includes(noteMod),
						},
					]}
					style:width={`${scaleX}px`}
					style:height={`${scaleY}px`}
					style:left={`${x * scaleX}px`}
					style:top={`${y * scaleY}px`}
					onpointerdown={() => playNote(note)}
				>
					{labels[noteMod]}
				</div>
			{/each}
		</div>
	</div>
	<div class="absolute left-2 top-2 bg-black/60 px-1.5 py-1 rounded-md">
		<div>x: {wheelX | 0}</div>
		<div>y: {wheelY | 0}</div>
		<!-- <div>{map}</div> -->
	</div>
</main>
