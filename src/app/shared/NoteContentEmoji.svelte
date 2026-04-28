<script lang="ts">
  import {imgproxy} from "src/engine"

  export let url: string = undefined
  export let name: string
</script>

{#if url}
  <img
    loading="lazy"
    decoding="async"
    src={imgproxy(url, {w: 24, h: 24})}
    class="-mt-0.5 inline h-[1em] min-w-[1em] align-middle" />
{:else}
  :{name}:
{/if}
