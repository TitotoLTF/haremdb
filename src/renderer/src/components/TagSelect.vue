<template>
  <div ref="rootRef" class="tag-select" :class="{ 'is-open': isOpen }" @click="focusInput">
    <div class="tag-select__control">
      <el-tag
        v-for="tag in selectedTags"
        :key="tag.$loki"
        size="small"
        closable
        :disable-transitions="false"
        @close.stop="removeTag(tag)"
      >
        {{ tag.name }}
      </el-tag>

      <el-input
        ref="inputRef"
        v-model="searchText"
        class="tag-select__input"
        :class="inputSizeClass"
        :placeholder="placeholder"
        :spellcheck="false"
        clearable
        @focus="openPanel"
        @blur="closePanel"
        @clear="openPanel"
        @keydown="handleKeydown"
        @input="handleInput"
      />

      <el-button
        v-if="clearable && selectedTags.length > 0"
        class="tag-select__clear"
        text
        size="small"
        @mousedown.prevent
        @click.stop="clearAll"
      >
        清空
      </el-button>
    </div>

    <transition name="el-fade-in-linear">
      <div v-if="isOpen" class="tag-select__dropdown">
        <div v-if="visibleTags.length === 0" class="tag-select__empty">没有匹配的标签</div>

        <div
          v-for="(tag, index) in visibleTags"
          :key="tag.optionKey"
          class="tag-select__option"
          :class="{ active: index === activeIndex, selected: isSelected(tag) }"
          @mouseenter="activeIndex = index"
          @mousedown.prevent="pickTag(tag)"
        >
          <el-tag size="small">{{ tag.name }}</el-tag>
          <el-text v-if="tag.groups && tag.groups.length > 0" type="info" size="small">
            {{ getReadableGroups(tag.groups) }}
          </el-text>
        </div>
      </div>
    </transition>
  </div>
</template>

<script setup>
import { computed, onMounted, ref, watch } from 'vue'
import { ElMessage } from 'element-plus'
import { pinyin } from 'pinyin-pro'

const props = defineProps({
  allowCreate: {
    type: Boolean,
    default: false
  },
  placeholder: {
    type: String,
    default: '选择或输入标签'
  },
  size: {
    type: String,
    default: 'default'
  },
  clearable: {
    type: Boolean,
    default: false
  },
  autoFocus: {
    type: Boolean,
    default: false
  }
})

const model = defineModel({ type: Array })
const emit = defineEmits(['confirm'])
const rootRef = ref(null)
const inputRef = ref(null)
const allTags = ref([])
const searchText = ref('')
const isOpen = ref(false)
const activeIndex = ref(-1)

const selectedTags = computed(() => model.value || [])
const selectedTagIds = computed(() => new Set(selectedTags.value.map((tag) => tag.$loki)))
const inputSizeClass = computed(() => (props.size === 'small' ? 'is-small' : 'is-default'))

watch(
  () => [searchText.value, allTags.value.length, props.allowCreate],
  () => {
    if (!isOpen.value) return
    activeIndex.value = visibleTags.value.length > 0 ? 0 : -1
  }
)

onMounted(() => {
  fetchData()

  if (props.autoFocus) {
    focusInput()
  }
})

const visibleTags = computed(() => {
  const keyword = normalizeKeyword(searchText.value)

  if (!keyword) {
    return []
  }

  const tags = rankTags(keyword).map(toOptionTag)

  if (shouldShowCreateOption(keyword)) {
    return [createTagOption(keyword), ...tags]
  }

  return tags
})

function openPanel() {
  isOpen.value = true
  if (allTags.value.length === 0) {
    fetchData()
  }

  focusInput()
}

function closePanel() {
  isOpen.value = false
  activeIndex.value = -1
}

function focusInput() {
  const input = inputRef.value
  if (!input) return

  input.focus?.()

  const nativeInput = input.input ?? input?.$el?.querySelector?.('input')
  nativeInput?.setSelectionRange?.(nativeInput.value.length, nativeInput.value.length)
}

defineExpose({
  focusInput
})

function handleInput() {
  if (!isOpen.value) {
    isOpen.value = true
  }
}

function handleKeydown(event) {
  if (event.key === 'ArrowDown') {
    event.preventDefault()
    openPanel()
    moveActive(1)
    return
  }

  if (event.key === 'ArrowUp') {
    event.preventDefault()
    openPanel()
    moveActive(-1)
    return
  }

  if (event.key === 'Enter') {
    event.preventDefault()

    if (!normalizeKeyword(searchText.value)) {
      closePanel()
      emit('confirm', selectedTags.value)
      return
    }

    openPanel()
    pickActiveTag()
    return
  }

  if (event.key === 'Backspace' && !searchText.value) {
    event.preventDefault()
    removeLastTag()
    return
  }

  if (event.key === 'Escape') {
    closePanel()
  }
}

function moveActive(step) {
  const options = visibleTags.value
  if (options.length === 0) {
    activeIndex.value = -1
    return
  }

  if (activeIndex.value < 0) {
    activeIndex.value = step > 0 ? 0 : options.length - 1
    return
  }

  activeIndex.value = (activeIndex.value + step + options.length) % options.length
}

function pickActiveTag() {
  const tag = visibleTags.value[activeIndex.value] || visibleTags.value[0]
  if (tag) {
    void pickTag(tag)
  }
}

