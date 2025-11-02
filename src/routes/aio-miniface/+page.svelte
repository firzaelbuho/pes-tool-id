<script lang="ts">
	import { onMount } from 'svelte';

	let playerId = '';
	let pesVersion: 'PES 2017' | 'PES 2021' = 'PES 2017';
	let file: File | null = null;
	let img: HTMLImageElement | null = null;
	let previewCanvas: HTMLCanvasElement;
	let previewCtx: CanvasRenderingContext2D | null = null;

	const logicalSize = 128;
	const displaySize = 256;

	let scale = 1;
	let offsetX = 0;
	let offsetY = 0;

	let isPanning = false;
	let lastPanX = 0;
	let lastPanY = 0;

	let naturalW = 0;
	let naturalH = 0;
	let downloadReady = false;
	let lastDDSBlob: Blob | null = null;

	onMount(() => {
		previewCtx = previewCanvas.getContext('2d');
		drawPreview();
	});

	function handleFileChange(e: Event) {
		const input = e.target as HTMLInputElement;
		if (!input.files || input.files.length === 0) return;
		file = input.files[0];

		downloadReady = false;
		lastDDSBlob = null;

		const url = URL.createObjectURL(file);
		img = new Image();
		img.onload = () => {
			naturalW = img!.naturalWidth;
			naturalH = img!.naturalHeight;

			// auto-fit
			scale = Math.max(logicalSize / naturalW, logicalSize / naturalH);
			offsetX = 0;
			offsetY = 0;

			drawPreview();
			URL.revokeObjectURL(url);
		};
		img.src = url;
	}

	function drawPreview() {
		if (!previewCtx) return;

		previewCanvas.width = logicalSize;
		previewCanvas.height = logicalSize;
		previewCanvas.style.width = displaySize + 'px';
		previewCanvas.style.height = displaySize + 'px';

		previewCtx.clearRect(0, 0, logicalSize, logicalSize);
		drawChecker(previewCtx, logicalSize, logicalSize);

		if (!img) return;

		const scaledW = naturalW * scale;
		const scaledH = naturalH * scale;

		const cx = logicalSize / 2 + offsetX;
		const cy = logicalSize / 2 + offsetY;

		const drawX = Math.round(cx - scaledW / 2);
		const drawY = Math.round(cy - scaledH / 2);

		previewCtx.drawImage(img, drawX, drawY, scaledW, scaledH);
	}

	function drawChecker(ctx: CanvasRenderingContext2D, w: number, h: number) {
		const size = 8;
		for (let y = 0; y < h; y += size) {
			for (let x = 0; x < w; x += size) {
				const isLight = (x / size + y / size) % 2 === 0;
				ctx.fillStyle = isLight ? '#2a2a2a' : '#1a1a1a';
				ctx.fillRect(x, y, size, size);
			}
		}
	}

	// Auto redraw
	// resize preview when controls change
	$: {
		// Dengan menyebutkan variabel ini di dalam blok reaktif,
		// Svelte akan memantau perubahan mereka.
		scale;
		offsetX;
		offsetY;
		img; // Kita juga perlu menggambar ulang jika gambar berubah

		// Panggil drawPreview() HANYA jika canvas sudah siap
		if (previewCtx) {
			drawPreview();
		}
	}

	async function convertAndPrepareDownload() {
		if (!previewCanvas) return;

		const off = document.createElement('canvas');
		off.width = logicalSize;
		off.height = logicalSize;

		const ctx = off.getContext('2d');
		if (!ctx) return;

		ctx.clearRect(0, 0, logicalSize, logicalSize);

		if (img) {
			const scaledW = naturalW * scale;
			const scaledH = naturalH * scale;
			const cx = logicalSize / 2 + offsetX;
			const cy = logicalSize / 2 + offsetY;
			const drawX = Math.round(cx - scaledW / 2);
			const drawY = Math.round(cy - scaledH / 2);
			ctx.drawImage(img, drawX, drawY, scaledW, scaledH);
		}

		const imageData = ctx.getImageData(0, 0, logicalSize, logicalSize);
		const ddsBlob = createUncompressedDDS(logicalSize, logicalSize, imageData.data);
		lastDDSBlob = ddsBlob;
		downloadReady = true;
	}

	function triggerDownload() {
		if (!lastDDSBlob) return;
		const name =
			pesVersion === 'PES 2017'
				? `player_${playerId || 'unknown'}.dds`
				: `${playerId || 'unknown'}.dds`;

		const url = URL.createObjectURL(lastDDSBlob);
		const a = document.createElement('a');
		a.href = url;
		a.download = name;
		document.body.appendChild(a);
		a.click();
		a.remove();
		URL.revokeObjectURL(url);
	}

	// Mouse pan handlers
	function startPan(e: MouseEvent) {
		isPanning = true;
		lastPanX = e.clientX;
		lastPanY = e.clientY;
	}

	function doPan(e: MouseEvent) {
		if (!isPanning) return;
		const dx = e.clientX - lastPanX;
		const dy = e.clientY - lastPanY;

		const panRatio = logicalSize / displaySize;
		offsetX += dx * panRatio;
		offsetY += dy * panRatio;

		lastPanX = e.clientX;
		lastPanY = e.clientY;
	}

	function endPan() {
		isPanning = false;
	}

	// === DDS Builder (no change) ===
	function createUncompressedDDS(width: number, height: number, rgba: Uint8ClampedArray) {
		const header = new ArrayBuffer(128);
		const dv = new DataView(header);

		dv.setUint8(0, 0x44);
		dv.setUint8(1, 0x44);
		dv.setUint8(2, 0x53);
		dv.setUint8(3, 0x20);

		dv.setUint32(4, 124, true);
		dv.setUint32(8, 0x00021007, true);
		dv.setUint32(12, height, true);
		dv.setUint32(16, width, true);
		dv.setUint32(20, width * 4, true);

		dv.setUint32(76, 32, true);
		dv.setUint32(80, 0x00000041, true);
		dv.setUint32(84, 0, true);
		dv.setUint32(88, 32, true);
		dv.setUint32(92, 0x00ff0000, true);
		dv.setUint32(96, 0x0000ff00, true);
		dv.setUint32(100, 0x000000ff, true);
		dv.setUint32(104, 0xff000000, true);

		dv.setUint32(108, 0x00001000, true);

		const pixelData = new Uint8Array(width * height * 4);

		for (let i = 0; i < width * height; i++) {
			const r = rgba[i * 4];
			const g = rgba[i * 4 + 1];
			const b = rgba[i * 4 + 2];
			const a = rgba[i * 4 + 3];
			pixelData[i * 4] = b;
			pixelData[i * 4 + 1] = g;
			pixelData[i * 4 + 2] = r;
			pixelData[i * 4 + 3] = a;
		}

		const out = new Uint8Array(header.byteLength + pixelData.length);
		out.set(new Uint8Array(header), 0);
		out.set(pixelData, header.byteLength);
		return new Blob([out], { type: 'application/octet-stream' });
	}
