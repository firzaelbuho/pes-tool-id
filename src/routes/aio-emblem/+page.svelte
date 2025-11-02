<script lang="ts">
  import { onMount, onDestroy } from 'svelte';
  import JSZip from 'jszip';
  import pica from 'pica';

  // --- State Form ---
  let teamId = '';
  let pesVersion: 'PES 2017' | 'PES 2021' = 'PES 2017';
  let file: File | null = null;
  
  // PERUBAHAN 1: Ubah tipe 'img' untuk menerima ImageBitmap.
  // Kita tidak lagi memerlukan HTMLImageElement untuk input.
  let img: ImageBitmap | null = null;
  // 'imgUrl' tidak lagi diperlukan.

  // --- State Kanvas & UI ---
  let previewCanvas: HTMLCanvasElement;
  let previewCtx: CanvasRenderingContext2D | null = null;
  const logicalPreview = 512;
  const displayPreview = 384;

  // --- State Kontrol ---
  let scale = 1;
  let offsetX = 0;
  let offsetY = 0;
  let naturalW = 0;
  let naturalH = 0;

  // --- State Panning ---
  let isPanning = false;
  let lastPanX = 0;
  let lastPanY = 0;

  // --- State Konversi ---
  let working = false;
  let lastZipUrl: string | null = null;

  // --- Instance Pica ---
  let p: ReturnType<typeof pica>;

  onMount(() => {
    p = pica(); // Inisialisasi di sisi client
    previewCtx = previewCanvas.getContext('2d');
    previewCanvas.width = logicalPreview;
    previewCanvas.height = logicalPreview;
    previewCanvas.style.width = displayPreview + 'px';
    previewCanvas.style.height = displayPreview + 'px';
    drawPreview();
  });

  onDestroy(() => {
    // PERUBAHAN 2: Hapus pembersihan 'imgUrl'
    if (lastZipUrl) URL.revokeObjectURL(lastZipUrl);
  });

  // PERUBAHAN 3: Ubah 'handleFileChange' menjadi 'async' dan gunakan 'createImageBitmap'
  async function handleFileChange(e: Event) {
    const input = e.target as HTMLInputElement;
    if (!input.files || input.files.length === 0) {
      file = null;
      img = null; // 'img' sekarang ImageBitmap atau null
      naturalW = 0;
      naturalH = 0;
      return;
    }
    file = input.files[0];

    // Hapus semua logika 'createObjectURL' dan 'new Image()'
    
    try {
      // Ini adalah pengganti utama:
      // Langsung buat ImageBitmap dari file.
      // Ini 100% aman untuk kanvas dan tidak akan 'taint'.
      const bitmap = await createImageBitmap(file);

      naturalW = bitmap.width;
      naturalH = bitmap.height;

      // Auto-fit
      const fitScale = Math.max(logicalPreview / naturalW, logicalPreview / naturalH);
      scale = fitScale;
      offsetX = 0;
      offsetY = 0;

      // Simpan bitmap. drawPreview akan dipicu secara reaktif.
      img = bitmap;

    } catch (err) {
      alert('Gagal memuat file gambar. File mungkin rusak atau tidak didukung.');
      console.error(err);
      file = null;
      img = null;
      naturalW = 0;
      naturalH = 0;
    }
  }

  function drawPreview() {
    if (!previewCtx) return;

    previewCtx.clearRect(0, 0, logicalPreview, logicalPreview);
    drawChecker(previewCtx, logicalPreview, logicalPreview);

    // 'img' sekarang adalah ImageBitmap, 'naturalW'/'naturalH' sudah di-set
    if (!img || !naturalW || !naturalH) return;

    const scaledW = naturalW * scale;
    const scaledH = naturalH * scale;
    const cx = logicalPreview / 2 + offsetX;
    const cy = logicalPreview / 2 + offsetY;
    const drawX = Math.round(cx - scaledW / 2);
    const drawY = Math.round(cy - scaledH / 2);

    // ctx.drawImage() bekerja sempurna dengan ImageBitmap
    previewCtx.drawImage(img, drawX, drawY, scaledW, scaledH);
  }

  function drawChecker(ctx: CanvasRenderingContext2D, w: number, h: number) {
    const size = 16;
    for (let y = 0; y < h; y += size) {
      for (let x = 0; x < w; x += size) {
        const isLight = (x / size + y / size) % 2 === 0;
        ctx.fillStyle = isLight ? '#2a2a2a' : '#1a1a1a';
        ctx.fillRect(x, y, size, size);
      }
    }
  }

  $: if (previewCtx) {
    const _deps = [img, scale, offsetX, offsetY, naturalW, naturalH];
    drawPreview();
  }

  // --- Event Handlers (Tidak Berubah) ---

  function startPan(e: PointerEvent) {
    (e.target as HTMLElement)?.setPointerCapture?.(e.pointerId);
    isPanning = true;
    lastPanX = e.clientX;
    lastPanY = e.clientY;
  }

  function doPan(e: PointerEvent) {
    if (!isPanning) return;
    const dx = e.clientX - lastPanX;
    const dy = e.clientY - lastPanY;
    const ratio = logicalPreview / displayPreview;
    offsetX += dx * ratio;
    offsetY += dy * ratio;
    lastPanX = e.clientX;
    lastPanY = e.clientY;
  }

  function endPan(e: PointerEvent) {
    const target = e.target as HTMLElement;
    if (target?.hasPointerCapture?.(e.pointerId)) {
      target.releasePointerCapture(e.pointerId);
    }
    isPanning = false;
  }

  function onWheel(e: WheelEvent) {
    if (!img) return;
    e.preventDefault();
    const delta = Math.sign(e.deltaY) * -0.08;
    const oldScale = scale;
    const newScale = Math.max(0.05, Math.min(8, scale * (1 + delta)));
    const rect = previewCanvas.getBoundingClientRect();
    const cx = (e.clientX - rect.left) * (logicalPreview / rect.width);
    const cy = (e.clientY - rect.top) * (logicalPreview / rect.height);
    const factor = newScale / oldScale;
    offsetX = (cx - logicalPreview / 2) * (1 - factor) + offsetX * factor;
    offsetY = (cy - logicalPreview / 2) * (1 - factor) + offsetY * factor;
    scale = newScale;
  }

  // --- Logika Konversi (Tidak Berubah) ---

  function normalizedTeamId(raw: string) {
    if (pesVersion === 'PES 2021') {
      const onlyDigits = raw.replace(/\D/g, '');
      return onlyDigits.padStart(6, '0');
    } else {
      return raw;
    }
  }

  async function convertAndDownload() {
    if (!p) {
      alert('Pica library not loaded yet.');
      return;
    }
    if (!img) return alert('Silakan upload gambar emblem terlebih dahulu.');
    if (!teamId.trim()) return alert('Silakan masukkan Team ID.');

    working = true;
    if (lastZipUrl) URL.revokeObjectURL(lastZipUrl);
    lastZipUrl = null;

    const normalized = normalizedTeamId(teamId.trim());
    const rawName = teamId.trim();

    try {
      const canvas512 = document.createElement('canvas');
      canvas512.width = logicalPreview;
      canvas512.height = logicalPreview;
      const ctx512 = canvas512.getContext('2d')!;

      // Gambar komposisi akhir (img adalah ImageBitmap)
      ctx512.clearRect(0, 0, logicalPreview, logicalPreview);
      const scaledW = naturalW * scale;
      const scaledH = naturalH * scale;
      const cx = logicalPreview / 2 + offsetX;
      const cy = logicalPreview / 2 + offsetY;
      const drawX = Math.round(cx - scaledW / 2);
      const drawY = Math.round(cy - scaledH / 2);
      ctx512.drawImage(img!, drawX, drawY, scaledW, scaledH); // Ini sekarang 100% aman

      // Buat kanvas target
      const canvas256 = document.createElement('canvas');
      canvas256.width = 256;
      canvas256.height = 256;
      const canvas128 = document.createElement('canvas');
      canvas128.width = 128;
      canvas128.height = 128;

      // Pica sekarang akan bekerja karena canvas512 tidak 'tainted'
      await p.resize(canvas512, canvas256, { unsharpAmount: 80 });
      await p.resize(canvas256, canvas128, { unsharpAmount: 80 });

      // Konversi kanvas ke blob
      const blob512: Blob = await canvasToBlob(canvas512);
      const blob256: Blob = await canvasToBlob(canvas256);
      const blob128: Blob = await canvasToBlob(canvas128);

      // Siapkan zip
      const zip = new JSZip();
      const name_r_ll = `emblem_${normalized}_r_ll.png`;
      const name_r_l = `emblem_${normalized}_r_l.png`;
      const name_r = `emblem_${normalized}_r.png`;
      zip.file(name_r_ll, blob512);
      zip.file(name_r_l, blob256);
      zip.file(name_r, blob128);

      const zipBlob = await zip.generateAsync({ type: 'blob' });
      const zipName = `${rawName || normalized}.zip`;
      const url = URL.createObjectURL(zipBlob);
      lastZipUrl = url;

      // Picu download
      const a = document.createElement('a');
      a.href = url;
      a.download = zipName;
      document.body.appendChild(a);
      a.click();
      a.remove();
    } catch (err) {
      console.error(err);
      alert('Konversi gagal: ' + (err as Error).message);
    } finally {
      working = false;
    }
  }

  function canvasToBlob(canvas: HTMLCanvasElement): Promise<Blob> {
    return new Promise((resolve, reject) => {
      canvas.toBlob((b) => {
        if (b) resolve(b);
        else reject(new Error('toBlob returned null'));
      }, 'image/png');
    });
  }
