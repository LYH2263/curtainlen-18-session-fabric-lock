<script setup>
import { computed, onMounted, ref, watch } from 'vue'
import { getJSON, postJSON } from '../api'
import PanelCut from '../components/PanelCut.vue'
// 会话锁布：锁状态只存 sessionStorage（会话级），不落库、不写 calc_runs
const LOCK_KEY = 'curtainlen.fabricLock'
const windows = ref([]); const fabrics = ref([]); const wid = ref(1); const fid = ref(1); const out = ref(null)
const lockedFid = ref(null)
const locked = computed(() => lockedFid.value !== null)
// 该窗关联默认优先，否则系统默认（首个 clean 布）
function defaultFid(w) {
  if (w && w.default_fabric_id && fabrics.value.some(x => x.id === w.default_fabric_id)) return w.default_fabric_id
  return fabrics.value.length ? fabrics.value[0].id : fid.value
}
function applyFabric() {
  const w = windows.value.find(x => x.id === wid.value)
  fid.value = locked.value ? lockedFid.value : defaultFid(w)
}
onMounted(async () => {
  windows.value = (await getJSON('/api/windows')).items.filter(x=>x.data_quality==='clean')
  fabrics.value = (await getJSON('/api/fabrics')).items.filter(x=>x.data_quality==='clean')
  try {
    const saved = JSON.parse(sessionStorage.getItem(LOCK_KEY) || 'null')
    if (saved && fabrics.value.some(x => x.id === saved.fabric_id)) lockedFid.value = saved.fabric_id
    else if (saved) sessionStorage.removeItem(LOCK_KEY)
  } catch { sessionStorage.removeItem(LOCK_KEY) }
  if (windows.value.length) wid.value = windows.value[0].id
  applyFabric()
})
watch(wid, applyFabric)
function lock() { lockedFid.value = fid.value; sessionStorage.setItem(LOCK_KEY, JSON.stringify({ fabric_id: fid.value })) }
function unlock() { lockedFid.value = null; sessionStorage.removeItem(LOCK_KEY) }
async function go(save){ out.value = save ? await postJSON('/api/estimate',{window_id:wid.value,fabric_id:fid.value,save:true}) : await getJSON(`/api/estimate?window_id=${wid.value}&fabric_id=${fid.value}`) }
</script>
<template><div class="page"><h1>算料</h1>
<select v-model.number="wid"><option v-for="x in windows" :key="x.id" :value="x.id">{{ x.name }}</option></select>
<select v-model.number="fid" :disabled="locked"><option v-for="x in fabrics" :key="x.id" :value="x.id">{{ x.name }}</option></select>
<button v-if="!locked" @click="lock">锁布</button><button v-else @click="unlock">解锁</button>
<span v-if="locked" class="lock-tag">已锁布</span>
<button @click="go(false)">试算</button><button @click="go(true)">保存</button>
<PanelCut v-if="out" :panels="out.panels" :cut-height="out.cut_height" :meters="out.meters" />
</div></template>
