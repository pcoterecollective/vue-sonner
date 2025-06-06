<template>
  <li
    ref="toastRef"
    :aria-live="toast.important ? 'assertive' : 'polite'"
    aria-atomic="true"
    role="status"
    tabindex="0"
    data-sonner-toast="true"
    :class="
      cn(
        props.class,
        toastClass,
        classes?.toast,
        toast.classes?.toast,
        // @ts-ignore
        classes?.[toastType],
        // @ts-ignore
        toast?.classes?.[toastType]
      )
    "
    :data-rich-colors="toast.richColors ?? defaultRichColors"
    :data-styled="!Boolean(toast.component || toast?.unstyled || unstyled)"
    :data-mounted="mounted"
    :data-promise="Boolean(toast.promise)"
    :data-removed="removed"
    :data-visible="isVisible"
    :data-y-position="y"
    :data-x-position="x"
    :data-index="index"
    :data-front="isFront"
    :data-swiping="swiping"
    :data-dismissible="dismissible"
    :data-type="toastType"
    :data-invert="invert"
    :data-swipe-out="swipeOut"
    :data-expanded="Boolean(expanded || (expandByDefault && mounted))"
    :style="{
      '--index': index,
      '--toasts-before': index,
      '--z-index': toasts.length - index,
      '--offset': `${removed ? offsetBeforeRemove : offset}px`,
      '--initial-height': expandByDefault ? 'auto' : `${initialHeight}px`,
      ...style,
      ...toastStyle
    }"
    @pointerdown="onPointerDown"
    @pointerup="onPointerUp"
    @pointermove="onPointerMove"
  >
    <template v-if="closeButton && !toast.component">
      <button
        :aria-label="closeButtonAriaLabel || 'Close toast'"
        :data-disabled="disabled"
        data-close-button="true"
        :class="cn(classes?.closeButton, toast?.classes?.closeButton)"
        @click="handleCloseToast"
      >
        <template v-if="icons?.close">
          <component :is="icons?.close" />
        </template>
        <template v-else>
          <slot name="close-icon" />
        </template>
      </button>
    </template>

    <template v-if="toast.component">
      <component
        :is="toast.component"
        v-bind="toast.componentProps"
        :onCloseToast="handleCloseToast"
      />
    </template>

    <template v-else>
      <template v-if="toastType !== 'default' || toast.icon || toast.promise">
        <div data-icon="" :class="cn(classes?.icon, toast?.classes?.icon)">
          <component :is="toast.icon" v-if="toast.icon" />
          <Transition v-else name="fade" mode="out-in">
            <slot v-if="toastType === 'loading'" name="loading-icon" />
            <slot v-else-if="toastType === 'success'" name="success-icon" />
            <slot v-else-if="toastType === 'error'" name="error-icon" />
            <slot v-else-if="toastType === 'warning'" name="warning-icon" />
            <slot v-else-if="toastType === 'info'" name="info-icon" />
          </Transition>
        </div>
      </template>

      <div
        data-content=""
        :class="cn(classes?.content, toast?.classes?.content)"
      >
        <div data-title="" :class="cn(classes?.title, toast.classes?.title)">
          <template v-if="isStringOfTitle">
            <component :is="toast.title" v-bind="toast.componentProps" />
          </template>
          <template v-else>
            {{ toast.title }}
          </template>
        </div>

        <template v-if="toast.description">
          <div
            data-description=""
            :class="
              cn(
                descriptionClass,
                toastDescriptionClass,
                classes?.description,
                toast.classes?.description
              )
            "
          >
            <template v-if="isStringOfDescription">
              <component
                :is="toast.description"
                v-bind="toast.componentProps"
              />
            </template>
            <template v-else>
              {{ toast.description }}
            </template>
          </div>
        </template>
      </div>
      <template v-if="toast.cancel">
        <button
          :style="toast.cancelButtonStyle || cancelButtonStyle"
          :class="cn(classes?.cancelButton, toast.classes?.cancelButton)"
          data-button
          data-cancel
          @click="
            (event) => {
              if (!isAction(toast.cancel!)) return;
              if (!dismissible) return;
              toast.cancel.onClick?.(event);
              deleteToast();
            }
          "
        >
          {{ isAction(toast.cancel) ? toast.cancel?.label : toast.cancel }}
        </button>
      </template>
      <template v-if="toast.action">
        <button
          :style="toast.actionButtonStyle || actionButtonStyle"
          :class="cn(classes?.actionButton, toast.classes?.actionButton)"
          data-button
          data-action
          @click="
            (event) => {
              if (!isAction(toast.action!)) return;
              if (event.defaultPrevented) return;
              toast.action.onClick?.(event);
              if (event.defaultPrevented) return;
              deleteToast();
            }
          "
        >
          {{ isAction(toast.action) ? toast.action?.label : toast.action }}
        </button>
      </template>
    </template>
  </li>
