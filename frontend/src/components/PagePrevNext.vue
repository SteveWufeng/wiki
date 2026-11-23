<template>
  <!--
    The page's place in the sidebar, read as a sequence. The list is flattened server menus down into
    links only: a reader reaches the next or previous doc without opening every folder in the menu by
    hand, and the chrome stays quiet when the page is nowhere in the menu the sidebar is showing.
  -->
  <div v-if="prev || next" class="page-prev-next mt-8">
    <w-separator class="mb-4" />
    <div class="flex flex-wrap gap-2">
      <div class="min-w-0 flex-1" v-if="prev">
        <w-btn
          :to="itemHref(prev.path)"
          :aria-label="t('common.page.prevDocAria', { title: prev.title })"
          outline
          no-caps
          padding="sm md"
          color="primary">
          <w-icon class="me-2" name="la:arrow-left" />
          <div class="flex flex-col items-start text-left">
            <div class="text-caption text-grey-6">{{ t('common.page.prevDoc') }}</div>
            <div class="text-body2 truncate">{{ prev.title }}</div>
          </div>
        </w-btn>
      </div>
      <div class="min-w-0 flex-1 text-end" v-if="next">
        <w-btn
          :to="itemHref(next.path)"
          :aria-label="t('common.page.nextDocAria', { title: next.title })"
          outline
          no-caps
          padding="sm md"
          color="primary">
          <div class="flex flex-col items-end text-left">
            <div class="text-caption text-grey-6">{{ t('common.page.nextDoc') }}</div>
            <div class="text-body2 truncate">{{ next.title }}</div>
          </div>
          <w-icon class="ms-2" name="la:arrow-right" />
        </w-btn>
      </div>
    </div>
  </div>
</template>

<script setup>
import { computed, onMounted, ref, watch } from 'vue'
import { useI18n } from 'vue-i18n'

import { usePageStore } from '@/stores/page'
import { useSiteStore } from '@/stores/site'

/**
 * Previous/Next links for the page list shown in the sidebar.
 *
 * The sidebar opens folders lazily, so the bottom of the page needs the flattened order. It asks for
 * the menu currently driving the sidebar, follows `menu` items into their submenus, and then uses the
 * current page's path to decide which neighbours, if any, are useful.
 */

const pageStore = usePageStore()
const siteStore = useSiteStore()
const { t } = useI18n()

const sequence = ref([])

const currentPath = computed(() => pageStore.path || 'home')

const currentIndex = computed(() => sequence.value.findIndex((item) => item.path === currentPath.value))

const prev = computed(() => (currentIndex.value > 0 ? sequence.value[currentIndex.value - 1] : null))
const next = computed(() =>
  currentIndex.value >= 0 && currentIndex.value < sequence.value.length - 1
    ? sequence.value[currentIndex.value + 1]
    : null
)

let request = 0

async function load() {
  const navId = siteStore.nav.displayId || siteStore.nav.currentId
  const currentRequest = ++request
  if (!navId || !pageStore.id) {
    sequence.value = []
    return
  }
  const result = await siteStore.fetchNavigationSequence(navId, pageStore.locale)
  if (currentRequest === request) {
    sequence.value = result
  }
}

function itemHref(path) {
  return `${siteStore.localeUrlPrefix(pageStore.locale)}/${path}`
}

onMounted(load)
watch(
  () => [
    pageStore.id,
    pageStore.locale,
    pageStore.path,
    siteStore.nav.displayId,
    siteStore.nav.currentId
  ],
  load
)
</script>

<style lang="scss" scoped>
.page-prev-next {
  :deep(.w-btn) {
    max-width: 100%;
  }
}
</style>
