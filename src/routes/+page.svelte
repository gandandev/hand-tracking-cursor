<script lang="ts">
  import { fly } from 'svelte/transition'
  import { cubicIn } from 'svelte/easing'
  import {
    HandLandmarker,
    FilesetResolver,
    type HandLandmarkerResult
  } from '@mediapipe/tasks-vision'

  let showStartScreen = $state(true)
  let loadingWebcam = $state(true)
  let videoSource: HTMLVideoElement | null = $state(null)
  let canvasElement: HTMLCanvasElement | null = $state(null)

  let handLandmarker: HandLandmarker | null = $state(null)
  let isTracking = $state(false)
  let lastVideoTime = $state(-1)

  let cursorX = $state(0)
  let mirroredCursorX = $derived(typeof window !== 'undefined' ? window.innerWidth - cursorX : 0)
  let cursorY = $state(0)
  let pinchLength = $state(0)
  let clicking = $derived(pinchLength < 0.04)

  async function initHandTracker() {
    const vision = await FilesetResolver.forVisionTasks(
      'https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@0.10.22-rc.20250304/wasm'
    )
    handLandmarker = await HandLandmarker.createFromOptions(vision, {
      baseOptions: {
        modelAssetPath: '/models/hand_landmarker.task',
        delegate: 'GPU'
      },
      runningMode: 'VIDEO',
      numHands: 2,
      minHandDetectionConfidence: 0.5,
      minHandPresenceConfidence: 0.5,
      minTrackingConfidence: 0.5
    })
  }

  function detectHands() {
    if (!videoSource || !handLandmarker || !canvasElement) return

    const videoTime = videoSource.currentTime
    if (videoTime === lastVideoTime) {
      if (isTracking) {
        requestAnimationFrame(detectHands)
      }
      return
    }
    lastVideoTime = videoTime

    const results = handLandmarker.detectForVideo(videoSource, videoTime)
    drawHandLandmarks(results)

    if (isTracking) {
      requestAnimationFrame(detectHands)
    }
  }

  function drawHandLandmarks(results: HandLandmarkerResult) {
    if (!canvasElement) return

    const ctx = canvasElement.getContext('2d')!
    ctx.clearRect(0, 0, canvasElement.width, canvasElement.height)

    results.landmarks.forEach((landmarks, handIndex) => {
      // Draw connection lines between landmarks
      drawConnections(ctx, landmarks)

      // Draw individual landmarks
      landmarks.forEach((landmark) => {
        const x = landmark.x * canvasElement!.width
        const y = landmark.y * canvasElement!.height

        ctx.beginPath()
        ctx.arc(x, y, 3, 0, 2 * Math.PI)
        ctx.fillStyle = handIndex === 0 ? '#00ff00' : '#ff0000'
        ctx.fill()
      })

      if (landmarks[8] && landmarks[4]) {
        // Thumb and index finger tip
        const videoAspect = videoSource!.videoWidth / videoSource!.videoHeight
        const screenAspect = window.innerWidth / window.innerHeight

        let scaleX,
          scaleY,
          offsetX = 0,
          offsetY = 0

        if (videoAspect > screenAspect) {
          // Left/right cropped
          scaleY = 1
          scaleX = screenAspect / videoAspect
          offsetX = (1 - scaleX) / 2
        } else {
          // Top/bottom cropped
          scaleX = 1
          scaleY = videoAspect / screenAspect
          offsetY = (1 - scaleY) / 2
        }

        const thumbX = landmarks[4].x
        const thumbY = landmarks[4].y
        const indexFingerX = landmarks[8].x
        const indexFingerY = landmarks[8].y

        pinchLength = Math.hypot(thumbX - indexFingerX, thumbY - indexFingerY)

        const midpointX = (thumbX + indexFingerX) / 2
        const midpointY = (thumbY + indexFingerY) / 2

        const normalizedX = (midpointX - offsetX) / scaleX
        const normalizedY = (midpointY - offsetY) / scaleY

        cursorX = normalizedX * window.innerWidth
        cursorY = normalizedY * window.innerHeight
      }
    })
  }

  function drawConnections(ctx: CanvasRenderingContext2D, landmarks: any[]) {
    const connections = [
      [0, 1],
      [1, 2],
      [2, 3],
      [3, 4], // Thumb
      [0, 5],
      [5, 6],
      [6, 7],
      [7, 8], // Index finger
      [0, 9],
      [9, 10],
      [10, 11],
      [11, 12], // Middle finger
      [0, 13],
      [13, 14],
      [14, 15],
      [15, 16], // Ring finger
      [0, 17],
      [17, 18],
      [18, 19],
      [19, 20], // Pinky
      [5, 9],
      [9, 13],
      [13, 17] // Palm connections
    ]

    ctx.strokeStyle = '#ffffff'
    ctx.lineWidth = 2

    connections.forEach(([start, end]) => {
      const startPoint = landmarks[start]
      const endPoint = landmarks[end]

      if (startPoint && endPoint) {
        ctx.beginPath()
        ctx.moveTo(startPoint.x * canvasElement!.width, startPoint.y * canvasElement!.height)
        ctx.lineTo(endPoint.x * canvasElement!.width, endPoint.y * canvasElement!.height)
        ctx.stroke()
      }
    })
  }

  async function startWebcam() {
    try {
      showStartScreen = false
      loadingWebcam = true

      await initHandTracker()

      const stream = await navigator.mediaDevices.getUserMedia({
        video: { width: 1280, height: 720 }
      })

      videoSource!.srcObject = stream
      await videoSource!.play()

      if (canvasElement) {
        canvasElement.width = videoSource!.videoWidth
        canvasElement.height = videoSource!.videoHeight
      }

      loadingWebcam = false
      isTracking = true
      detectHands()
    } catch (error) {
      console.error('Error starting webcam or hand tracking:', error)
    }
  }

  function stopTracking() {
    isTracking = false
  }
</script>

<div class="relative h-screen w-full">
  {#if showStartScreen}
    <div
      class="absolute flex h-screen w-full flex-col items-center justify-center text-center"
      transition:fly={{ y: -50, duration: 100, easing: cubicIn }}
    >
      <h1 class="text-3xl font-semibold">Hand tracking cursor demo</h1>
      <p class="mt-2 text-neutral-500">
        Track your hand movements to control the cursor using computer vision.
      </p>
      <button
        class="mt-4 rounded-full bg-neutral-100 px-4 py-2 duration-150 hover:bg-neutral-200 active:scale-98"
        onclick={startWebcam}
      >
        Get started
      </button>
    </div>
  {:else}
    <!-- svelte-ignore a11y_media_has_caption -->
    <video
      bind:this={videoSource}
      class="absolute h-full w-full -scale-x-100 object-cover duration-1000"
      class:opacity-0={loadingWebcam}
      class:blur-lg={loadingWebcam}
      onplaying={() => {
        loadingWebcam = false
      }}
    ></video>

    <canvas
      bind:this={canvasElement}
      class="pointer-events-none absolute h-full w-full -scale-x-100 object-cover"
    ></canvas>

    <div
      class="absolute size-6 -translate-x-1/2 -translate-y-1/2 rounded-full bg-white transition-[scale,opacity] duration-300"
      class:scale-70={clicking}
      class:opacity-50={!clicking}
      style:left={`${mirroredCursorX}px`}
      style:top={`${cursorY}px`}
    ></div>

    <div class="absolute right-0 bottom-1 left-0 flex flex-col items-center">
      <span>Pinch length: {pinchLength}</span>
      <div class="h-1 rounded-full bg-white" style:width={`${pinchLength * 100}%`}></div>
    </div>
  {/if}
</div>
