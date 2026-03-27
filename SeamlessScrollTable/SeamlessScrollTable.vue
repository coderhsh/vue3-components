<template>
  <!--
    根容器
    1. rootStyle 用来注入行高 CSS 变量
    2. mouseenter / mouseleave 控制 hover 暂停
    3. wheel.prevent 拦截默认滚轮行为，把滚轮交给组件内部处理
  -->
  <div
    class="seamless-scroll-table"
    :style="rootStyle"
    @mouseenter="handleMouseEnter"
    @mouseleave="handleMouseLeave"
    @wheel.prevent="handleWheel"
  >
    <!-- 表头区域：列布局和 body 保持一致 -->
    <div class="table-header" :style="gridStyle">
      <div
        v-for="(col, index) in columns"
        :key="`${col.key}-${index}`"
        class="th"
        :style="getCellStyle(col)"
      >
        <!--
          表头插槽优先级：
          1. header-列key    -> 单独控制某一列
          2. header          -> 通用表头
          3. col.title       -> 默认标题
        -->
        <slot :name="`header-${col.key}`" :column="col" :index="index">
          <slot name="header" :column="col" :index="index">
            {{ col.title }}
          </slot>
        </slot>
      </div>
    </div>

    <!--
      表体可视区域
      高度固定 = rowHeight * visibleCount
      真正滚动的是内部 virtual-track
    -->
    <div ref="bodyRef" class="table-body" :style="bodyStyle">
      <div class="scroll-viewport">
        <!--
          虚拟列表轨道：
          这里只渲染可视区附近的少量行，而不是整张表
          transform 通过 currentOffset 计算，实现“无限循环滚动”
        -->
        <div class="virtual-track" :style="trackStyle">
          <div
            v-for="item in virtualRows"
            :key="item.renderKey"
            class="tr"
            :class="item.rowClass"
            :style="[gridStyle, getRowStyle(item.top)]"
          >
            <div
              v-for="(col, colIndex) in columns"
              :key="`${col.key}-${colIndex}`"
              class="td"
              :style="getCellStyle(col)"
            >
              <!--
                行内容插槽优先级建议：
                1. row-列key    -> 单独控制某一列
                2. row          -> 通用单元格渲染
                3. 默认文本渲染
              -->
              <slot
                :row="item.row"
                :value="item.row[col.key]"
                :column="col"
                :row-index="item.dataIndex"
                :col-index="colIndex"
              >
                <slot
                  :name="`row-${col.key}`"
                  :row="item.row"
                  :value="item.row[col.key]"
                  :column="col"
                  :row-index="item.dataIndex"
                  :col-index="colIndex"
                >
                  <slot
                    name="row"
                    :row="item.row"
                    :value="item.row[col.key]"
                    :column="col"
                    :row-index="item.dataIndex"
                    :col-index="colIndex"
                  >
                    {{ item.row[col.key] ?? '--' }}
                  </slot>
                </slot>
              </slot>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
  import { computed, nextTick, onBeforeUnmount, onMounted, ref, watch } from 'vue'

  const props = defineProps({
    /** 列配置 */
    columns: {
      type: Array,
      default: () => [],
    },
    /** 表格数据 */
    data: {
      type: Array,
      default: () => [],
    },
    /** 单行高度，虚拟列表和滚动逻辑都依赖这个值 */
    rowHeight: {
      type: Number,
      default: 40,
    },
    /** 可视区展示多少行 */
    visibleCount: {
      type: Number,
      default: 6,
    },
    /**
     * 滚动模式
     * continuous = 连续滚动
     * step = 按行停顿滚动
     */
    mode: {
      type: String,
      default: 'continuous',
    },
    /**
     * 自动滚动速度，单位 px/s
     */
    speed: {
      type: Number,
      default: 30,
    },
    /**
     * step 模式下每滚动一行后停顿多久，单位 ms
     */
    pauseDuration: {
      type: Number,
      default: 1200,
    },
    /** 鼠标悬浮时是否暂停自动滚动 */
    pauseOnHover: {
      type: Boolean,
      default: true,
    },
    /** 是否显示斑马纹 */
    stripe: {
      type: Boolean,
      default: true,
    },
    /** 行唯一标识字段 */
    rowKey: {
      type: String,
      default: '',
    },

    /**
     * 虚拟列表缓冲区行数
     * 实际渲染数 = visibleCount + buffer * 2 + 2
     * 适当的 buffer 可以避免滚动时边缘闪烁
     */
    buffer: {
      type: Number,
      default: 3,
    },

    /** 是否启用鼠标滚轮控制 */
    wheelControl: {
      type: Boolean,
      default: true,
    },

    /**
     * 用户滚轮操作后，自动滚动需要暂停多久
     * 避免“用户刚滚完，自动滚动立刻抢控制权”
     */
    wheelPauseDuration: {
      type: Number,
      default: 1800,
    },

    /**
     * continuous 模式下滚轮灵敏度
     * deltaY * wheelStep = 本次滚轮注入的位移
     */
    wheelStep: {
      type: Number,
      default: 0.9,
    },

    /**
     * step 模式下滚轮每次移动多少行
     */
    wheelStepRows: {
      type: Number,
      default: 1,
    },

    /**
     * 手动目标推进速度倍率
     * 用在 scrollToRow / wheel 惯性追赶等场景
     */
    manualSpeedRatio: {
      type: Number,
      default: 10,
    },

    /**
     * 滚轮停止多久后开始自动吸附到最近整行
     */
    wheelSnapDelay: {
      type: Number,
      default: 120,
    },

    /**
     * 吸附到整行时的速度倍率
     */
    snapSpeedRatio: {
      type: Number,
      default: 12,
    },
  })

  /** 表体 DOM，用于 ResizeObserver 监听尺寸变化 */
  const bodyRef = ref(null)

  /**
   * 当前滚动偏移量，单位 px
   * 它表示“逻辑列表”已经向上滚动了多少距离
   */
  const currentOffset = ref(0)

  /** hover 暂停标记 */
  const isHoverPaused = ref(false)

  /**
   * 手动暂停标记
   * 对外 expose 的 play / pause 操作的是它
   */
  const isManualPaused = ref(false)

  /** requestAnimationFrame id */
  let rafId = 0

  /** 上一帧时间戳 */
  let lastTimestamp = 0

  /**
   * 自动滚动暂停截止时间
   * 常用于滚轮、scrollToRow 后短时间暂停自动滚动
   */
  let pauseUntil = 0

  /** 尺寸监听器 */
  let resizeObserver = null

  /**
   * 手动控制目标位移
   * 不为 null 时，表示当前需要“朝某个目标位置推进”
   */
  let manualTargetOffset = null

  /**
   * 滚轮活跃截止时间
   * 在这个时间之前不触发吸附，避免用户连续滚轮时反复吸附
   */
  let wheelActiveUntil = 0

  /**
   * 吸附目标
   * continuous 模式下，滚轮结束后自动吸附到最近整行
   */
  let snapTargetOffset = null

  /** 规范化 mode，只允许 continuous / step 两种 */
  const normalizedMode = computed(() => (props.mode === 'step' ? 'step' : 'continuous'))

  /** 行高兜底，避免出现 0 或非法值 */
  const rowHeightNum = computed(() => Math.max(1, Number(props.rowHeight) || 40))

  /** 可视区总高度 */
  const viewportHeight = computed(() => rowHeightNum.value * props.visibleCount)

  /** 整个数据列表逻辑总高度 */
  const listHeight = computed(() => props.data.length * rowHeightNum.value)

  /** 是否需要滚动：只有数据量超过 visibleCount 才滚 */
  const shouldScroll = computed(() => props.data.length > props.visibleCount)

  /** buffer 兜底 */
  const safeBuffer = computed(() => Math.max(1, Number(props.buffer) || 1))

  /**
   * 实际渲染行数
   * 不滚动时渲染所有数据
   * 滚动时只渲染可视区前后的一小段
   */
  const renderCount = computed(() => {
    if (!props.data.length) return 0
    if (!shouldScroll.value) return props.data.length
    return props.visibleCount + safeBuffer.value * 2 + 2
  })

  /**
   * grid 列模板
   * 支持数字宽度（自动补 px）和字符串宽度（如 1fr / 120px / 20%）
   */
  const gridTemplateColumns = computed(() => {
    if (!props.columns.length) return '1fr'

    return props.columns
      .map(col => {
        if (col.width !== undefined && col.width !== null && col.width !== '') {
          return typeof col.width === 'number' ? `${col.width}px` : col.width
        }
        return '1fr'
      })
      .join(' ')
  })

  /** header / row 公共 grid 布局 */
  const gridStyle = computed(() => ({
    gridTemplateColumns: gridTemplateColumns.value,
  }))

  /** 注入行高 CSS 变量 */
  const rootStyle = computed(() => ({
    '--sst-row-height': `${rowHeightNum.value}px`,
  }))

  /** 表体高度 */
  const bodyStyle = computed(() => ({
    height: `${viewportHeight.value}px`,
  }))

  /**
   * 轨道整体位移
   *
   * currentOffset 可以拆成两部分：
   * 1. 整数行部分：决定当前应该从哪一行开始渲染
   * 2. 小数位移部分：决定轨道再向上平移多少 px
   *
   * 这里只保留“当前行内的小数位移”，整数行部分交给 virtualRows 处理
   */
  const bodyTranslateY = computed(() => {
    if (!shouldScroll.value) return 0
    const fractional = currentOffset.value % rowHeightNum.value
    return -(safeBuffer.value * rowHeightNum.value + fractional)
  })

  /** 轨道样式 */
  const trackStyle = computed(() => ({
    height: `${renderCount.value * rowHeightNum.value}px`,
    transform: `translate3d(0, ${bodyTranslateY.value}px, 0)`,
  }))

  /**
   * 单元格对齐方式
   * flex + textAlign 双保险，保证文本和插槽内容都对齐
   */
  function getCellStyle(col) {
    return {
      justifyContent:
        col.align === 'right' ? 'flex-end' : col.align === 'center' ? 'center' : 'flex-start',
      textAlign: col.align || 'left',
    }
  }

  /** 每一行在虚拟轨道中的绝对位置 */
  function getRowStyle(top) {
    return {
      transform: `translate3d(0, ${top}px, 0)`,
    }
  }

  /** 计算行 key */
  function getRowKey(row, index) {
    if (props.rowKey && row?.[props.rowKey] != null) return row[props.rowKey]
    return row?.id ?? index
  }

  /** 生成斑马纹 class */
  function getStripeClass(index) {
    if (!props.stripe) return ''
    return index % 2 === 0 ? 'is-striped is-odd' : 'is-striped is-even'
  }

  /** 安全取模，兼容负数 */
  function mod(n, m) {
    if (m <= 0) return 0
    return ((n % m) + m) % m
  }

  /**
   * 规范化 offset
   * 让 offset 永远落在 [0, listHeight) 内
   * 这是“无缝循环”的关键之一
   */
  function normalizeOffset(value) {
    const total = listHeight.value
    if (total <= 0) return 0
    return mod(value, total)
  }

  /** 找到离当前位置最近的“整行起点” */
  function getNearestRowOffset(value) {
    const rh = rowHeightNum.value
    return Math.round(value / rh) * rh
  }

  /** 根据数据索引计算该行起始 offset */
  function getRowStartOffsetByIndex(index) {
    const len = props.data.length
    if (!len) return 0
    const normalizedIndex = mod(index, len)
    return normalizedIndex * rowHeightNum.value
  }

  /**
   * 虚拟列表核心：
   * 根据 currentOffset 计算“当前应渲染哪些行”
   *
   * 思路：
   * 1. 先算出当前滚动到哪一行 startIndex
   * 2. 前后再多渲染 buffer 行
   * 3. 用 mod 把逻辑索引映射回真实 dataIndex
   * 4. top 只在当前虚拟窗口内部递增
   */
  const virtualRows = computed(() => {
    const rows = props.data
    const len = rows.length
    if (!len) return []

    // 数据不足一屏时，不走虚拟滚动，直接全量渲染
    if (!shouldScroll.value) {
      return rows.map((row, index) => ({
        row,
        dataIndex: index,
        top: index * rowHeightNum.value,
        renderKey: `${getRowKey(row, index)}-${index}`,
        rowClass: getStripeClass(index),
      }))
    }

    const rh = rowHeightNum.value
    const startIndex = Math.floor(currentOffset.value / rh) - safeBuffer.value
    const result = []

    for (let i = 0; i < renderCount.value; i++) {
      const logicalIndex = startIndex + i
      const dataIndex = mod(logicalIndex, len)
      const row = rows[dataIndex]

      result.push({
        row,
        dataIndex,
        top: i * rh,
        renderKey: `${getRowKey(row, dataIndex)}-${logicalIndex}-${i}`,
        rowClass: getStripeClass(dataIndex),
      })
    }

    return result
  })

  /** 停止 raf 动画 */
  function stopAnimation() {
    if (rafId) {
      cancelAnimationFrame(rafId)
      rafId = 0
    }
  }

  /** 重置所有运行时状态 */
  function resetRuntime() {
    lastTimestamp = 0
    pauseUntil = 0
    manualTargetOffset = null
    wheelActiveUntil = 0
    snapTargetOffset = null
    currentOffset.value = 0
  }

  /**
   * 朝手动目标位移推进
   *
   * 用在：
   * - scrollToRow
   * - scrollNext / scrollPrev
   * - 滚轮触发后的惯性追赶
   *
   * 返回值：
   * true  -> 当前帧已经处理了手动控制
   * false -> 当前没有手动控制
   */
  function moveTowardTarget(deltaMs) {
    if (manualTargetOffset == null) return false

    const total = listHeight.value
    if (total <= 0) {
      manualTargetOffset = null
      return false
    }

    const current = currentOffset.value
    const target = normalizeOffset(manualTargetOffset)

    // 计算向前/向后两种路径，选择更短的那条
    let forward = target - current
    if (forward < 0) forward += total

    let backward = current - target
    if (backward < 0) backward += total

    let direction = 1
    let distance = forward

    if (backward < forward) {
      direction = -1
      distance = backward
    }

    // 足够接近时直接吸附到目标
    if (distance <= 0.5) {
      currentOffset.value = target
      manualTargetOffset = null
      return false
    }

    const baseSpeed = Math.max(120, props.speed * props.manualSpeedRatio)
    const move = Math.min(distance, (baseSpeed * deltaMs) / 1000)

    currentOffset.value = normalizeOffset(current + direction * move)
    return true
  }

  /**
   * 朝吸附目标推进
   * continuous 模式下，滚轮结束后吸附到整行
   */
  function moveTowardSnapTarget(deltaMs) {
    if (snapTargetOffset == null) return false

    const total = listHeight.value
    if (total <= 0) {
      snapTargetOffset = null
      return false
    }

    const current = currentOffset.value
    const target = normalizeOffset(snapTargetOffset)

    let forward = target - current
    if (forward < 0) forward += total

    let backward = current - target
    if (backward < 0) backward += total

    let direction = 1
    let distance = forward

    if (backward < forward) {
      direction = -1
      distance = backward
    }

    if (distance <= 0.5) {
      currentOffset.value = target
      snapTargetOffset = null
      return false
    }

    const baseSpeed = Math.max(160, props.speed * props.snapSpeedRatio)
    const move = Math.min(distance, (baseSpeed * deltaMs) / 1000)

    currentOffset.value = normalizeOffset(current + direction * move)
    return true
  }

  /**
   * 判断当前是否应该开始“吸附到整行”
   * 仅 continuous 模式需要
   */
  function tryStartSnap(timestamp) {
    if (!shouldScroll.value || !props.data.length) return false
    if (normalizedMode.value !== 'continuous') return false
    if (manualTargetOffset != null) return false

    if (snapTargetOffset != null) {
      return true
    }

    // wheelActiveUntil 之前，认为用户还在持续滚轮，不吸附
    if (!wheelActiveUntil) return false
    if (timestamp < wheelActiveUntil) return false

    snapTargetOffset = normalizeOffset(getNearestRowOffset(currentOffset.value))
    wheelActiveUntil = 0
    return true
  }

  /** continuous 模式自动滚动 */
  function runContinuous(deltaMs) {
    const distance = (Math.max(props.speed, 0) * deltaMs) / 1000
    currentOffset.value = normalizeOffset(currentOffset.value + distance)
  }

  /** step 模式自动滚动 */
  function runStep(timestamp, deltaMs) {
    if (pauseUntil && timestamp < pauseUntil) return

    const rh = rowHeightNum.value
    const current = currentOffset.value
    const currentRow = Math.floor(current / rh)
    const target = (currentRow + 1) * rh
    const remain = target - current

    // 已经足够接近下一行时，直接停到整行
    if (remain <= 0.5) {
      currentOffset.value = normalizeOffset(target)
      pauseUntil = timestamp + Math.max(0, props.pauseDuration)
      return
    }

    const distance = (Math.max(props.speed, 0) * deltaMs) / 1000
    currentOffset.value = normalizeOffset(current + Math.min(distance, remain))

    if (target - currentOffset.value <= 0.5) {
      currentOffset.value = normalizeOffset(target)
      pauseUntil = timestamp + Math.max(0, props.pauseDuration)
    }
  }

  /**
   * 主动画循环
   *
   * 优先级：
   * 1. 手动目标推进
   * 2. 吸附推进
   * 3. 自动滚动
   *
   * hover / pause 只影响“自动滚动”，不会挡住手动滚轮和吸附
   */
  function frameLoop(timestamp) {
    if (!shouldScroll.value || listHeight.value <= 0) return

    if (!lastTimestamp) {
      lastTimestamp = timestamp
    }

    const deltaMs = Math.min(timestamp - lastTimestamp, 34)
    lastTimestamp = timestamp

    const hoverPaused = props.pauseOnHover && isHoverPaused.value
    const paused = isManualPaused.value

    const hasManualControl = moveTowardTarget(deltaMs)
    tryStartSnap(timestamp)
    const hasSnapControl = moveTowardSnapTarget(deltaMs)

    if (!hasManualControl && !hasSnapControl && !hoverPaused && !paused) {
      if (normalizedMode.value === 'step') {
        runStep(timestamp, deltaMs)
      } else {
        if (!(pauseUntil && timestamp < pauseUntil)) {
          runContinuous(deltaMs)
        }
        return
      }
    }

    rafId = requestAnimationFrame(frameLoop)
  }

  /**
   * 启动动画
   * keepOffset = true 时尽量保留当前滚动位置
   */
  async function startAnimation({ keepOffset = false } = {}) {
    const prev = currentOffset.value

    stopAnimation()
    await nextTick()

    if (!keepOffset) {
      resetRuntime()
    } else {
      lastTimestamp = 0
      pauseUntil = 0
      manualTargetOffset = null
      wheelActiveUntil = 0
      snapTargetOffset = null
      currentOffset.value = normalizeOffset(prev)
    }

    if (!shouldScroll.value || listHeight.value <= 0) return
    rafId = requestAnimationFrame(frameLoop)
  }

  /** hover 进入：暂停自动滚动 */
  function handleMouseEnter() {
    if (props.pauseOnHover) {
      isHoverPaused.value = true
    }
  }

  /** hover 离开：恢复自动滚动 */
  function handleMouseLeave() {
    isHoverPaused.value = false
    lastTimestamp = 0
  }

  /**
   * 滚轮处理
   *
   * step 模式：
   * - 按整行移动
   *
   * continuous 模式：
   * - 立即位移
   * - 再加一点 manualTargetOffset 做惯性追赶
   * - 停下后自动吸附到整行
   */
  function handleWheel(event) {
    if (!props.wheelControl) return
    if (!shouldScroll.value || !props.data.length) return

    const now =
      typeof performance !== 'undefined' && performance.now ? performance.now() : Date.now()

    wheelActiveUntil = now + Math.max(80, props.wheelSnapDelay)
    pauseUntil = now + Math.max(0, props.wheelPauseDuration)
    lastTimestamp = 0
    snapTargetOffset = null

    if (normalizedMode.value === 'step') {
      const direction = event.deltaY > 0 ? 1 : -1
      const rows = Math.max(1, Math.floor(Math.abs(props.wheelStepRows || 1)))
      const base = getNearestRowOffset(currentOffset.value)
      const target = normalizeOffset(base + direction * rows * rowHeightNum.value)

      manualTargetOffset = target

      // 给一点立即反馈，让滚轮更“跟手”
      currentOffset.value = normalizeOffset(
        currentOffset.value +
          direction * Math.min(rowHeightNum.value * rows, rowHeightNum.value * 0.35)
      )
      return
    }

    const delta = event.deltaY * (Number(props.wheelStep) || 0.9)
    currentOffset.value = normalizeOffset(currentOffset.value + delta)
    manualTargetOffset = normalizeOffset(currentOffset.value + delta * 0.35)
  }

  /** ========= expose methods ========= */

  /** 临时暂停自动滚动，常用于外部主动滚动场景 */
  function pauseAutoScrollTemporarily() {
    const now =
      typeof performance !== 'undefined' && performance.now ? performance.now() : Date.now()

    pauseUntil = now + Math.max(0, props.wheelPauseDuration)
    wheelActiveUntil = 0
    snapTargetOffset = null
    lastTimestamp = 0
  }

  /**
   * 滚动到指定行
   * options:
   * - immediate: 是否立即跳转
   * - align: start | center | end
   */
  function scrollToRow(index, options = {}) {
    const len = props.data.length
    if (!len) return

    const { immediate = false, align = 'start' } = options
    let target = getRowStartOffsetByIndex(index)

    if (align === 'center') {
      const centerOffset = ((props.visibleCount - 1) / 2) * rowHeightNum.value
      target = normalizeOffset(target - centerOffset)
    } else if (align === 'end') {
      const endOffset = (props.visibleCount - 1) * rowHeightNum.value
      target = normalizeOffset(target - endOffset)
    } else {
      target = normalizeOffset(target)
    }

    pauseAutoScrollTemporarily()

    if (immediate || !shouldScroll.value) {
      currentOffset.value = target
      manualTargetOffset = null
      snapTargetOffset = null
      return
    }

    manualTargetOffset = target
  }

  /** 向后滚动指定行数 */
  function scrollNext(step = 1, options = {}) {
    const len = props.data.length
    if (!len) return

    const count = Math.max(1, Number(step) || 1)
    const currentIndex = Math.round(currentOffset.value / rowHeightNum.value)
    scrollToRow(currentIndex + count, options)
  }

  /** 向前滚动指定行数 */
  function scrollPrev(step = 1, options = {}) {
    const len = props.data.length
    if (!len) return

    const count = Math.max(1, Number(step) || 1)
    const currentIndex = Math.round(currentOffset.value / rowHeightNum.value)
    scrollToRow(currentIndex - count, options)
  }

  /** 获取当前最接近的行索引 */
  function getCurrentRowIndex() {
    const len = props.data.length
    if (!len) return -1
    return mod(Math.round(currentOffset.value / rowHeightNum.value), len)
  }

  /** 恢复自动播放 */
  function play() {
    isManualPaused.value = false
    lastTimestamp = 0
  }

  /** 暂停自动播放 */
  function pause() {
    isManualPaused.value = true
  }

  /** 对外暴露方法 */
  defineExpose({
    scrollToRow,
    scrollNext,
    scrollPrev,
    getCurrentRowIndex,
    play,
    pause,
  })

  /**
   * 结构型参数变化时，尽量保留当前位置重启动画
   * 这里只监听长度和关键参数，避免 deep watch 过重
   */
  watch(
    () => [
      props.data.length,
      props.columns.length,
      props.rowHeight,
      props.visibleCount,
      props.mode,
      props.speed,
      props.pauseDuration,
      props.buffer,
      props.wheelStep,
      props.wheelStepRows,
      props.manualSpeedRatio,
      props.wheelSnapDelay,
      props.snapSpeedRatio,
    ],
    () => {
      startAnimation({ keepOffset: true })
    }
  )

  /**
   * data 引用整体变化时，重启动画
   * 比 deep watch 更轻
   */
  watch(
    () => props.data,
    () => {
      startAnimation({ keepOffset: true })
    }
  )

  onMounted(() => {
    startAnimation()

    // 容器尺寸变化时重新计算虚拟列表和位移
    if (window.ResizeObserver && bodyRef.value) {
      resizeObserver = new ResizeObserver(() => {
        startAnimation({ keepOffset: true })
      })
      resizeObserver.observe(bodyRef.value)
    }
  })

  onBeforeUnmount(() => {
    stopAnimation()
    if (resizeObserver) {
      resizeObserver.disconnect()
      resizeObserver = null
    }
  })
