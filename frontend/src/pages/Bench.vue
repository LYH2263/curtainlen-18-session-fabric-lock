<script setup>
import { onMounted, ref, watch } from 'vue'
import { getJSON, postJSON } from '../api'
import PanelCut from '../components/PanelCut.vue'
const windows = ref([]); const fabrics = ref([]); const wid = ref(1); const fid = ref(1); const out = ref(null)
// 会话锁布：仅存内存，不落库、不写 calc_runs、不污染窗户主数据
const locked = ref(false)
let defaultFabricId = null
function resetFabricToDefault(){ if (defaultFabricId != null) fid.value = defaultFabricId }
onMounted(async () => {
  windows.value = (await getJSON('/api/windows')).items.filter(x=>x.data_quality==='clean')
  fabrics.value = (await getJSON('/api/fabrics')).items.filter(x=>x.data_quality==='clean')
  if (windows.value.length) wid.value = windows.value[0].id
  const sys = (await getJSON('/api/settings')).default_fabric_id
  defaultFabricId = (sys != null && fabrics.value.some(x=>x.id===Number(sys)))
    ? Number(sys) : (fabrics.value[0]?.id ?? null)
  if (defaultFabricId != null) fid.value = defaultFabricId
})
// 换窗：锁定则保持当前布；未锁定则回到系统默认布（解锁本身不重置，下次换窗才回默认）
watch(wid, () => { if (!locked.value) resetFabricToDefault() })
function toggleLock(){ locked.value = !locked.value }
async function go(save){ out.value = save ? await postJSON('/api/estimate',{window_id:wid.value,fabric_id:fid.value,save:true}) : await getJSON(`/api/estimate?window_id=${wid.value}&fabric_id=${fid.value}`) }
</script>
<template><div class="page"><h1>算料</h1>
<select v-model.number="wid"><option v-for="x in windows" :key="x.id" :value="x.id">{{ x.name }}</option></select>
<select v-model.number="fid" :disabled="locked"><option v-for="x in fabrics" :key="x.id" :value="x.id">{{ x.name }}</option></select>
<button class="lock-btn" :class="{locked}" @click="toggleLock">{{ locked ? '解锁' : '锁定' }}</button>
<span v-if="locked" class="locked-hint">已锁定：换窗仍用此布</span>
<button @click="go(false)">试算</button><button @click="go(true)">保存</button>
<PanelCut v-if="out" :panels="out.panels" :cut-height="out.cut_height" :meters="out.meters" />
</div></template>
