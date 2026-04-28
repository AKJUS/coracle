<script lang="ts">
  import cx from "classnames"
  import Image from "src/partials/Image.svelte"

  export let url
  export let alt = "Link preview"
  export let fullSize = false

  const className = cx("object-contain object-center", {
    "max-h-96": !fullSize,
    "max-h-screen": fullSize,
  })
</script>

<Image src={url} {alt} class={className} />