</template>

<script lang="ts" setup>
import './styles.css'

import {
  computed,
  onBeforeUnmount,
  onMounted,
  ref,
  watch,
  watchEffect
} from 'vue'
import { type HeightT, type ToastProps, type ToastT, isAction } from './types'
import { useIsDocumentHidden } from './hooks'

const props = defineProps<ToastProps>()

const emit = defineEmits<{
  (e: 'update:heights', heights: HeightT[]): void
  (e: 'removeToast', toast: ToastT): void
}>()

// Default lifetime of a toasts (in ms)
const TOAST_LIFETIME = 4000

const SWIPE_THRESHOLD = 20

const TIME_BEFORE_UNMOUNT = 200

const mounted = ref(false)
const removed = ref(false)
const swiping = ref(false)
const swipeOut = ref(false)
const swiped = ref(false)
const offsetBeforeRemove = ref(0)
const initialHeight = ref(0)
const remainingTime = ref(
  props.toast.duration || props.duration || TOAST_LIFETIME
)
const dragStartTime = ref<Date | null>(null)
const toastRef = ref<HTMLLIElement | null>(null)
const isFront = computed(() => props.index === 0)
const isVisible = computed(() => props.index + 1 <= props.visibleToasts)
const toastType = computed(() => props.toast.type)
const dismissible = computed(() => props.toast.dismissible !== false)
const toastClass = computed(() => props.toast.class || '')
const toastDescriptionClass = computed(() => props.descriptionClass || '')

const toastStyle = props.toast.style || {}

// Height index is used to calculate the offset as it gets updated before the toast array, which means we can calculate the new layout faster.
const heightIndex = computed(
  () =>
    props.heights.findIndex((height) => height.toastId === props.toast.id) || 0
)
const closeButton = computed(() => props.toast.closeButton ?? props.closeButton)
const duration = computed(
  () => props.toast.duration || props.duration || TOAST_LIFETIME
)

const closeTimerStartTimeRef = ref(0)
const lastCloseTimerStartTimeRef = ref(0)
const pointerStartRef = ref<{ x: number; y: number } | null>(null)
const coords = computed(() => props.position.split('-'))
const y = computed(() => coords.value[0])
const x = computed(() => coords.value[1])
const isStringOfTitle = computed(() => typeof props.toast.title !== 'string')
const isStringOfDescription = computed(
  () => typeof props.toast.description !== 'string'
)

const toastsHeightBefore = computed(() => {
  return props.heights.reduce((prev, curr, reducerIndex) => {
    // Calculate offset up untill current  toast
    if (reducerIndex >= heightIndex.value) {
      return prev
    }

    return prev + curr.height
  }, 0)
})
const isDocumentHidden = useIsDocumentHidden()
const invert = computed(() => props.toast.invert || props.invert)
const disabled = computed(() => toastType.value === 'loading')

const offset = computed(() => {
  return heightIndex.value * props.gap! + toastsHeightBefore.value || 0
})

onMounted(() => {
  if (!mounted.value) return

  const toastNode = toastRef.value
  const originalHeight = toastNode?.style.height
  toastNode!.style.height = 'auto'
  const newHeight = toastNode!.getBoundingClientRect().height
  toastNode!.style.height = originalHeight as string

  initialHeight.value = newHeight

  let newHeightArr

  const alreadyExists = props.heights.find(
    (height) => height.toastId === props.toast.id
  )

  if (!alreadyExists) {
    newHeightArr = [
      {
        toastId: props.toast.id,
        height: newHeight,
        position: props.toast.position
      },
      ...props.heights
    ]
  } else {
    newHeightArr = props.heights.map((height) =>
      height.toastId === props.toast.id
        ? { ...height, height: newHeight }
        : height
    )
  }

  emit('update:heights', newHeightArr as HeightT[])
})

