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
	import LanguageSelector from '$lib/components/ui/selector/language-selector.svelte';
	import ThemeSelector from '$lib/components/ui/selector/theme-selector.svelte';
	import ChevronDown from '@lucide/svelte/icons/chevron-down';

	// Types
	interface CompressionTarget {
		label: string;
		value: number;
		description: string;
	}

	interface VideoMetadata {
		duration: number;
		bitrate: number;
		resolution: string;
		size: number;
		fps: number;
		hasMotion: boolean;
	}

	// State
	let ffmpeg = $state<FFmpeg>();
	let isLoaded = $state(false);
	let loadPromise = $state<Promise<void> | null>(null); // Queued loading promise
	let isProcessing = $state(false);
	let progress = $state(0);
	let selectedFile = $state<File | null>(null);
	let processedVideo = $state<Uint8Array | null>(null);
	let originalSize = $state(0);
	let compressedSize = $state(0);
	let errorMessage = $state('');
	let videoMetadata = $state<VideoMetadata | null>(null);
	let statusMessage = $state('Preparing...');
	let startTime = $state<number>(0);
	let isChromium = $state(false);

	const compressionTargets: CompressionTarget[] = [
		{ label: '8 MB', value: 8 * 1024 * 1024, description: 'Discord Basic Limit' },
		{ label: '25 MB', value: 25 * 1024 * 1024, description: 'Discord Nitro Basic' },
		{ label: '50 MB', value: 50 * 1024 * 1024, description: 'High Quality' },
		{ label: '100 MB', value: 100 * 1024 * 1024, description: 'Discord Nitro' }
	];
	let selectedTargetValue = $state('25 MB');
	let selectedTarget = $state(compressionTargets[1]);

	onMount(() => {
		// Silent load in background
		loadPromise = initFFmpeg();
		isChromium = !!(navigator as any).userAgentData;
	});

	const initFFmpeg = async (): Promise<void> => {
		try {
			ffmpeg = new FFmpeg();
			ffmpeg.on('progress', ({ progress: prog }: ProgressEvent) => {
				progress = Math.round(prog * 100);
			});
			await ffmpeg.load({
				coreURL: await toBlobURL(`ffmpeg/ffmpeg-core.js`, 'text/javascript'),
				wasmURL: await toBlobURL(`ffmpeg/ffmpeg-core.wasm`, 'application/wasm'),
				workerURL: await toBlobURL(`ffmpeg/ffmpeg-core.worker.js`, 'text/javascript')
			});
			isLoaded = true;
		} catch (error) {
			console.error('FFmpeg Load Error:', error);
			errorMessage = 'Hardware acceleration not supported or engine failed to load.';
			throw error;
		}
	};

	const handleFileSelect = async (event: Event) => {
		const target = event.target as HTMLInputElement;
		const file = target.files?.[0];
		if (file) {
			selectedFile = file;
			originalSize = file.size;
			processedVideo = null;
			await extractMetadata(file);
		}
	};

	const extractMetadata = async (file: File) => {
		const video = document.createElement('video');
		video.src = URL.createObjectURL(file);
		video.onloadedmetadata = () => {
			videoMetadata = {
				duration: video.duration,
				bitrate: Math.round((file.size * 8) / video.duration / 1000),
				resolution: `${video.videoWidth}x${video.videoHeight}`,
				size: file.size,
				fps: 30, // Default baseline
				hasMotion: true
			};
			URL.revokeObjectURL(video.src);
		};
	};

	const compressVideo = async () => {
		if (!selectedFile || !videoMetadata) return;

		isProcessing = true;
		progress = 0;
		errorMessage = '';

		try {
			// QUEUE LOGIC: Wait for engine if not ready
			if (!isLoaded && loadPromise) {
				statusMessage = 'Initializing engine...';
				await loadPromise;
			}

			if (!ffmpeg) throw new Error('Engine Error');

			statusMessage = 'Processing video...';
			const inputName = 'input_' + selectedFile.name;
			await ffmpeg.writeFile(inputName, await fetchFile(selectedFile));

			// Simple Discord-optimized settings
			const targetBitrate = Math.floor(((selectedTarget.value * 8) / videoMetadata.duration / 1000) * 0.9);
			const videoBitrate = Math.max(150, targetBitrate - 128);

			await ffmpeg.exec([
				'-i', inputName,
				'-c:v', 'libx264',
				'-b:v', `${videoBitrate}k`,
				'-preset', 'veryfast',
				'-vf', 'scale=iw*min(1\\,1280/iw):-2', // Max 720p for Discord
				'-c:a', 'aac',
				'-b:a', '128k',
				'-movflags', '+faststart',
				'output.mp4'
			]);

			const data = (await ffmpeg.readFile('output.mp4')) as Uint8Array;
			processedVideo = data;
			compressedSize = data.length;
			statusMessage = 'Done!';
		} catch (e) {
			errorMessage = 'Compression failed. The file might be too large or incompatible.';
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
		a.download = `discord_${selectedTarget.label}_${selectedFile?.name || 'video.mp4'}`;
		a.click();
	};
</script>

<svelte:head>
	<title>Discord Video Compressor - Compress to 8MB / 25MB Online</title>
	<meta name="description" content="Free online Discord video compressor. Reduce video size to 8MB or 25MB for Discord upload without quality loss. Privacy-focused, works in your browser." />
	<meta name="keywords" content="discord video compressor, 8mb video compressor, 25mb video compressor, compress video for discord, video size reducer, discord nitro compressor" />
	<meta property="og:title" content="Discord Video Compressor - Fast & Secure" />
	<meta property="og:description" content="Compress your videos to fit Discord's file limits (8MB, 25MB, 50MB) instantly." />
	<meta property="og:type" content="website" />
</svelte:head>

<div class="container mx-auto max-w-4xl p-6 min-h-screen">
	<header class="mb-12 flex flex-col items-center text-center">
		<div class="flex items-center gap-3 mb-4">
			<h1 class="text-4xl font-extrabold tracking-tight lg:text-5xl text-primary">
				Discord Compressor
			</h1>
			<Badge variant="outline" class="hidden sm:block">v2.0</Badge>
		</div>
		<p class="text-muted-foreground text-lg max-w-lg">
			Compress your videos to meet Discord's 8MB or 25MB upload limits.
			<strong>100% Privacy</strong>: Files never leave your computer.
		</p>
		<div class="mt-4 flex gap-2">
			<LanguageSelector />
			<ThemeSelector />
		</div>
	</header>

	{#if errorMessage}
		<Alert.Root class="mb-6 border-destructive bg-destructive/10">
			<Alert.Description>{errorMessage}</Alert.Description>
		</Alert.Root>
	{/if}

	<div class="grid grid-cols-1 gap-8 lg:grid-cols-2">
		<Card.Root class="shadow-xl border-2">
			<Card.Header>
				<Card.Title>Compressor Settings</Card.Title>
			</Card.Header>
			<Card.Content class="space-y-6">
				<div class="space-y-2">
					<Label for="video-upload">Select Video File</Label>
					<Input
						id="video-upload"
						type="file"
						accept="video/*"
						onchange={handleFileSelect}
						class="cursor-pointer"
					/>
				</div>

				{#if videoMetadata}
					<div class="bg-muted p-3 rounded-lg text-sm grid grid-cols-2 gap-2">
						<span class="text-muted-foreground">Original Size:</span>
						<span class="font-mono">{(originalSize / (1024 * 1024)).toFixed(2)} MB</span>
						<span class="text-muted-foreground">Resolution:</span>
						<span>{videoMetadata.resolution}</span>
					</div>
				{/if}

				<div class="space-y-2">
					<Label>Target File Size</Label>
					<Select.Root type="single" value={selectedTargetValue} onValueChange={(v) => {
						selectedTargetValue = v || '25 MB';
						selectedTarget = compressionTargets.find(t => t.label === v) || compressionTargets[1];
					}}>
						<Select.Trigger class="w-full">
							{selectedTargetValue}
						</Select.Trigger>
						<Select.Content>
							{#each compressionTargets as target}
								<Select.Item value={target.label}>{target.label} - {target.description}</Select.Item>
							{/each}
						</Select.Content>
					</Select.Root>
				</div>

				<Button
					onclick={compressVideo}
					class="w-full h-12 text-lg font-bold"
					disabled={!selectedFile || isProcessing}
				>
					{#if isProcessing}
						<Loader class="mr-2 h-5 w-5 animate-spin" />
						{statusMessage}
					{:else}
						Compress Video
					{/if}
				</Button>

				{#if isProcessing && progress > 0}
					<div class="space-y-2">
						<div class="flex justify-between text-xs font-bold uppercase tracking-wider text-primary">
							<span>Encoding Progress</span>
							<span>{progress}%</span>
						</div>
						<Progress value={progress} class="h-3" />
					</div>
				{/if}
			</Card.Content>
		</Card.Root>

		<Card.Root class="bg-accent/30 border-dashed border-2">
			<Card.Header>
				<Card.Title>Download Result</Card.Title>
			</Card.Header>
			<Card.Content class="flex flex-col items-center justify-center min-h-[300px]">
				{#if processedVideo}
					<div class="text-center space-y-4 w-full">
						<div class="p-6 bg-background rounded-full inline-block mb-2 shadow-sm">
							<Badge class="text-xl px-4 py-1" variant="default">
								{(compressedSize / (1024 * 1024)).toFixed(2)} MB
							</Badge>
						</div>
						<p class="text-sm text-muted-foreground">Successfully compressed for Discord!</p>
						<Button onclick={downloadVideo} size="lg" class="w-full bg-green-600 hover:bg-green-700">
							Download MP4
						</Button>
					</div>
				{:else}
					<div class="text-center text-muted-foreground p-8">
						<div class="mb-4 opacity-20">
							<ChevronDown size={64} class="mx-auto" />
						</div>
						<p>Your compressed video will appear here.</p>
					</div>
				{/if}
			</Card.Content>
		</Card.Root>
	</div>

	<section class="mt-20 space-y-12 border-t pt-12">
		<div class="text-center">
			<h2 class="text-3xl font-bold mb-4">Frequently Asked Questions</h2>
			<p class="text-muted-foreground">Everything you need to know about DiscordCompressor.com</p>
		</div>

		<div class="grid grid-cols-1 md:grid-cols-2 gap-8">
			<div class="space-y-3">
				<h3 class="text-xl font-semibold italic text-primary">What is the Discord file size limit?</h3>
				<p class="text-muted-foreground leading-relaxed">
					As of 2024, standard Discord users have a **25MB** upload limit (recently increased from 8MB).
					Nitro Basic users can upload up to **50MB**, and Nitro subscribers can upload up to **500MB**.
					Our tool helps you hit these targets exactly.
				</p>
			</div>

			<div class="space-y-3">
				<h3 class="text-xl font-semibold italic text-primary">How do I compress a video to 8MB?</h3>
				<p class="text-muted-foreground leading-relaxed">
					Simply upload your file, select the "8 MB" target from our dropdown, and click Compress.
					The tool will automatically calculate the best bitrate to ensure your video fits
					the legacy Discord limit while maintaining the best possible quality.
				</p>
			</div>

			<div class="space-y-3">
				<h3 class="text-xl font-semibold italic text-primary">Is my data secure?</h3>
				<p class="text-muted-foreground leading-relaxed">
					**Yes.** Unlike other online converters, we use WebAssembly (WASM) to run the
					compression engine inside your browser. Your video is never uploaded to any server.
					It stays on your machine throughout the entire process.
				</p>
			</div>

			<div class="space-y-3">
				<h3 class="text-xl font-semibold italic text-primary">What formats are supported?</h3>
				<p class="text-muted-foreground leading-relaxed">
					We support all major video formats including MP4, MOV, MKV, and WebM.
					The output is always optimized as an **MP4 (H.264)**, which is the most
					compatible format for Discord's mobile and desktop players.
				</p>
			</div>
		</div>
	</section>

	<footer class="mt-20 pb-10 text-center text-sm text-muted-foreground border-t pt-6">
		<p>© 2025 DiscordCompressor.com - Private & Fast Browser Compression</p>
	</footer>
</div>

<style>
	:global(body) {
		background-attachment: fixed;
	}
</style>