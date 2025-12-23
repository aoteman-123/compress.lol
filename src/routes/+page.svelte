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
	import * as m from '$lib/paraglide/messages.js';
	import LanguageSelector from '$lib/components/ui/selector/language-selector.svelte';
	import ThemeSelector from '$lib/components/ui/selector/theme-selector.svelte';
	import Settings from '@lucide/svelte/icons/settings';
	import ChevronDown from '@lucide/svelte/icons/chevron-down';

	interface CompressionTarget {
		label: string;
		value: number;
		description: string;
	}

	interface VideoMetadata {
		duration: number;
		bitrate: number;
		resolution: string;
		codec: string;
		size: number;
		fps: number;
		hasMotion: boolean;
	}

	interface CompressionSettings {
		videoBitrate: string;
		audioBitrate: string;
		resolution: string;
		crf: number;
		preset: string;
		tune: string;
		bufferSize: string;
		refs: number;
		bframes: number;
		targetFps: number;
	}

	// 状态变量
	let ffmpeg = $state<FFmpeg>();
	let isLoaded = $state(false);
	let loadPromise = $state<Promise<void> | null>(null); // [优化] 用于统一追踪加载状态
	let isProcessing = $state(false);
	let progress = $state(0);
	let selectedFile = $state<File | null>(null);
	let processedVideo = $state<Uint8Array | null>(null);
	let originalSize = $state(0);
	let compressedSize = $state(0);
	let errorMessage = $state('');
	let videoMetadata = $state<VideoMetadata | null>(null);
	let message = $state('Initializing...');
	let startTime = $state<number>(0);
	let estimatedTimeRemaining = $state<number>(0);
	let isChromium = $state(false);
	let showAdvancedSettings = $state(false);
	let muteSound = $state(false);
	let audioOnlyMode = $state(false);
	let preserveOriginalFps = $state(false);

	const getOptimalThreadCount = (): number => {
		try {
			const cores = navigator.hardwareConcurrency || 2;
			const isIsolated = (globalThis as any).crossOriginIsolated === true;
			if (!isIsolated) return 1;
			const baseline = Math.max(1, cores - 1);
			const cap = 4;
			return Math.min(baseline, cap);
		} catch {
			return 1;
		}
	};

	const isChromiumByFeatures = (): boolean => {
		try {
			return !!(navigator as any).userAgentData;
		} catch {
			return false;
		}
	};

	const compressionTargets: CompressionTarget[] = [
		{ label: '8 MB', value: 8 * 1024 * 1024, description: 'Ultra compression' },
		{ label: '25 MB', value: 25 * 1024 * 1024, description: 'High compression' },
		{ label: '50 MB', value: 50 * 1024 * 1024, description: 'Medium compression' },
		{ label: '100 MB', value: 100 * 1024 * 1024, description: 'Low compression' }
	];
	let selectedTargetValue = $state('25 MB');
	let selectedTarget = $state(compressionTargets[1]);

	onMount(() => {
		// [优化] 立即启动加载并保存 Promise
		loadPromise = loadFFmpeg();
		isChromium = isChromiumByFeatures();
	});

	const loadFFmpeg = async (): Promise<void> => {
		try {
			ffmpeg = new FFmpeg();
			message = 'Loading ffmpeg-core.js';

			ffmpeg.on('log', ({ message: msg }: LogEvent) => {
				message = msg;
			});

			ffmpeg.on('progress', ({ progress: prog }: ProgressEvent) => {
				progress = Math.round(prog * 100);
				if (startTime > 0 && progress > 5) {
					const elapsed = (Date.now() - startTime) / 1000;
					const rate = progress / elapsed;
					if (rate > 0) {
						estimatedTimeRemaining = Math.round((100 - progress) / rate);
					}
				}
			});

			await ffmpeg.load({
				coreURL: await toBlobURL(`ffmpeg/ffmpeg-core.js`, 'text/javascript'),
				wasmURL: await toBlobURL(`ffmpeg/ffmpeg-core.wasm`, 'application/wasm'),
				workerURL: await toBlobURL(`ffmpeg/ffmpeg-core.worker.js`, 'text/javascript')
			});

			isLoaded = true;
			message = 'Ready to compress videos!';
		} catch (error) {
			console.error('Failed to load FFmpeg:', error);
			errorMessage = 'Failed to load FFmpeg. Please refresh the page.';
			message = 'Failed to load FFmpeg';
			throw error; // 抛出错误以通知 await 该 Promise 的逻辑 [cite: 25]
		}
	};

	const handleFileSelect = (event: Event): void => {
		const target = event.target as HTMLInputElement;
		const file = target.files?.[0];
		if (
			file &&
			(file.type.startsWith('video/') ||
				file.type === 'video/x-matroska' ||
				file.type === 'application/x-matroska' ||
				file.name.match(/\.(mp4|avi|mov|wmv|flv|webm|mkv|m4v|3gp|ogv)$/i))
		) {
			const maxSize = 5 * 1024 * 1024 * 1024;
			if (file.size > maxSize) {
				errorMessage = m.file_size_limit_error();
				target.value = '';
				return;
			}

			selectedFile = file;
			originalSize = file.size;
			errorMessage = '';
			processedVideo = null;
			getVideoMetadata(file); // 元数据获取不依赖 FFmpeg，可以立即运行
		} else {
			errorMessage = m.select_valid_video();
		}
	};

	const detectVideoFps = async (file: File): Promise<number> => {
		// [优化] 如果用户在加载完之前上传，等待加载完成再检测 FPS
		if (!isLoaded && loadPromise) {
			await loadPromise.catch(() => {});
		}

		if (!ffmpeg || !isLoaded) return 30;

		const inputName = `fps_detect_${Date.now()}.mp4`;
		try {
			await ffmpeg.writeFile(inputName, await fetchFile(file));
			let detectedFps = 30;
			let foundFps = false;
			const logHandler = ({ message: msg }: LogEvent) => {
				if (!foundFps) {
					const fpsMatch = msg.match(/,\s*(\d+\.?\d*)\s*fps/i);
					const tbrMatch = msg.match(/(\d+\.?\d*)\s*tbr/i);
					if (fpsMatch) {
						detectedFps = Math.round(parseFloat(fpsMatch[1]));
						foundFps = true;
					} else if (tbrMatch && !foundFps) {
						detectedFps = Math.round(parseFloat(tbrMatch[1]));
						foundFps = true;
					}
				}
			};
			ffmpeg.on('log', logHandler);
			try {
				await ffmpeg.exec(['-i', inputName, '-frames:v', '1', '-f', 'null', '-']);
			} catch (e) {}
			ffmpeg.off('log', logHandler);
			await ffmpeg.deleteFile(inputName);
			return detectedFps;
		} catch (error) {
			return 30;
		}
	};

	const getVideoMetadata = async (file: File): Promise<void> => {
		try {
			const video = document.createElement('video');
			video.src = URL.createObjectURL(file);
			await new Promise<void>((resolve) => {
				video.onloadedmetadata = () => {
					const estimatedBitrate = Math.round((file.size * 8) / video.duration / 1000);
					const pixelCount = video.videoWidth * video.videoHeight;
					const bitratePerPixel = (estimatedBitrate / pixelCount) * 1000;
					const hasMotion = bitratePerPixel > 0.1 || estimatedBitrate > 3000;

					videoMetadata = {
						duration: video.duration,
						bitrate: estimatedBitrate,
						resolution: `${video.videoWidth}x${video.videoHeight}`,
						codec: 'detecting',
						size: file.size,
						fps: 30,
						hasMotion
					};
					URL.revokeObjectURL(video.src);
					resolve();
				};
			});

			detectVideoFps(file).then((fps) => {
				if (videoMetadata) videoMetadata = { ...videoMetadata, fps };
			});
		} catch (error) {
			console.error('Metadata failed:', error);
		}
	};

	const calculateOptimalResolution = (w: number, h: number, max: number): string => {
		if (w <= max) return `${w}x${h}`;
		const ratio = w / h;
		const newW = max;
		const newH = Math.round(newW / ratio);
		return `${newW % 2 === 0 ? newW : newW - 1}x${newH % 2 === 0 ? newH : newH - 1}`;
	};

	const calculateCompressionSettings = (targetSize: number, metadata: VideoMetadata, preserveFps: boolean): CompressionSettings => {
		const efficiency = metadata.hasMotion ? 0.8 : 0.85;
		const targetBitrate = Math.round(((targetSize * 8) / metadata.duration / 1000) * efficiency);
		const audioBitrate = Math.min(128, Math.round(targetBitrate * 0.12));
		const videoBitrate = Math.max(200, targetBitrate - audioBitrate);

		let resolution = metadata.resolution;
		let crf = 23;
		let fpsCap = 30;
		const [width, height] = metadata.resolution.split('x').map(Number);

		if (targetSize <= 8 * 1024 * 1024) {
			const maxWidth = metadata.hasMotion ? 1024 : 854;
			if (width > maxWidth) resolution = calculateOptimalResolution(width, height, maxWidth);
			crf = metadata.hasMotion ? 18 : 26;
			fpsCap = 24;
		} else if (targetSize <= 25 * 1024 * 1024) {
			const maxWidth = metadata.hasMotion ? 1440 : 1280;
			if (width > maxWidth) resolution = calculateOptimalResolution(width, height, maxWidth);
			crf = metadata.hasMotion ? 16 : 24;
		}

		return {
			videoBitrate: `${videoBitrate}k`,
			audioBitrate: `${audioBitrate}k`,
			resolution,
			crf,
			preset: 'veryfast',
			tune: 'film',
			bufferSize: metadata.hasMotion ? `${videoBitrate * 3}k` : `${videoBitrate * 2}k`,
			refs: 1,
			bframes: 0,
			targetFps: preserveFps ? metadata.fps : Math.min(metadata.fps, fpsCap)
		};
	};

	const compressVideo = async (): Promise<void> => {
		if (!selectedFile || !videoMetadata) return;

		isProcessing = true;
		progress = 0;
		errorMessage = '';
		startTime = Date.now();

		try {
			// [核心优化] 如果引擎还没加载完，在此排队等待
			if (!isLoaded && loadPromise) {
				message = 'Waiting for engine to initialize...';
				await loadPromise;
			}

			if (!ffmpeg) throw new Error('FFmpeg failed to initialize');

			const inputDir = '/input';
			await ffmpeg.createDir(inputDir);
			message = 'Mounting input file...';
			await ffmpeg.mount('WORKERFS' as any, { files: [selectedFile] }, inputDir);

			if (audioOnlyMode) {
				message = 'Processing audio only...';
				const args = ['-i', `${inputDir}/${selectedFile.name}`, '-c:v', 'copy'];
				if (muteSound) args.push('-an');
				else args.push('-c:a', 'copy');
				args.push('-movflags', '+faststart', '-f', 'mp4', '-y', 'output.mp4');
				await ffmpeg.exec(args);
			} else {
				const settings = calculateCompressionSettings(selectedTarget.value, videoMetadata, preserveOriginalFps);
				const threadCount = isChromium ? getOptimalThreadCount() : 0;
				message = 'Starting compression...';

				const args = [
					'-i', `${inputDir}/${selectedFile.name}`,
					'-c:v', 'libx264',
					'-preset', 'veryfast',
					'-tune', 'film',
					'-crf', settings.crf.toString(),
					'-maxrate', settings.videoBitrate,
					'-bufsize', settings.bufferSize,
					'-threads', threadCount.toString()
				];

				if (!muteSound) args.push('-c:a', 'aac', '-b:a', settings.audioBitrate, '-ac', '2');
				else args.push('-an');

				let vFilters: string[] = [];
				if (settings.resolution !== videoMetadata.resolution) vFilters.push(`scale=${settings.resolution}:flags=fast_bilinear`);
				if (settings.targetFps < videoMetadata.fps) vFilters.push(`fps=${settings.targetFps}`);

				if (vFilters.length > 0) args.splice(args.indexOf('-i') + 2, 0, '-vf', vFilters.join(','));
				args.push('-movflags', '+faststart', '-f', 'mp4', '-y', 'output.mp4');
				await ffmpeg.exec(args);
			}

			const data = (await ffmpeg.readFile('output.mp4')) as Uint8Array;
			processedVideo = data;
			compressedSize = data.length;
			await ffmpeg.unmount(inputDir);
			await ffmpeg.deleteDir(inputDir);
			await ffmpeg.deleteFile('output.mp4');
			message = 'Completed successfully!';
		} catch (error) {
			errorMessage = 'Compression failed. Please try again.';
			message = 'Failed';
		} finally {
			isProcessing = false;
			progress = 0;
			startTime = 0;
		}
	};

	const downloadVideo = (): void => {
		if (!processedVideo) return;
		const blob = new Blob([new Uint8Array(processedVideo)], { type: 'video/mp4' });
		const url = URL.createObjectURL(blob);
		const a = document.createElement('a');
		const filename = `compressed_${selectedTarget.label}_${selectedFile?.name || 'video.mp4'}`;
		a.download = filename;
		a.href = url;
		a.click();
		URL.revokeObjectURL(url);
	};

	const formatFileSize = (b: number) => {
		if (b === 0) return '0 B';
		const i = Math.floor(Math.log(b) / Math.log(1024));
		return (b / Math.pow(1024, i)).toFixed(2) + ' ' + ['B', 'KB', 'MB', 'GB'][i];
	};

	const compressionRatio = $derived(compressedSize > 0 && originalSize > 0 ? (1 - compressedSize / originalSize) * 100 : 0);
	const isFileSmallerThanTarget = $derived(!!selectedTarget && originalSize > 0 && originalSize < selectedTarget.value);

	const handleTargetChange = (val: string | undefined) => {
		if (!val) return;
		selectedTargetValue = val;
		const t = compressionTargets.find((t) => t.label === val);
		if (t) selectedTarget = t;
	};
