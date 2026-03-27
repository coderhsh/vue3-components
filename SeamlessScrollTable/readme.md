# seamless-scroll-table-vue3

一个基于 **Vue 3** 的无缝轮播表格组件，支持：

- 虚拟列表渲染
- 连续滚动 / 按行停顿滚动
- 鼠标悬浮暂停
- 鼠标滚轮接管滚动
- 滚轮停止后自动吸附到整行
- 表头插槽 / 单元格插槽
- 外部方法控制滚动

适合用于：

- 大屏数据看板
- 实时监控列表
- 轮播公告表格
- 自动滚动排行榜
- 数据驾驶舱

---

## 特性

- **Vue 3 原生支持**
- **虚拟列表**，大数据量下性能更稳
- **无缝循环滚动**
- **支持 continuous / step 两种模式**
- **滚轮即时生效**
- **滚轮后自动吸附到整行**
- **支持自定义表头和单元格插槽**
- **支持外部 expose 方法调用**
- **无额外重型依赖**

---

## 安装

```bash
npm install seamless-scroll-table-vue3
```

或

```bash
pnpm add seamless-scroll-table-vue3
```

或

```bash
yarn add seamless-scroll-table-vue3
```

---

## 引入方式

### 全局注册

```ts
import { createApp } from 'vue'
import App from './App.vue'
import SeamlessScrollTable from 'seamless-scroll-table-vue3'

const app = createApp(App)

app.component('SeamlessScrollTable', SeamlessScrollTable)
app.mount('#app')
```

### 局部引入

```vue
<script setup lang="ts">
  import SeamlessScrollTable from 'seamless-scroll-table-vue3'
</script>
```

---

## 基础示例

```vue
<template>
  <SeamlessScrollTable
    :columns="columns"
    :data="tableData"
    :row-height="44"
    :visible-count="6"
    mode="continuous"
    :speed="36"
    row-key="id"
  />
</template>

<script setup lang="ts">
  import SeamlessScrollTable from 'seamless-scroll-table-vue3'

  const columns = [
    { title: '名称', key: 'name' },
    { title: '区域', key: 'area', align: 'center' },
    { title: '数值', key: 'value', align: 'right', width: 120 },
  ]

  const tableData = [
    { id: 1, name: '设备A', area: '华东', value: 126 },
    { id: 2, name: '设备B', area: '华南', value: 98 },
    { id: 3, name: '设备C', area: '华北', value: 302 },
    { id: 4, name: '设备D', area: '西南', value: 156 },
    { id: 5, name: '设备E', area: '华中', value: 219 },
    { id: 6, name: '设备F', area: '东北', value: 87 },
    { id: 7, name: '设备G', area: '西北', value: 164 },
    { id: 8, name: '设备H', area: '华东', value: 143 },
  ]
</script>
```

---

## 按行滚动示例

```vue
<template>
  <SeamlessScrollTable
    :columns="columns"
    :data="tableData"
    :row-height="44"
    :visible-count="6"
    mode="step"
    :speed="80"
    :pause-duration="1000"
    row-key="id"
  />
</template>
```

---

## 滚轮控制示例

```vue
<template>
  <SeamlessScrollTable
    :columns="columns"
    :data="tableData"
    mode="continuous"
    :wheel-control="true"
    :wheel-step="1"
    :wheel-pause-duration="1500"
    :manual-speed-ratio="12"
    :snap-speed-ratio="16"
    row-key="id"
  />
</template>
```

---

## 表头插槽示例

组件支持两种表头插槽：

- 通用表头插槽：`header`
- 指定列表头插槽：`header-${col.key}`

### 指定列自定义表头

```vue
<template>
  <SeamlessScrollTable :columns="columns" :data="tableData" row-key="id">
    <template #header-name="{ column, index }">
      <span style="color: #ffd66b"> {{ index + 1 }}. {{ column.title }} </span>
    </template>
  </SeamlessScrollTable>
</template>
```

### 通用表头

```vue
<template>
  <SeamlessScrollTable :columns="columns" :data="tableData" row-key="id">
    <template #header="{ column }">
      <span>{{ column.title }}</span>
    </template>
  </SeamlessScrollTable>
</template>
```

### 表头插槽优先级

```txt
header-${col.key} > header > 默认标题 col.title
```

---

## 单元格插槽示例

组件支持两种单元格插槽：

- 通用单元格插槽：`row`
- 指定列单元格插槽：`row-${col.key}`