</script>

<style>
  /* small helper for slider thumbs in dark */
  input[type='range']::-webkit-slider-thumb { -webkit-appearance: none; }
</style>

<div class="min-h-screen p-6 bg-gray-900 text-gray-100 font-sans">
  <div class="max-w-3xl mx-auto">
    <h1 class="text-2xl font-semibold mb-4">AIO Emblem Converter</h1>

    <div class="bg-gray-800 rounded-xl p-6 shadow-md mb-6">
      <form on:submit|preventDefault={convertAndDownload} class="space-y-4">
        <div>
          <label class="block text-sm">Team ID</label>
          <input
            class="w-full mt-1 bg-gray-900 border border-gray-700 rounded px-3 py-2 text-gray-100"
            bind:value={teamId}
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
          <label class="block text-sm">Emblem Image</label>
          <input class="mt-2" type="file" accept="image/*" on:change={handleFileChange} />
        </div>

        <div class="flex gap-2 items-center">
          <button
            type="submit"
            class="bg-green-600 hover:bg-green-500 px-4 py-2 rounded disabled:opacity-50"
            disabled={working}
          >
            {working ? 'Processing...' : 'Convert & Download ZIP'}
          </button>
        </div>
      </form>
    </div>

    <div class="grid grid-cols-1 gap-6">
      <div class="bg-gray-800 rounded-xl p-4">
        <h2 class="text-md font-medium mb-2">Preview (square)</h2>
        <div class="flex flex-col sm:flex-row gap-4 items-start">
          <canvas
            bind:this={previewCanvas}
            class="rounded border border-gray-700 shadow-inner cursor-grab active:cursor-grabbing"
            on:pointerdown={startPan}
            on:pointermove={doPan}
            on:pointerup={endPan}
            on:pointercancel={endPan}
            on:wheel={onWheel}
          ></canvas>

          <div class="flex-1 w-full">
            <div class="mb-2 text-xs text-gray-400">
              Gunakan slider atau drag untuk menyesuaikan area emblem. Scroll zoom in/out.
            </div>

            <div class="mb-2">
              <label class="text-xs">Scale: {scale.toFixed(2)}</label>
              <input
                type="range"
                min="0.05"
                max="8"
                step="0.01"
                bind:value={scale}
                class="w-full h-2 bg-gray-700 rounded-lg appearance-none cursor-pointer"
              />
            </div>

            <div class="mb-2">
              <label class="text-xs">Offset X: {offsetX.toFixed(0)}px</label>
              <input
                type="range"
                min={-1024}
                max={1024}
                step="1"
                bind:value={offsetX}
                class="w-full"
              />
            </div>

            <div>
              <label class="text-xs">Offset Y: {offsetY.toFixed(0)}px</label>
              <input
                type="range"
                min={-1024}
                max={1024}
                step="1"
                bind:value={offsetY}
                class="w-full"
              />
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</div>