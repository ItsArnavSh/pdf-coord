<script lang="ts">
	import { onMount } from 'svelte';
	import { Upload, Plus, X, ChevronLeft, ChevronRight, Download } from 'lucide-svelte';

	let pdfFile: File | null = null;
	let labels: string[] = [];
	let newLabel = '';
	let currentPage = 1;
	let totalPages = 0;
	let annotations: { page: number; x: number; y: number; label: string }[] = [];
	let hoverCoords: { x: number; y: number } | null = null;
	let selectedLabel = '';
	let lastClick: { page: number; x: number; y: number } | null = null;

	let canvas: HTMLCanvasElement;
	let pdfDoc: any = null;
	let currentScale = 1.5;
	let pageHeight = 0;

	// Load PDF.js from CDN
	onMount(async () => {
		const script = document.createElement('script');
		script.src = 'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.min.js';
		script.onload = () => {
			window.pdfjsLib.GlobalWorkerOptions.workerSrc =
				'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.worker.min.js';
		};
		document.head.appendChild(script);
	});

	function handleFileUpload(event: Event) {
		const input = event.target as HTMLInputElement;
		const file = input.files?.[0];
		if (file && file.type === 'application/pdf') {
			pdfFile = file;
			loadPDF();
		}
	}

	async function loadPDF() {
		if (!pdfFile || !window.pdfjsLib) return;

		const arrayBuffer = await pdfFile.arrayBuffer();
		const pdf = await window.pdfjsLib.getDocument(arrayBuffer).promise;
		pdfDoc = pdf;
		totalPages = pdf.numPages;
		currentPage = 1;
		annotations = [];
		renderPage(currentPage);
	}

	async function renderPage(pageNum: number) {
		if (!pdfDoc || !canvas) return;

		const page = await pdfDoc.getPage(pageNum);
		const viewport = page.getViewport({ scale: currentScale });

		canvas.width = viewport.width;
		canvas.height = viewport.height;
		pageHeight = viewport.height;

		const context = canvas.getContext('2d');
		if (!context) return;

		const renderContext = {
			canvasContext: context,
			viewport: viewport
		};

		await page.render(renderContext).promise;
	}

	function goToPrevPage() {
		if (currentPage > 1) {
			currentPage -= 1;
			renderPage(currentPage);
		}
	}

	function goToNextPage() {
		if (currentPage < totalPages) {
			currentPage += 1;
			renderPage(currentPage);
		}
	}

	function handleCanvasMouseMove(e: MouseEvent) {
		if (!canvas) return;
		const rect = canvas.getBoundingClientRect();

		// Get coordinates in canvas space (accounting for any CSS scaling)
		const scaleX = canvas.width / rect.width;
		const scaleY = canvas.height / rect.height;

		const x = Math.round((e.clientX - rect.left) * scaleX);
		const y = Math.round((e.clientY - rect.top) * scaleY);

		// Convert to PDF coordinate system (origin at bottom-left)
		const pdfX = x / currentScale;
		const pdfY = (pageHeight - y) / currentScale;

		hoverCoords = { x: Math.round(pdfX), y: Math.round(pdfY) };
	}

	function handleCanvasClick(e: MouseEvent) {
		if (!canvas) return;
		const rect = canvas.getBoundingClientRect();

		// Get coordinates in canvas space (accounting for any CSS scaling)
		const scaleX = canvas.width / rect.width;
		const scaleY = canvas.height / rect.height;

		const x = Math.round((e.clientX - rect.left) * scaleX);
		const y = Math.round((e.clientY - rect.top) * scaleY);

		// Convert to PDF coordinate system (origin at bottom-left)
		const pdfX = x / currentScale;
		const pdfY = (pageHeight - y) / currentScale;

		lastClick = {
			page: currentPage,
			x: Math.round(pdfX),
			y: Math.round(pdfY)
		};
		selectedLabel = '';
	}

	function addLabel() {
		if (newLabel.trim() && !labels.includes(newLabel.trim())) {
			labels = [...labels, newLabel.trim()];
			newLabel = '';
		}
	}

	function removeLabel(label: string) {
		labels = labels.filter((l) => l !== label);
		// Remove annotations with this label
		annotations = annotations.filter((a) => a.label !== label);
	}

	function saveAnnotation() {
		if (lastClick && selectedLabel) {
			annotations = [...annotations, { ...lastClick, label: selectedLabel }];
			lastClick = null;
			selectedLabel = '';
		}
	}

	function exportJSON() {
		const exportData = {
			pdf_file: pdfFile?.name || 'unknown.pdf',
			scale_used: currentScale,
			annotations: annotations
		};
		const json = JSON.stringify(exportData, null, 2);
		const blob = new Blob([json], { type: 'application/json' });
		const url = URL.createObjectURL(blob);
		const a = document.createElement('a');
		a.href = url;
		a.download = 'annotations.json';
		a.click();
		URL.revokeObjectURL(url);
	}
</script>

