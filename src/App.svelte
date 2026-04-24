<script lang="ts">
  import { onDestroy, onMount, tick } from 'svelte'

  type Mode = 'photo' | 'video'
  type Facing = 'user' | 'environment'
  type BootState = 'loading' | 'ready' | 'blocked' | 'nocam' | 'error'

  type Capture = {
    type: Mode
    blob: Blob
    url: string
    filename: string
    mime: string
  }

  const SETTINGS_KEY = 'nxr-cam-settings-v1'
  const VIDEO_LIMITS = [15, 20, 30]

  let mode: Mode = $state('photo')
  let facing: Facing = $state('user')
  let mirrorPreview = $state(true)
  let audioEnabled = $state(false)
  let soundEnabled = $state(false)
  let preferFullscreen = $state(true)
  let maxVideoSeconds = $state(20)

  let bootState: BootState = $state('loading')
  let message = $state('Starting camera...')
  let note = $state('')
  let noteTimeout: number | undefined
  let privacyVisible = $state(false)
  let privacyTimeout: number | undefined

  let settingsOpen = $state(false)
  let reviewOpen = $state(false)
  let shutterRingActive = $state(false)

  let isRecording = $state(false)
  let recordingSeconds = $state(0)

  let lastCapture: Capture | null = $state(null)

  let videoEl: HTMLVideoElement | null = null
  let stream: MediaStream | null = $state(null)
  let recorder: MediaRecorder | null = null
  let recordedChunks: Blob[] = []

  let videoInputs: MediaDeviceInfo[] = $state([])
  let currentDeviceId = ''

  let recordingInterval: number | undefined
  let autoStopTimeout: number | undefined
  let fallbackMimeUsed = $state(false)

  let wakeLock: any = null
  let fullscreenActive = $state(false)

  let zoomSupported = $state(false)
  let zoomMin = $state(1)
  let zoomMax = $state(1)
  let zoomStep = $state(0.1)
  let zoomValue = $state(1)
  let pinchStartDistance = $state(0)
  let pinchStartZoom = $state(1)

  function setNote(text: string, timeout = 2400) {
    note = text
    window.clearTimeout(noteTimeout)
    noteTimeout = window.setTimeout(() => {
      note = ''
    }, timeout)
  }

  function loadSettings() {
    try {
      const raw = localStorage.getItem(SETTINGS_KEY)
      if (!raw) return
      const parsed = JSON.parse(raw)

      if (parsed.mode === 'photo' || parsed.mode === 'video') mode = parsed.mode
      if (parsed.facing === 'user' || parsed.facing === 'environment') facing = parsed.facing
      if (typeof parsed.mirrorPreview === 'boolean') mirrorPreview = parsed.mirrorPreview
      if (typeof parsed.audioEnabled === 'boolean') audioEnabled = parsed.audioEnabled
      if (typeof parsed.soundEnabled === 'boolean') soundEnabled = parsed.soundEnabled
      if (typeof parsed.preferFullscreen === 'boolean') preferFullscreen = parsed.preferFullscreen
      if (VIDEO_LIMITS.includes(parsed.maxVideoSeconds)) maxVideoSeconds = parsed.maxVideoSeconds
      if (typeof parsed.currentDeviceId === 'string') currentDeviceId = parsed.currentDeviceId
    } catch {
      setNote('Could not restore settings')
    }
  }

  $effect(() => {
    try {
      localStorage.setItem(
        SETTINGS_KEY,
        JSON.stringify({
          mode,
          facing,
          mirrorPreview,
          audioEnabled,
          soundEnabled,
          preferFullscreen,
          maxVideoSeconds,
          currentDeviceId,
        })
      )
    } catch {
      // no-op
    }
  })

  function timestamp() {
    return new Date().toISOString().replace(/[:.]/g, '-')
  }

  function filenameFor(kind: Mode, mime: string) {
    const ext = mime.includes('mp4') ? 'mp4' : mime.includes('webm') ? 'webm' : kind === 'photo' ? 'jpg' : 'mp4'
    return `${kind}-${timestamp()}.${ext}`
  }

  function stopStream() {
    if (!stream) return
    for (const track of stream.getTracks()) track.stop()
    stream = null
  }

  async function bindStreamToVideoElement() {
    if (!videoEl || !stream) return

    if (videoEl.srcObject !== stream) {
      videoEl.srcObject = stream
    }

    try {
      await videoEl.play()
    } catch {
      // autoplay blocked until user interaction
    }
  }

  async function refreshVideoDevices() {
    const devices = await navigator.mediaDevices.enumerateDevices()
    videoInputs = devices.filter((d) => d.kind === 'videoinput')
  }

  function resetZoomState() {
    zoomSupported = false
    zoomMin = 1
    zoomMax = 1
    zoomStep = 0.1
    zoomValue = 1
    pinchStartDistance = 0
    pinchStartZoom = 1
  }

  async function detectZoomSupport() {
    resetZoomState()

    const track = stream?.getVideoTracks()[0]
    if (!track) return

    const capabilities = track.getCapabilities() as any
    if (capabilities?.zoom && typeof capabilities.zoom.min === 'number') {
      zoomSupported = true
      zoomMin = capabilities.zoom.min
      zoomMax = capabilities.zoom.max
      zoomStep = capabilities.zoom.step || 0.1

      const settings = track.getSettings() as any
      zoomValue = typeof settings.zoom === 'number' ? settings.zoom : zoomMin
    }
  }

  async function applyZoom(value: number) {
    if (!zoomSupported) return
    const track = stream?.getVideoTracks()[0]
    if (!track) return

    const clamped = Math.max(zoomMin, Math.min(zoomMax, value))
    try {
      await track.applyConstraints({
        advanced: [{ zoom: clamped } as any],
      })
      zoomValue = clamped
    } catch {
      // ignore unsupported constraint errors
    }
  }

  function touchDistance(touchA: Touch, touchB: Touch) {
    const dx = touchA.clientX - touchB.clientX
    const dy = touchA.clientY - touchB.clientY
    return Math.sqrt(dx * dx + dy * dy)
  }

  function handleTouchStart(event: TouchEvent) {
    if (!zoomSupported || event.touches.length !== 2) return
    pinchStartDistance = touchDistance(event.touches[0], event.touches[1])
    pinchStartZoom = zoomValue
  }

  async function handleTouchMove(event: TouchEvent) {
    if (!zoomSupported || event.touches.length !== 2 || pinchStartDistance <= 0) return
    const nextDistance = touchDistance(event.touches[0], event.touches[1])
    const scale = nextDistance / pinchStartDistance
    const rawZoom = pinchStartZoom * scale
    await applyZoom(rawZoom)
  }

  function handleTouchEnd() {
    pinchStartDistance = 0
  }

  async function startCamera() {
    bootState = 'loading'
    message = 'Starting camera...'

    stopStream()
    resetZoomState()

    try {
      let constraints: MediaStreamConstraints = {
        video: currentDeviceId
          ? { deviceId: { exact: currentDeviceId } }
          : {
              facingMode: { ideal: facing },
              width: { ideal: 1920 },
              height: { ideal: 1080 },
            },
        audio: audioEnabled,
      }

      try {
        stream = await navigator.mediaDevices.getUserMedia(constraints)
      } catch (initialError) {
        if (currentDeviceId) {
          currentDeviceId = ''
          constraints = {
            video: {
              facingMode: { ideal: facing },
              width: { ideal: 1920 },
              height: { ideal: 1080 },
            },
            audio: audioEnabled,
          }
          stream = await navigator.mediaDevices.getUserMedia(constraints)
        } else {
          throw initialError
        }
      }

      if (!stream) {
        throw new Error('camera-stream-unavailable')
      }

      await refreshVideoDevices()
      if (videoInputs.length === 0) {
        stopStream()
        bootState = 'nocam'
        message = 'No camera device detected.'
        return
      }

      const activeTrack = stream.getVideoTracks()[0]
      const settings = activeTrack?.getSettings()
      if (settings?.deviceId) currentDeviceId = settings.deviceId

      bootState = 'ready'
      reviewOpen = false
      await bindStreamToVideoElement()
      await detectZoomSupport()
      await requestWakeLock()

      if (preferFullscreen) {
        setNote('Tap viewfinder for fullscreen')
      }
    } catch (error: any) {
      if (error?.name === 'NotAllowedError' || error?.name === 'SecurityError') {
        bootState = 'blocked'
        message = 'Camera access denied. Enable permission in browser settings and retry.'
        return
      }

      if (error?.name === 'NotFoundError' || error?.name === 'DevicesNotFoundError') {
        bootState = 'nocam'
        message = 'No camera available on this device.'
        return
      }

      bootState = 'error'
      message = 'Could not start camera. Close other camera apps and retry.'
    }
  }

  async function switchCamera() {
    if (bootState !== 'ready') return

    if (videoInputs.length > 1) {
      const currentIndex = videoInputs.findIndex((d) => d.deviceId === currentDeviceId)
      const nextIndex = currentIndex < 0 ? 0 : (currentIndex + 1) % videoInputs.length
      currentDeviceId = videoInputs[nextIndex]?.deviceId || ''
    } else {
      facing = facing === 'user' ? 'environment' : 'user'
      currentDeviceId = ''
    }

    await startCamera()
  }

  function triggerCaptureFeedback() {
    shutterRingActive = false
    window.setTimeout(() => {
      shutterRingActive = true
      window.setTimeout(() => {
        shutterRingActive = false
      }, 300)
    }, 0)

    if (soundEnabled) {
      const audioCtx = new AudioContext()
      const oscillator = audioCtx.createOscillator()
      const gainNode = audioCtx.createGain()

      oscillator.type = 'triangle'
      oscillator.frequency.value = 1200
      gainNode.gain.value = 0.015

      oscillator.connect(gainNode)
      gainNode.connect(audioCtx.destination)
      oscillator.start()
      oscillator.stop(audioCtx.currentTime + 0.035)
      oscillator.onended = () => {
        void audioCtx.close()
      }
    }
  }

  async function capturePhoto() {
    if (bootState !== 'ready' || !videoEl) return

    if (preferFullscreen) {
      await enterFullscreen()
    }

    const width = videoEl.videoWidth
    const height = videoEl.videoHeight
    if (!width || !height) return

    const canvas = document.createElement('canvas')
    canvas.width = width
    canvas.height = height

    const context = canvas.getContext('2d')
    if (!context) return

    context.drawImage(videoEl, 0, 0, width, height)

    const blob: Blob | null = await new Promise((resolve) => {
      canvas.toBlob((b) => resolve(b), 'image/jpeg', 0.92)
    })

    if (!blob) return

    triggerCaptureFeedback()

    const capture: Capture = {
      type: 'photo',
      blob,
      url: URL.createObjectURL(blob),
      filename: filenameFor('photo', blob.type || 'image/jpeg'),
      mime: blob.type || 'image/jpeg',
    }

    disposeCapture()
    lastCapture = capture
    reviewOpen = true
  }

  function supportedVideoMime() {
    const options = [
      'video/mp4;codecs=h264,aac',
      'video/mp4',
      'video/webm;codecs=vp9,opus',
      'video/webm;codecs=vp8,opus',
      'video/webm',
    ]

    for (const candidate of options) {
      if (MediaRecorder.isTypeSupported(candidate)) return candidate
    }

    return ''
  }

  function clearRecordingTimers() {
    window.clearInterval(recordingInterval)
    window.clearTimeout(autoStopTimeout)
    recordingInterval = undefined
    autoStopTimeout = undefined
  }

  function startRecordingTimers() {
    recordingSeconds = 0
    recordingInterval = window.setInterval(() => {
      recordingSeconds += 1
    }, 1000)

    autoStopTimeout = window.setTimeout(() => {
      void stopRecording()
    }, maxVideoSeconds * 1000)
  }

  async function startRecording() {
    if (bootState !== 'ready' || !stream || isRecording) return

    if (preferFullscreen) {
      await enterFullscreen()
    }

    const mimeType = supportedVideoMime()
    if (!mimeType) {
      setNote('This browser cannot record video')
      return
    }

    fallbackMimeUsed = false
    if (!mimeType.includes('mp4')) {
      fallbackMimeUsed = true
      setNote('Recording in WebM')
    }

    recordedChunks = []

    recorder = new MediaRecorder(stream, mimeType ? { mimeType } : undefined)
    recorder.ondataavailable = (event) => {
      if (event.data.size > 0) recordedChunks.push(event.data)
    }

    recorder.onstop = () => {
      clearRecordingTimers()
      isRecording = false

      if (recordedChunks.length === 0) return

      const finalMime = recorder?.mimeType || (fallbackMimeUsed ? 'video/webm' : 'video/mp4')
      const blob = new Blob(recordedChunks, { type: finalMime })

      triggerCaptureFeedback()

      const capture: Capture = {
        type: 'video',
        blob,
        url: URL.createObjectURL(blob),
        filename: filenameFor('video', finalMime),
        mime: finalMime,
      }

      disposeCapture()
      lastCapture = capture
      reviewOpen = true
    }

    recorder.start()
    isRecording = true
    reviewOpen = false
    startRecordingTimers()
  }

  async function stopRecording() {
    console.log('[CAM] stopRecording called, recorder:', !!recorder, 'isRecording:', isRecording)
    if (!recorder || !isRecording) {
      console.log('[CAM] stopRecording early return')
      return
    }
    recorder.stop()
    console.log('[CAM] recorder.stop() called')
  }

  async function toggleRecording() {
    console.log('[CAM] toggleRecording isRecording:', isRecording)
    if (isRecording) {
      console.log('[CAM] Calling stopRecording')
      await stopRecording()
    } else {
      console.log('[CAM] Calling startRecording')
      await startRecording()
    }
  }

  function disposeCapture() {
    if (lastCapture?.url) URL.revokeObjectURL(lastCapture.url)
    lastCapture = null
  }

  function closeReview() {
    reviewOpen = false
  }

  function downloadCapture() {
    if (!lastCapture) return

    const link = document.createElement('a')
    link.href = lastCapture.url
    link.download = lastCapture.filename
    document.body.append(link)
    link.click()
    link.remove()
  }

  async function shareCapture() {
    if (!lastCapture || !navigator.share) return

    const file = new File([lastCapture.blob], lastCapture.filename, { type: lastCapture.mime })
    const canShareFiles = navigator.canShare?.({ files: [file] }) ?? false
    if (!canShareFiles) {
      setNote('Share not available')
      return
    }

    try {
      await navigator.share({ files: [file], title: lastCapture.filename })
    } catch {
      // user canceled share
    }
  }

  async function toggleAudio() {
    audioEnabled = !audioEnabled
    await startCamera()
  }

  async function enterFullscreen() {
    if (!document.fullscreenElement) {
      try {
        await document.documentElement.requestFullscreen()
      } catch {
        setNote('Fullscreen blocked')
      }
    }
  }

  async function exitFullscreen() {
    if (!document.fullscreenElement) return
    try {
      await document.exitFullscreen()
    } catch {
      // ignore
    }
  }

  async function requestWakeLock() {
    if (!('wakeLock' in navigator) || wakeLock) return
    try {
      wakeLock = await (navigator as any).wakeLock.request('screen')
      wakeLock.addEventListener('release', () => {
        wakeLock = null
      })
    } catch {
      // ignore unsupported/blocked wake lock
    }
  }

  async function releaseWakeLock() {
    if (!wakeLock) return
    try {
      await wakeLock.release()
    } catch {
      // ignore
    }
    wakeLock = null
  }

  function handleVisibilityChange() {
    if (document.visibilityState === 'visible') {
      void requestWakeLock()
    }
  }

  function handleFullscreenChange() {
    fullscreenActive = Boolean(document.fullscreenElement)
  }

  function handleViewfinderTap() {
    if (preferFullscreen) {
      void enterFullscreen()
    }
  }

  function formatTimer(total: number) {
    const minutes = Math.floor(total / 60)
      .toString()
      .padStart(2, '0')
    const seconds = (total % 60).toString().padStart(2, '0')
    return `${minutes}:${seconds}`
  }

  function availableShare() {
    return Boolean(lastCapture && navigator.share)
  }

  async function runPrimaryAction() {
    if (mode === 'photo') {
      await capturePhoto()
    } else {
      await toggleRecording()
    }
  }

  function handleKeydown(event: KeyboardEvent) {
    if (bootState !== 'ready') return

    const target = event.target as HTMLElement | null
    if (target && ['INPUT', 'SELECT', 'TEXTAREA', 'BUTTON'].includes(target.tagName)) return

    if (event.code === 'Space') {
      event.preventDefault()
      void runPrimaryAction()
      return
    }

    if (event.key.toLowerCase() === 'v') {
      event.preventDefault()
      mode = mode === 'photo' ? 'video' : 'photo'
    }
  }

  async function retryCamera() {
    await startCamera()
  }

  async function zoomIn() {
    await applyZoom(zoomValue + zoomStep)
  }

  async function zoomOut() {
    await applyZoom(zoomValue - zoomStep)
  }

  onMount(async () => {
    privacyVisible = true
    privacyTimeout = window.setTimeout(() => {
      privacyVisible = false
    }, 3000)

    loadSettings()
    await startCamera()
    fullscreenActive = Boolean(document.fullscreenElement)
  })

  onDestroy(() => {
    clearRecordingTimers()
    window.clearTimeout(noteTimeout)
    window.clearTimeout(privacyTimeout)

    if (isRecording) recorder?.stop()
    stopStream()
    disposeCapture()
    void releaseWakeLock()
  })

  $effect(() => {
    if (mode === 'photo' && isRecording) {
      void stopRecording()
    }
    if (bootState === 'ready' && stream) {
      ;(async () => {
        await tick()
        await tick()
        if (videoEl) {
          await bindStreamToVideoElement()
        }
      })()
    }
  })
