<script setup lang="ts">
import { useDebounceFn } from '@vueuse/core'
import { LewPopover, LewTree, LewTooltip } from 'lew-ui'
import type { TreeDataSource } from '../../tree'
import { object2class, numFormat } from 'lew-ui/utils'
import { treeSelectProps } from './props'
import { cloneDeep, isString } from 'lodash-es'
import Icon from 'lew-ui/utils/Icon.vue'

// 获取app
const app = getCurrentInstance()?.appContext.app
if (app && !app.directive('tooltip')) {
  app.use(LewTooltip)
}
const props = defineProps(treeSelectProps)
const emit = defineEmits(['change', 'blur', 'clear'])
const treeSelectValue: Ref<any> = defineModel()

// 校验 Model
if (!isString(treeSelectValue.value)) {
  throw new Error('tree-select modelValue must be a string')
}

const lewSelectRef = ref()
const inputRef = ref()
const lewPopoverRef = ref()
const lewTreeRef = ref()

const state = reactive({
  selectWidth: 0, // 选择框宽度
  visible: false, // 弹出框是否显示
  searchLoading: false, // 树加载
  initLoading: true, // 初始化 loading
  treeList: [], // 树列表
  hideBySelect: false, // 记录是否通过选择隐藏
  keyword: props.defaultValue || (treeSelectValue.value as any), // 搜索关键字
  keywordBackup: props.defaultValue as any // 搜索关键字备份
})

const getSelectWidth = () => {
  state.selectWidth = lewSelectRef.value?.clientWidth - 12
}

const show = () => {
  lewPopoverRef.value.show()
}

const hide = () => {
  lewPopoverRef.value.hide()
}

const searchDebounce = useDebounceFn(async (e: any) => {
  search(e)
}, props.searchDelay)

const search = async (e: any) => {
  const keyword = e.target.value
  if (props.searchable) {
    state.searchLoading = true
    await lewTreeRef.value.init(keyword)
    state.treeList = lewTreeRef.value.getTreeList()
    state.searchLoading = false
  }
}

const change = ({ item }: { item: TreeDataSource }) => {
  if (item.disabled) {
    return
  }

  state.hideBySelect = true
  // @ts-ignore
  emit('change', item[props.keyField])
  setTimeout(() => {
    hide()
  }, 100)
}

const clearHandle = () => {
  treeSelectValue.value = undefined
  state.keyword = ''
  state.keywordBackup = ''
  emit('clear')
  emit('change')
}

const getValueStyle = computed(() => {
  return state.visible ? 'opacity:0.6' : ''
})

const findKeyword = () => {
  if (lewTreeRef.value && treeSelectValue.value) {
    state.treeList = lewTreeRef.value.getTreeList()
    const treeItem: any = state.treeList.find((e: TreeDataSource) => {
      // @ts-ignore
      return e[props.keyField] === treeSelectValue.value
    })
    if (treeItem !== undefined) {
      if (
        props.showAllLevels &&
        treeItem.labelPaths &&
        treeItem.labelPaths.length > 0
      ) {
        state.keyword = treeItem.labelPaths.join(' / ')
      } else {
        state.keyword = treeItem.label[0]
      }
    }
  }
}

findKeyword()

const getSelectClassName = computed(() => {
  let { clearable, size, align, disabled, readonly, searchable } = props
  clearable = clearable ? !!treeSelectValue.value : false
  const focus = state.visible

  return object2class('lew-select', {
    clearable,
    size,
    align,
    disabled,
    readonly,
    searchable,
    focus
  })
})

const getBodyClassName = computed(() => {
  const { size, disabled } = props
  return object2class('lew-select-body', { size, disabled })
})

const getIconSize = computed(() => {
  const size: any = {
    small: 14,
    medium: 15,
    large: 16
  }
  return size[props.size]
})

const showHandle = () => {
  state.visible = true
  state.keywordBackup = cloneDeep(state.keyword)
  if (props.searchable) {
    state.keyword = ''
  }
  state.hideBySelect = false // 重置
  getSelectWidth()
  if (props.searchable && state.treeList.length === 0) {
    search({ target: { value: '' } })
  }
}

