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

	// State Variables
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
		// Start loading the heavy WASM engine silently in background
		loadPromise = initFFmpeg();
	});

	const initFFmpeg = async (): Promise<void> => {
		try {
			ffmpeg = new FFmpeg();
			ffmpeg.on('progress', ({ progress: prog }: ProgressEvent) => {
				// We allocate 15%-100% of the progress bar to actual FFmpeg task
				progress = 15 + Math.round(prog * 85);
			});
			await ffmpeg.load({
				coreURL: await toBlobURL(`ffmpeg/ffmpeg-core.js`, 'text/javascript'),
				wasmURL: await toBlobURL(`ffmpeg/ffmpeg-core.wasm`, 'application/wasm'),
				workerURL: await toBlobURL(`ffmpeg/ffmpeg-core.worker.js`, 'text/javascript')
			});
			isLoaded = true;
		} catch (error) {
			console.error(error);
			errorMessage = 'Hardware acceleration is disabled or unsupported in this browser.';
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

			// Quick metadata check using native browser video element
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
			// Phase 1: Fake progress if WASM is still downloading
			if (!isLoaded && loadPromise) {
				const interval = setInterval(() => {
					if (progress < 14) progress += 1;
				}, 600);
				await loadPromise;
				clearInterval(interval);
			}

			if (!ffmpeg) throw new Error('Engine failed');

			// Phase 2: Start real FFmpeg processing
			progress = 15;
			statusMessage = 'Optimizing bitrate for Discord...';

			const inputName = 'input_video';
			await ffmpeg.writeFile(inputName, await fetchFile(selectedFile));

			// Calculate bitrate to hit the target size (Target / Duration * Safety Factor)
			const targetBitrateKbps = Math.floor(((selectedTarget.value * 8) / (videoDuration || 10) / 1000) * 0.88);
			const vBitrate = Math.max(100, targetBitrateKbps - 128); // Reserve 128k for audio

			await ffmpeg.exec([
				'-i', inputName,
				'-c:v', 'libx264',
				'-b:v', `${vBitrate}k`,
				'-preset', 'ultrafast',
				'-vf', 'scale=iw*min(1\\,1280/iw):-2', // Ensure max 720p for Discord preview
				'-c:a', 'aac',
				'-b:a', '128k',
				'-movflags', '+faststart',
				'output.mp4'
			]);

			const data = (await ffmpeg.readFile('output.mp4')) as Uint8Array;
			processedVideo = data;
			compressedSize = data.length;
			statusMessage = 'Finished!';
		} catch (e) {
			errorMessage = 'Processing failed. Your file may be corrupted or too complex.';
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
		a.download = `discord_ready_${selectedFile?.name || 'video.mp4'}`;
		a.click();
	};
</script>

<svelte:head>
	<title>Discord Video Compressor - Compress to 8MB / 25MB for Free</title>
	<meta name="description" content="Fast, private online video compressor for Discord. Reduce your video size to 8MB or 25MB instantly in your browser. No uploads, 100% secure." />
	<meta name="keywords" content="discord compressor, video compressor 8mb, compress video for discord, 25mb video compressor, discord nitro video size" />

	<meta property="og:title" content="Discord Video Compressor" />
	<meta property="og:description" content="Fit your videos into Discord's upload limit without losing quality." />
	<meta property="og:type" content="website" />
</svelte:head>

<div class="min-h-screen bg-background flex flex-col items-center p-4 md:p-10">
	<header class="w-full max-w-5xl flex justify-between items-center mb-16">
		<div class="flex items-center gap-2">
			<Zap class="text-primary h-8 w-8 fill-primary" />
			<h1 class="text-2xl font-extrabold tracking-tighter">DISCORD COMPRESSOR</h1>
		</div>
		<ThemeSelector />
	</header>

	<main class="w-full max-w-5xl grid grid-cols-1 lg:grid-cols-12 gap-10">
		<div class="lg:col-span-7 space-y-8">
			<Card.Root class="border-2 shadow-xl overflow-hidden">
				<Card.Header class="bg-accent/30">
					<Card.Title>Video Optimizer</Card.Title>
					<p class="text-sm text-muted-foreground">Select a file to begin the compression process.</p>
				</Card.Header>
				<Card.Content class="p-8 space-y-6">
					<div class="relative group">
						<input
							type="file"
							accept="video/*"
							onchange={handleFileSelect}
							class="absolute inset-0 w-full h-full opacity-0 cursor-pointer z-10"
						/>
						<div class="border-2 border-dashed rounded-xl py-12 text-center transition-all group-hover:border-primary group-hover:bg-primary/5">
							{#if selectedFile}
								<p class="font-bold text-primary truncate px-4">{selectedFile.name}</p>
								<p class="text-xs text-muted-foreground mt-2">Original: {(originalSize / 1024 / 1024).toFixed(2)} MB</p>
							{:else}
								<p class="text-lg font-bold">Select or Drop Video</p>
								<p class="text-sm text-muted-foreground">MP4, MOV, WebM (Max 500MB)</p>
							{/if}
						</div>
					</div>

					<div class="flex flex-col sm:flex-row gap-4">
						<div class="flex-1 space-y-2">
							<Label>Discord Target Limit</Label>
							<Select.Root type="single" value={selectedTargetValue} onValueChange={(v) => {
								selectedTargetValue = v || '25 MB';
								selectedTarget = compressionTargets.find(t => t.label === v) || compressionTargets[1];
							}}>
								<Select.Trigger class="w-full">{selectedTargetValue}</Select.Trigger>
								<Select.Content>
									{#each compressionTargets as t}
										<Select.Item value={t.label}>{t.label}</Select.Item>
									{/each}
								</Select.Content>
							</Select.Root>
						</div>
						<div class="flex items-end sm:w-48">
							<Button
								onclick={compressVideo}
								disabled={!selectedFile || isProcessing}
								class="w-full font-bold h-10 shadow-lg"
							>
								{#if isProcessing}
									<Loader class="mr-2 h-4 w-4 animate-spin" />
									{progress}%
								{:else}
									Compress
								{/if}
							</Button>
						</div>
					</div>

					{#if isProcessing}
						<div class="space-y-3">
							<Progress value={progress} class="h-2.5" />
							<p class="text-xs text-center font-medium animate-pulse uppercase tracking-widest text-muted-foreground">
								{statusMessage}
							</p>
						</div>
					{/if}

					{#if errorMessage}
						<Alert.Root variant="destructive">
							<Alert.Description>{errorMessage}</Alert.Description>
						</Alert.Root>
					{/if}
				</Card.Content>
			</Card.Root>

			{#if processedVideo}
				<Card.Root class="border-primary/40 border-2 bg-primary/5">
					<Card.Content class="p-6 flex flex-col sm:flex-row items-center justify-between gap-4">
						<div class="text-center sm:text-left">
							<Badge variant="outline" class="mb-1">Ready for Upload</Badge>
							<p class="text-lg font-black">Size: {(compressedSize / 1024 / 1024).toFixed(2)} MB</p>
						</div>
						<Button onclick={downloadVideo} size="lg" class="w-full sm:w-auto px-8 bg-green-600 hover:bg-green-700">
							Download MP4
						</Button>
					</Card.Content>
				</Card.Root>
			{/if}
		</div>

		<div class="lg:col-span-5 space-y-8">
			<div class="bg-card border-2 p-6 rounded-2xl shadow-sm space-y-4">
				<div class="flex items-center gap-3">
					<ShieldCheck class="text-green-500 h-6 w-6" />
					<h3 class="font-bold text-lg">Local Processing</h3>
				</div>
				<p class="text-sm text-muted-foreground leading-relaxed">
					Unlike other websites, we don't upload your videos to a server. All compression happens <strong>inside your browser</strong>. Your privacy is guaranteed because your data never leaves your computer.
				</p>
			</div>

			<div class="p-6 rounded-2xl border-2 space-y-4">
				<h3 class="font-bold text-lg">Discord Upload Limits</h3>
				<div class="space-y-2 text-sm">
					<div class="flex justify-between border-b pb-2">
						<span class="text-muted-foreground">Free Users</span>
						<span class="font-mono font-bold">25 MB</span>
					</div>
					<div class="flex justify-between border-b pb-2">
						<span class="text-muted-foreground">Nitro Basic</span>
						<span class="font-mono font-bold">50 MB</span>
					</div>
					<div class="flex justify-between border-b pb-2">
						<span class="text-muted-foreground">Nitro Pro</span>
						<span class="font-mono font-bold">500 MB</span>
					</div>
				</div>
			</div>
		</div>
	</main>

	<section class="w-full max-w-5xl mt-24 pb-20">
		<h2 class="text-3xl font-black mb-12 text-center tracking-tight italic">FREQUENTLY ASKED QUESTIONS</h2>

		<div class="grid grid-cols-1 md:grid-cols-2 gap-10 text-left">
			<div class="space-y-3">
				<h4 class="font-bold text-primary">How to compress a video for Discord to 8MB?</h4>
				<p class="text-sm text-muted-foreground">
					Simply select the "8 MB" option in our tool. Our algorithm will automatically adjust the bitrate and scale to ensure the final output is under the legacy Discord limit while preserving as much quality as possible.
				</p>
			</div>

			<div class="space-y-3">
				<h4 class="font-bold text-primary">Is this Discord compressor safe?</h4>
				<p class="text-sm text-muted-foreground">
					Yes, it is safer than most. Since we use FFmpeg.WASM, the video file is processed locally on your machine. We never see your files, and no data is sent to our servers.
				</p>
			</div>

			<div class="space-y-3">
				<h4 class="font-bold text-primary">Why is the processing speed slow?</h4>
				<p class="text-sm text-muted-foreground">
					Because compression happens in your browser, it depends on your CPU power. If it's taking too long, try closing other browser tabs or using a device with a faster processor.
				</p>
			</div>

			<div class="space-y-3">
				<h4 class="font-bold text-primary">Which formats are supported?</h4>
				<p class="text-sm text-muted-foreground">
					We support MP4, MOV, WEBM, and AVI. The output is always an MP4 file using the H.264 codec, which is guaranteed to be playable directly within the Discord chat window on both mobile and desktop.
				</p>
			</div>
		</div>
	</section>

	<footer class="mt-auto py-8 text-center text-xs text-muted-foreground border-t w-full max-w-5xl">
		<p>© 2025 DiscordCompressor.com | Fast & Secure Browser-Side Video Tools</p>
	</footer>
</div>

<style>
	:global(html) {
		scroll-behavior: smooth;
	}
</style>