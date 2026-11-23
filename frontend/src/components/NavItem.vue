<template>
  <w-item-label
    v-if="item.type === 'header'"
    class="sidebar-nav-header text-caption wrap-anywhere"
    header
    >{{ item.label }}</w-item-label
  >
  <w-separator v-else-if="item.type === 'separator'" dark />
  <!--
    A group: either a folder (`menu`) or an authored link that carries children. Both open in place as
    an accordion. When the group also has a page of its own, the row is split: the label links to that
    page and the chevron opens the submenu. A folder's own submenu is not in the item until it is first
    opened, so it is fetched from the menu it names at that moment.
  -->
  <div v-else-if="isGroup" class="w-expansion-item">
    <div v-if="ownDestination" class="sidebar-nav-split">
      <w-item class="sidebar-nav-split__link" dense v-bind="ownDestination">
        <w-item-section side><w-icon :name="groupIcon" color="white" /></w-item-section>
        <w-item-section class="wrap-anywhere text-white">{{ item.label }}</w-item-section>
      </w-item>
      <w-item
        class="sidebar-nav-split__menu"
        dense
        clickable
        :aria-label="t('navEdit.showMenu', { label: item.label })"
        :aria-expanded="String(expanded)"
        :aria-controls="contentId"
        @click="toggleExpanded">
        <w-item-section side>
          <w-icon
            name="mdi:chevron-down"
            class="w-expansion-item__arrow"
            :class="expanded ? 'rotate-180' : ''"
            color="white" />
        </w-item-section>
      </w-item>
    </div>
    <w-item
      v-else
      class="w-expansion-item__header select-none"
      dense
      clickable
      :aria-expanded="String(expanded)"
      :aria-controls="contentId"
      @click="toggleExpanded">
      <w-item-section side><w-icon :name="groupIcon" color="white" /></w-item-section>
      <w-item-section class="wrap-anywhere text-white">{{ item.label }}</w-item-section>
      <w-item-section side>
        <w-icon
          name="mdi:chevron-down"
          class="w-expansion-item__arrow"
          :class="expanded ? 'rotate-180' : ''" />
      </w-item-section>
    </w-item>
    <div v-show="expanded" :id="contentId" class="w-expansion-item__content">
      <w-list clickable dense dark>
        <NavItem v-for="child in children" :key="child.id" :item="child" />
      </w-list>
    </div>
  </div>
  <!-- -> Anything else is a leaf: a link to a page, an address elsewhere, or a new tab -->
  <w-item v-else v-bind="destination(item)">
    <w-item-section side
      ><w-icon :name="item.icon || `la:file-alt`" color="white"
    /></w-item-section>
    <w-item-section class="wrap-anywhere text-white">{{ item.label }}</w-item-section>
  </w-item>
</template>

<script setup>
import { computed, inject, onMounted, ref, useId, watch } from 'vue'
import { useI18n } from 'vue-i18n'
import { useRouter, useRoute } from 'vue-router'

import { decodeUrlPath } from '@/helpers/pagePaths'
import { routableHref } from '@/helpers/renderedContent'

import { usePageStore } from '@/stores/page'
import { useSiteStore } from '@/stores/site'

/**
 * One row of the sidebar navigation, and -- because a group renders its children through this same
 * component -- the whole subtree beneath it. A group is an in-place accordion; a leaf is a link.
 *
 * Which groups stand open is held in a set the sidebar provides, keyed per group, so the open/closed
 * state a reader set survives moving between pages rather than being rebuilt from the page each time.
 * A group is opened on arrival when its own submenu is flagged to, or when the page being read is one of
 * its (already-loaded) children.
 */
const props = defineProps({
  item: {
    type: Object,
    required: true
  }
})

const router = useRouter()
const route = useRoute()
const { t } = useI18n()
const pageStore = usePageStore()
const siteStore = useSiteStore()

const navExpanded = inject('navExpanded')
const contentId = useId()

const loading = ref(false)

const isGroup = computed(
  () => props.item.type === 'menu' || (props.item.type === 'link' && props.item.children?.length > 0)
)

const children = computed(() => props.item.children ?? [])

const groupIcon = computed(
  () => props.item.icon || (props.item.type === 'menu' ? 'mdi:folder-outline' : 'mdi:text-box-outline')
)