const hideHandle = () => {
  state.visible = false
  if (!state.hideBySelect && treeSelectValue.value) {
    findKeyword()
  }
  if (!treeSelectValue.value && state.keyword) {
    state.keyword = ''
    state.keywordBackup = ''
  }
  inputRef.value.blur()
  emit('blur')
}

watch(
  () => treeSelectValue.value,
  () => {
    findKeyword()
  }
)

const searchCount = computed(() => {
  return state.treeList.filter((e: TreeDataSource) => {
    return e.level === 0
  }).length
})

defineExpose({ show, hide })
</script>

<template>
  <lew-popover
    ref="lewPopoverRef"
    popoverBodyClassName="lew-select-popover-body"
    class="lew-select-view"
    :trigger="trigger"
    :disabled="disabled || readonly"
    placement="bottom-start"
    style="width: 100%"
    :loading="state.searchLoading"
    @show="showHandle"
    @hide="hideHandle"
  >
    <template #trigger>
      <div ref="lewSelectRef" class="lew-select" :class="getSelectClassName">
        <Icon
          v-if="!readonly && !state.initLoading"
          :size="getIconSize"
          type="chevron-down"
          class="icon-select"
          :class="{ 'icon-select-hide': clearable && state.keyword }"
        />
        <Icon
          v-else-if="state.initLoading"
          loading
          type="loader"
          :size="getIconSize"
          class="icon-loader"
        />
        <transition name="lew-form-icon-ani">
          <Icon
            v-if="clearable && state.keyword && !readonly"
            :size="getIconSize"
            type="close"
            class="lew-form-icon-close"
            :class="{
              'lew-form-icon-close-focus': state.visible
            }"
            @click.stop="clearHandle"
          />
        </transition>
        <input
          ref="inputRef"
          v-model="state.keyword"
          class="value"
          :style="getValueStyle"
          :readonly="!searchable"
          :placeholder="state.keywordBackup || props.placeholder"
          @input="searchDebounce"
        />
      </div>
    </template>
    <template #popover-body>
      <div
        class="lew-select-body"
        :class="getBodyClassName"
        :style="`width:${state.selectWidth}px`"
      >
        <slot name="header"></slot>

        <div class="lew-select-options-box">
          <div
            v-if="searchable && (state.treeList || []).length > 0"
            class="result-count"
          >
            共
            {{ numFormat(searchCount) }}
            条结果
          </div>
          <div class="tree-select-wrapper lew-scrollbar">
            <lew-tree
              ref="lewTreeRef"
              v-model="treeSelectValue"
              v-bind="{
                keyField,
                labelField,
                disabledField,
                showLine,
                showCheckbox,
                dataSource,
                loadMethod,
                initTree,
                expandAll
              }"
              :is-select="true"
              @init-end="state.initLoading = false"
              @change="change"
            >
              <template v-if="$slots.empty" #empty>
                <slot name="empty"></slot>
              </template>
              <template v-if="$slots.item" #item="{ props }">
                <slot name="item" :props="props"></slot>
              </template>
            </lew-tree>
          </div>
        </div>
        <slot name="footer"></slot>
      </div>
    </template>
  </lew-popover>
</template>

