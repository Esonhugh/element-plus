<template>
  <div
    :style="style"
    :class="[
      ns.b(),
      ns.is(isSimple ? 'simple' : parent.props.direction),
      ns.is('flex', isLast && !space && !isCenter),
      ns.is('center', isCenter && !isVertical && !isSimple),
    ]"
  >
    <!-- icon & line -->
    <div :class="[ns.e('head'), ns.is(currentStatus)]">
      <div v-if="!isSimple" :class="ns.e('line')">
        <i :class="ns.e('line-inner')" :style="lineStyle"></i>
      </div>

      <div :class="[ns.e('icon'), ns.is(icon ? 'icon' : 'text')]">
        <slot
          v-if="currentStatus !== 'success' && currentStatus !== 'error'"
          name="icon"
        >
          <el-icon v-if="icon" :class="ns.e('icon-inner')">
            <component :is="icon" />
          </el-icon>
          <div v-if="!icon && !isSimple" :class="ns.e('icon-inner')">
            {{ index + 1 }}
          </div>
        </slot>
        <el-icon v-else :class="[ns.e('icon-inner'), ns.is('status')]">
          <check v-if="currentStatus === 'success'" />
          <close v-else />
        </el-icon>
      </div>
    </div>
    <!-- title & description -->
    <div :class="ns.e('main')">
      <div :class="[ns.e('title'), ns.is(currentStatus)]">
        <slot name="title">{{ title }}</slot>
      </div>
      <div v-if="isSimple" :class="ns.e('arrow')"></div>
      <div v-else :class="[ns.e('description'), ns.is(currentStatus)]">
        <slot name="description">{{ description }}</slot>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import {
  computed,
  defineComponent,
  getCurrentInstance,
  inject,
  onBeforeUnmount,
  onMounted,
  ref,
  reactive,
  watch,
} from 'vue'
import { ElIcon } from '@element-plus/components/icon'
import { Close, Check } from '@element-plus/icons-vue'

import { useNamespace } from '@element-plus/hooks'
import type { Ref, PropType, Component } from 'vue'

export interface IStepsProps {
  space: number | string
  active: number
  direction: string
  alignCenter: boolean
  simple: boolean
  finishStatus: string
  processStatus: string
}

export interface StepItemState {
  uid: number
  currentStatus: string
  setIndex: (val: number) => void
  calcProgress: (status: string) => void
}

export interface IStepsInject {
  props: IStepsProps
  steps: Ref<StepItemState[]>
}

export default defineComponent({
  name: 'ElStep',
  components: {
    ElIcon,
    Close,
    Check,
  },
  props: {
    title: {
      type: String,
      default: '',
    },
    icon: {
      type: [String, Object] as PropType<string | Component>,
      default: '',
    },
    description: {
      type: String,
      default: '',
    },
    status: {
      type: String,
      default: '',
      validator: (val: string): boolean =>
        ['', 'wait', 'process', 'finish', 'error', 'success'].includes(val),
    },
  },
  setup(props) {
    const ns = useNamespace('step')
    const index = ref(-1)
    const lineStyle = ref({})
    const internalStatus = ref('')
    const parent: IStepsInject = inject('ElSteps')
    const currentInstance = getCurrentInstance()

    onMounted(() => {
      watch(
        [
          () => parent.props.active,
          () => parent.props.processStatus,
          () => parent.props.finishStatus,
        ],
        ([active]) => {
          updateStatus(active)
        },
        { immediate: true }
      )
    })

    onBeforeUnmount(() => {
      parent.steps.value = parent.steps.value.filter(
        (instance) => instance.uid !== currentInstance.uid
      )
    })

    const currentStatus = computed(() => {
      return props.status || internalStatus.value
    })
    const prevStatus = computed(() => {
      const prevStep = parent.steps.value[index.value - 1]
      return prevStep ? prevStep.currentStatus : 'wait'
    })
    const isCenter = computed(() => {
      return parent.props.alignCenter
    })
    const isVertical = computed(() => {
      return parent.props.direction === 'vertical'
    })
    const isSimple = computed(() => {
      return parent.props.simple
    })
    const stepsCount = computed(() => {
      return parent.steps.value.length
    })
    const isLast = computed(() => {
      return (
        parent.steps.value[stepsCount.value - 1]?.uid === currentInstance.uid
      )
    })
    const space = computed(() => {
      return isSimple.value ? '' : parent.props.space
    })
    const style = computed(() => {
      const style: Record<string, unknown> = {
        flexBasis:
          typeof space.value === 'number'
            ? `${space.value}px`
            : space.value
            ? space.value
            : `${100 / (stepsCount.value - (isCenter.value ? 0 : 1))}%`,
      }
      if (isVertical.value) return style
      if (isLast.value) {
        style.maxWidth = `${100 / stepsCount.value}%`
      }
      return style
    })

    const setIndex = (val) => {
      index.value = val
    }
    const calcProgress = (status) => {
      let step = 100
      const style: Record<string, unknown> = {}

      style.transitionDelay = `${150 * index.value}ms`
      if (status === parent.props.processStatus) {
        step = 0
      } else if (status === 'wait') {
        step = 0
        style.transitionDelay = `${-150 * index.value}ms`
      }
      style.borderWidth = step && !isSimple.value ? '1px' : 0
      style[
        parent.props.direction === 'vertical' ? 'height' : 'width'
      ] = `${step}%`
      lineStyle.value = style
    }
    const updateStatus = (activeIndex) => {
      if (activeIndex > index.value) {
        internalStatus.value = parent.props.finishStatus
      } else if (activeIndex === index.value && prevStatus.value !== 'error') {
        internalStatus.value = parent.props.processStatus
      } else {
        internalStatus.value = 'wait'
      }
      const prevChild = parent.steps.value[stepsCount.value - 1]
      if (prevChild) prevChild.calcProgress(internalStatus.value)
    }

    const stepItemState = reactive({
      uid: computed(() => currentInstance.uid),
      currentStatus,
      setIndex,
      calcProgress,
    })
    parent.steps.value = [...parent.steps.value, stepItemState]

    return {
      ns,
      index,
      lineStyle,
      currentStatus,
      isCenter,
      isVertical,
      isSimple,
      isLast,
      space,
      style,
      parent,
      setIndex,
      calcProgress,
      updateStatus,
    }
  },
})
</script>