<main class="mx-auto max-w-6xl p-6 font-sans">
	<h1 class="mb-8 text-center text-3xl font-bold">PDF Coordinate Annotator</h1>

	<!-- Upload Section -->
	{#if !pdfFile}
		<div class="py-16 text-center">
			<label
				class="inline-flex cursor-pointer items-center gap-3 rounded-lg bg-blue-600 px-6 py-4 text-white transition hover:bg-blue-700"
			>
				<Upload size={24} />
				<span class="text-lg">Upload PDF</span>
				<input type="file" accept="application/pdf" on:change={handleFileUpload} class="hidden" />
			</label>
			<p class="mt-4 text-sm text-gray-600">
				Coordinates use PDF coordinate system (origin at bottom-left)
			</p>
		</div>
	{:else}
		<!-- Labels Management -->
		<section class="mb-8 rounded-lg bg-gray-100 p-6">
			<h2 class="mb-4 text-xl font-semibold">Labels</h2>
			<div class="mb-4 flex gap-3">
				<input
					type="text"
					bind:value={newLabel}
					on:keypress={(e) => e.key === 'Enter' && addLabel()}
					placeholder="Enter label name"
					class="flex-1 rounded-lg border border-gray-300 px-4 py-2 focus:ring-2 focus:ring-blue-500 focus:outline-none"
				/>
				<button
					on:click={addLabel}
					class="flex items-center gap-2 rounded-lg bg-green-600 px-4 py-2 text-white hover:bg-green-700"
				>
					<Plus size={20} />
					Add
				</button>
			</div>
			<div class="flex flex-wrap gap-2">
				{#each labels as label}
					<span
						class="inline-flex items-center gap-2 rounded-full bg-blue-100 px-3 py-1 text-blue-800"
					>
						{label}
						<button on:click={() => removeLabel(label)} class="hover:text-red-600">
							<X size={16} />
						</button>
					</span>
				{/each}
			</div>
		</section>

		<!-- PDF Viewer -->
		<section class="mb-8 rounded-lg bg-gray-900 p-6">
			<div class="mb-4 flex items-center justify-between">
				<h2 class="text-xl font-semibold text-white">PDF Viewer</h2>
				<div class="flex items-center gap-4">
					<button
						on:click={goToPrevPage}
						disabled={currentPage === 1}
						class="rounded-lg bg-gray-700 p-2 text-white hover:bg-gray-600 disabled:opacity-50"
					>
						<ChevronLeft size={24} />
					</button>
					<span class="text-white">
						Page {currentPage} of {totalPages}
					</span>
					<button
						on:click={goToNextPage}
						disabled={currentPage === totalPages}
						class="rounded-lg bg-gray-700 p-2 text-white hover:bg-gray-600 disabled:opacity-50"
					>
						<ChevronRight size={24} />
					</button>
				</div>
			</div>

			<div class="overflow-hidden rounded-lg bg-white shadow-2xl">
				<canvas
					bind:this={canvas}
					on:mousemove={handleCanvasMouseMove}
					on:click={handleCanvasClick}
					on:mouseleave={() => (hoverCoords = null)}
					class="mx-auto block cursor-crosshair"
				></canvas>
			</div>

			{#if hoverCoords}
				<p class="mt-3 text-center text-gray-300">
					PDF Coordinates: x={hoverCoords.x}, y={hoverCoords.y} (origin: bottom-left)
				</p>
			{/if}
		</section>

		<!-- Pending Annotation -->
		{#if lastClick}
			<section class="mb-8 rounded-lg border border-amber-300 bg-amber-50 p-6">
				<h3 class="mb-3 text-lg font-semibold">Add Annotation</h3>
				<p class="mb-3">
					Page {lastClick.page}, x={lastClick.x}, y={lastClick.y}
				</p>
				<div class="flex items-end gap-3">
					<div class="flex-1">
						<label class="mb-1 block text-sm font-medium">Select label</label>
						<select
							bind:value={selectedLabel}
							class="w-full rounded-lg border border-gray-300 px-4 py-2 focus:ring-2 focus:ring-blue-500 focus:outline-none"
						>
							<option value="">-- Choose a label --</option>
							{#each labels as label}
								<option value={label}>{label}</option>
							{/each}
						</select>
					</div>
					<button
						on:click={saveAnnotation}
						disabled={!selectedLabel}
						class="rounded-lg bg-blue-600 px-6 py-2 text-white hover:bg-blue-700 disabled:opacity-50"
					>
						Save
					</button>
				</div>
			</section>
		{/if}

		<!-- Annotations List -->
		{#if annotations.length > 0}
			<section class="rounded-lg bg-gray-100 p-6">
				<div class="mb-4 flex items-center justify-between">
					<h2 class="text-xl font-semibold">Annotations ({annotations.length})</h2>
					<button
						on:click={exportJSON}
						class="flex items-center gap-2 rounded-lg bg-green-600 px-4 py-2 text-white hover:bg-green-700"
					>
						<Download size={20} />
						Export JSON
					</button>
				</div>
				<ul class="space-y-2">
					{#each annotations as ann}
						<li class="rounded border bg-white p-3">
							Page {ann.page}: ({ann.x}, {ann.y}) — <strong>{ann.label}</strong>
						</li>
					{/each}
				</ul>
			</section>
		{/if}
	{/if}
</main>

<style>
	:global(body) {
		background-color: #f3f4f6;
	}
</style>
