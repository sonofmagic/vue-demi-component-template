<script lang="ts" setup>
import { h, nextTick, onBeforeMount, ref } from 'vue-demi'
import { Checkbox, MessageBox } from 'element-ui'
import dayjs from 'dayjs'
// @ts-expect-error
import { cloneDeep } from 'lodash-es'
import type { ContextMenuSlotContext, ICellAttrs, IScrollOffset, ItemComponentProps, VSheetType } from '@sonofmagic/ui'
import { Popover, Sheet, SheetCell, VirtualList, useDataSource, usePopover, vScrollbar } from '@sonofmagic/ui'
import Item from './item.vue'
import yAxisItem from './yAxisItem.vue'
const sheetRef = ref<VSheetType>()
const page = ref(0)
const { columns, dataSource, transform } = useDataSource(() => {
  const columns = []

  const columnsLength = 30
  const firstDay = dayjs().startOf('M')
  for (let i = 0; i < columnsLength; i++) {
    columns.push({
      width: 120,
      title: firstDay.add(i, 'day').format('YYYY-MM-DD'),
      key: i,
    })
  }

  return {
    columns,
    // rows,
    // childrenKey: 'shiftId',
    rowKey: 'personId',
    children: 'shiftList',
  }
})
let mockData
onBeforeMount(async () => {
  const { data } = await import('./mock.json')
  mockData = data
  for (let i = 0; i < 2; i++) {
    dataSource.value.push(...transform(data.map((x, j) => {
      return {
        ...x,
        personId: x.personId + i + j,
      }
    })))
  }
})
const { context: subPopoverContext } = usePopover()
const dom = ref<HTMLDivElement>()
const syncScroll = ({ scrollLeft, scrollTop }: IScrollOffset) => {
  if (dom.value) {
    dom.value.scrollLeft = scrollLeft
    dom.value.scrollTop = scrollTop
  }
}

const onPrompt = ({ defaultValue, isSingle }: { isSingle: boolean; defaultValue?: string }) => {
  return MessageBox.prompt('', '添加备注', {
    inputType: 'textArea',
    inputValue: defaultValue,
    inputPlaceholder: '请输入备注',
    closeOnClickModal: false,
    closeOnPressEscape: false,
  })
}

const closeContextMenu = ({ menuContext }: ContextMenuSlotContext) => {
  menuContext.close()
}

function doLock({ menuContext, selectedCellSet }: ContextMenuSlotContext) {
  selectedCellSet?.forEach((x) => {
    x.locked = true
  })
  menuContext.close()
}

function unlock({ menuContext, selectedCellSet }: ContextMenuSlotContext) {
  selectedCellSet?.forEach((x) => {
    x.locked = false
  })
  menuContext.close()
}

async function doNote({ menuContext, selectedCellSet }: ContextMenuSlotContext) {
  if (selectedCellSet) {
    const isSingle = selectedCellSet.size === 1
    // @ts-expect-error
    const defaultValue = isSingle ? Array.from(selectedCellSet.values())[0].value.remark : ''
    const res = await onPrompt({
      defaultValue,
      isSingle,
    })

    selectedCellSet?.forEach((x) => {
      // @ts-expect-error
      x.value.remark = res.value
    })
  }
}

function doSetValue({ menuContext, selectedCellSet }: ContextMenuSlotContext, value?: unknown) {
  selectedCellSet.forEach((x) => {
    x.value = cloneDeep(value)
  })
  menuContext.close()
}

function setDisabled({ selectedCellSet, menuContext }: ContextMenuSlotContext, flag: boolean) {
  selectedCellSet.forEach((x) => {
    x.disabled = flag
  })
  menuContext.close()
}

const selectValue: (e: MouseEvent, attrs: ICellAttrs, value: unknown) => void = (e, attrs, value) => {
  e.stopPropagation()

  if (attrs)
    attrs.item.value = value
}

