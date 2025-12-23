<script lang="ts">
	import { FFmpeg } from '@ffmpeg/ffmpeg';
	// @ts-ignore
	import type { LogEvent, ProgressEvent } from '@ffmpeg/ffmpeg/dist/esm/types';
	import { fetchFile, toBlobURL } from '@ffmpeg/util';
	import { onMount } from 'svelte';
	import * as Card from '$lib/components/ui/card/index.js';
	import * as Select from '$lib/components/ui/select/index.js';
	import { Button } from '$lib/components/ui/button/index.js';
	import { Progress } from '$lib/components/ui/progress/index.js';
	import { Badge } from '$lib/components/ui/badge/index.js';
	import { Label } from '$lib/components/ui/label/index.js';
	import { Input } from '$lib/components/ui/input/index.js';
	import * as Alert from '$lib/components/ui/alert/index.js';
	import Loader from '@lucide/svelte/icons/loader-circle';
	import ThemeSelector from '$lib/components/ui/selector/theme-selector.svelte';
	import Zap from '@lucide/svelte/icons/zap';
	import ShieldCheck from '@lucide/svelte/icons/shield-check';

	// Types
	interface CompressionTarget {
		label: string;
		value: number;
	}

	// State
	let ffmpeg = $state<FFmpeg>();
	let isLoaded = $state(false);
	let loadPromise = $state<Promise<void> | null>(null);
	let isProcessing = $state(false);
	let progress = $state(0);
	let selectedFile = $state<File | null>(null);
	let processedVideo = $state<Uint8Array | null>(null);
	let originalSize = $state(0);
	let compressedSize = $state(0);
	let errorMessage = $state('');
	let statusMessage = $state('Initializing...');
	let videoDuration = $state(0);

	const compressionTargets: CompressionTarget[] = [
		{ label: '8 MB', value: 8 * 1024 * 1024 },
		{ label: '25 MB', value: 25 * 1024 * 1024 },
		{ label: '50 MB', value: 50 * 1024 * 1024 }
	];
	let selectedTargetValue = $state('25 MB');
	let selectedTarget = $state(compressionTargets[1]);

	onMount(() => {
		// Start engine load silently
		loadPromise = initFFmpeg();
	});

	const initFFmpeg = async (): Promise<void> => {
		try {
			ffmpeg = new FFmpeg();
			ffmpeg.on('progress', ({ progress: prog }: ProgressEvent) => {
				// Scale real progress from 15% to 100%
				progress = 15 + Math.round(prog * 85);
			});
			// Listen for logs to catch potential errors early
			ffmpeg.on('log', ({ message }) => {
				if (message.includes('Error')) console.warn('FFmpeg Log:', message);
			});

			await ffmpeg.load({
				coreURL: await toBlobURL(`ffmpeg/ffmpeg-core.js`, 'text/javascript'),
				wasmURL: await toBlobURL(`ffmpeg/ffmpeg-core.wasm`, 'application/wasm'),
				workerURL: await toBlobURL(`ffmpeg/ffmpeg-core.worker.js`, 'text/javascript')
			});
			isLoaded = true;
		} catch (error) {
			console.error('WASM Load Fail:', error);
			errorMessage = 'Engine failed to start. Please use a modern browser like Chrome.';
		}
	};

	const handleFileSelect = (event: Event) => {
		const target = event.target as HTMLInputElement;
		const file = target.files?.[0];
		if (file) {
			selectedFile = file;
			originalSize = file.size;
			processedVideo = null;
			errorMessage = '';

			const v = document.createElement('video');
			v.src = URL.createObjectURL(file);
			v.onloadedmetadata = () => {
				videoDuration = v.duration;
				URL.revokeObjectURL(v.src);
			};
		}
	};

	const compressVideo = async () => {
		if (!selectedFile) return;
		isProcessing = true;
		progress = 1;
		errorMessage = '';
		statusMessage = 'Analyzing video structure...';

		try {
			// Fake progress while waiting for JS/WASM loading
			if (!isLoaded && loadPromise) {
				const interval = setInterval(() => {
					if (progress < 14) progress += 1;
				}, 700);
				await loadPromise;
				clearInterval(interval);
			}

			if (!ffmpeg) throw new Error('FFmpeg not ready');

			progress = 15;
			statusMessage = 'Optimizing frames for Discord...';

			const inputName = 'input_file';
			const outputName = 'output.mp4';

			// Clean up previous files if any
			try { await ffmpeg.deleteFile(outputName); } catch(e) {}

			await ffmpeg.writeFile(inputName, await fetchFile(selectedFile));

			// Bitrate Calculation with 12% safety margin
			const targetBitrateKbps = Math.floor(((selectedTarget.value * 8) / (videoDuration || 1) / 1000) * 0.88);
			const videoBitrate = Math.max(150, targetBitrateKbps - 128);

			// Core Command: Optimized for browser WASM limits
			await ffmpeg.exec([
				'-i', inputName,
				'-c:v', 'libx264',
				'-b:v', `${videoBitrate}k`,
				'-preset', 'ultrafast', // Crucial for preventing WASM crashes
				'-profile:v', 'baseline', // Best compatibility for Discord mobile
				'-level', '3.0',
				'-vf', 'scale=iw*min(1\\,1280/iw):-2,format=yuv420p',
				'-c:a', 'aac',
				'-b:a', '128k',
				'-ac', '2',
				'-movflags', '+faststart',
				outputName
			]);

			const data = (await ffmpeg.readFile(outputName)) as Uint8Array;
			processedVideo = data;
			compressedSize = data.length;
			statusMessage = 'Successfully Optimized!';

			// Auto cleanup
			await ffmpeg.deleteFile(inputName);
		} catch (e) {
			console.error('Task Error:', e);
			errorMessage = 'Optimization failed. Try a smaller file or a different browser tab.';
		} finally {
			isProcessing = false;
		}
	};

	const downloadVideo = () => {
		if (!processedVideo) return;
		const blob = new Blob([processedVideo], { type: 'video/mp4' });
		const url = URL.createObjectURL(blob);
		const a = document.createElement('a');
		a.href = url;
		a.download = `Discord_${selectedTarget.label}_${selectedFile?.name || 'video.mp4'}`;
		a.click();
		URL.revokeObjectURL(url);
	};
