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
	import ShieldCheck from '@lucide/svelte/icons/shield-check';
	import Zap from '@lucide/svelte/icons/zap';

	// Types
	interface CompressionTarget {
		label: string;
		value: number;
		description: string;
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
	let statusMessage = $state('Preparing...');
	let videoDuration = $state(0);

	const compressionTargets: CompressionTarget[] = [
		{ label: '8 MB', value: 8 * 1024 * 1024, description: 'Discord Legacy Limit' },
		{ label: '25 MB', value: 25 * 1024 * 1024, description: 'Discord Standard' },
		{ label: '50 MB', value: 50 * 1024 * 1024, description: 'Nitro Basic' }
	];
	let selectedTargetValue = $state('25 MB');
	let selectedTarget = $state(compressionTargets[1]);

	onMount(() => {
		// Silent load background
		loadPromise = initFFmpeg();
	});

	const initFFmpeg = async (): Promise<void> => {
		try {
			ffmpeg = new FFmpeg();
			ffmpeg.on('progress', ({ progress: prog }: ProgressEvent) => {
				// Actual compression takes up 15% to 100% of the bar
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
			errorMessage = 'Browser acceleration error. Please try Chrome or Edge.';
		}
	};

	const handleFileSelect = (event: Event) => {
		const target = event.target as HTMLInputElement;
		const file = target.files?.[0];
		if (file) {
			selectedFile = file;
			originalSize = file.size;
			processedVideo = null;
			// Quick duration check
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
		progress = 2; // Immediate feedback
		errorMessage = '';
		statusMessage = 'Analyzing video frames...';

		try {
			// If engine isn't ready, "fake" the progress during loading
			if (!isLoaded && loadPromise) {
				const interval = setInterval(() => {
					if (progress < 12) progress += 1;
				}, 800);
				await loadPromise;
				clearInterval(interval);
			}

			if (!ffmpeg) throw new Error();

			progress = 15;
			statusMessage = 'Optimizing for Discord...';

			const inputName = 'input.mp4';
			await ffmpeg.writeFile(inputName, await fetchFile(selectedFile));

			// Logic: Calculate bitrate based on target size and duration
			const targetBitrate = Math.floor(((selectedTarget.value * 8) / (videoDuration || 10) / 1000) * 0.85);
			const videoBitrate = Math.max(100, targetBitrate - 128);

			await ffmpeg.exec([
				'-i', inputName,
				'-c:v', 'libx264',
				'-b:v', `${videoBitrate}k`,
				'-preset', 'ultrafast',
				'-vf', 'scale=iw*min(1\\,1280/iw):-2', // Ensure it fits mobile screens
				'-c:a', 'aac',
				'-b:a', '128k',
				'-movflags', '+faststart',
				'out.mp4'
			]);

			const data = (await ffmpeg.readFile('out.mp4')) as Uint8Array;
			processedVideo = data;
			compressedSize = data.length;
			statusMessage = 'Optimization complete!';
		} catch (e) {
			errorMessage = 'Error: Video codec incompatible or file too large.';
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
		a.download = `discord_compressed_${selectedFile?.name || 'video.mp4'}`;
		a.click();
	};
</script>

<svelte:head>
	<title>Discord Video Compressor - Compress to 8MB / 25MB Fast & Private</title>
	<meta name="description" content="The fastest way to compress videos for Discord. Reduce size to 8MB or 25MB directly in your browser. No uploads, 100% private, Nitro optimized." />
	<meta name="keywords" content="discord compressor, video compressor 8mb, 25mb video compressor, discord video size limit, nitro video compressor" />

	<meta property="og:title" content="Discord Video Compressor" />
	<meta property="og:description" content="Compress your videos for Discord instantly without leaving your browser." />
	<meta property="og:url" content="https://discordcompressor.com" />
</svelte:head>

<div class="min-h-screen bg-background text-foreground flex flex-col items-center p-4 md:p-8">
	<header class="w-full max-w-4xl flex justify-between items-center mb-12">
		<div class="flex items-center gap-2">
			<div class="bg-primary p-2 rounded-lg">
				<Zap class="text-primary-foreground h-6 w-6" />
			</div>
			<span class="text-2xl font-black tracking-tighter">DISCORD COMPRESSOR</span>
		</div>
		<div class="flex gap-2">
			<LanguageSelector />
			<ThemeSelector />
		</div>
	</header>

	<main class="w-full max-w-4xl grid grid-cols-1 lg:grid-cols-5 gap-8">
		<div class="lg:col-span-3 space-y-6">
			<Card.Root class="overflow-hidden border-2">
				<Card.Header class="pb-4">
					<Card.Title class="text-2xl">Optimize Your Video</Card.Title>
					<p class="text-sm text-muted-foreground">Privacy first: your video stays on your device.</p>
				</Card.Header>
				<Card.Content class="space-y-6">
					<div
						class="border-2 border-dashed rounded-xl p-8 text-center transition-colors hover:border-primary/50 bg-accent/20"
						role="button"
						tabindex="0"
					>
						<Input
							type="file"
							accept="video/*"
							onchange={handleFileSelect}
							class="hidden"
							id="file-input"
						/>
						<label for="file-input" class="cursor-pointer">
							{#if selectedFile}
								<p class="font-bold text-primary truncate">{selectedFile.name}</p>
								<p class="text-xs text-muted-foreground mt-1">{(originalSize / 1024 / 1024).toFixed(2)} MB</p>
							{:else}
								<p class="text-lg font-medium">Drop video or click to upload</p>
								<p class="text-xs text-muted-foreground">Support MP4, MOV, WEBM up to 500MB</p>
							{/if}
						</label>
					</div>

					<div class="grid grid-cols-2 gap-4">
						<div class="space-y-2">
							<Label>Target Size</Label>
							<Select.Root type="single" value={selectedTargetValue} onValueChange={(v) => {
								selectedTargetValue = v || '25 MB';
								selectedTarget = compressionTargets.find(t => t.label === v) || compressionTargets[1];
							}}>
								<Select.Trigger>{selectedTargetValue}</Select.Trigger>
								<Select.Content>
									{#each compressionTargets as t}
										<Select.Item value={t.label}>{t.label}</Select.Item>
									{/each}
								</Select.Content>
							</Select.Root>
						</div>
						<div class="flex items-end">
							<Button
								onclick={compressVideo}
								disabled={!selectedFile || isProcessing}
								class="w-full font-bold h-10"
							>
								{#if isProcessing}
									<Loader class="mr-2 h-4 w-4 animate-spin" />
									{progress}%
								{:else}
									Compress Now
								{/if}
							</Button>
						</div>
					</div>

					{#if isProcessing}
						<div class="space-y-2">
							<Progress value={progress} class="h-2" />
							<p class="text-xs text-center animate-pulse text-muted-foreground italic">
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
				<Card.Root class="bg-primary/5 border-primary/20 border-2">
					<Card.Content class="p-6 flex items-center justify-between">
						<div>
							<p class="text-sm font-bold">Ready for Discord!</p>
							<p class="text-xs text-muted-foreground">New size: {(compressedSize / 1024 / 1024).toFixed(2)} MB</p>
						</div>
						<Button onclick={downloadVideo} variant="default" class="bg-green-600 hover:bg-green-700">
							Download Video
						</Button>
					</Card.Content>
				</Card.Root>
			{/if}
		</div>

		<div class="lg:col-span-2 space-y-6">
			<div class="bg-card rounded-xl p-6 border shadow-sm">
				<h3 class="font-bold flex items-center gap-2 mb-4">
					<ShieldCheck class="text-green-500" /> No Uploading Needed
				</h3>
				<p class="text-sm text-muted-foreground leading-relaxed">
					DiscordCompressor.com uses your computer's own power to shrink videos. Your files never touch our servers, meaning it's <strong>100% private</strong> and your data is safe.
				</p>
			</div>

			<div class="bg-card rounded-xl p-6 border shadow-sm">
				<h3 class="font-bold mb-4">Discord Limits (2025)</h3>
				<ul class="text-xs space-y-3">
					<li class="flex justify-between border-b pb-1">
						<span>Standard Users</span>
						<span class="font-bold">25 MB</span>
					</li>
					<li class="flex justify-between border-b pb-1">
						<span>Nitro Basic</span>
						<span class="font-bold">50 MB</span>
					</li>
					<li class="flex justify-between border-b pb-1">
						<span>Nitro Pro</span>
						<span class="font-bold">500 MB</span>
					</li>
				</ul>
			</div>
		</div>
	</main>

	<section class="w-full max-w-4xl mt-20 space-y-12">
		<div class="text-center">
			<h2 class="text-3xl font-bold italic">Frequently Asked Questions</h2>
		</div>

		<div class="grid grid-cols-1 md:grid-cols-2 gap-8">
			<div class="space-y-2">
				<h4 class="font-bold text-primary">How do I compress a video for Discord without Nitro?</h4>
				<p class="text-sm text-muted-foreground">
					Standard Discord accounts are limited to 25MB. Simply upload your video to our tool, select "25MB" as the target, and we will adjust the bitrate to ensure it fits perfectly while keeping the resolution at 720p or 1080p.
				</p>
			</div>

			<div class="space-y-2">
				<h4 class="font-bold text-primary">Is there a video length limit?</h4>
				<p class="text-sm text-muted-foreground">
					No, but longer videos will require more compression to fit into the 8MB or 25MB limit, which might lower the visual quality. For the best result on Discord, we recommend videos under 5 minutes.
				</p>
			</div>

			<div class="space-y-2">
				<h4 class="font-bold text-primary">Why is this compressor faster than others?</h4>
				<p class="text-sm text-muted-foreground">
					Most "online" compressors upload your file to a server, process it, and then you have to download it back. Our tool works <strong>locally in your browser</strong> using FFmpeg WASM technology, skipping the upload/download time entirely.
				</p>
			</div>

			<div class="space-y-2">
				<h4 class="font-bold text-primary">Will it work on my phone?</h4>
				<p class="text-sm text-muted-foreground">
					Yes! DiscordCompressor.com is compatible with modern mobile browsers on Android and iOS (iPhone), allowing you to shrink videos captured on your phone instantly.
				</p>
			</div>
		</div>
	</section>

	<footer class="mt-20 py-8 border-t w-full max-w-4xl text-center text-xs text-muted-foreground">
		<p>© 2025 DiscordCompressor.com | Private Browser-Based Video Optimization</p>
	</footer>
</div>

<style>
	:global(body) {
		background: radial-gradient(circle at top right, rgba(88, 101, 242, 0.05), transparent),
		            radial-gradient(circle at bottom left, rgba(88, 101, 242, 0.05), transparent);
	}
</style>