<script lang="ts">
  import { open } from '@tauri-apps/api/dialog';
  import { readTextFile } from '@tauri-apps/api/fs';
  import { dirname, join } from '@tauri-apps/api/path';
  import { convertFileSrc } from '@tauri-apps/api/tauri';
  import { onMount } from 'svelte';
  import LottiePlayer from './components/LottiePlayer.svelte';

  let lottiePlayer: LottiePlayer;
  let button: HTMLButtonElement;
  let storyPath: string;
  let forcedSize;
  onMount(() => {
    document.body.onresize = function (ev: UIEvent) {
      forcedSize = Math.min(window.innerWidth, window.innerHeight - button.clientHeight * 2);
    };
    document.body.onresize(null);
  });
  let audioPlayer: HTMLAudioElement;

  let defaultPath: string = null;
  let filter: string = null;
  let multiple = false;
  let directory = false;

  async function getOrCreateAudioPlayer() {
    if (storyPath) {
      const audioPath = await join(storyPath, 'audio.mp3');
      console.log('play', audioPath);
      audioPlayer = new Audio(convertFileSrc(audioPath));
    }
  }

  let playing = false;
  async function onPlay() {
    playing = true;
    if (!audioPlayer) {
      await getOrCreateAudioPlayer();
    }
    if (audioPlayer) {
      audioPlayer.play();
    }
  }
  function onPause() {
    playing = false;
    console.log('pause');
    if (audioPlayer) {
      audioPlayer.pause();
    }
  }
  function onComplete() {
    console.log('pause');
    if (audioPlayer) {
      audioPlayer.pause();
      audioPlayer.currentTime = 0;
    }
  }
  function onSeek(event) {
    console.log('seek', event);
    if (audioPlayer) {
      try {
        const progress = event.detail.frame / event.detail.totalFrames;

        audioPlayer.currentTime = progress * audioPlayer.duration;
        console.log(
          'seek',
          event.detail.frame,
          event.detail.totalFrames,
          progress,
          audioPlayer.duration,
          audioPlayer.currentTime
        );
      } catch (error) {
        console.error('currentTime', error);
      }
    }
  }

  async function prepareLottie(filePath) {
    const data = JSON.parse(await readTextFile(filePath));
    storyPath = await dirname(filePath);
    const storyImagesFolder = convertFileSrc(await join(storyPath, 'images'));
    const assets = {};
    const assetsToAdd = [];
    data.w = 228;
    data.h = 192;
    data.assets.forEach((a) => {
      assets[a.id] = a.layers;
      a.layers.forEach((layer) => {
        const cleaned = layer.nm.split('.')[0];
        if (!assets[layer.refId]) {
          const asset = (assets[layer.refId] = {
            id: layer.refId,
            w: 228,
            h: 192,
            e: 0,
            u: storyImagesFolder,
            p: '/' + cleaned + '.png',
          });
          assetsToAdd.push(asset);
        }
      });
    });

    data.layers.forEach((layer) => {
      if (!assets[layer.refId]) {
        const cleaned = layer.nm.split('.')[0];
        const asset = (assets[layer.refId] = {
          id: layer.refId,
          w: 228,
          h: 192,
          e: 0,
          u: storyImagesFolder,
          p: '/' + cleaned + '.png',
        });
        assetsToAdd.push(asset);
        // }
      }
    });
    data.assets.push(...assetsToAdd);
    getOrCreateAudioPlayer();
    return data;
  }

  async function openDialog() {
    try {
      const res = await open({
        defaultPath,
        filters: filter
          ? [
              {
                name: 'Tauri Example',
                extensions: filter.split(',').map((f) => f.trim()),
              },
            ]
          : [],
        multiple,
        directory,
      });
      console.log('test', res);

      const data = await prepareLottie(res);
      console.log(data);
      lottiePlayer.load(JSON.stringify(data));
    } catch (error) {
      console.error(error);
    }
  }
</script>

<div
  style={`overflow:hidden; display: flex; flex-flow: column;width=${forcedSize}px; height=${forcedSize}px;`}
>
  <button bind:this={button} type="button" id="open-dialog" on:click={openDialog}
    >Selectionner le fichier de composition</button
  >
  <LottiePlayer
    bind:this={lottiePlayer}
    autoplay={false}
    loop={false}
    controls={true}
    width={forcedSize}
    height={forcedSize}
    renderer="svg"
    style="display: flex; flex-flow: column;margin: 0 auto;"
    background="transparent"
    controlsLayout={[
      'previousFrame',
      'playpause',
      'stop',
      'nextFrame',
      'progress',
      'frame',
      'spacer',
      'snapshot',
      'info',
      'zoom',
    ]}
    on:play={onPlay}
    on:seek={onSeek}
    on:pause={onPause}
    on:complete={onComplete}
  />
</div>
