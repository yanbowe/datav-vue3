<template>
  <div ref="scrollRankingBoard" class="dv-scroll-ranking-board">
    <div
      v-for="(item, i) in state.rows"
      :key="item.toString() + item.scroll"
      class="row-item"
      :style="`height: ${state.heights[i]}px;`"
    >
      <div class="ranking-info">
        <div class="rank">
          No.{{ item.ranking }}
        </div>
        <div class="info-name" v-html="item.name" />
        <div class="ranking-value">
          {{ state.mergedConfig.valueFormatter ? state.mergedConfig.valueFormatter(item) : item.value + state.mergedConfig.unit }}
        </div>
      </div>

      <div class="ranking-column">
        <div
          class="inside-column"
          :style="`width: ${item.percent}%;`"
        >
          <div class="shine" />
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import autoResize from 'packages/datav-vue3/utils/autoResize'
import { deepClone, deepMerge } from 'packages/datav-vue3/utils'

const props = defineProps({
  config: {
    type: Object,
    default: () => ({}),
  },
})

const scrollRankingBoard = ref(null)

const { width, height } = autoResize(scrollRankingBoard, onResize, afterAutoResizeMixinInit)

const state = reactive({
  defaultConfig: {
    /**
         * @description Board data
         * @type {Array<Object>}
         * @default data = []
         */
    data: [],
    /**
         * @description Row num
         * @type {Number}
         * @default rowNum = 5
         */
    rowNum: 5,
    /**
         * @description Scroll wait time
         * @type {Number}
         * @default waitTime = 2000
         */
    waitTime: 2000,
    /**
         * @description Carousel type
         * @type {String}
         * @default carousel = 'single'
         * @example carousel = 'single' | 'page'
         */
    carousel: 'single',
    /**
         * @description Value unit
         * @type {String}
         * @default unit = ''
         * @example unit = 'ton'
         */
    unit: '',
    /**
         * @description Auto sort by value
         * @type {Boolean}
         * @default sort = true
         */
    sort: true,
    /**
         * @description Value formatter
         * @type {Function}
         * @default valueFormatter = null
         */
    valueFormatter: null,
  },

  mergedConfig: null,

  rowsData: [],

  rows: [],

  heights: [],

  avgHeight: 0,

  animationIndex: 0,

  animationHandler: '',

  updater: 0,
})

watch(() => props.config, () => {
  stopAnimation()

  calcData()
}, {
  deep: true,
})

onUnmounted(() => {
  stopAnimation()
})

function afterAutoResizeMixinInit() {
  calcData()
}

function onResize() {
  if (!state.mergedConfig)
    return

  calcHeights(true)
}

function calcData() {
  mergeConfig()

  calcRowsData()

  calcHeights()

  animation(true)
}

function mergeConfig() {
  state.mergedConfig = deepMerge(deepClone(state.defaultConfig, true), props.config || {})
}

function calcRowsData() {
  let { data } = state.mergedConfig
  const { rowNum, sort } = state.mergedConfig

  sort && data.sort(({ value: a }, { value: b }) => {
    if (a > b)
      return -1
    else if (a < b)
      return 1
    else
      return 0
  })

  const value = data.map(({ value }) => value)

  const min = Math.min(...value) || 0

  // abs of min
  const minAbs = Math.abs(min)

  const max = Math.max(...value) || 0

  // abs of max
  const maxAbs = Math.abs(max)

  const total = max + minAbs

  data = data.map((row, i) => ({ ...row, ranking: i + 1, percent: (row.value + minAbs) / total * 100 }))

  const rowLength = data.length

  if (rowLength > rowNum && rowLength < 2 * rowNum)
    data = [...data, ...data]

  data = data.map((d, i) => ({ ...d, scroll: i }))

  state.rowsData = data
  state.rows = data
}

function calcHeights(onresize = false) {
  const { rowNum, data } = state.mergedConfig

  const avgHeight = height.value / rowNum

  state.avgHeight = avgHeight

  if (!onresize)
    state.heights = new Array(data.length).fill(avgHeight)
}

async function animation(start = false) {
  // let { avgHeight, animationIndex, mergedConfig, rowsData, animation, updater } = this

  const { waitTime, carousel, rowNum } = state.mergedConfig

  const rowLength = state.rowsData.length

  if (rowNum >= rowLength)
    return

  const { updater } = state
  if (start) {
    await new Promise(resolve => setTimeout(resolve, waitTime))
    if (updater !== state.updater)
      return
  }

  const animationNum = carousel === 'single' ? 1 : rowNum

  const rows = state.rowsData.slice(state.animationIndex)
  rows.push(...state.rowsData.slice(0, state.animationIndex))

  state.rows = rows.slice(0, rowNum + 1)
  state.heights = new Array(rowLength).fill(state.avgHeight)

  await new Promise(resolve => setTimeout(resolve, 300))
  if (updater !== state.updater)
    return

  state.heights.splice(0, animationNum, ...new Array(animationNum).fill(0))

  state.animationIndex += animationNum

  const back = state.animationIndex - rowLength
  if (back >= 0)
    state.animationIndex = back

  state.animationHandler = setTimeout(animation, waitTime - 300)
}

function stopAnimation() {
  state.updater = (state.updater + 1) % 999999

  if (!state.animationHandler)
    return

  clearTimeout(state.animationHandler)
}
</script>

<style lang="less">
@color: #1370fb;

.dv-scroll-ranking-board {
  width: 100%;
  height: 100%;
  color: #fff;
  overflow: hidden;

  .row-item {
    transition: all 0.3s;
    display: flex;
    flex-direction: column;
    justify-content: center;
    overflow: hidden;
  }

  .ranking-info {
    display: flex;
    width: 100%;
    font-size: 13px;

    .rank {
      width: 40px;
      color: @color;
    }

    .info-name {
      flex: 1;
    }
  }

  .ranking-column {
    border-bottom: 2px solid fade(@color, 50);
    margin-top: 5px;

    .inside-column {
      position: relative;
      height: 6px;
      background-color: @color;
      margin-bottom: 2px;
      border-radius: 1px;
      overflow: hidden;
    }

    .shine {
      position: absolute;
      left: 0%;
      top: 2px;
      height: 2px;
      width: 50px;
      transform: translateX(-100%);
      background: radial-gradient(rgb(40, 248, 255) 5%, transparent 80%);
      animation: shine 3s ease-in-out infinite alternate;
    }
  }
}

@keyframes shine {
  80% {
    left: 0%;
    transform: translateX(-100%);
  }

  100% {
    left: 100%;
    transform: translateX(0%);
  }
}
</style>