<style lang="scss" scoped>
.lew-select-view {
  width: 100%;

  > div {
    width: 100%;
  }

  .lew-select {
    position: relative;
    width: 100%;
    display: inline-flex;
    align-items: center;
    overflow: hidden;
    white-space: nowrap;
    text-overflow: ellipsis;
    cursor: pointer;
    user-select: none;
    box-sizing: border-box;
    border-radius: var(--lew-border-radius-small);
    background-color: var(--lew-form-bgcolor);
    transition: all var(--lew-form-transition-ease);
    box-sizing: border-box;
    border: var(--lew-form-border-width) var(--lew-form-border-color) solid;
    box-shadow: var(--lew-form-box-shadow);

    .icon-select {
      position: absolute;
      top: 50%;
      right: 9px;
      transform: translateY(-50%) rotate(0deg);
      transition: all var(--lew-form-transition-bezier);
      padding: 2px;
    }

    .icon-loader {
      position: absolute;
      top: 50%;
      right: 9px;
      animation: icon-spin 1s infinite linear;
    }
    @keyframes icon-spin {
      0% {
        transform: translateY(-50%) rotate(0deg);
      }
      100% {
        transform: translateY(-50%) rotate(360deg);
      }
    }

    .icon-select {
      opacity: var(--lew-form-icon-opacity);
    }

    .icon-select-hide {
      opacity: 0;
      transform: translate(100%, -50%);
    }

    .value {
      width: calc(100% - 24px);
      display: inline-flex;
      align-items: center;
      box-sizing: border-box;
      transition: all 0.2s;
      border: none;
      outline: none;
      background: none;
      cursor: pointer;
      overflow: hidden;
      text-overflow: ellipsis;
      color: var(--lew-text-color-1);
    }

    .value::placeholder {
      color: rgb(165, 165, 165);
    }
  }
  .lew-select-align-left {
    text-align: left;
  }

  .lew-select-align-center {
    text-align: center;
  }

  .lew-select-align-right {
    text-align: right;
  }

  .lew-select-placeholder {
    color: rgb(165, 165, 165);
  }

  .lew-select:hover {
    background-color: var(--lew-form-bgcolor-hover);
  }

  .lew-select:active {
    background-color: var(--lew-form-bgcolor-active);
  }

  .lew-select-focus {
    background-color: var(--lew-form-bgcolor-focus);
    border: var(--lew-form-border-width) var(--lew-form-border-color-focus)
      solid;

    .icon-select {
      transform: translateY(-50%) rotate(180deg);
      color: var(--lew-text-color-1);
    }

    .icon-select-hide {
      opacity: 0;
      transform: translate(100%, -50%) rotate(180deg);
    }
  }

  .lew-select-focus:hover {
    background-color: var(--lew-form-bgcolor-focus-hover);
  }

  .lew-select-disabled {
    opacity: var(--lew-disabled-opacity);
    pointer-events: none; //鼠标点击不可修改
  }

  .lew-select-readonly {
    pointer-events: none; //鼠标点击不可修改

    .lew-select {
      user-select: text;
    }
  }
  .lew-select-searchable {
    .lew-select {
      .value {
        cursor: text;
      }
    }
  }
  .lew-select-disabled:hover {
    background-color: var(--lew-form-bgcolor);
    border: var(--lew-form-border-width) var(--lew-form-border-color) solid;
  }
  .lew-select-size-small {
    height: var(--lew-form-item-height-small);
    line-height: var(--lew-form-input-line-height-small);
    .value {
      padding: var(--lew-form-input-padding-small);
      font-size: var(--lew-form-font-size-small);
    }
  }

  .lew-select-size-medium {
    height: var(--lew-form-item-height-medium);
    line-height: var(--lew-form-input-line-height-medium);
    .value {
      padding: var(--lew-form-input-padding-medium);

      font-size: var(--lew-form-font-size-medium);
    }
  }

  .lew-select-size-large {
    height: var(--lew-form-item-height-large);
    line-height: var(--lew-form-input-line-height-large);
    .value {
      padding: var(--lew-form-input-padding-large);

      font-size: var(--lew-form-font-size-large);
    }
  }
}
</style>
<style lang="scss">
.lew-select-popover-body {
  padding: 6px;
}
.lew-select-body {
  width: 100%;
  box-sizing: border-box;
  .tree-select-wrapper {
    padding: 5px 0px;
    max-height: 280px;
    overflow: auto;
  }
  .result-count {
    padding-left: 8px;
    margin: 5px 0px;
    opacity: 0.4;
    font-size: 13px;
  }

  .lew-select-options-box {
    overflow-y: auto;
    overflow-x: hidden;
    height: auto;
    box-sizing: border-box;
    transition: all 0.25s ease;
    margin-top: -2px;
    margin-bottom: -2px;
  }
}
</style>
