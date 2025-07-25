<template>
  <div :style="style">
    <slot />
  </div>
</template>

<script setup lang="ts">
import { ref, watch, useSlots, computed, provide, onBeforeMount, onMounted, onUnmounted, onDeactivated, onActivated } from 'vue';
import type { VNode, VNodeArrayChildren } from 'vue';
import type { BreakPoint } from '@/components/Grid/interface';

defineOptions({
  name: 'Grid'
});

type Props = {
  cols?: number | Record<BreakPoint, number>;
  collapsed?: boolean;
  collapsedRows?: number;
  gap?: [number, number] | number;
};

const props = withDefaults(defineProps<Props>(), {
  cols: () => ({ xs: 1, sm: 2, md: 2, lg: 3, xl: 4 }),
  collapsed: false,
  collapsedRows: 1,
  gap: 0
});

onBeforeMount(() => props.collapsed && findIndex());
onMounted(() => {
  resize({ target: { innerWidth: window.innerWidth } } as unknown as Event);
  window.addEventListener('resize', resize);
});
onActivated(() => {
  resize({ target: { innerWidth: window.innerWidth } } as unknown as Event);
  window.addEventListener('resize', resize);
});
onUnmounted(() => {
  window.removeEventListener('resize', resize);
});
onDeactivated(() => {
  window.removeEventListener('resize', resize);
});

// 监听屏幕变化
const resize = (e: Event) => {
  let width = (e.target as Window).innerWidth;
  switch (true) {
    case width < 768:
      breakPoint.value = 'xs';
      break;
    case width >= 768 && width < 992:
      breakPoint.value = 'sm';
      break;
    case width >= 992 && width < 1200:
      breakPoint.value = 'md';
      break;
    case width >= 1200 && width < 1920:
      breakPoint.value = 'lg';
      break;
    case width >= 1920:
      breakPoint.value = 'xl';
      break;
  }
};

// 注入 gap 间距
provide('gap', Array.isArray(props.gap) ? props.gap[0] : props.gap);

// 注入响应式断点
let breakPoint = ref<BreakPoint>('xl');
provide('breakPoint', breakPoint);

// 注入要开始折叠的 index
const hiddenIndex = ref(-1);
provide('shouldHiddenIndex', hiddenIndex);

// 注入 cols
const gridCols = computed(() => {
  if (typeof props.cols === 'object') return props.cols[breakPoint.value] ?? props.cols;
  return props.cols;
});
provide('cols', gridCols);

// 寻找需要开始折叠的字段 index
const slots = (() => {
  const s = useSlots();
  return typeof s.default === 'function' ? s.default() : [];
})();

const findIndex = () => {
  let fields: VNodeArrayChildren = [];
  let suffix: VNode | null = null;
  slots.forEach((slot: any) => {
    // suffix
    if (typeof slot.type === 'object' && slot.type.name === 'GridItem' && slot.props?.suffix !== undefined) suffix = slot;
    // slot children
    if (typeof slot.type === 'symbol' && Array.isArray(slot.children)) fields.push(...slot.children);
  });

  // 计算 suffix 所占用的列
  let suffixCols = 0;
  if (suffix) {
    suffixCols =
      ((suffix as VNode).props?.[breakPoint.value]?.span ?? (suffix as VNode).props?.span ?? 1) +
      ((suffix as VNode).props?.[breakPoint.value]?.offset ?? (suffix as VNode).props?.offset ?? 0);
  }

  let total = 0;
  let found = false;
  const colsNum = Number(gridCols.value) || 1;
  for (let index = 0; index < fields.length; index++) {
    const current = fields[index] as VNode;
    const span = current.props?.[breakPoint.value]?.span ?? current.props?.span ?? 1;
    const offset = current.props?.[breakPoint.value]?.offset ?? current.props?.offset ?? 0;
    total += Number(span) + Number(offset);
    if (total > props.collapsedRows * colsNum - suffixCols) {
      hiddenIndex.value = index;
      found = true;
      break;
    }
  }
  if (!found) hiddenIndex.value = -1;
};

// 断点变化时 执行 findIndex
watch(
  () => breakPoint.value,
  () => {
    if (props.collapsed) findIndex();
  }
);

// 监听 collapsed
watch(
  () => props.collapsed,
  value => {
    if (value) return findIndex();
    hiddenIndex.value = -1;
  }
);

// 设置间距
const gridGap = computed(() => {
  if (typeof props.gap === 'number') return `${props.gap}px`;
  if (Array.isArray(props.gap)) return `${props.gap[1]}px ${props.gap[0]}px`;
  return 'unset';
});

// 设置 style
const style = computed(() => {
  return {
    display: 'grid',
    gridGap: gridGap.value,
    gridTemplateColumns: `repeat(${gridCols.value}, minmax(0, 1fr))`
  };
});

defineExpose({ breakPoint });
</script>
