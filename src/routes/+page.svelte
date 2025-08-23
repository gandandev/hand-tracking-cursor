<script lang="ts">
  let showStartScreen = $state(true)
  let loadingWebcam = $state(false)

  let videoSource: HTMLVideoElement | null = $state(null)

  async function startWebcam() {
    try {
      showStartScreen = false
      loadingWebcam = true

      const stream = await navigator.mediaDevices.getUserMedia({
        video: true
      })
      videoSource!.srcObject = stream
      videoSource!.play()

      loadingWebcam = false
    } catch (error) {
      console.log(error)
    }
  }
</script>

{#if showStartScreen}
  <div class="flex h-screen w-full flex-col items-center justify-center text-center">
    <h1 class="text-3xl font-semibold">Hand tracking cursor demo</h1>
    <p class="mt-2 text-neutral-500">
      Track your hand movements to control the cursor using computer vision.
    </p>
    <button
      class="mt-4 rounded-full bg-neutral-100 px-4 py-2 duration-150 hover:bg-neutral-200 active:scale-98"
      onclick={startWebcam}
    >
      {loadingWebcam ? 'Loading...' : 'Get started'}
    </button>
  </div>
{:else}
  <!-- svelte-ignore a11y_media_has_caption -->
  <video bind:this={videoSource} class="h-full w-full -scale-x-100 object-cover"></video>
{/if}