/** Stable per group: a folder by the menu it opens, an authored link by its target. */
const groupKey = computed(() =>
  props.item.type === 'menu'
    ? `m:${props.item.targetNavId}`
    : `l:${props.item.target ?? props.item.id}`
)

const expanded = computed(() => navExpanded.has(groupKey.value))

// METHODS

/**
 * Where a leaf points, as the props that take a reader there. External addresses, `mailto:` and new tabs
 * go out as plain anchors; only an in-wiki path is the router's. The same `routableHref` the page body
 * and the folder list use, so all three agree on what is routable.
 */
function destination(item) {
  const address = item.target ?? '/'
  const target = item.openInNewWindow ? '_blank' : undefined
  let url
  try {
    url = new URL(address, globalThis.location.href)
  } catch {
    return {}
  }
  if (!['http:', 'https:', 'mailto:'].includes(url.protocol)) {
    return {}
  }
  const routable = routableHref({ href: url.href, target }, globalThis.location)
  return routable ? { to: routable } : { href: address, target }
}

/** Where a page path in the menu leads, in the locale being read. */
function pageHref(path) {
  const prefix = siteStore.localeUrlPrefix(pageStore.locale)
  return path === 'home' ? prefix || '/' : `${prefix}/${path}`
}

/**
 * Whether a group is also a page: an authored link with a usable target, or a folder menu whose
 * target has a page this reader may open. Split rows need this to know which half links.
 */
const ownDestination = computed(() => {
  if (props.item.type === 'link') {
    const dest = destination(props.item)
    return Object.keys(dest).length > 0 ? dest : null
  }
  if (props.item.type === 'menu' && props.item.targetPage) {
    return { to: pageHref(props.item.targetPage) }
  }
  return null
})

const ownCurrent = computed(() => {
  const { to } = ownDestination.value ?? {}
  return Boolean(to) && decodeUrlPath(router.resolve(to).path) === decodeUrlPath(route.path)
})

/** Whether an item leads to the page being read. */
function isCurrent(item) {
  const { to } = destination(item)
  return Boolean(to) && decodeUrlPath(router.resolve(to).path) === decodeUrlPath(route.path)
}

/** ...and whether the current page sits anywhere inside a group's already-loaded children. */
function containsCurrent(item) {
  return (item.children ?? []).some(
    (child) =>
      child.type === 'link' &&
      (child.children?.length ? containsCurrent(child) : isCurrent(child))
  )
}

/**
 * Fetch a folder's submenu the first time it is opened. Kept on the item so a re-render (or a close and
 * reopen) does not ask again.
 */
async function ensureLoaded() {
  if (props.item.type !== 'menu' || props.item.children || loading.value) {
    return
  }
  loading.value = true
  try {
    const items = await API_CLIENT.get(`sites/${siteStore.id}/navigation/${props.item.targetNavId}`).json()
    props.item.children = items ?? []
  } catch (err) {
    console.warn(err.message)
    props.item.children = []
  } finally {
    loading.value = false
  }
}

function setExpanded(value) {
  if (value) {
    navExpanded.add(groupKey.value)
  } else {
    navExpanded.delete(groupKey.value)
  }
}

function toggleExpanded() {
  setExpanded(!expanded.value)
}

// WATCHERS

/*
  A persisted open folder can return without its submenu in hand: the menu on screen is refetched after a
  page load or refresh, and the item object then starts with no children again. Fetching here -- not only
  on mount -- is what lets the reader's open folders survive navigation and refreshes of the same menu.
*/
watch(
  () => [expanded.value, props.item.id, props.item.targetNavId],
  ([isExpanded]) => {
    if (isExpanded) {
      ensureLoaded()
    }
  },
  { immediate: true }
)

// MOUNTED

onMounted(() => {
  // -> Seed a group that says to start open, one whose child is the page being read, or one whose own
  //    page is being read. A folder menu with its own page can still be checked on arrival; the rest
  //    of a folder's children are not here yet.
  if (
    !expanded.value &&
    (props.item.expandByDefault || ownCurrent.value || containsCurrent(props.item))
  ) {
    navExpanded.add(groupKey.value)
  }
})
</script>