### 指定列自定义内容

```vue
<template>
  <SeamlessScrollTable :columns="columns" :data="tableData" row-key="id">
    <template #row-value="{ value }">
      <span :style="{ color: value > 200 ? '#ff7b7b' : '#7bffb2' }">
        {{ value }}
      </span>
    </template>
  </SeamlessScrollTable>
</template>
```

### 通用单元格插槽

```vue
<template>
  <SeamlessScrollTable :columns="columns" :data="tableData" row-key="id">
    <template #row="{ value }">
      <span>{{ value ?? '--' }}</span>
    </template>
  </SeamlessScrollTable>
</template>
```

### 单元格插槽优先级

```txt
row-${col.key} > row > 默认值渲染
```

---

## 外部方法控制示例

组件通过 `ref` 暴露了以下方法：

- `scrollToRow(index, options?)`
- `scrollNext(step?, options?)`
- `scrollPrev(step?, options?)`
- `getCurrentRowIndex()`
- `play()`
- `pause()`

### 示例

```vue
<template>
  <div>
    <div style="margin-bottom: 12px">
      <button @click="handlePrev">上一条</button>
      <button @click="handleNext">下一条</button>
      <button @click="handleTo10">跳到第 10 条</button>
      <button @click="handlePause">暂停</button>
      <button @click="handlePlay">播放</button>
      <button @click="handleLog">获取当前行</button>
    </div>

    <SeamlessScrollTable
      ref="tableRef"
      :columns="columns"
      :data="tableData"
      :row-height="44"
      :visible-count="6"
      mode="continuous"
      row-key="id"
    />
  </div>
</template>

<script setup lang="ts">
  import { ref } from 'vue'
  import SeamlessScrollTable from 'seamless-scroll-table-vue3'

  const tableRef = ref()

  const columns = [
    { title: '名称', key: 'name' },
    { title: '区域', key: 'area', align: 'center' },
    { title: '数值', key: 'value', align: 'right', width: 120 },
  ]

  const tableData = Array.from({ length: 30 }).map((_, i) => ({
    id: i + 1,
    name: `设备${i + 1}`,
    area: ['华东', '华南', '华北'][i % 3],
    value: 100 + i,
  }))

  function handlePrev() {
    tableRef.value?.scrollPrev()
  }

  function handleNext() {
    tableRef.value?.scrollNext()
  }

  function handleTo10() {
    tableRef.value?.scrollToRow(9, { align: 'start' })
  }

  function handlePause() {
    tableRef.value?.pause()
  }

  function handlePlay() {
    tableRef.value?.play()
  }

  function handleLog() {
    console.log(tableRef.value?.getCurrentRowIndex())
  }
</script>
```

---

## Props

| 参数               | 说明                                  | 类型      | 默认值         |
| ------------------ | ------------------------------------- | --------- | -------------- |
| columns            | 列配置                                | `Array`   | `[]`           |
| data               | 表格数据                              | `Array`   | `[]`           |
| rowHeight          | 行高                                  | `number`  | `40`           |
| visibleCount       | 可视区显示行数                        | `number`  | `6`            |
| mode               | 滚动模式，可选 `continuous` / `step`  | `string`  | `'continuous'` |
| speed              | 自动滚动速度，单位 px/s               | `number`  | `30`           |
| pauseDuration      | `step` 模式下每行停顿时长，单位 ms    | `number`  | `1200`         |
| pauseOnHover       | 鼠标悬浮时是否暂停自动滚动            | `boolean` | `true`         |
| stripe             | 是否显示斑马纹                        | `boolean` | `true`         |
| rowKey             | 行唯一标识字段                        | `string`  | `''`           |
| buffer             | 虚拟列表缓冲区行数                    | `number`  | `3`            |
| wheelControl       | 是否启用鼠标滚轮控制                  | `boolean` | `true`         |
| wheelPauseDuration | 滚轮操作后自动滚动暂停多久，单位 ms   | `number`  | `1800`         |
| wheelStep          | `continuous` 模式下滚轮灵敏度         | `number`  | `0.9`          |
| wheelStepRows      | `step` 模式下滚轮每次移动几行         | `number`  | `1`            |
| manualSpeedRatio   | 手动目标推进速度倍率                  | `number`  | `10`           |
| wheelSnapDelay     | 滚轮停止多久后开始吸附到整行，单位 ms | `number`  | `120`          |
| snapSpeedRatio     | 吸附到整行时的速度倍率                | `number`  | `12`           |