const itemScopedSlots = {
  default: (props: ItemComponentProps) => {
    return h(Item, {
      props,
    })
  },
}

function wait(ts = 1000) {
  return new Promise((resolve) => {
    setTimeout(() => {
      resolve(ts)
    }, ts)
  })
}

function expandSubPopover(ctx: ContextMenuSlotContext, e: MouseEvent) {
  console.log(ctx, e)
}

async function onScroll2Bottom({ yAxisScrollbar }) {
  // console.log(yAxisScrollbar)
  await wait()
  // console.log(mockData)
  dataSource.value.push(...transform(cloneDeep(mockData.map((x, j) => {
    return {
      ...x,
      personId: x.personId + j + Math.random(),
    }
  }))))
  nextTick(() => {
    yAxisScrollbar?.update()
  })

  page.value++
  console.log('fetchNewRows')
}

function resetColumns(ctx: ContextMenuSlotContext) {
  const _columns = []

  const columnsLength = 10
  const firstDay = dayjs('2022-06-01').startOf('M')
  for (let i = 0; i < columnsLength; i++) {
    _columns.push({
      width: 120,
      title: firstDay.add(i, 'day').format('YYYY-MM-DD'),
      key: i,
    })
  }
  columns.value = _columns
}
</script>

<template>
  <div class="max-w-[100vw] max-h-[100vh]">
    <div class="flex relative p-16 h-screen">
      <div class="mr-2 flex-shrink-0 w-[200px] flex flex-col overflow-hidden">
        <div class="text-lg p-2 h-[48px] flex-shrink-0">
          阿斯蒂芬
        </div>
        <!-- <VirtualList
          ref="dom" class="flex-1 overflow-y-auto w-full" data-key="personId" :data-sources="rows"
          :data-component="yAxisItem"
        /> -->
        <div ref="dom" class="flex-1 overflow-y-hidden">
          <div class="table border-collapse w-full">
            <div class="table-row-group">
              <div v-for="(row, idx) in dataSource" :key="row.key" class="h-[48px] border table-row">
                <div class="table-cell p-2">
                  <Checkbox :true-label="true" :false-label="false" @change="(v) => sheetRef?.selectRow(idx, v)" />
                  选中第{{ idx + 1 }}行
                </div>
              </div>
            </div>
          </div>
        </div>
        <!-- <div class="flex-1 flex"> -->

        <!-- <div class="table border-collapse w-full">
            <div class="table-row-group">
              <div v-for="(row, idx) in rows" :key="row.key" class="h-[48px] border table-row">
                <div class="table-cell p-2">
                  <Checkbox
                    :true-label="true" :false-label="false"
                    @change="(v) => sheetRef?.selectRow(idx, v)"
                  />
                  选中第{{ idx + 1 }}行
                </div>
              </div>
            </div>
          </div> -->
        <!-- </div> -->
      </div>

      <!-- <Sheet :columns="columns" :dataSource="dataSource" @scroll="syncScroll"></Sheet> -->
      <Sheet
        ref="sheetRef" :item-scoped-slots="itemScopedSlots" :columns="columns" :data-source="dataSource"
        :item-component="SheetCell" :on-scroll-to-bottom="onScroll2Bottom" @scroll="syncScroll"
      >
        <template #context-menu="ctx">
          <div class="w-32 text-center">
            <div class="w-32 text-center">
              <!-- <div
                class="hover:bg-blue-200 hover:text-blue-600 px-4 py-1 cursor-pointer"
                @click="closeContextMenu(ctx)"
              >
                复制
              </div>
              <div
                class="hover:bg-blue-200 hover:text-blue-600 px-4 py-1 cursor-pointer"
                @click="closeContextMenu(ctx)"
              >
                粘贴
              </div> -->
              <div class="hover:bg-blue-200 hover:text-blue-600 px-4 py-1 cursor-pointer" @click="doLock(ctx)">
                锁定
              </div>
              <div class="hover:bg-blue-200 hover:text-blue-600 px-4 py-1 cursor-pointer" @click="unlock(ctx)">
                解锁
              </div>
              <div class="hover:bg-blue-200 hover:text-blue-600 px-4 py-1 cursor-pointer" @click="doNote(ctx)">
                备注
              </div>
              <div
                class="hover:bg-blue-200 hover:text-blue-600 px-4 py-1 cursor-pointer"
                @click="setDisabled(ctx, true)"
              >
                禁用
              </div>
              <div
                class="hover:bg-blue-200 hover:text-blue-600 px-4 py-1 cursor-pointer"
                @click="setDisabled(ctx, false)"
              >
                解禁
              </div>
              <div
                class="hover:bg-blue-200 hover:text-blue-600 px-4 py-1 cursor-pointer"
                @click="expandSubPopover(ctx, $event)"
              >
                行/列复制
              </div>

              <div
                class="hover:bg-blue-200 hover:text-blue-600 px-4 py-1 cursor-pointer"
                @click="closeContextMenu(ctx)"
              >
                复制上一区间
              </div>
              <div
                class="hover:bg-blue-200 hover:text-blue-600 px-4 py-1 cursor-pointer" @click="doSetValue(ctx, {
                  name: '测试数据',
                  startTime: '11:11',
                  endTime: '11:11',
                  remark: '',
                })"
              >
                set value
              </div>
              <div class="hover:bg-blue-200 hover:text-blue-600 px-4 py-1 cursor-pointer" @click="doSetValue(ctx)">
                clear
              </div>
              <div class="hover:bg-blue-200 hover:text-blue-600 px-4 py-1 cursor-pointer" @click="resetColumns(ctx)">
                reset columns
              </div>
              <Popover :context="subPopoverContext" placement="right-start">
                <div class="w-32 text-center">
                  <div class="w-32 text-center">
                    <div class="hover:bg-blue-200 hover:text-blue-600 px-4 py-1 cursor-pointer" @click="doLock(ctx)">
                      从上往下
                    </div>
                    <div class="hover:bg-blue-200 hover:text-blue-600 px-4 py-1 cursor-pointer" @click="unlock(ctx)">
                      从下往上
                    </div>
                    <div class="hover:bg-blue-200 hover:text-blue-600 px-4 py-1 cursor-pointer" @click="doNote(ctx)">
                      从左往右
                    </div>
                    <div
                      class="hover:bg-blue-200 hover:text-blue-600 px-4 py-1 cursor-pointer"
                      @click="setDisabled(ctx, true)"
                    >
                      从右往左
                    </div>
                  </div>
                </div>
              </Popover>
            </div>
          </div>
        </template>
        <template #detail="{ attrs }">
          <div class="bg-white w-[160px] text-xs border px-2 py-1 space-y-1">
            <div class="text-[13px] text-[#333333]">
              {{ attrs?.item.value.name }}
            </div>
            <div class="text-[#333333]">
              {{ attrs?.item.value.startTime }}-{{ attrs?.item.value.endTime }} {{ attrs?.item.value.duration }}h
            </div>
            <div v-if="attrs?.item.value.remark" class="text-[#B1B9CC]">
              备注:{{ attrs?.item.value.remark }}
            </div>
          </div>
        </template>
        <template #value-selector="{ attrs }">
          <div class="bg-white w-[360px] p-2 border">
            <div>未定义</div>
            <input class="border" placeholder="请输入">
            <div class="overflow-auto h-[200px]">
              <div
                v-for="i in 30" :key="i" class="flex justify-around cursor-pointer hover:bg-blue-300"
                @click="selectValue($event, attrs, i)"
              >
                <div class="flex-1">
                  撒大声地
                </div>
                <div class="flex-1">
                  {{ i }}
                </div>
              </div>
            </div>
          </div>
        </template>
      </Sheet>
    </div>
  </div>
</template>

<style lang="scss">
.vue-dom-sheet-context-menu {
  @apply border bg-white;
}
</style>
