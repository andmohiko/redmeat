<!-- CameraComponent.svelte -->
<script lang="ts">
  import { onMount } from 'svelte';

  let video: HTMLVideoElement;
  let canvas: HTMLCanvasElement;
  let redPercentage = 0;
  let stopProcessing = false;

  onMount(async () => {
    await getVideoDevices();
    await startVideo();
  });

  $: if (selectedDeviceId) {
    startVideo();
  }

  async function startVideo() {
    try {
      const stream = await navigator.mediaDevices.getUserMedia({ video: { deviceId: selectedDeviceId ? { exact: selectedDeviceId } : undefined, facingMode: selectedDeviceId ? undefined : 'environment' } });
      video.srcObject = stream;
      video.addEventListener('loadedmetadata', () => {
        video.play();
        // ビデオのメタデータが読み込まれた後にキャンバスのサイズを設定
        canvas.width = video.videoWidth;
        canvas.height = video.videoHeight;
      });
    } catch (err) {
      console.error("カメラへのアクセスが拒否されました:", err);
    }
  }


  function processFrame() {
    if (stopProcessing) return;
    const context = canvas.getContext('2d');
    if (!context) {
      console.error('2D コンテキストの取得に失敗しました');
      return;
    }
    context.drawImage(video, 0, 0, canvas.width, canvas.height);
    highlightRedAreas();
    calculateRedPercentage();
    requestAnimationFrame(processFrame);
  }

  function stopFrameProcessing() {
    stopProcessing = true;
    const context = canvas.getContext('2d');
    if (!context) {
      console.error('2D コンテキストの取得に失敗しました');
      return;
    }
    // 画面を初期状態に戻すためにキャンバスをクリア
    context.clearRect(0, 0, canvas.width, canvas.height);
    redPercentage = 0;
  }

  function calculateRedPercentage() {
    const context = canvas.getContext('2d');
    if (!context) {
      console.error('2D コンテキストの取得に失敗しました');
      return;
    }
    const imageData = context.getImageData(0, 0, canvas.width, canvas.height);
    const data = imageData.data;

    let redCount = 0;
    for (let i = 0; i < data.length; i += 4) {
      const r = data[i];
      const g = data[i + 1];
      const b = data[i + 2];

      if (isRed(r, g, b)) {
        redCount++;
      }
    }

    redPercentage = (redCount / (data.length / 4)) * 100;
  }

  function highlightRedAreas() {
    const context = canvas.getContext('2d');
    if (!context) {
      console.error('2D コンテキストの取得に失敗しました');
      return;
    }
    const imageData = context.getImageData(0, 0, canvas.width, canvas.height);
    const data = imageData.data;

    for (let i = 0; i < data.length; i += 4) {
      const r = data[i];
      const g = data[i + 1];
      const b = data[i + 2];

      if (isRed(r, g, b)) {
        // 強調表示するために赤色をさらに強くし、他の色を抑える
        data[i] = 255;     // 赤
        data[i + 1] = 0;   // 緑
        data[i + 2] = 0;   // 青
      } else {
        // 赤色でない部分は少し暗くする
        data[i] *= 0.9;
        data[i + 1] *= 0.9;
        data[i + 2] *= 0.9;
      }
    }

    context.putImageData(imageData, 0, 0);
  }

  function isRed(r: number, g: number, b: number) {
    return r > 100 && g <= 70 && b <= 70;
  }

  let devices: MediaDeviceInfo[] = [];
  let selectedDeviceId = '';

  async function getVideoDevices() {
    try {
      const deviceInfos = await navigator.mediaDevices.enumerateDevices();
      devices = deviceInfos.filter(device => device.kind === 'videoinput');
      if (devices.length > 0) {
        // デフォルトで背面カメラを選択するように試みる
        const backCamera = devices.find(device => device.label.toLowerCase().includes('back'));
        selectedDeviceId = backCamera ? backCamera.deviceId : devices[0].deviceId;
      }
    } catch (err) {
      console.error("デバイスの取得に失敗しました:", err);
    }
  }

  function handleDeviceChange(event: Event) {
    selectedDeviceId = (event.target as HTMLSelectElement).value;
    stopFrameProcessing();
    startVideo().then(() => {
      stopProcessing = false;
      requestAnimationFrame(processFrame);
    });
  }

</script>

<video bind:this={video} autoplay playsinline style="object-fit: cover; width: 100vw; height: 100vh;">
  <track kind="captions" srclang="en" label="English captions">
</video>
<select on:change={handleDeviceChange} style="position: absolute; top: 10px; left: 300px; z-index: 2;">
  {#each devices as device}
    <option value={device.deviceId} selected={device.deviceId === selectedDeviceId}>{device.label || 'カメラ ' + (devices.indexOf(device) + 1)}</option>
  {/each}
</select>
<canvas bind:this={canvas} style="position: absolute; top: 0; left: 0; width: 100vw; height: 100vh; z-index: 1;"></canvas>
<button on:click={() => requestAnimationFrame(processFrame)} style="position: absolute; top: 10px; left: 10px; z-index: 2;">赤色の割合を計算</button>
<button on:click={stopFrameProcessing} style="position: absolute; top: 10px; left: 150px; z-index: 2;">赤色の検知を停止</button>
<p style="position: absolute; top: 50px; left: 10px; z-index: 2; background: rgba(255, 255, 255, 0.7); padding: 5px;">赤色の割合: {redPercentage.toFixed(2)}%</p>

<style>
  :global(body) {
    margin: 0;
    overflow: hidden;
  }
</style>