---

## columns 配置项

`columns` 为数组，每一项支持以下字段：

| 字段  | 说明                                       | 类型               | 是否必填 |
| ----- | ------------------------------------------ | ------------------ | -------- |
| title | 列标题                                     | `string`           | 是       |
| key   | 数据字段名                                 | `string`           | 是       |
| width | 列宽，支持数字和字符串                     | `number \| string` | 否       |
| align | 对齐方式，可选 `left` / `center` / `right` | `string`           | 否       |

### 示例

```ts
const columns = [
  { title: '名称', key: 'name' },
  { title: '区域', key: 'area', align: 'center' },
  { title: '数值', key: 'value', align: 'right', width: 120 },
  { title: '占比', key: 'rate', width: '120px', align: 'right' },
  { title: '说明', key: 'desc', width: '2fr' },
]
```

---

## Slots

### 表头插槽

#### `header`

通用表头插槽。

**插槽参数：**

| 参数   | 说明       |
| ------ | ---------- |
| column | 当前列配置 |
| index  | 当前列索引 |

#### `header-${col.key}`

指定列表头插槽。

例如列 `key = name`，插槽名就是：

```vue
#header-name
```

---

### 行内容插槽

#### `row`

通用单元格插槽。

**插槽参数：**

| 参数      | 说明           |
| --------- | -------------- |
| row       | 当前行完整数据 |
| value     | 当前单元格值   |
| column    | 当前列配置     |
| row-index | 当前数据索引   |
| col-index | 当前列索引     |

#### `row-${col.key}`

指定列单元格插槽。

例如列 `key = value`，插槽名就是：

```vue
#row-value
```

---

## Expose Methods

### `scrollToRow(index, options?)`

滚动到指定行。

#### 参数

| 参数              | 说明                                      | 类型      | 默认值    |
| ----------------- | ----------------------------------------- | --------- | --------- |
| index             | 目标行索引，从 `0` 开始                   | `number`  | -         |
| options.immediate | 是否立即跳转                              | `boolean` | `false`   |
| options.align     | 对齐方式，可选 `start` / `center` / `end` | `string`  | `'start'` |

#### 示例

```ts
tableRef.value?.scrollToRow(9)
tableRef.value?.scrollToRow(19, { align: 'center' })
tableRef.value?.scrollToRow(0, { immediate: true })
```

---

### `scrollNext(step?, options?)`

向后滚动若干行。

#### 参数

| 参数    | 说明             | 类型     | 默认值 |
| ------- | ---------------- | -------- | ------ |
| step    | 向后滚动行数     | `number` | `1`    |
| options | 同 `scrollToRow` | `object` | `{}`   |

#### 示例

```ts
tableRef.value?.scrollNext()
tableRef.value?.scrollNext(3)
```

---

### `scrollPrev(step?, options?)`

向前滚动若干行。

#### 参数

| 参数    | 说明             | 类型     | 默认值 |
| ------- | ---------------- | -------- | ------ |
| step    | 向前滚动行数     | `number` | `1`    |
| options | 同 `scrollToRow` | `object` | `{}`   |

#### 示例

```ts
tableRef.value?.scrollPrev()
tableRef.value?.scrollPrev(2)
```

---

### `getCurrentRowIndex()`

获取当前最接近的行索引。

#### 返回值

```ts
number
```

当数据为空时返回：

```ts
;-1
```

#### 示例

```ts
const index = tableRef.value?.getCurrentRowIndex()
console.log(index)
```

---

### `play()`

恢复自动滚动。

```ts
tableRef.value?.play()
```

---

### `pause()`

暂停自动滚动。

```ts
tableRef.value?.pause()
```

---

## 推荐配置

### 1. 看板大屏常用配置

```vue
<SeamlessScrollTable
  :columns="columns"
  :data="tableData"
  :row-height="44"
  :visible-count="6"
  mode="continuous"
  :speed="36"
  :buffer="4"
  row-key="id"
/>
```

### 2. 公告类、逐条阅读型配置

```vue
<SeamlessScrollTable
  :columns="columns"
  :data="tableData"
  :row-height="44"
  :visible-count="6"
  mode="step"
  :speed="80"
  :pause-duration="1200"
  row-key="id"
/>
```

### 3. 滚轮交互更强的配置

