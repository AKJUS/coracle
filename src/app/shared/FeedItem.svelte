<script lang="ts">
  import {remove, first, spec} from "@welshman/lib"
  import {deriveEvents} from "@welshman/store"
  import {getIdOrAddress, getIdFilters, getReplyFilters, isChildOf} from "@welshman/util"
  import type {TrustedEvent} from "@welshman/util"
  import {Router, addMaximalFallbacks} from "@welshman/router"
  import {repository} from "@welshman/app"
  import type {Thunk} from "@welshman/app"
  import {onMount, setContext} from "svelte"
  import {derived} from "svelte/store"
  import {quantify} from "src/util/misc"
  import {replyKinds} from "src/util/nostr"
  import AltColor from "src/partials/AltColor.svelte"
  import Popover from "src/partials/Popover.svelte"
  import Spinner from "src/partials/Spinner.svelte"
  import {fly, slide} from "src/util/transition"
  import NoteMeta from "src/app/shared/NoteMeta.svelte"
  import Note from "src/app/shared/Note.svelte"
  import {
    ensureUnwrapped,
    getSetting,
    isEventMuted,
    sortEventsDesc,
    userSettings,
    myRequest,
  } from "src/engine"

  export let note
  export let relays = []
  export let getContext = (event: TrustedEvent) => repository.query(getReplyFilters([event]))
  export let depth = 0
  export let anchor = null
  export let pinned = false
  export let topLevel = false
  export let isLastReply = false
  export let showMeta = false
  export let showParent = true
  export let showLoading = false
  export let showMedia = getSetting("show_media")

  setContext("topLevel", topLevel)

  let ready = false
  let event = note
  let replyIsOpen = false
  let showMutedReplies = false
  let collapsed = depth === 0
  let showHiddenReplies = anchor === getIdOrAddress(event)
  let pendingReplies: Thunk[] = []

  const showEntire = showHiddenReplies

  const addPendingReply = (thunk: Thunk) => {
    pendingReplies = [...pendingReplies, thunk]
  }

  const removePendingReply = (thunk: Thunk) => {
    pendingReplies = remove(thunk, pendingReplies)
  }

  const replies = derived(
    deriveEvents(repository, {filters: getReplyFilters([event], {kinds: replyKinds})}),
    events => sortEventsDesc(events.filter(e => isChildOf(e, event))),
  )

  let mutedReplies, hiddenReplies, visibleReplies

  $: {
    mutedReplies = []
    hiddenReplies = []
    visibleReplies = []

    for (const e of $replies) {
      if ($isEventMuted(e)) {
        mutedReplies.push(e)
      } else if (pendingReplies.some(thunk => thunk.event.id === e.id)) {
        visibleReplies.push(e)
      } else if (collapsed) {
        hiddenReplies.push(e)
      } else if (!showHiddenReplies && !getContext(event).some(spec({id: e.id}))) {
        hiddenReplies.push(e)
      } else {
        visibleReplies.push(e)
      }
    }

    if (depth === 0) {
      mutedReplies.splice(0)
      hiddenReplies.splice(0)
    }

    if (!showHiddenReplies && visibleReplies.length === 0) {
      mutedReplies.splice(0)
      hiddenReplies.splice(0)
    }

    if (showMutedReplies) {
      visibleReplies = visibleReplies.concat(mutedReplies.splice(0))
    }

    if (!showHiddenReplies) {
      hiddenReplies = hiddenReplies.concat(visibleReplies.slice(3))
      visibleReplies = visibleReplies.slice(0, 3)
    }
  }

  onMount(async () => {
    if (!event.pubkey) {
      event = first(
        await myRequest({
          autoClose: true,
          relays: Router.get().FromRelays(relays).policy(addMaximalFallbacks).getUrls(),
          filters: getIdFilters([event.id]),
        }),
      )
    }

    event = await ensureUnwrapped(event)

    if (event.pubkey) {
      ready = true
    }
  })
</script>