</script>

<svelte:window
  onkeydown={handleKeydown}
  onvisibilitychange={handleVisibilityChange}
  onfullscreenchange={handleFullscreenChange}
/>

<main class="app">
  {#if bootState === 'ready'}
    <div
      class="camera-shell"
      ontouchstart={handleTouchStart}
      ontouchmove={handleTouchMove}
      ontouchend={handleTouchEnd}
      ontouchcancel={handleTouchEnd}
      onclick={handleViewfinderTap}
      role="button"
      tabindex="0"
      aria-label="Tap for fullscreen"
    >
      <video bind:this={videoEl} autoplay playsinline muted class:mirror={mirrorPreview}></video>
      <div class="shutter-ring" class:active={shutterRingActive}></div>
    </div>
  {:else}
    <section class="fallback">
      <div class="fallback-card">
        {#if bootState === 'loading'}
          <div class="loader-ring"></div>
        {/if}
        <p class="fallback-title">{bootState === 'loading' ? 'Initializing' : 'Camera unavailable'}</p>
        <p class="fallback-text">{message}</p>
        {#if bootState !== 'loading'}
          <button class="review-btn" type="button" onclick={retryCamera}>Retry</button>
        {/if}
      </div>
    </section>
  {/if}

  {#if bootState === 'ready'}
    <div class="status-text">
      {mode === 'photo' ? 'PHOTO' : 'REC'}
      {#if isRecording}
        {formatTimer(recordingSeconds)}
      {/if}
    </div>

    <button
      class="settings-toggle"
      class:active={settingsOpen}
      type="button"
      onclick={() => (settingsOpen = !settingsOpen)}
      aria-label="Settings"
      aria-expanded={settingsOpen}
      title="Settings"
    >
      {settingsOpen ? '×' : '●'}
    </button>

    {#if isRecording}
      <div class="record-progress">
        <div
          class="record-progress-fill"
          style={`width: ${(recordingSeconds / maxVideoSeconds) * 100}%`}
        ></div>
      </div>
    {/if}

    <div class="bottom-bar">
      <div class="mode-selector">
        <button
          class="mode-btn"
          class:active={mode === 'photo'}
          type="button"
          onclick={() => (mode = 'photo')}
        >
          Photo
        </button>
        <button
          class="mode-btn"
          class:active={mode === 'video'}
          type="button"
          onclick={() => (mode = 'video')}
        >
          Video
        </button>
      </div>

      <div class="controls-row">
        <button
          class="thumb-preview"
          type="button"
          onclick={() => (reviewOpen = Boolean(lastCapture))}
          disabled={!lastCapture}
          aria-label="View last capture"
        >
          {#if lastCapture?.type === 'photo'}
            <img src={lastCapture.url} alt="Last capture" loading="lazy" />
          {:else if lastCapture?.type === 'video'}
            <video src={lastCapture.url} muted playsinline></video>
          {:else}
            <span class="thumb-placeholder">IMG</span>
          {/if}
        </button>

        <button
          class="shutter-btn"
          class:active={isRecording}
          class:recording={isRecording}
          type="button"
          onclick={runPrimaryAction}
          aria-label={mode === 'photo' ? 'Capture photo' : isRecording ? 'Stop recording' : 'Start recording'}
        ></button>

        <div class="zoom-controls">
          {#if zoomSupported}
            <button class="zoom-btn" type="button" onclick={zoomOut} aria-label="Zoom out">−</button>
            <button class="zoom-btn" type="button" onclick={zoomIn} aria-label="Zoom in">+</button>
          {:else}
            <button class="zoom-btn" type="button" onclick={switchCamera} aria-label="Switch camera">↻</button>
          {/if}
        </div>
      </div>
    </div>
  {/if}

  <section class="settings-sheet" class:open={settingsOpen} aria-label="Settings">
    <div class="settings-header">
      <span class="settings-title">Settings</span>
      <button class="settings-close" type="button" onclick={() => (settingsOpen = false)}>×</button>
    </div>

    <div class="settings-group">
      <div class="setting-row">
        <span class="setting-label">Audio</span>
        <button
          class="toggle"
          class:active={audioEnabled}
          type="button"
          onclick={toggleAudio}
          aria-pressed={audioEnabled}
          aria-label="Toggle audio"
        ></button>
      </div>

      <div class="setting-row">
        <span class="setting-label">Mirror</span>
        <button
          class="toggle"
          class:active={mirrorPreview}
          type="button"
          onclick={() => (mirrorPreview = !mirrorPreview)}
          aria-pressed={mirrorPreview}
          aria-label="Toggle mirror"
        ></button>
      </div>

      <div class="setting-row">
        <span class="setting-label">Sound</span>
        <button
          class="toggle"
          class:active={soundEnabled}
          type="button"
          onclick={() => (soundEnabled = !soundEnabled)}
          aria-pressed={soundEnabled}
          aria-label="Toggle sound"
        ></button>
      </div>

      <div class="setting-row">
        <span class="setting-label">Fullscreen</span>
        <button
          class="toggle"
          class:active={preferFullscreen}
          type="button"
          onclick={() => (preferFullscreen = !preferFullscreen)}
          aria-pressed={preferFullscreen}
          aria-label="Toggle fullscreen"
        ></button>
      </div>

      <div class="setting-row">
        <span class="setting-label">Video Limit</span>
        <select
          class="select"
          bind:value={maxVideoSeconds}
        >
          {#each VIDEO_LIMITS as value}
            <option {value}>{value}s</option>
          {/each}
        </select>
      </div>

      {#if fullscreenActive}
        <div class="setting-row">
          <span class="setting-label">Fullscreen</span>
          <button class="review-btn" type="button" onclick={exitFullscreen}>Exit</button>
        </div>
      {/if}

      {#if videoInputs.length > 1}
        <div class="setting-row">
          <span class="setting-label">Switch Lens</span>
          <button class="review-btn" type="button" onclick={switchCamera}>Switch</button>
        </div>
      {/if}
    </div>
  </section>

  <div class="review-overlay" class:open={reviewOpen} role="dialog" aria-label="Capture review">
    {#if lastCapture}
      <div class="review-media">
        {#if lastCapture.type === 'photo'}
          <img src={lastCapture.url} alt="Captured preview" />
        {:else}
          <!-- svelte-ignore a11y_media_has_caption -->
          <video src={lastCapture.url} controls playsinline></video>
        {/if}
      </div>

      <div class="review-actions">
        <button class="review-btn" type="button" onclick={closeReview}>Retake</button>
        <button class="review-btn primary" type="button" onclick={downloadCapture}>Download</button>
        {#if availableShare()}
          <button class="review-btn" type="button" onclick={shareCapture}>Share</button>
        {/if}
      </div>
    {/if}
  </div>

  {#if note}
    <div class="note">{note}</div>
  {/if}

  <div class="privacy-note" class:visible={privacyVisible}>
    Local only · Space capture · V video
  </div>
</main>