```vue
<SeamlessScrollTable
  :columns="columns"
  :data="tableData"
  mode="continuous"
  :wheel-control="true"
  :wheel-step="1.2"
  :manual-speed-ratio="14"
  :snap-speed-ratio="16"
  :wheel-pause-duration="1500"
  row-key="id"
/>
```

---

## 工作原理简述

### 1. 虚拟列表

组件不会一次性渲染全部行，而是只渲染：

```txt
visibleCount + buffer * 2 + 2
```

附近的一小段数据，所以在大量数据场景下性能更稳。

### 2. 无缝循环

内部会把滚动距离规范到 `[0, listHeight)` 范围内，利用取模逻辑实现循环滚动，因此不会出现真实滚动条到底后中断的问题。

### 3. 连续滚动与按行滚动

- `continuous`：持续平滑滚动
- `step`：逐行滚动，每行可停顿一段时间

### 4. 鼠标滚轮控制

滚轮时组件会接管默认滚动行为：

- 在 `continuous` 模式下立即位移，并追加短惯性
- 在 `step` 模式下按整行移动
- 停止滚轮后，连续模式会自动吸附到最近整行

---

## 注意事项

### 1. `rowHeight` 要与真实样式一致

组件的虚拟列表和滚动逻辑都基于 `rowHeight` 计算。

如果你设置：

```vue
:row-height="44"
```

那实际行高最好也就是 `44px`。
如果样式高度和传入值差异太大，会出现定位不准、吸附偏移等问题。

---

### 2. 建议传 `rowKey`

为了保证渲染稳定性，建议给数据提供唯一键，并传入：

```vue
row-key="id"
```

---

### 3. 数据量不足一屏时不会滚动

当：

```txt
data.length <= visibleCount
```

组件会自动退化成普通静态表格，不会启动滚动动画。

---

### 4. slot 内尽量避免超重组件

虽然支持自定义单元格插槽，但如果你在每个单元格里放非常重的组件，还是会影响性能。

更推荐放：

- 文本
- 图标
- 小型状态标签

---

## 常见问题

### Q1：为什么滚轮没有效果？

请确认：

1. `wheelControl` 是否为 `true`
2. 数据是否超过 `visibleCount`
3. 模板事件是否为：

```vue
@wheel.prevent="handleWheel"
```

而不是：

```vue
@wheel.passive="handleWheel"
```

---

### Q2：为什么滚动吸附不准？

通常是因为 `rowHeight` 和实际行高不一致。

请确保：

- `rowHeight` 配置值正确
- `.tr` 的真实高度与传入值保持一致

---

### Q3：为什么数据更新后重新计算了位置？

组件会在关键参数变化时重新启动动画，但会尽量保留当前位置。

如果你希望更新数据后手动跳到某一行，可以在数据更新后调用：

```ts
tableRef.value?.scrollToRow(0, { immediate: true })
```

---

### Q4：如何自定义主题？

组件默认提供了一套深色看板风格样式，你可以通过覆盖类名来自定义：

- `.seamless-scroll-table`
- `.table-header`
- `.th`
- `.tr`
- `.td`

如果你希望组件库支持更完整的主题能力，建议在发包时额外提供：

- CSS 变量主题
- 暗色 / 亮色主题
- 自定义 className 或 theme prop

---

## TypeScript 提示

如果你在组件库里导出了类型，推荐额外提供这两个类型：

```ts
export interface SeamlessScrollTableColumn {
  title: string
  key: string
  width?: number | string
  align?: 'left' | 'center' | 'right'
}

export interface ScrollToRowOptions {
  immediate?: boolean
  align?: 'start' | 'center' | 'end'
}
```

---

## 版本兼容

| 依赖 | 版本建议        |
| ---- | --------------- |
| Vue  | `^3.3.0` 或更高 |

---

## License

MIT

---

## 更新日志建议

发包后建议维护一个 `CHANGELOG.md`，例如：

```md
# Changelog

## 1.0.0

- 初始版本
- 支持虚拟列表
- 支持 continuous / step 模式
- 支持滚轮控制
- 支持自动吸附到整行
- 支持 expose 方法
- 支持表头 / 单元格插槽
```

---

## 仓库示例信息

你可以在 README 顶部再补这些内容：

```md
- 在线示例：xxx
- 文档站点：xxx
- NPM 地址：xxx
- 更新日志：xxx
```

如果有演示图，也可以加：

```md
![demo](./docs/demo.gif)
```