</script>

<svelte:head>
	<title>{m.app_title()} - {m.app_subtitle()}</title>
</svelte:head>

<div class="container mx-auto max-w-4xl p-6">
	<div class="mb-2 flex items-center justify-center gap-2">
		<h1 class="mr-4 mb-2 text-4xl font-bold">{m.app_title()}</h1>
		<LanguageSelector />
		<ThemeSelector />
	</div>

	<div class="mb-8 text-center text-muted-foreground">
		<p>{m.app_subtitle()}</p>
	</div>

	{#if !isLoaded && !isProcessing}
		<div class="mb-6 flex items-center justify-center gap-2 rounded-lg bg-accent/50 p-3">
			<Loader class="h-4 w-4 animate-spin text-primary" />
			<span class="text-sm font-medium">Downloading engine (WASM)...</span>
		</div>
	{/if}

	{#if errorMessage}
		<Alert.Root class="mb-6 border-destructive">
			<Alert.Description>{errorMessage}</Alert.Description>
		</Alert.Root>
	{/if}

	<div class="grid grid-cols-1 gap-6 lg:grid-cols-2">
		<Card.Root>
			<Card.Header>
				<Card.Title>{m.upload_video()}</Card.Title>
				<Card.Description>{m.upload_description()}</Card.Description>
			</Card.Header>
			<Card.Content class="space-y-4">
				<div>
					<Label for="video-upload">{m.choose_video_file()}</Label>
					<Input
						id="video-upload"
						type="file"
						accept="video/*"
						onchange={handleFileSelect}
						disabled={false}
						class="mt-2"
					/>
				</div>

				{#if videoMetadata}
					<div class="space-y-2 rounded-md border p-3">
						<h4 class="text-sm font-semibold">{m.video_information()}</h4>
						<div class="grid grid-cols-2 gap-2 text-xs">
							<div>Resolution: {videoMetadata.resolution}</div>
							<div>FPS: {videoMetadata.fps}</div>
							<div class="col-span-2">Size: {formatFileSize(videoMetadata.size)}</div>
						</div>
					</div>
				{/if}

				<div>
					<Label>{m.target_size()}</Label>
					<Select.Root type="single" value={selectedTargetValue} onValueChange={handleTargetChange}>
						<Select.Trigger class="mt-2 w-full">
							{selectedTargetValue || m.select_target_size()}
						</Select.Trigger>
						<Select.Content>
							{#each compressionTargets as target}
								<Select.Item value={target.label}>{target.label}</Select.Item>
							{/each}
						</Select.Content>
					</Select.Root>
				</div>

				<Button
					onclick={compressVideo}
					disabled={!selectedFile || isProcessing}
					class="w-full"
				>
					{#if isProcessing}
						<Loader class="mr-2 h-4 w-4 animate-spin" />
						{message.length > 25 ? message.substring(0, 25) + '...' : message}
					{:else}
						{audioOnlyMode ? m.process_audio_only() : m.compress_video()}
					{/if}
				</Button>

				{#if isProcessing && progress > 0}
					<div class="space-y-2">
						<div class="flex justify-between text-xs font-medium">
							<span>Processing</span>
							<span>{progress}%</span>
						</div>
						<Progress value={progress} class="h-2 w-full" />
					</div>
				{/if}
			</Card.Content>
		</Card.Root>

		<Card.Root>
			<Card.Header>
				<Card.Title>{m.results()}</Card.Title>
			</Card.Header>
			<Card.Content class="space-y-4">
				{#if processedVideo}
					<div class="space-y-3">
						<div class="flex justify-between text-sm">
							<span>Compressed:</span>
							<Badge variant={compressedSize <= selectedTarget.value ? 'default' : 'destructive'}>
								{formatFileSize(compressedSize)}
							</Badge>
						</div>
						<div class="flex justify-between text-sm">
							<span>Reduction:</span>
							<Badge variant="outline">{compressionRatio.toFixed(1)}%</Badge>
						</div>
						<Button onclick={downloadVideo} class="w-full">
							{m.download_compressed()}
						</Button>
					</div>
				{:else}
					<div class="py-12 text-center text-sm text-muted-foreground italic">
						{m.upload_compress_message()}
					</div>
				{/if}
			</Card.Content>
		</Card.Root>
	</div>

    <footer class="mt-12 text-center text-xs text-muted-foreground">
		<p>{@html m.footer_text()}</p>
	</footer>
</div>