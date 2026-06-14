<template>
  <el-select
    v-model="selectedValues"
    multiple
    placeholder="选择或输入标签"
    filterable
    default-first-option
    :filter-method="filterTags"
    :reserve-keyword="false"
    :allow-create="false"
    :loading="isLoading"
    tag-type="primary"
    @visible-change="onDropdown"
  >
    <el-option
      v-for="tag in visibleTags"
      :key="tag.optionKey"
      :label="tag.name"
      :value="tag.optionValue"
    >
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
import { computed, ref } from 'vue'
import { ElMessage } from 'element-plus'
import { pinyin } from 'pinyin-pro'

const allTags = ref([])
const searchText = ref('')
const isLoading = ref(false)
const props = defineProps({
  allowCreate: {
    type: Boolean,
    default: false
  }
})
const model = defineModel({ type: Array })
const selectedValues = computed({
  get() {
    return (model.value || []).map(resolveTagValue)
  },
  set(nextValues) {
    void syncSelection(nextValues)
  }
})

const tagMap = computed(() => new Map(allTags.value.map((tag) => [tag.$loki, tag])))
const visibleTags = computed(() => {
  const keyword = normalizeKeyword(searchText.value)
  const filteredTags = rankTags(keyword).map(toOptionTag)

  if (!shouldShowCreateOption(keyword)) {
    return filteredTags
  }

  return [buildCreateTagOption(keyword), ...filteredTags]
})

function onDropdown(isShow) {
  if (isShow) {
    fetchData()
    return
  }

  searchText.value = ''
}

function fetchData() {
  isLoading.value = true
  window.electron.ipcRenderer
    .invoke('db:get-tags', { currentPage: 1, pageSize: 9999, joinGroup: true })
    .then((result) => {
      if (result.isSuccess) {
        allTags.value = result.data.list
      } else {
        ElMessage.error(result.msg)
        console.error(result.data)
      }
    })
    .finally(() => {
      isLoading.value = false
    })
}

function getPinyinInitials(str) {
  if (!str) return ''
  return pinyin(str, { pattern: 'first', toneType: 'none', type: 'string' }).replace(/\s+/g, '')
}

function filterTags(query) {
  searchText.value = query || ''
}

function getReadableGroups(groups) {
  return groups.map((g) => g.name).join(', ')
}

function rankTags(query) {
  const keyword = normalizeKeyword(query)

  if (!keyword) {
    return sortTagsByName(allTags.value)
  }

  const buckets = createRankBuckets()

  for (const tag of allTags.value) {
    pushMatchedTag(buckets, tag, keyword)
  }

  return collectRankedTags(buckets)
}

function buildCreateTagOption(name) {
  return {
    name: name.trim(),
    groups: [],
    optionKey: `create:${name.trim().toLowerCase()}`,
    optionValue: `create:${name.trim()}`,
    __isCreateOption: true
  }
}

function normalizeKeyword(query) {
  return (query || '').trim().toLowerCase()
}

function sortTagsByName(tags) {
  return [...tags].sort((a, b) => a.name.localeCompare(b.name, 'zh-CN'))
}

function createRankBuckets() {
  return { ne: [], np: [], pe: [], pp: [], nc: [], pc: [], gm: [] }
}

function pushMatchedTag(buckets, tag, keyword) {
  const name = (tag.name || '').toLowerCase()
  const pinyinInitials = getPinyinInitials(tag.name || '')

  // 先比名称，再比拼音，再比包含关系，最后才看分组名。
  if (name === keyword) buckets.ne.push(tag)
  else if (name.startsWith(keyword)) buckets.np.push(tag)
  else if (pinyinInitials === keyword) buckets.pe.push(tag)
  else if (pinyinInitials.startsWith(keyword)) buckets.pp.push(tag)
  else if (name.includes(keyword)) buckets.nc.push(tag)
  else if (pinyinInitials.includes(keyword)) buckets.pc.push(tag)
  else if (matchesGroupName(tag, keyword)) buckets.gm.push(tag)
}

function matchesGroupName(tag, keyword) {
  const groupNames = (tag.groups || []).map((group) => (group.name || '').toLowerCase()).join(', ')
  return groupNames.includes(keyword)
}

function collectRankedTags(buckets) {
  return Object.values(buckets).reduce((result, bucket) => {
    result.push(...sortTagsByName(bucket))
    return result
  }, [])
}

function shouldShowCreateOption(keyword) {
  return props.allowCreate && keyword && !hasExactTag(keyword)
}

function toOptionTag(tag) {
  return {
    ...tag,
    optionKey: tag.$loki,
    optionValue: tag.$loki
  }
}

function resolveTagValue(tag) {
  if (tag && typeof tag === 'object') {
    return tag.$loki
  }

  if (typeof tag === 'string') {
    const matchedTag = allTags.value.find((item) => item.name === tag)
    return matchedTag ? matchedTag.$loki : tag
  }

  return tag
}

function hasExactTag(query) {
  const keyword = normalizeKeyword(query)
  return allTags.value.some((tag) => (tag.name || '').trim().toLowerCase() === keyword)
}

async function syncSelection(nextValues) {
  if (!Array.isArray(nextValues)) {
    model.value = []
    return
  }

  const nextTags = []

  for (const value of nextValues) {
    if (typeof value === 'string' && value.startsWith('create:')) {
      const name = value.slice('create:'.length).trim()
      if (!name) continue

      const createdTag = await ensureTag(name)
      if (createdTag && !nextTags.some((tag) => tag.$loki === createdTag.$loki)) {
        nextTags.push(createdTag)
      }
      continue
    }

    const tag = tagMap.value.get(value)
    if (tag && !nextTags.some((item) => item.$loki === tag.$loki)) {
      nextTags.push(tag)
    }
  }

  model.value = nextTags
}

async function ensureTag(name) {
  const normalizedName = name.trim()
  const existingTag = allTags.value.find((tag) => (tag.name || '').trim().toLowerCase() === normalizedName.toLowerCase())

  if (existingTag) {
    return existingTag
  }

  const result = await window.electron.ipcRenderer.invoke('db:add-tag', { name: normalizedName })
  if (result.isSuccess) {
    const addedTag = result.data
    allTags.value = [addedTag, ...allTags.value.filter((tag) => tag.$loki !== addedTag.$loki)]
    return addedTag
  }

  ElMessage.error(result.msg)
  console.error(result.data)
  return null
}
</script>

<style scoped></style>