{#if ready}
  <div>
    {#if showMeta}
      <NoteMeta context={getContext(note)} />
    {/if}
    <div class="note relative">
      {#if !showParent && !topLevel}
        <AltColor let:isAlt>
          <svg height="36" width="36" class="absolute -left-[18px] top-1">
            <circle
              cx="18"
              cy="18"
              r="14"
              fill="transparent"
              stroke-width="4"
              stroke-dashoffset="54"
              stroke-dasharray="100 100"
              transform-origin="center"
              class={isAlt ? "stroke-tinted-700" : "stroke-neutral-800"} />
          </svg>
        </AltColor>
        {#if isLastReply}
          <AltColor background class="absolute -left-4 -top-4 h-10 w-1" let:isAlt />
        {:else}
          <AltColor background class="absolute -bottom-4 -left-4 top-0 w-1" let:isAlt />
        {/if}
      {/if}
      <Note
        {event}
        {pinned}
        {showEntire}
        {showParent}
        {showMedia}
        bind:replyIsOpen
        {addPendingReply}
        {removePendingReply} />
      {#if !replyIsOpen && (visibleReplies.length > 0 || collapsed) && !showEntire && depth > 0}
        <div class="relative">
          <AltColor
            background
            class="absolute left-0 top-0 -mr-2 -mt-6 flex h-8 w-8 cursor-pointer items-center
                   justify-center rounded-full"
            on:click={() => {
              collapsed = !collapsed
            }}>
            <Popover triggerType="mouseenter">
              <div slot="trigger">
                <i
                  class="fa fa-arrow-up transition-all"
                  class:text-tinted-500={!collapsed}
                  class:text-tinted-100={collapsed}
                  class:rotate-180={collapsed} />
              </div>
              <div slot="tooltip">
                {collapsed ? "Show replies" : "Hide replies"}
              </div>
            </Popover>
          </AltColor>
        </div>
      {/if}

      {#if visibleReplies.length > 0 || hiddenReplies.length > 0 || mutedReplies.length > 0}
        <div
          class="note-children relative ml-4 mt-4 flex flex-col gap-4"
          in:fly|local={{y: 20}}
          out:slide|local>
          {#if hiddenReplies.length > 0}
            <button
              class="cursor-pointer rounded-md bg-gradient-to-l from-transparent via-tinted-700 to-tinted-700 py-2 text-neutral-100 outline-0 transition-colors hover:bg-tinted-700"
              on:click={() => {
                showHiddenReplies = true
              }}>
              <i class="fa fa-up-down pr-2 text-sm" />
              Show {quantify(hiddenReplies.length, "other reply", "more replies")}
            </button>
            {#if visibleReplies.length > 0}
              <AltColor background class="absolute -left-4 -top-10 h-28 w-1" />
            {/if}
          {:else if visibleReplies.length > 0}
            <AltColor background class="absolute -left-4 -top-10 h-14 w-1" />
          {/if}
          {#if visibleReplies.length}
            {#key showHiddenReplies}
              {#each visibleReplies as r, i (r.id)}
                <div in:fly={{y: 20}}>
                  <svelte:self
                    isLastReply={i === visibleReplies.length - 1}
                    showParent={false}
                    note={r}
                    depth={depth - 1}
                    {getContext}
                    {anchor} />
                </div>
              {/each}
            {/key}
          {/if}
          {#if showHiddenReplies && mutedReplies.length > 0 && !$userSettings.ignore_muted_content}
            <button
              class="cursor-pointer rounded-md bg-gradient-to-l from-transparent via-tinted-700 to-tinted-700 py-2 text-neutral-100 outline-0 transition-colors hover:bg-tinted-700"
              on:click={() => {
                showMutedReplies = true
              }}>
              <i class="fa fa-up-down pr-2 text-sm" />
              Show {quantify(mutedReplies.length, "hidden reply", "hidden replies")}
            </button>
          {/if}
        </div>
      {/if}
    </div>
  </div>
{:else if showLoading}
  <Spinner />
{/if}