</script>

<svelte:head>
	<title>Discord Video Compressor - Resize MP4 to 8MB/25MB Online</title>
	<meta name="description" content="Free and private Discord video compressor. Shrink your videos to 8MB, 25MB or 50MB directly in your browser. No uploads, no storage, pure privacy." />
	<meta name="keywords" content="discord compressor, video compressor 8mb, 25mb video reducer, nitro video compressor, online video resizer discord" />
</svelte:head>

<div class="min-h-screen bg-slate-50 text-slate-900 flex flex-col items-center p-4 md:p-12 font-sans">
	<header class="w-full max-w-4xl flex justify-between items-center mb-16">
		<div class="flex items-center gap-2">
			<div class="bg-indigo-600 p-2 rounded-xl shadow-lg">
				<Zap class="text-white h-6 w-6 fill-white" />
			</div>
			<h1 class="text-xl font-black tracking-tighter">DISCORD COMPRESSOR</h1>
		</div>
		<ThemeSelector />
	</header>

	<main class="w-full max-w-4xl grid grid-cols-1 lg:grid-cols-12 gap-12">
		<div class="lg:col-span-7 space-y-6">
			<Card.Root class="border-none shadow-2xl bg-white overflow-hidden rounded-3xl">
				<Card.Header class="pb-2">
					<Card.Title class="text-2xl font-bold">Compress Video</Card.Title>
					<p class="text-sm text-slate-400 font-medium tracking-tight">Fits any video into Discord's limits.</p>
				</Card.Header>
				<Card.Content class="p-8 space-y-8">
					<div class="relative">
						<input
							type="file"
							accept="video/*"
							onchange={handleFileSelect}
							class="absolute inset-0 w-full h-full opacity-0 cursor-pointer z-10"
						/>
						<div class="border-2 border-dashed border-slate-200 rounded-2xl py-14 text-center transition-all hover:bg-indigo-50 hover:border-indigo-300">
							{#if selectedFile}
								<div class="px-4">
									<p class="font-bold text-indigo-600 truncate">{selectedFile.name}</p>
									<p class="text-xs text-slate-400 mt-1 uppercase font-bold tracking-widest">
										Original: {(originalSize / 1024 / 1024).toFixed(2)} MB
									</p>
								</div>
							{:else}
								<div class="space-y-2">
									<div class="mx-auto w-12 h-12 bg-indigo-100 rounded-full flex items-center justify-center mb-4">
										<ShieldCheck class="text-indigo-600 w-6 h-6" />
									</div>
									<p class="text-lg font-bold">Upload Video</p>
									<p class="text-xs text-slate-400 font-medium">MP4, MOV, WEBM (UP TO 500MB)</p>
								</div>
							{/if}
						</div>
					</div>

					<div class="space-y-4">
						<div class="space-y-2">
							<Label class="text-xs font-bold uppercase text-slate-400">Target File Size</Label>
							<Select.Root type="single" value={selectedTargetValue} onValueChange={(v) => {
								selectedTargetValue = v || '25 MB';
								selectedTarget = compressionTargets.find(t => t.label === v) || compressionTargets[1];
							}}>
								<Select.Trigger class="w-full h-12 rounded-xl font-bold">{selectedTargetValue}</Select.Trigger>
								<Select.Content>
									{#each compressionTargets as t}
										<Select.Item value={t.label} class="font-medium">{t.label}</Select.Item>
									{/each}
								</Select.Content>
							</Select.Root>
						</div>

						<Button
							onclick={compressVideo}
							disabled={!selectedFile || isProcessing}
							class="w-full font-black h-14 rounded-2xl text-lg shadow-indigo-200 shadow-xl transition-transform active:scale-95 bg-indigo-600 hover:bg-indigo-700"
						>
							{#if isProcessing}
								<Loader class="mr-2 h-5 w-5 animate-spin" />
								{progress}%
							{:else}
								START COMPRESSION
							{/if}
						</Button>
					</div>

					{#if isProcessing}
						<div class="space-y-3 pt-2">
							<Progress value={progress} class="h-2 rounded-full bg-slate-100" />
							<p class="text-[10px] text-center font-black animate-pulse uppercase tracking-[0.2em] text-indigo-400">
								{statusMessage}
							</p>
						</div>
					{/if}

					{#if errorMessage}
						<Alert.Root variant="destructive" class="rounded-xl border-none bg-red-50 text-red-600">
							<Alert.Description class="font-bold text-xs">{errorMessage}</Alert.Description>
						</Alert.Root>
					{/if}
				</Card.Content>
			</Card.Root>

			{#if processedVideo}
				<div class="bg-emerald-50 border-2 border-emerald-100 p-6 rounded-3xl flex items-center justify-between animate-in fade-in slide-in-from-bottom-4">
					<div>
						<p class="text-xs font-black text-emerald-600 uppercase tracking-widest mb-1">Compression Result</p>
						<p class="text-2xl font-black text-emerald-900">{(compressedSize / 1024 / 1024).toFixed(2)} MB</p>
					</div>
					<Button onclick={downloadVideo} size="lg" class="rounded-2xl px-10 bg-emerald-600 hover:bg-emerald-700 font-bold shadow-lg shadow-emerald-100">
						DOWNLOAD
					</Button>
				</div>
			{/if}
		</div>

		<div class="lg:col-span-5 space-y-10">
			<section class="space-y-4">
				<h3 class="text-sm font-black uppercase tracking-widest text-indigo-600">Why choose us?</h3>
				<div class="grid gap-4">
					<div class="bg-white p-5 rounded-2xl shadow-sm border border-slate-100">
						<p class="font-bold text-slate-800 text-sm mb-1">Total Privacy</p>
						<p class="text-xs text-slate-400 leading-relaxed">Videos are processed locally in your RAM. Your files never leave your device.</p>
					</div>
					<div class="bg-white p-5 rounded-2xl shadow-sm border border-slate-100">
						<p class="font-bold text-slate-800 text-sm mb-1">No File Limits</p>
						<p class="text-xs text-slate-400 leading-relaxed">Compress large files for Discord Nitro or fit them into the 25MB standard limit.</p>
					</div>
				</div>
			</section>

			<section class="bg-indigo-600 rounded-3xl p-8 text-white shadow-xl shadow-indigo-100">
				<h3 class="font-black text-xl mb-4 italic uppercase tracking-tighter">Discord Limits 2025</h3>
				<div class="space-y-4 font-bold text-sm">
					<div class="flex justify-between border-b border-indigo-500 pb-2">
						<span class="opacity-80">Free / Standard</span>
						<span>25 MB</span>
					</div>
					<div class="flex justify-between border-b border-indigo-500 pb-2">
						<span class="opacity-80">Nitro Basic</span>
						<span>50 MB</span>
					</div>
					<div class="flex justify-between">
						<span class="opacity-80">Nitro Pro</span>
						<span>500 MB</span>
					</div>
				</div>
			</section>
		</div>
	</main>

	<section class="w-full max-w-4xl mt-32 space-y-16 border-t border-slate-200 pt-20">
		<h2 class="text-4xl font-black text-center tracking-tighter italic">FAQ</h2>

		<div class="grid grid-cols-1 md:grid-cols-2 gap-12">
			<div class="space-y-3">
				<h4 class="font-black text-indigo-600 uppercase text-xs tracking-widest">How to compress video to 8MB?</h4>
				<p class="text-sm text-slate-500 leading-relaxed font-medium">
					Discord recently increased the limit to 25MB, but some servers still prefer smaller files. Select "8 MB" in our settings, and we'll handle the complex calculations to make it fit perfectly.
				</p>
			</div>
			<div class="space-y-3">
				<h4 class="font-black text-indigo-600 uppercase text-xs tracking-widest">Why is my file still too big?</h4>
				<p class="text-sm text-slate-500 leading-relaxed font-medium">
					If the video is extremely long, even high compression might struggle. We suggest trimming videos to under 3 minutes for the best results on Discord.
				</p>
			</div>
			<div class="space-y-3">
				<h4 class="font-black text-indigo-600 uppercase text-xs tracking-widest">Does it work on Mobile?</h4>
				<p class="text-sm text-slate-500 leading-relaxed font-medium">
					Yes! Our tool works on Chrome and Safari for iOS and Android. It uses your phone's processor to compress videos on-the-go.
				</p>
			</div>
			<div class="space-y-3">
				<h4 class="font-black text-indigo-600 uppercase text-xs tracking-widest">Is it really free?</h4>
				<p class="text-sm text-slate-500 leading-relaxed font-medium">
					Absolutely. There are no subscriptions or hidden fees. We don't even have servers to store your videos, which keeps our costs low and your data safe.
				</p>
			</div>
		</div>
	</section>

	<footer class="mt-32 pb-12 text-center text-[10px] font-bold uppercase tracking-[0.3em] text-slate-300">
		© 2025 DiscordCompressor.com | Browser-Based Optimization
	</footer>
</div>

<style>
	:global(html) {
		scroll-behavior: smooth;
		background-color: #f8fafc;
	}
</style>