</script>

<div class="min-h-screen p-6 bg-gray-900 text-gray-100 font-sans">
	<div class="max-w-3xl mx-auto">
		<h1 class="text-2xl font-semibold mb-4">AIO Miniface Converter</h1>
		<div class="bg-gray-800 rounded-xl p-6 shadow-md">
			<form on:submit|preventDefault={convertAndPrepareDownload} class="space-y-4">
				<div>
					<label class="block text-sm">Player ID</label>
					<input
						class="w-full mt-1 bg-gray-900 border border-gray-700 rounded px-3 py-2 text-gray-100"
						bind:value={playerId}
						placeholder="e.g. 12345"
					/>
				</div>

				<div>
					<label class="block text-sm">PES Version</label>
					<select
						class="w-full mt-1 bg-gray-900 border border-gray-700 rounded px-3 py-2"
						bind:value={pesVersion}
					>
						<option>PES 2017</option>
						<option>PES 2021</option>
					</select>
				</div>

				<div>
					<label class="block text-sm">Face Image</label>
					<input class="mt-2" type="file" accept="image/*" on:change={handleFileChange} />
				</div>

				<div class="flex gap-2 items-center">
					<button type="submit" class="bg-green-600 hover:bg-green-500 px-4 py-2 rounded">
						Convert
					</button>
					{#if downloadReady}
						<button
							type="button"
							class="bg-blue-600 hover:bg-blue-500 px-4 py-2 rounded"
							on:click={triggerDownload}
						>
							Download Miniface
						</button>
					{/if}
				</div>
			</form>
		</div>

		<div class="mt-6">
			<div class="bg-gray-800 rounded-xl p-4">
				<h2 class="text-md font-medium mb-2">Preview (128×128)</h2>
				<div class="flex gap-4 items-start">
					<canvas
						bind:this={previewCanvas}
						class="rounded border border-gray-700 shadow-inner cursor-grab active:cursor-grabbing"
						on:mousedown={startPan}
						on:mousemove={doPan}
						on:mouseup={endPan}
						on:mouseleave={endPan}
					/>

					<div class="flex-1">
						<div class="mb-2 text-xs text-gray-400">Zoom dan geser dalam area 128x128.</div>

						<div class="mb-2">
							<label class="text-xs">Scale: {scale.toFixed(2)}</label>
							<input
								type="range"
								min="0.1"
								max="3"
								step="0.01"
								bind:value={scale}
								class="w-full"
							/>
						</div>

						<div class="mb-2">
							<label class="text-xs">Offset X: {offsetX.toFixed(0)}px</label>
							<input type="range" min="-200" max="200" step="1" bind:value={offsetX} class="w-full" />
						</div>

						<div>
							<label class="text-xs">Offset Y: {offsetY.toFixed(0)}px</label>
							<input type="range" min="-200" max="200" step="1" bind:value={offsetY} class="w-full" />
						</div>
					</div>
				</div>
			</div>

			</div>
	</div>
</div>