async function pickTag(tag) {
  if (tag.__isCreateOption) {
    const createdTag = await ensureTag(tag.name)
    if (createdTag) {
      toggleTag(createdTag)
    }
  } else {
    toggleTag(tag)
  }

  searchText.value = ''
  activeIndex.value = -1
  openPanel()
  focusInput()
}

function toggleTag(tag) {
  const nextTags = [...selectedTags.value]
  const index = nextTags.findIndex((item) => item.$loki === tag.$loki)

  if (index === -1) {
    nextTags.push(tag)
  } else {
    nextTags.splice(index, 1)
  }

  model.value = nextTags
}

function removeTag(tag) {
  model.value = selectedTags.value.filter((item) => item.$loki !== tag.$loki)
}

function removeLastTag() {
  if (selectedTags.value.length === 0) return
  model.value = selectedTags.value.slice(0, -1)
}

function clearAll() {
  model.value = []
  searchText.value = ''
  activeIndex.value = -1
  openPanel()
  focusInput()
}

function fetchData() {
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
    })
}

function normalizeKeyword(query) {
  return (query || '').trim().toLowerCase()
}

function rankTags(keyword) {
  if (!keyword) {
    return [...allTags.value].sort((a, b) => a.name.localeCompare(b.name, 'zh-CN'))
  }

  return [...allTags.value]
    .filter((tag) => getTagRank(tag, keyword) < 99)
    .sort((a, b) => {
      const rankDiff = getTagRank(a, keyword) - getTagRank(b, keyword)
      if (rankDiff !== 0) return rankDiff
      return a.name.localeCompare(b.name, 'zh-CN')
    })
}

function getTagRank(tag, keyword) {
  const name = (tag.name || '').toLowerCase()
  const initials = getPinyinInitials(tag.name || '')
  const groupNames = (tag.groups || []).map((group) => (group.name || '').toLowerCase()).join(', ')

  // 排序顺序是固定的：完全命中 > 前缀命中 > 拼音命中 > 包含命中 > 分组命中。
  if (name === keyword) return 0
  if (name.startsWith(keyword)) return 1
  if (initials === keyword) return 2
  if (initials.startsWith(keyword)) return 3
  if (name.includes(keyword)) return 4
  if (initials.includes(keyword)) return 5
  if (groupNames.includes(keyword)) return 6
  return 99
}

function getPinyinInitials(str) {
  if (!str) return ''
  return pinyin(str, { pattern: 'first', toneType: 'none', type: 'string' }).replace(/\s+/g, '')
}

function createTagOption(name) {
  return {
    $loki: `create:${name}`,
    optionKey: `create:${name}`,
    name,
    groups: [],
    __isCreateOption: true
  }
}

function shouldShowCreateOption(keyword) {
  return props.allowCreate && Boolean(keyword) && !hasExactTag(keyword)
}

function hasExactTag(keyword) {
  return allTags.value.some((tag) => (tag.name || '').trim().toLowerCase() === keyword)
}

function toOptionTag(tag) {
  return {
    ...tag,
    optionKey: tag.$loki
  }
}

function isSelected(tag) {
  return selectedTagIds.value.has(tag.$loki)
}

function getReadableGroups(groups) {
  return groups.map((group) => group.name).join(', ')
}

async function ensureTag(name) {
  const normalizedName = name.trim()
  if (!normalizedName) return null

  const existingTag = allTags.value.find(
    (tag) => (tag.name || '').trim().toLowerCase() === normalizedName.toLowerCase()
  )
  if (existingTag) return existingTag

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

<style scoped>
.tag-select {
  position: relative;
  width: 100%;
}

.tag-select__control {
  min-height: 32px;
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  gap: 6px;
  padding: 4px 8px;
  border: 1px solid var(--el-border-color);
  border-radius: 6px;
  background: var(--el-fill-color-blank);
  transition:
    border-color 0.2s ease,
    box-shadow 0.2s ease;
}

.tag-select.is-open .tag-select__control {
  border-color: var(--el-color-primary);
  box-shadow: 0 0 0 1px var(--el-color-primary-light-7) inset;
}

.tag-select__input {
  min-width: 140px;
  flex: 1 1 140px;
}

.tag-select__input.is-small :deep(.el-input__wrapper) {
  min-height: 24px;
}

.tag-select__input.is-default :deep(.el-input__wrapper) {
  min-height: 28px;
}

.tag-select__clear {
  margin-left: auto;
  color: var(--el-color-danger);
}

.tag-select__dropdown {
  position: absolute;
  z-index: 20;
  top: calc(100% + 6px);
  left: 0;
  right: 0;
  max-height: 260px;
  overflow: auto;
  padding: 4px;
  border: 1px solid var(--el-border-color-light);
  border-radius: 8px;
  background: var(--el-bg-color);
  box-shadow: var(--el-box-shadow-light);
}

.tag-select__option {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  padding: 8px 10px;
  border-radius: 6px;
  cursor: pointer;
}

.tag-select__option.active {
  background: var(--el-color-primary-light-9);
}

.tag-select__option.selected {
  color: var(--el-color-primary);
}

.tag-select__empty {
  padding: 10px;
  color: var(--el-text-color-secondary);
  text-align: center;
}
</style>