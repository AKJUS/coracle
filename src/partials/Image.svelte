<script lang="ts">
  import {onDestroy} from "svelte"
  import {ensurePlural} from "@welshman/lib"
  import {imgproxy} from "src/engine"

  export let src
  export let onClick = undefined

  const urls = ensurePlural(src)

  let i = 0
  let proxyFailed = false
  let loading = true
  let timeout = null

  const clearTimer = () => {
    if (timeout) {
      clearTimeout(timeout)
      timeout = null
    }
  }

  const handleFailure = () => {
    if (!proxyFailed) {
      proxyFailed = true
    } else if (i < urls.length - 1) {
      i++
      proxyFailed = false
    }
  }

  const onLoad = () => {
    loading = false
    clearTimer()
  }

  $: displayedSrc = proxyFailed ? urls[i] : imgproxy(urls[i])

  $: {
    clearTimer()
    if (!proxyFailed && displayedSrc !== urls[i]) {
      timeout = setTimeout(handleFailure, 8000)
    }
  }

  onDestroy(clearTimer)
</script>

<img
  {...$$props}
  loading="lazy"
  decoding="async"
  class:opacity-0={loading}
  on:error={handleFailure}
  on:load={onLoad}
  on:click={onClick}
  src={displayedSrc} />

{#if loading}
  <div class="shimmer h-64 w-96" />
{/if}
