<template>
  <el-select
    v-model="model"
    multiple
    placeholder="选择或输入标签"
    filterable
    :filter-method="filterTags"
    :reserve-keyword="false"
    value-key="$loki"
    allow-create
    default-first-option
    tag-type="primary"
    @visible-change="onDropdown"
  >
    <el-option v-for="tag in displayTags" :key="tag.$loki" :label="tag.name" :value="tag">
      <div class="flex items-center justify-between h-full">
        <el-tag size="small">{{ tag.name }}</el-tag>
        <el-text v-if="tag && tag.groups" type="info" size="small">
          {{ getReadableGroups(tag.groups) }}
        </el-text>
      </div>
    </el-option>
  </el-select>
</template>

<script setup>
import { watch, ref } from 'vue'
import { ElMessage } from 'element-plus'
import { pinyin } from 'pinyin-pro'

const allTags = ref([])
const displayTags = ref([])
const model = defineModel({ type: Array })
const isDropdownShow = ref(false)

watch(
  () => model.value,
  () => {
    if (!isDropdownShow.value) displayTags.value = model.value
  },
  { immediate: true }
)

function onDropdown(isShow) {
  isDropdownShow.value = isShow
  if (isShow) fetchData()
}

function fetchData() {
  window.electron.ipcRenderer
    .invoke('db:get-tags', { currentPage: 1, pageSize: 9999, joinGroup: true })
    .then((result) => {
      if (result.isSuccess) {
        allTags.value = result.data.list
        displayTags.value = result.data.list
      } else {
        ElMessage.error(result.msg)
        console.error(result.data)
      }
    })
}

function getPinyinInitials(str) {
  if (!str) return ''
  return pinyin(str, { pattern: 'first', toneType: 'none', type: 'string' }).replace(/\s+/g, '')
}

/**
 * 按输入文本前缀匹配排序，支持拼音首字母搜索。
 * 优先级：名称完全 > 名称前缀 > 拼音完全 > 拼音前缀 > 名称包含 > 拼音包含 > 分组名包含
 */
function filterTags(query) {
  if (!query || !query.trim()) {
    displayTags.value = allTags.value
    return
  }

  const keyword = query.trim().toLowerCase()
  const buckets = { ne: [], np: [], pe: [], pp: [], nc: [], pc: [], gm: [] }

  for (const tag of allTags.value) {
    const name = (tag.name || '').toLowerCase()
    const py = getPinyinInitials(tag.name || '')

    if (name === keyword) buckets.ne.push(tag)
    else if (name.startsWith(keyword)) buckets.np.push(tag)
    else if (py === keyword) buckets.pe.push(tag)
    else if (py.startsWith(keyword)) buckets.pp.push(tag)
    else if (name.includes(keyword)) buckets.nc.push(tag)
    else if (py.includes(keyword)) buckets.pc.push(tag)
    else {
      const groupNames = (tag.groups || []).map((g) => (g.name || '').toLowerCase()).join(', ')
      if (groupNames.includes(keyword)) buckets.gm.push(tag)
    }
  }

  const sortByName = (a, b) => a.name.localeCompare(b.name, 'zh-CN')
  displayTags.value = Object.values(buckets)
    .reduce((arr, bucket) => (bucket.sort(sortByName), arr.push(...bucket), arr), [])
}

function getReadableGroups(groups) {
  return groups.map((g) => g.name).join(', ')
}
</script>

<style scoped></style>
