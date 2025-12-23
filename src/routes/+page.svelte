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
    import * as Alert from '$lib/components/ui/alert/index.js';
    import Loader from '@lucide/svelte/icons/loader-circle';
    import Zap from '@lucide/svelte/icons/zap';
    import ShieldCheck from '@lucide/svelte/icons/shield-check';
    import AlertTriangle from '@lucide/svelte/icons/alert-triangle';
    import FileVideo from '@lucide/svelte/icons/file-video-2';
    import Download from '@lucide/svelte/icons/download';

    // Types
    interface CompressionTarget {
       label: string;
       value: number;
    }

    // State
    let ffmpeg = $state<FFmpeg | null>(null);
    let isLoaded = $state(false);
    let engineLoadPromise = $state<Promise<void> | null>(null);
    let isProcessing = $state(false);
    let progress = $state(0);
    let selectedFile = $state<File | null>(null);
    let processedVideo = $state<Uint8Array | null>(null);
    let originalSize = $state(0);
    let compressedSize = $state(0);
    let errorMessage = $state('');
    let statusMessage = $state('Ready');
    let videoDuration = $state(0);

    const compressionTargets: CompressionTarget[] = [
       { label: '8 MB', value: 8 * 1024 * 1024 },
       { label: '25 MB', value: 25 * 1024 * 1024 },
       { label: '50 MB', value: 50 * 1024 * 1024 }
    ];
    let selectedTargetValue = $state('25 MB');
    let selectedTarget = $derived(compressionTargets.find(t => t.label === selectedTargetValue) || compressionTargets[1]);

    onMount(() => {
        engineLoadPromise = initFFmpeg();
    });

    const initFFmpeg = async () => {
       try {
          if (typeof SharedArrayBuffer === 'undefined') {
             errorMessage = "Browser Error: SharedArrayBuffer is missing. Please use Chrome desktop or ensure COOP/COEP headers are set.";
             throw new Error("SharedArrayBuffer missing");
          }

          const _ffmpeg = new FFmpeg();

          _ffmpeg.on('log', ({ message }) => console.log('FFmpeg:', message));

          _ffmpeg.on('progress', ({ progress: prog }: ProgressEvent) => {
             // 算法：当前进度 + (FFmpeg进度 * 剩余可用空间)
             // 这样无论模拟进度到了多少，都会平滑地向100%靠拢，绝不会溢出或倒退
             const currentP = progress;
             if (currentP >= 100) return;

             const nextP = Math.round(currentP + (prog * (100 - currentP)));
             if (nextP > progress && nextP < 100) {
                progress = nextP;
             }
          });

          const baseURL = 'https://unpkg.com/@ffmpeg/core@0.12.6/dist/esm';
          await _ffmpeg.load({
             coreURL: await toBlobURL(`${baseURL}/ffmpeg-core.js`, 'text/javascript'),
             wasmURL: await toBlobURL(`${baseURL}/ffmpeg-core.wasm`, 'application/wasm'),
          });

          ffmpeg = _ffmpeg;
          isLoaded = true;
       } catch (error: any) {
          console.error('Init Error:', error);
          if (!errorMessage) errorMessage = "Engine failed to load. Please refresh or try Chrome Desktop.";
          throw error;
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

    const simulateLoading = async () => {
        const steps = [5, 15, 30, 45, 60, 70, 75];
        for (const step of steps) {
            if (isLoaded || !isProcessing) break;
            if (step > progress) progress = step;
            statusMessage = "Initializing Engine...";
            await new Promise(r => setTimeout(r, 400));
        }
    };

    const compressVideo = async () => {
       if (!selectedFile) return;

       // 重置状态：解决连续压缩进度超过100%的关键
       isProcessing = true;
       progress = 0;
       errorMessage = '';
       processedVideo = null;
       statusMessage = 'Preparing...';

       try {
          if (!isLoaded && engineLoadPromise) {
             simulateLoading();
             await engineLoadPromise;
          }

          if (!ffmpeg) throw new Error("Engine not ready");

          const inputName = 'input.mp4';
          const outputName = 'output.mp4';

          await ffmpeg.writeFile(inputName, await fetchFile(selectedFile));

          const targetBitrateKbps = Math.floor(((selectedTarget.value * 8) / (videoDuration || 1) / 1000) * 0.88);
          const videoBitrate = Math.max(120, targetBitrateKbps - 64);

          statusMessage = 'Optimizing for Discord...';

          await ffmpeg.exec([
             '-i', inputName,
             '-c:v', 'libx264',
             '-b:v', `${videoBitrate}k`,
             '-preset', 'ultrafast',
             '-profile:v', 'baseline',
             '-level', '3.0',
             '-vf', 'scale=iw*min(1\\,720/iw):-2,format=yuv420p',
             '-c:a', 'aac', '-b:a', '64k', '-ac', '2',
             '-movflags', '+faststart',
             outputName
          ]);

          const data = (await ffmpeg.readFile(outputName)) as Uint8Array;
          processedVideo = data;
          compressedSize = data.length;
          statusMessage = 'Done!';
          progress = 100;

          await ffmpeg.deleteFile(inputName);
          await ffmpeg.deleteFile(outputName);

       } catch (e: any) {
          console.error('Task Error:', e);
          errorMessage = isLoaded
            ? "Compression failed. File might be too complex for browser memory."
            : "Engine failed to initialize. Please check your connection.";
          isProcessing = false;
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

<div class="min-h-screen bg-[#F2F3F5] dark:bg-[#2B2D31] text-[#2E3338] dark:text-[#DBDEE1] p-4 md:p-12 font-sans transition-colors">
    <header class="max-w-5xl mx-auto flex justify-between items-center mb-12">
       <div class="flex items-center gap-3 group cursor-default">
          <div class="bg-[#5865F2] p-2.5 rounded-2xl shadow-lg shadow-indigo-500/20 rotate-[-4deg] group-hover:rotate-0 transition-all">
             <Zap class="text-white h-6 w-6 fill-current" />
          </div>
          <h1 class="text-2xl font-black tracking-tighter text-[#5865F2] dark:text-white uppercase">Discord Compressor</h1>
       </div>
    </header>

    <main class="max-w-5xl mx-auto grid grid-cols-1 lg:grid-cols-12 gap-8">
       <div class="lg:col-span-7 space-y-6">
          <Card.Root class="border-none shadow-xl bg-white dark:bg-[#313338] rounded-[32px] overflow-hidden">
             <Card.Header class="px-8 pt-8 pb-0">
                 <Card.Title class="text-xl font-black uppercase tracking-tight italic">
                    Optimize Video
                 </Card.Title>
             </Card.Header>
             <Card.Content class="p-8 md:p-10 space-y-8">
                <div class="relative group">
                   <input type="file" accept="video/*" onchange={handleFileSelect} class="absolute inset-0 w-full h-full opacity-0 cursor-pointer z-10" />
                   <div class="border-2 border-dashed border-slate-200 dark:border-[#4E5058] rounded-[24px] py-16 text-center transition-all group-hover:bg-slate-50 dark:group-hover:bg-[#383A40] group-hover:border-[#5865F2]">
                      {#if selectedFile}
                         <div class="flex flex-col items-center px-6">
                            <div class="bg-[#5865F2] p-4 rounded-full mb-4 shadow-lg shadow-indigo-500/30">
                               <FileVideo class="text-white h-8 w-8" />
                            </div>
                            <p class="font-bold dark:text-white truncate max-w-full italic">{selectedFile.name}</p>
                            <Badge variant="secondary" class="mt-2 bg-slate-100 dark:bg-[#1E1F22] text-[#5865F2] font-black italic uppercase tracking-widest text-[10px]">
                               Original: {(originalSize / 1024 / 1024).toFixed(2)} MB
                            </Badge>
                         </div>
                      {:else}
                         <div class="space-y-3">
                            <div class="mx-auto w-14 h-14 bg-slate-100 dark:bg-[#383A40] rounded-full flex items-center justify-center mb-4">
                               <ShieldCheck class="text-slate-400 w-7 h-7" />
                            </div>
                            <p class="text-lg font-black dark:text-white uppercase italic tracking-tighter">Upload Video</p>
                            <p class="text-[10px] text-slate-400 font-bold uppercase tracking-[0.2em]">Private Browser Processing</p>
                         </div>
                      {/if}
                   </div>
                </div>

                <div class="grid grid-cols-1 md:grid-cols-2 gap-6">
                   <div class="space-y-2">
                      <Label class="text-[10px] font-black uppercase text-slate-400 tracking-[0.2em] ml-1">Target Size</Label>
                      <Select.Root type="single" value={selectedTargetValue} onValueChange={(v) => { if(v) selectedTargetValue = v; }}>
                         <Select.Trigger class="w-full h-14 rounded-2xl border-none bg-slate-100 dark:bg-[#1E1F22] font-black text-lg focus:ring-2 ring-[#5865F2] transition-all">
                            {selectedTargetValue}
                         </Select.Trigger>
                         <Select.Content class="rounded-xl border-none shadow-2xl">
                            {#each compressionTargets as t}
                               <Select.Item value={t.label} class="font-bold py-3">{t.label}</Select.Item>
                            {/each}
                         </Select.Content>
                      </Select.Root>
                   </div>

                   <div class="flex items-end">
                      <Button
                         onclick={compressVideo}
                         disabled={!selectedFile || isProcessing}
                         class="w-full h-14 rounded-2xl font-black text-lg bg-[#5865F2] hover:bg-[#4752C4] text-white shadow-lg shadow-indigo-500/20 transition-all active:scale-95 italic disabled:opacity-50"
                      >
                         {#if isProcessing}
                            <Loader class="mr-2 h-6 w-6 animate-spin" />
                            {progress}%
                         {:else}
                            OPTIMIZE NOW
                         {/if}
                      </Button>
                   </div>
                </div>

                {#if isProcessing}
                   <div class="space-y-3 pt-2">
                      <Progress value={progress} class="h-3 rounded-full bg-slate-100 dark:bg-[#1E1F22]" />
                      <p class="text-[10px] text-center font-black animate-pulse uppercase tracking-[0.3em] text-[#5865F2]">
                         {statusMessage}
                      </p>
                   </div>
                {/if}

                {#if errorMessage}
                   <Alert.Root variant="destructive" class="rounded-2xl border-none bg-red-50 dark:bg-red-950/20 text-red-600 animate-in fade-in slide-in-from-top-2">
                      <Alert.Description class="font-bold text-xs flex flex-col gap-2 uppercase tracking-tight">
                         <div class="flex items-center gap-2">
                            <AlertTriangle class="h-4 w-4 shrink-0" /> Error Detected
                         </div>
                         <span class="normal-case font-medium opacity-80">{errorMessage}</span>
                      </Alert.Description>
                   </Alert.Root>
                {/if}
             </Card.Content>
          </Card.Root>

          {#if processedVideo}
             <div class="bg-[#23A559] p-6 rounded-[28px] flex flex-col md:flex-row items-center justify-between gap-6 shadow-xl shadow-green-500/10 animate-in zoom-in-95">
                <div class="text-white text-center md:text-left">
                   <p class="text-[10px] font-black uppercase tracking-[0.2em] opacity-80 mb-1">Compression Done</p>
                   <p class="text-3xl font-black italic">{(compressedSize / 1024 / 1024).toFixed(2)} MB</p>
                </div>
                <Button onclick={downloadVideo} size="lg" class="w-full md:w-auto rounded-2xl px-12 bg-white text-[#23A559] hover:bg-slate-100 font-black italic shadow-lg gap-2">
                   <Download class="h-5 w-5" /> DOWNLOAD
                </Button>
             </div>
          {/if}
       </div>

       <div class="lg:col-span-5 space-y-8">
          <section class="bg-white dark:bg-[#313338] p-8 rounded-[32px] shadow-sm">
             <h3 class="text-[11px] font-black uppercase tracking-[0.2em] text-[#5865F2] mb-6">Discord Limits 2025</h3>
             <div class="space-y-5 font-bold">
                <div class="flex justify-between items-center text-sm border-b border-slate-50 dark:border-[#383A40] pb-4">
                   <span class="text-slate-500">Free Users</span>
                   <Badge variant="outline" class="rounded-lg border-slate-200 dark:border-[#4E5058] font-black">25 MB</Badge>
                </div>
                <div class="flex justify-between items-center text-sm border-b border-slate-50 dark:border-[#383A40] pb-4">
                   <span class="text-slate-500">Nitro Basic</span>
                   <Badge variant="outline" class="rounded-lg border-indigo-200 dark:border-[#5865F2] text-[#5865F2] font-black">50 MB</Badge>
                </div>
                <div class="flex justify-between items-center text-sm">
                   <span class="text-slate-500">Nitro Pro</span>
                   <Badge class="bg-gradient-to-r from-pink-500 to-indigo-500 border-none font-black text-white">500 MB</Badge>
                </div>
             </div>
          </section>

          <section class="bg-[#5865F2] p-8 rounded-[32px] text-white shadow-xl shadow-indigo-500/20">
             <h3 class="font-black text-xl mb-4 italic uppercase tracking-tighter">Zero-Upload Privacy</h3>
             <p class="text-sm opacity-90 leading-relaxed font-medium">
                Unlike other compressors, we don't own servers. Your video is processed in your **RAM**. Once you close the tab, the video is gone forever.
             </p>
          </section>
       </div>
    </main>

    <section class="max-w-5xl mx-auto mt-32 space-y-16 border-t border-slate-200 dark:border-[#4E5058] pt-20">
       <h2 class="text-5xl font-black text-center tracking-tighter italic uppercase text-slate-800 dark:text-white">FAQ</h2>

       <div class="grid grid-cols-1 md:grid-cols-2 gap-x-12 gap-y-16">
          <div class="space-y-4">
             <h4 class="font-black text-[#5865F2] uppercase text-xs tracking-[0.2em]">How to compress video to 8MB?</h4>
             <p class="text-sm text-slate-500 dark:text-slate-400 leading-relaxed font-medium">
                Discord recently increased the limit to 25MB, but some servers still prefer smaller files. Select "8 MB" in our settings, and we'll handle the complex bitrate calculations to ensure it fits perfectly without crashing the Discord mobile app.
             </p>
          </div>
          <div class="space-y-4">
             <h4 class="font-black text-[#5865F2] uppercase text-xs tracking-[0.2em]">Why is my file still too big?</h4>
             <p class="text-sm text-slate-500 dark:text-slate-400 leading-relaxed font-medium">
                If the video is extremely long (e.g., over 10 minutes), even high compression might struggle to reach 8MB while maintaining quality. We suggest trimming videos to under 3 minutes for the best results on Discord.
             </p>
          </div>
          <div class="space-y-4">
             <h4 class="font-black text-[#5865F2] uppercase text-xs tracking-[0.2em]">Does it work on Mobile?</h4>
             <p class="text-sm text-slate-500 dark:text-slate-400 leading-relaxed font-medium">
                Yes! Our tool works on Chrome and Safari for iOS and Android. It uses your phone's internal processor. Note that older phones might struggle with high-resolution files.
             </p>
          </div>
          <div class="space-y-4">
             <h4 class="font-black text-[#5865F2] uppercase text-xs tracking-[0.2em]">Is it really free?</h4>
             <p class="text-sm text-slate-500 dark:text-slate-400 leading-relaxed font-medium">
                Absolutely. There are no subscriptions, hidden fees, or watermarks. By processing everything locally, we eliminate server costs and pass that benefit directly to you.
             </p>
          </div>
       </div>
    </section>

    <footer class="mt-32 pb-12 text-center text-[10px] font-black uppercase tracking-[0.5em] text-slate-400">
       © 2025 DiscordCompressor.com | Browser-Based Optimization
    </footer>
</div>

<style>
    :global(html) {
       scroll-behavior: smooth;
       -webkit-tap-highlight-color: transparent;
    }
</style>