<!--
  Fixes two issues:
  1. Root "/" auto-redirects to latest version based on browser language
  2. Language switcher path mangling (e.g. /zh/en/v0.4/... -> /en/v0.4/...)
-->
<script setup lang="ts">
import { useRoute, useRouter } from 'vitepress'
import { onMounted, ref } from 'vue'

const router = useRouter()
const route = useRoute()

const LATEST = 'v0.4'
let redirected = false

function detectLang(): string {
  if (typeof navigator === 'undefined') return 'en'
  const lang = (navigator.language || (navigator as any).userLanguage || 'en').toLowerCase()
  return lang.startsWith('zh') ? 'zh' : 'en'
}

function fixPath(path: string): string | null {
  const clean = path.replace(/\.html$/, '').replace(/\/$/, '') || '/'

  // Root -> redirect to latest version in browser language
  if (clean === '/') {
    return `/${detectLang()}/${LATEST}/`
  }

  // Fix mangled paths: /zh/en/... or /en/zh/...
  const mangled = clean.match(/^\/(zh|en)\/(zh|en)\/(v\d+\.\d+)(\/.*)?$/)
  if (mangled) {
    const innerLang = mangled[2]
    const version = mangled[3]
    const rest = mangled[4] || '/'
    return `/${innerLang}/${version}${rest}`
  }

  return null
}

onMounted(() => {
  if (redirected) return
  const target = fixPath(route.path)
  if (target) {
    redirected = true
    router.go(target)
  }
})
</script>
