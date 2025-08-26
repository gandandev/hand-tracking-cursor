<script lang="ts">
  import { fly } from 'svelte/transition'

  let { children, value } = $props()
</script>

<div class="grid w-full">
  {#key value}
    <div
      class="[grid-area:1/1]"
      in:fly={{ y: 24, duration: 500 }}
      out:fly={{ y: -24, duration: 500 }}
    >
      {@render children?.()}
    </div>
  {/key}
</div>
