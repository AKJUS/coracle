<script lang="ts">
  import cx from "classnames"
  import {imgproxy} from "src/engine"

  export let src: string

  const url = imgproxy(src, {w: 280, h: 280})
</script>

<div
  class={cx($$props.class, "shrink-0 overflow-hidden rounded-full bg-cover bg-center")}
  style={$$props.style || ""}>
  <img loading="lazy" decoding="async" src={url} class="h-full w-full object-cover" alt="" />
</div>
