<!--
  Handles:
  1. Mangled paths from language switcher (e.g. /zh/en/v0.4/... -> /en/v0.4/...)
  2. Paths with language but no version (e.g. /zh/getting-started -> /zh/v0.4/getting-started)
  3. Paths without any prefix (e.g. /getting-started -> /en/v0.4/getting-started)
  4. Real 404s (path has language + version but page doesn't exist)
-->
<script setup lang="ts">
import { useData, useRoute, useRouter, withBase } from 'vitepress'
import { computed, onMounted, ref } from 'vue'

const router = useRouter()
const route = useRoute()
const { theme } = useData()

const redirecting = ref(false)

const LATEST = 'v0.4'

function detectLang(): string {
  if (typeof navigator === 'undefined') return 'en'
  const lang = (navigator.language || (navigator as any).userLanguage || 'en').toLowerCase()
  return lang.startsWith('zh') ? 'zh' : 'en'
}

function resolveTarget(rawPath: string): string | null {
  const cleanPath = rawPath.replace(/\.html$/, '')

  // Fix mangled paths from language switcher: /zh/en/v0.4/... or /en/zh/v0.4/...
  const mangled = cleanPath.match(/^\/(zh|en)\/(zh|en)\/(v\d+\.\d+)(\/.*)?$/)
  if (mangled) {
    const innerLang = mangled[2]
    const version = mangled[3]
    const rest = mangled[4] || '/'
    return `/${innerLang}/${version}${rest}`
  }

  // Already has language + version -> real 404
  if (/^\/(en|zh)\/v\d/.test(cleanPath)) return null

  // Has language but no version -> add latest version
  const langMatch = cleanPath.match(/^\/(en|zh)(\/|$)/)
  if (langMatch) {
    const lang = langMatch[1]
    const rest = cleanPath.replace(/^\/(en|zh)/, '') || '/getting-started'
    return `/${lang}/${LATEST}${rest}`
  }

  // No language prefix -> detect from browser
  const lang = detectLang()
  return `/${lang}/${LATEST}${cleanPath}`
}

onMounted(() => {
  const target = resolveTarget(route.path)
  if (target) {
    redirecting.value = true
    router.go(target)
  }
})

const homeLink = computed(() => {
  const lang = detectLang()
  return `/${lang}/${LATEST}/`
})
</script>

<template>
  <div v-if="!redirecting" class="NotFound">
    <p class="code">{{ theme.notFound?.code ?? "404" }}</p>
    <h1 class="title">{{ theme.notFound?.title ?? "PAGE NOT FOUND" }}</h1>
    <div class="divider" />
    <blockquote class="quote">
      {{
        theme.notFound?.quote ??
        "But if you don't change your direction, and if you keep looking, you may end up where you are heading."
      }}
    </blockquote>

    <div class="action">
      <a
        class="link"
        :href="withBase(homeLink)"
        :aria-label="theme.notFound?.linkLabel ?? 'go to home'"
      >
        {{ theme.notFound?.linkText ?? "Take me home" }}
      </a>
    </div>
  </div>
</template>

<style scoped>
.NotFound {
  padding: 64px 24px 96px;
  text-align: center;
}

@media (min-width: 768px) {
  .NotFound {
    padding: 96px 32px 168px;
  }
}

.code {
  line-height: 64px;
  font-size: 64px;
  font-weight: 600;
}

.title {
  padding-top: 12px;
  letter-spacing: 2px;
  line-height: 20px;
  font-size: 20px;
  font-weight: 700;
}

.divider {
  margin: 24px auto 18px;
  width: 64px;
  height: 1px;
  background-color: var(--vp-c-divider);
}

.quote {
  margin: 0 auto;
  max-width: 256px;
  font-size: 14px;
  font-weight: 500;
  color: var(--vp-c-text-2);
}

.action {
  padding-top: 20px;
}

.link {
  display: inline-block;
  border: 1px solid var(--vp-c-brand-1);
  border-radius: 16px;
  padding: 3px 16px;
  font-size: 14px;
  font-weight: 500;
  color: var(--vp-c-brand-1);
  transition:
    border-color 0.25s,
    color 0.25s;
}

.link:hover {
  border-color: var(--vp-c-brand-2);
  color: var(--vp-c-brand-2);
}
</style>