function deleteToast() {
  // Save the offset for the exit swipe animation
  removed.value = true
  offsetBeforeRemove.value = offset.value
  const height = props.heights.filter(
    (height) => height.toastId !== props.toast.id
  )
  emit('update:heights', height)

  setTimeout(() => {
    emit('removeToast', props.toast)
  }, TIME_BEFORE_UNMOUNT)
}

function handleCloseToast() {
  if (disabled.value || !dismissible.value) {
    return {}
  }

  deleteToast()
  props.toast.onDismiss?.(props.toast)
}

function onPointerDown(event: PointerEvent) {
  if (disabled.value || !dismissible.value) return
  dragStartTime.value = new Date()
  offsetBeforeRemove.value = offset.value
  // Ensure we maintain correct pointer capture even when going outside of the toast (e.g. when swiping)
  ;(event.target as HTMLElement).setPointerCapture(event.pointerId)
  if ((event.target as HTMLElement).tagName === 'BUTTON') return
  swiping.value = true
  pointerStartRef.value = { x: event.clientX, y: event.clientY }
}

function onPointerUp() {
  if (swipeOut.value || !dismissible) return
  pointerStartRef.value = null

  const swipeAmount = Number(
    toastRef.value?.style
      .getPropertyValue('--swipe-amount')
      .replace('px', '') || 0
  )

  const timeTaken = new Date().getTime() - dragStartTime.value?.getTime()!
  const velocity = Math.abs(swipeAmount) / timeTaken

  // Remove only if treshold is met
  if (Math.abs(swipeAmount) >= SWIPE_THRESHOLD || velocity > 0.11) {
    offsetBeforeRemove.value = offset.value
    props.toast.onDismiss?.(props.toast)
    deleteToast()
    swipeOut.value = true
    swiped.value = false
    return
  }

  toastRef.value?.style.setProperty('--swipe-amount', '0px')
  swiping.value = false
}

function onPointerMove(event: PointerEvent) {
  if (!pointerStartRef.value || !dismissible.value) return

  const yPosition = event.clientY - pointerStartRef.value.y

  // @ts-expect-error
  const isHighlighted = window.getSelection()?.toString().length > 0

  const swipeAmount =
    y.value === 'top' ? Math.min(0, yPosition) : Math.max(0, yPosition)

  if (Math.abs(swipeAmount) > 0) {
    swiped.value = true
  }

  if (isHighlighted) return

  toastRef.value?.style.setProperty('--swipe-amount', `${swipeAmount}px`)
}

watchEffect((onInvalidate) => {
  if (
    (props.toast.promise && toastType.value === 'loading') ||
    props.toast.duration === Infinity ||
    props.toast.type === 'loading'
  ) {
    return
  }
  let timeoutId: ReturnType<typeof setTimeout>

  // Pause the timer on each hover
  const pauseTimer = () => {
    if (lastCloseTimerStartTimeRef.value < closeTimerStartTimeRef.value) {
      // Get the elapsed time since the timer started
      const elapsedTime = new Date().getTime() - closeTimerStartTimeRef.value

      remainingTime.value = remainingTime.value - elapsedTime
    }

    lastCloseTimerStartTimeRef.value = new Date().getTime()
  }

  const startTimer = () => {
    if (remainingTime.value === Infinity) return
    closeTimerStartTimeRef.value = new Date().getTime()

    // Let the toast know it has started
    timeoutId = setTimeout(() => {
      props.toast.onAutoClose?.(props.toast)
      deleteToast()
    }, remainingTime.value)
  }

  if (
    props.expanded ||
    props.interacting ||
    (props.pauseWhenPageIsHidden && isDocumentHidden)
  ) {
    pauseTimer()
  } else {
    startTimer()
  }

  onInvalidate(() => {
    clearTimeout(timeoutId)
  })
})

watch(
  () => props.toast.delete,
  () => {
    if (props.toast.delete) {
      deleteToast()
    }
  },
  {
    deep: true
  }
)

onMounted(() => {
  mounted.value = true

  if (toastRef.value) {
    const height = toastRef.value.getBoundingClientRect().height
    // Add toast height tot heights array after the toast is mounted
    initialHeight.value = height

    const newHeights = [
      { toastId: props.toast.id, height, position: props.toast.position! },
      ...props.heights
    ]
    emit('update:heights', newHeights)
  }
})

onBeforeUnmount(() => {
  if (toastRef.value) {
    const newHeights = props.heights.filter(
      (height) => height.toastId !== props.toast.id
    )
    emit('update:heights', newHeights)
  }
})
</script>