</script>

<style scoped>
  .seamless-scroll-table {
    width: 100%;
    color: #d9ecff;
    background: #08111f;
    border: 1px solid rgba(120, 180, 255, 0.18);
    border-radius: 10px;
    overflow: hidden;
  }

  .table-header,
  .tr {
    display: grid;
  }

  .table-header {
    height: 42px;
    background: linear-gradient(180deg, rgba(34, 87, 160, 0.85), rgba(20, 49, 92, 0.95));
    border-bottom: 1px solid rgba(120, 180, 255, 0.16);
  }

  .th,
  .td {
    display: flex;
    align-items: center;
    min-width: 0;
    padding: 0 14px;
    overflow: hidden;
    white-space: nowrap;
    text-overflow: ellipsis;
    box-sizing: border-box;
  }

  .th {
    font-size: 14px;
    font-weight: 600;
    color: #e8f3ff;
  }

  .table-body {
    position: relative;
    overflow: hidden;
  }

  .scroll-viewport {
    position: relative;
    height: 100%;
    overflow: hidden;
  }

  .virtual-track {
    position: relative;
    width: 100%;
    will-change: transform;
    transform: translate3d(0, 0, 0);
    backface-visibility: hidden;
  }

  .tr {
    position: absolute;
    left: 0;
    width: 100%;
    height: var(--sst-row-height);
    font-size: 13px;
    border-bottom: 1px solid rgba(120, 180, 255, 0.08);
    box-sizing: border-box;
  }

  .tr.is-odd {
    background: rgba(20, 45, 84, 0.35);
  }

  .tr.is-even {
    background: rgba(9, 24, 46, 0.65);
  }

  .td {
    color: #b8d9ff;
  }
</style>
