<script setup lang="ts">
import { computed, ref, nextTick, onMounted, onUnmounted, watch } from 'vue'

interface Props {
  // 当前需要按下的键（字符或 keyCode）
  nextKey?: string | null
  // 当前实际按下的键（用于按键反馈）
  pressedKey?: string | null
  // 是否显示手指指示
  showFingerGuide?: boolean
  // 是否显示键盘
  visible?: boolean
}

const props = withDefaults(defineProps<Props>(), {
  nextKey: null,
  pressedKey: null,
  showFingerGuide: true,
  visible: true,
})

// 监听 nextKey 变化
watch(() => props.nextKey, (newVal) => {
  // console.log('nextKey prop changed:', newVal)
}, { immediate: true })

// DOM 引用
const keyboardContainerRef = ref<HTMLElement>()
const guideLineRef = ref<SVGSVGElement>()
const activeFingerRef = ref<HTMLElement>()
const highlightedKeyRef = ref<HTMLElement>()

// 引导线路径数据
const guideLinePath = ref<string>('')
const guideLineVisible = ref(false)
const guideLineId = ref<string>('')

// 生成唯一的引导线ID（用于SVG动画引用）
const generateGuideLineId = () => {
  return `guideLine-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`
}

// QWERTY 键盘布局
const keyboardLayout = [
  [
    { key: '`', code: 'Backquote', finger: 'left-pinky' },
    { key: '1', code: 'Digit1', finger: 'left-pinky' },
    { key: '2', code: 'Digit2', finger: 'left-ring' },
    { key: '3', code: 'Digit3', finger: 'left-middle' },
    { key: '4', code: 'Digit4', finger: 'left-index' },
    { key: '5', code: 'Digit5', finger: 'left-index' },
    { key: '6', code: 'Digit6', finger: 'right-index' },
    { key: '7', code: 'Digit7', finger: 'right-index' },
    { key: '8', code: 'Digit8', finger: 'right-middle' },
    { key: '9', code: 'Digit9', finger: 'right-ring' },
    { key: '0', code: 'Digit0', finger: 'right-pinky' },
    { key: '-', code: 'Minus', finger: 'right-pinky' },
    { key: '=', code: 'Equal', finger: 'right-pinky' },
    { key: 'Backspace', code: 'Backspace', finger: 'right-pinky', width: 2, label: '←' },
  ],
  [
    { key: 'Tab', code: 'Tab', finger: 'left-pinky', width: 1.5, label: 'Tab' },
    { key: 'q', code: 'KeyQ', finger: 'left-pinky' },
    { key: 'w', code: 'KeyW', finger: 'left-ring' },
    { key: 'e', code: 'KeyE', finger: 'left-middle' },
    { key: 'r', code: 'KeyR', finger: 'left-index' },
    { key: 't', code: 'KeyT', finger: 'left-index' },
    { key: 'y', code: 'KeyY', finger: 'right-index' },
    { key: 'u', code: 'KeyU', finger: 'right-index' },
    { key: 'i', code: 'KeyI', finger: 'right-middle' },
    { key: 'o', code: 'KeyO', finger: 'right-ring' },
    { key: 'p', code: 'KeyP', finger: 'right-pinky' },
    { key: '[', code: 'BracketLeft', finger: 'right-pinky' },
    { key: ']', code: 'BracketRight', finger: 'right-pinky' },
    { key: '\\', code: 'Backslash', finger: 'right-pinky', width: 1.5 },
  ],
  [
    { key: 'Caps', code: 'CapsLock', finger: 'left-pinky', width: 1.75, label: 'Caps' },
    { key: 'a', code: 'KeyA', finger: 'left-pinky' },
    { key: 's', code: 'KeyS', finger: 'left-ring' },
    { key: 'd', code: 'KeyD', finger: 'left-middle' },
    { key: 'f', code: 'KeyF', finger: 'left-index' },
    { key: 'g', code: 'KeyG', finger: 'left-index' },
    { key: 'h', code: 'KeyH', finger: 'right-index' },
    { key: 'j', code: 'KeyJ', finger: 'right-index' },
    { key: 'k', code: 'KeyK', finger: 'right-middle' },
    { key: 'l', code: 'KeyL', finger: 'right-ring' },
    { key: ';', code: 'Semicolon', finger: 'right-pinky' },
    { key: "'", code: 'Quote', finger: 'right-pinky' },
    { key: 'Enter', code: 'Enter', finger: 'right-pinky', width: 2.25, label: 'Enter' },
  ],
  [
    { key: 'Shift', code: 'ShiftLeft', finger: 'left-pinky', width: 2.25, label: 'Shift' },
    { key: 'z', code: 'KeyZ', finger: 'left-pinky' },
    { key: 'x', code: 'KeyX', finger: 'left-ring' },
    { key: 'c', code: 'KeyC', finger: 'left-middle' },
    { key: 'v', code: 'KeyV', finger: 'left-index' },
    { key: 'b', code: 'KeyB', finger: 'left-index' },
    { key: 'n', code: 'KeyN', finger: 'right-index' },
    { key: 'm', code: 'KeyM', finger: 'right-index' },
    { key: ',', code: 'Comma', finger: 'right-middle' },
    { key: '.', code: 'Period', finger: 'right-ring' },
    { key: '/', code: 'Slash', finger: 'right-pinky' },
    { key: 'Shift', code: 'ShiftRight', finger: 'right-pinky', width: 2.75, label: 'Shift' },
  ],
  [
    { key: 'Ctrl', code: 'ControlLeft', finger: 'left-pinky', width: 1.25, label: 'Ctrl' },
    { key: 'Win', code: 'MetaLeft', finger: 'left-pinky', width: 1.25, label: 'Win' },
    { key: 'Alt', code: 'AltLeft', finger: 'left-thumb', width: 1.25, label: 'Alt' },
    { key: 'Space', code: 'Space', finger: 'thumb', width: 6.25, label: 'Space' },
    { key: 'Alt', code: 'AltRight', finger: 'right-thumb', width: 1.25, label: 'Alt' },
    { key: 'Win', code: 'MetaRight', finger: 'right-pinky', width: 1.25, label: 'Win' },
    { key: 'Ctrl', code: 'ControlRight', finger: 'right-pinky', width: 1.25, label: 'Ctrl' },
  ],
]

// 手指颜色映射（增强区分度）
const fingerColors: Record<string, string> = {
  'left-pinky': '#9c27b0',      // 紫色 - 左手小指
  'left-ring': '#673ab7',        // 深紫色 - 左手无名指
  'left-middle': '#3f51b5',      // 靛蓝色 - 左手中指
  'left-index': '#2196f3',       // 蓝色 - 左手食指
  'right-index': '#00bcd4',      // 青色 - 右手食指
  'right-middle': '#009688',     // 青绿色 - 右手中指
  'right-ring': '#4caf50',       // 绿色 - 右手无名指
  'right-pinky': '#8bc34a',      // 浅绿色 - 右手小指
  'left-thumb': '#ff5722',       // 橙红色 - 左手拇指
  'right-thumb': '#ff9800',      // 橙色 - 右手拇指
  'thumb': '#ff9800',            // 橙色 - 通用拇指
}

// 字符到 keyCode 的映射
function getKeyCodeFromChar(char: string): string | null {
  if (!char) return null
  
  const lowerChar = char.toLowerCase()
  
  // 处理空格
  if (char === ' ' || lowerChar === 'space') {
    return 'Space'
  }
  
  // 数字
  if (/[0-9]/.test(char)) {
    return `Digit${char}`
  }
  
  // 字母
  if (/[a-z]/i.test(char)) {
    return `Key${char.toUpperCase()}`
  }
  
  // 特殊字符映射
  const specialCharMap: Record<string, string> = {
    '`': 'Backquote',
    '-': 'Minus',
    '=': 'Equal',
    '[': 'BracketLeft',
    ']': 'BracketRight',
    '\\': 'Backslash',
    ';': 'Semicolon',
    "'": 'Quote',
    ',': 'Comma',
    '.': 'Period',
    '/': 'Slash',
  }
  
  return specialCharMap[lowerChar] || null
}

// 获取下一个按键的信息
const nextKeyInfo = computed(() => {
  if (!props.nextKey) {
    return null
  }

  const keyCode = getKeyCodeFromChar(props.nextKey)
  if (!keyCode) {
    return null
  }
  
  // 查找匹配的按键
  for (const row of keyboardLayout) {
    for (const keyInfo of row) {
      if (keyInfo.code === keyCode) {
        return keyInfo
      }
    }
  }

  return null
})

// 获取当前高亮的按键（下一个需要按下的）
const highlightedKey = computed(() => {
  return nextKeyInfo.value?.code || null
})

// 获取当前按下的按键（实际按键反馈）
const pressedKeyInfo = computed(() => {
  if (!props.pressedKey) return null
  
  // 查找匹配的按键
  for (const row of keyboardLayout) {
    for (const keyInfo of row) {
      if (keyInfo.code === props.pressedKey) {
        return keyInfo
      }
    }
  }
  
  return null
})

// 获取当前需要的手指
const activeFinger = computed(() => {
  return nextKeyInfo.value?.finger || null
})

// 获取按键的样式类
function getKeyClass(keyInfo: any) {
  const isHighlighted = highlightedKey.value === keyInfo.code
  const isPressed = pressedKeyInfo.value?.code === keyInfo.code
  const isActiveFinger = activeFinger.value === keyInfo.finger
  
  return {
    'key': true,
    'key-highlighted': isHighlighted,
    'key-pressed': isPressed,
    [`finger-${keyInfo.finger}`]: isActiveFinger && props.showFingerGuide,
  }
}

// 获取按键的样式
function getKeyStyle(keyInfo: any) {
  const width = keyInfo.width || 1
  const isHighlighted = highlightedKey.value === keyInfo.code
  const isPressed = pressedKeyInfo.value?.code === keyInfo.code
  const isActiveFinger = activeFinger.value === keyInfo.finger && props.showFingerGuide
  
  let backgroundColor = '#f5f5f5'
  let color = '#333'
  let borderColor = '#ddd'
  let transform = 'scale(1)'
  let boxShadow = 'none'
  
  // 优先级：按下 > 高亮 > 手指指示
  if (isPressed) {
    // 按键按下时的反馈（最高优先级）
    backgroundColor = '#ff9800'
    color = '#fff'
    borderColor = '#f57c00'
    transform = 'scale(0.95)'
    boxShadow = '0 2px 8px rgba(255, 152, 0, 0.6)'
  } else if (isHighlighted) {
    // 下一个需要按下的键（高优先级，使用绿色系，与橙色和蓝色区分）
    backgroundColor = '#4caf50'  // 绿色，与橙色和蓝色区分度大
    color = '#fff'
    borderColor = '#388e3c'  // 更深的绿色边框
    boxShadow = '0 4px 12px rgba(76, 175, 80, 0.5)'  // 添加阴影使其更突出
  } else if (isActiveFinger) {
    // 手指指示颜色（低优先级，使用更浅的颜色，避免与高亮混淆）
    // 将颜色转换为 rgba 格式，添加透明度
    const fingerColor = fingerColors[keyInfo.finger]
    // 简单的 hex 转 rgba（假设是 6 位 hex）
    const r = parseInt(fingerColor.slice(1, 3), 16)
    const g = parseInt(fingerColor.slice(3, 5), 16)
    const b = parseInt(fingerColor.slice(5, 7), 16)
    backgroundColor = `rgba(${r}, ${g}, ${b}, 0.25)`  // 25% 透明度，使其更浅
    borderColor = `rgba(${r}, ${g}, ${b}, 0.5)`  // 50% 透明度边框
    color = '#333'  // 保持深色文字，确保可读性
  }
  
  const style: any = {
    flex: width,
    backgroundColor,
    color,
    borderColor,
    transform,
    boxShadow,
    transition: 'all 0.1s ease',
  }

  return style
}

// 获取显示文本
function getKeyLabel(keyInfo: any) {
  return keyInfo.label || keyInfo.key.toUpperCase()
}

// 获取手指样式（用于手指指示器）
function getFingerStyle(finger: string, isActive: boolean) {
  if (!isActive) {
    return {
      backgroundColor: 'transparent',
      borderColor: '#ddd',
      borderWidth: '2px',
      boxShadow: 'none'
    }
  }
  
  const fingerColor = fingerColors[finger]
  // 将 hex 颜色转换为 rgba
  const r = parseInt(fingerColor.slice(1, 3), 16)
  const g = parseInt(fingerColor.slice(3, 5), 16)
  const b = parseInt(fingerColor.slice(5, 7), 16)
  
  return {
    backgroundColor: `rgba(${r}, ${g}, ${b}, 0.8)`,
    borderColor: fingerColor,
    borderWidth: '3px',
    boxShadow: `0 0 12px rgba(${r}, ${g}, ${b}, 0.5)`
  }
}

// 获取手指名称（中文）
function getFingerName(finger: string): string {
  const fingerNames: Record<string, string> = {
    'left-pinky': '左手小指',
    'left-ring': '左手无名指',
    'left-middle': '左手中指',
    'left-index': '左手食指',
    'right-index': '右手食指',
    'right-middle': '右手中指',
    'right-ring': '右手无名指',
    'right-pinky': '右手小指',
    'left-thumb': '左手拇指',
    'right-thumb': '右手拇指',
    'thumb': '拇指',
  }
  return fingerNames[finger] || finger
}

// 获取左手轮廓位置样式（始终显示在左侧红框）
function getLeftHandOutlineStyle() {
  return {
    position: 'absolute',
    bottom: '0px', // 在键盘下方空白区域的底部，不遮挡键盘按键
    left: '8%', // 左侧红框位置
    zIndex: 10,
  }
}

// 获取右手轮廓位置样式（始终显示在右侧红框）
function getRightHandOutlineStyle() {
  return {
    position: 'absolute',
    bottom: '0px', // 在键盘下方空白区域的底部，不遮挡键盘按键
    right: '8%', // 右侧红框位置
    zIndex: 10,
  }
}

// 获取文字提示位置样式（放在中间红框位置，键盘下方中央）
function getLabelBadgeStyle() {
  return {
    position: 'absolute',
    bottom: '5px', // 在键盘下方空白区域的底部，与手部轮廓同一水平线
    left: '50%',
    top: 'auto',
    transform: 'translateX(-50%)', // 水平居中，对应中间红框位置
    zIndex: 11,
  }
}

// 获取手指标签样式（根据活动手指颜色动态调整）
function getFingerLabelStyle() {
  if (!activeFinger.value) {
    return {}
  }
  
  const fingerColor = fingerColors[activeFinger.value] || '#42a5f5'
  
  // 将 hex 颜色转换为 rgba，用于半透明背景
  const r = parseInt(fingerColor.slice(1, 3), 16)
  const g = parseInt(fingerColor.slice(3, 5), 16)
  const b = parseInt(fingerColor.slice(5, 7), 16)
  
  return {
    background: `linear-gradient(135deg, rgba(${r}, ${g}, ${b}, 0.65), rgba(${r}, ${g}, ${b}, 0.55))`,
  }
}

// 获取标签位置（左侧还是右侧）
const labelPosition = computed(() => {
  if (!activeFinger.value) return 'left'
  
  // 左手手指：标签在右侧
  // 右手手指：标签在左侧
  return activeFinger.value.startsWith('left') ? 'right' : 'left'
})

// 计算并更新引导线路径
function updateGuideLine() {
  if (!props.showFingerGuide || !activeFinger.value || !highlightedKey.value) {
    guideLineVisible.value = false
    return
  }

  nextTick(() => {
    try {
      if (!keyboardContainerRef.value) {
        // 延迟重试，等待 DOM 渲染
        setTimeout(updateGuideLine, 100)
        return
      }
      
      // 使用整个虚拟键盘组件作为参考容器
      const containerRect = keyboardContainerRef.value.getBoundingClientRect()
      
      // 查找手指元素的中心点（从手部轮廓中活动手指的圆圈）
      // 由于手部轮廓位置固定在键盘下方，我们需要根据活动手指计算圆圈位置
      // SVG viewBox: 0 0 100 130, 实际大小: 140x182
      const svgWidth = 140
      const svgHeight = 182
      const viewBoxWidth = 100
      const viewBoxHeight = 130
      
      // 手指圆圈在SVG中的坐标（viewBox坐标系）
      const fingerCirclePositions: Record<string, { cx: number, cy: number }> = {
        'left-pinky': { cx: 22, cy: 48 },
        'left-ring': { cx: 35, cy: 43 },
        'left-middle': { cx: 48, cy: 40 },
        'left-index': { cx: 61, cy: 43 },
        'left-thumb': { cx: 38, cy: 115 },
        'right-pinky': { cx: 78, cy: 48 },
        'right-ring': { cx: 65, cy: 43 },
        'right-middle': { cx: 52, cy: 40 },
        'right-index': { cx: 39, cy: 43 },
        'right-thumb': { cx: 62, cy: 115 },
        'thumb': { cx: 50, cy: 115 },
      }
      
      let fingerX = 0, fingerY = 0
      
      const fingerPos = fingerCirclePositions[activeFinger.value] || { cx: 50, cy: 40 }
      
      // 将viewBox坐标转换为实际像素坐标
      const circleX = (fingerPos.cx / viewBoxWidth) * svgWidth
      const circleY = (fingerPos.cy / viewBoxHeight) * svgHeight
      
      // 手部轮廓容器位置（固定在键盘下方中央）
      if (activeFingerRef.value) {
        try {
          const fingerContainerRect = activeFingerRef.value.getBoundingClientRect()
          // 计算手指圆圈相对于键盘容器的位置
          fingerX = fingerContainerRect.left + circleX - containerRect.left
          fingerY = fingerContainerRect.top + circleY - containerRect.top
        } catch (e) {
          console.warn('Failed to get finger container rect:', e)
          setTimeout(updateGuideLine, 100)
          return
        }
      } else {
        // 如果找不到 ref，延迟重试
        setTimeout(updateGuideLine, 100)
        return
      }
      
      // 查找按键元素的中心点
      let keyX = 0, keyY = 0
      if (!highlightedKeyRef.value) {
        // 如果找不到按键元素，延迟重试
        setTimeout(updateGuideLine, 100)
        return
      }
      
      try {
        const keyRect = highlightedKeyRef.value.getBoundingClientRect()
        keyX = keyRect.left + keyRect.width / 2 - containerRect.left
        keyY = keyRect.top + keyRect.height / 2 - containerRect.top
      } catch (e) {
        console.warn('Failed to get key rect:', e)
        setTimeout(updateGuideLine, 100)
        return
      }
      
      // 确保坐标有效
      if (isNaN(fingerX) || isNaN(fingerY) || isNaN(keyX) || isNaN(keyY)) {
        console.warn('Invalid coordinates:', { fingerX, fingerY, keyX, keyY })
        guideLineVisible.value = false
        return
      }
      
      // 创建从手部轮廓（底部）到按键（上方）的贝塞尔曲线路径
      // 引导线从下往上，控制点向下偏移，形成向上的弧形
      const midX = (fingerX + keyX) / 2
      const midY = Math.max(fingerY, keyY) + Math.abs(keyY - fingerY) * 0.6
      
      guideLinePath.value = `M ${fingerX} ${fingerY} Q ${midX} ${midY} ${keyX} ${keyY}`
      guideLineId.value = generateGuideLineId()
      guideLineVisible.value = true
    } catch (error) {
      console.warn('Failed to update guide line:', error)
      guideLineVisible.value = false
    }
  })
}

// 监听相关值变化，更新引导线
watch([() => activeFinger.value, () => highlightedKey.value, () => props.showFingerGuide], () => {
  // 延迟更新，确保 DOM 已渲染
  setTimeout(updateGuideLine, 50)
})

onMounted(() => {
  // 延迟更新，确保所有 DOM 元素都已渲染
  setTimeout(updateGuideLine, 200)
  // 监听窗口大小变化
  window.addEventListener('resize', updateGuideLine)
})

onUnmounted(() => {
  window.removeEventListener('resize', updateGuideLine)
})
</script>

<template>
  <div class="virtual-keyboard" v-if="visible" ref="keyboardContainerRef">
    <!-- SVG 引导线覆盖层 - 覆盖整个虚拟键盘区域 -->
    <svg 
      v-if="guideLineVisible && showFingerGuide && activeFinger && highlightedKey && guideLinePath"
      ref="guideLineRef"
      class="guide-line-overlay"
      :style="{ 
        '--finger-color': activeFinger ? fingerColors[activeFinger] : '#2196f3',
        'width': '100%',
        'height': '100%'
      }"
    >
      <defs>
        <linearGradient :id="`guideGradient-${activeFinger}`" x1="0%" y1="0%" x2="100%" y2="100%">
          <stop offset="0%" :stop-color="activeFinger ? fingerColors[activeFinger] : '#2196f3'" stop-opacity="0.8" />
          <stop offset="100%" :stop-color="activeFinger ? fingerColors[activeFinger] : '#2196f3'" stop-opacity="0.4" />
        </linearGradient>
        <marker :id="`arrowhead-${activeFinger}`" markerWidth="10" markerHeight="10" refX="5" refY="5" orient="auto">
          <polygon points="0 0, 10 5, 0 10" :fill="activeFinger ? fingerColors[activeFinger] : '#2196f3'" />
        </marker>
        <!-- 发光效果 -->
        <filter id="glow">
          <feGaussianBlur stdDeviation="2" result="coloredBlur"/>
          <feMerge>
            <feMergeNode in="coloredBlur"/>
            <feMergeNode in="SourceGraphic"/>
          </feMerge>
        </filter>
      </defs>
      <!-- 引导线路径 -->
      <path
        :id="`path-${guideLineId}`"
        :d="guideLinePath"
        :stroke="`url(#guideGradient-${activeFinger})`"
        stroke-width="3"
        fill="none"
        stroke-linecap="round"
        stroke-linejoin="round"
        :marker-end="`url(#arrowhead-${activeFinger})`"
        class="guide-path"
      />
      <!-- 沿路径移动的手指指示器（圆形） -->
      <circle
        r="6"
        :fill="activeFinger ? fingerColors[activeFinger] : '#42a5f5'"
        opacity="0.9"
        filter="url(#glow)"
      >
        <animateMotion
          dur="1.5s"
          repeatCount="indefinite"
        >
          <mpath :href="`#path-${guideLineId}`" />
        </animateMotion>
      </circle>
    </svg>
    
    <div class="keyboard-container">
      
      <div class="keyboard-row" v-for="(row, rowIndex) in keyboardLayout" :key="rowIndex">
        <div
          v-for="(keyInfo, keyIndex) in row"
          :key="`${rowIndex}-${keyIndex}`"
          :ref="el => { if (highlightedKey === keyInfo.code && el) highlightedKeyRef = el }"
          :class="getKeyClass(keyInfo)"
          :style="getKeyStyle(keyInfo)"
        >
          <span class="key-label">{{ getKeyLabel(keyInfo) }}</span>
        </div>
      </div>
    </div>
    
    <!-- 左手轮廓 - 始终显示在左侧红框 -->
    <div 
      v-if="showFingerGuide && highlightedKey"
      class="finger-dot-container left-hand-container"
      :ref="el => { if (activeFinger && activeFinger.startsWith('left') && el) activeFingerRef = el }"
      :style="getLeftHandOutlineStyle()"
    >
      <!-- 手部轮廓（增强版，更清晰） -->
      <svg 
        class="hand-outline-svg"
        viewBox="0 0 100 130"
        width="140"
        height="182"
      >
        <!-- 左手轮廓 - 始终显示 -->
        <g class="hand-outline left-hand">
          <!-- 手掌 -->
          <ellipse cx="50" cy="90" rx="28" ry="18" fill="none" stroke="#666" stroke-width="3.5" opacity="0.6" />
          <!-- 小指 -->
          <g :class="{ 'finger-group-active': activeFinger === 'left-pinky' }">
            <line x1="28" y1="78" x2="22" y2="48" :stroke="fingerColors['left-pinky']" stroke-width="3" :opacity="activeFinger === 'left-pinky' ? 0.8 : 0.4" />
            <circle cx="22" cy="48" r="6" :fill="fingerColors['left-pinky']" 
                    :stroke="fingerColors['left-pinky']" stroke-width="3" :opacity="activeFinger === 'left-pinky' ? 0.95 : 0.5" />
          </g>
          <!-- 无名指 -->
          <g :class="{ 'finger-group-active': activeFinger === 'left-ring' }">
            <line x1="38" y1="75" x2="35" y2="43" :stroke="fingerColors['left-ring']" stroke-width="3" :opacity="activeFinger === 'left-ring' ? 0.8 : 0.4" />
            <circle cx="35" cy="43" r="6" :fill="fingerColors['left-ring']" 
                    :stroke="fingerColors['left-ring']" stroke-width="3" :opacity="activeFinger === 'left-ring' ? 0.95 : 0.5" />
          </g>
          <!-- 中指 -->
          <g :class="{ 'finger-group-active': activeFinger === 'left-middle' }" :ref="el => { if (activeFinger === 'left-middle' && el) activeFingerRef = el }">
            <line x1="48" y1="73" x2="48" y2="40" :stroke="fingerColors['left-middle']" stroke-width="3" :opacity="activeFinger === 'left-middle' ? 0.8 : 0.4" />
            <circle cx="48" cy="40" r="6" :fill="fingerColors['left-middle']" 
                    :stroke="fingerColors['left-middle']" stroke-width="3" :opacity="activeFinger === 'left-middle' ? 0.95 : 0.5" />
          </g>
          <!-- 食指（活动手指） -->
          <g :class="{ 'finger-group-active': activeFinger === 'left-index' }" :ref="el => { if (activeFinger === 'left-index' && el) activeFingerRef = el }">
            <line x1="58" y1="75" x2="61" y2="43" :stroke="fingerColors['left-index']" stroke-width="4.5" :opacity="activeFinger === 'left-index' ? 0.9 : 0.4" />
            <circle cx="61" cy="43" r="7" :fill="fingerColors['left-index']" 
                    :stroke="fingerColors['left-index']" stroke-width="3.5" :opacity="activeFinger === 'left-index' ? 1 : 0.5" />
          </g>
          <!-- 拇指 -->
          <g :class="{ 'finger-group-active': activeFinger === 'left-thumb' || activeFinger === 'thumb' }" :ref="el => { if ((activeFinger === 'left-thumb' || activeFinger === 'thumb') && el) activeFingerRef = el }">
            <line x1="50" y1="100" x2="38" y2="115" :stroke="fingerColors['left-thumb']" stroke-width="3" :opacity="(activeFinger === 'left-thumb' || activeFinger === 'thumb') ? 0.8 : 0.4" />
            <circle cx="38" cy="115" r="6" :fill="fingerColors['left-thumb']" 
                    :stroke="fingerColors['left-thumb']" stroke-width="3" :opacity="(activeFinger === 'left-thumb' || activeFinger === 'thumb') ? 0.95 : 0.5" />
          </g>
        </g>
      </svg>
      
      <!-- 手指指示圆点（用于引导线起点） -->
      <div class="finger-dot"></div>
    </div>
    
    <!-- 右手轮廓 - 始终显示在右侧红框 -->
    <div 
      v-if="showFingerGuide && highlightedKey"
      class="finger-dot-container right-hand-container"
      :ref="el => { if (activeFinger && activeFinger.startsWith('right') && el) activeFingerRef = el }"
      :style="getRightHandOutlineStyle()"
    >
      <!-- 手部轮廓（增强版，更清晰） -->
      <svg 
        class="hand-outline-svg"
        viewBox="0 0 100 130"
        width="140"
        height="182"
      >
        <!-- 右手轮廓 -->
        <g class="hand-outline right-hand">
          <!-- 手掌 -->
          <ellipse cx="50" cy="90" rx="28" ry="18" fill="none" stroke="#666" stroke-width="3.5" opacity="0.6" />
          <!-- 小指 -->
          <g :class="{ 'finger-group-active': activeFinger === 'right-pinky' }">
            <line x1="72" y1="78" x2="78" y2="48" :stroke="fingerColors['right-pinky']" stroke-width="3" :opacity="activeFinger === 'right-pinky' ? 0.8 : 0.4" />
            <circle cx="78" cy="48" r="6" :fill="fingerColors['right-pinky']" 
                    :stroke="fingerColors['right-pinky']" stroke-width="3" :opacity="activeFinger === 'right-pinky' ? 0.95 : 0.5" />
          </g>
          <!-- 无名指 -->
          <g :class="{ 'finger-group-active': activeFinger === 'right-ring' }">
            <line x1="62" y1="75" x2="65" y2="43" :stroke="fingerColors['right-ring']" stroke-width="3" :opacity="activeFinger === 'right-ring' ? 0.8 : 0.4" />
            <circle cx="65" cy="43" r="6" :fill="fingerColors['right-ring']" 
                    :stroke="fingerColors['right-ring']" stroke-width="3" :opacity="activeFinger === 'right-ring' ? 0.95 : 0.5" />
          </g>
          <!-- 中指 -->
          <g :class="{ 'finger-group-active': activeFinger === 'right-middle' }" :ref="el => { if (activeFinger === 'right-middle' && el) activeFingerRef = el }">
            <line x1="52" y1="73" x2="52" y2="40" :stroke="fingerColors['right-middle']" stroke-width="3" :opacity="activeFinger === 'right-middle' ? 0.8 : 0.4" />
            <circle cx="52" cy="40" r="6" :fill="fingerColors['right-middle']" 
                    :stroke="fingerColors['right-middle']" stroke-width="3" :opacity="activeFinger === 'right-middle' ? 0.95 : 0.5" />
          </g>
          <!-- 食指（活动手指） -->
          <g :class="{ 'finger-group-active': activeFinger === 'right-index' }" :ref="el => { if (activeFinger === 'right-index' && el) activeFingerRef = el }">
            <line x1="42" y1="75" x2="39" y2="43" :stroke="fingerColors['right-index']" stroke-width="4.5" :opacity="activeFinger === 'right-index' ? 0.9 : 0.4" />
            <circle cx="39" cy="43" r="7" :fill="fingerColors['right-index']" 
                    :stroke="fingerColors['right-index']" stroke-width="3.5" :opacity="activeFinger === 'right-index' ? 1 : 0.5" />
          </g>
          <!-- 拇指 -->
          <g :class="{ 'finger-group-active': activeFinger === 'right-thumb' || activeFinger === 'thumb' }" :ref="el => { if ((activeFinger === 'right-thumb' || activeFinger === 'thumb') && el) activeFingerRef = el }">
            <line x1="50" y1="100" x2="62" y2="115" :stroke="fingerColors['right-thumb']" stroke-width="3" :opacity="(activeFinger === 'right-thumb' || activeFinger === 'thumb') ? 0.8 : 0.4" />
            <circle cx="62" cy="115" r="6" :fill="fingerColors['right-thumb']" 
                    :stroke="fingerColors['right-thumb']" stroke-width="3" :opacity="(activeFinger === 'right-thumb' || activeFinger === 'thumb') ? 0.95 : 0.5" />
          </g>
        </g>
      </svg>
      
      <!-- 手指指示圆点（用于引导线起点） -->
      <div class="finger-dot"></div>
    </div>
    
    <!-- 手指名称标签 - 放在中间红框位置（只显示一个提示） -->
    <div 
      v-if="showFingerGuide && activeFinger && highlightedKey"
      class="finger-label-badge" 
      :style="{ ...getFingerLabelStyle(), ...getLabelBadgeStyle() }"
    >
      使用 {{ getFingerName(activeFinger) }} 按下此键
    </div>
  </div>
</template>

<style scoped lang="scss">
.virtual-keyboard {
  width: 100%;
  max-width: 800px;
  margin: 15px auto;
  padding: 15px;
  background: var(--color-bg-1, #fff);
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  position: relative;
  // 移除固定高度限制，让内容自适应，避免在不同屏幕上出现滚动条
  overflow: visible; // 允许内容正常显示，但确保不会溢出父容器
}

.keyboard-container {
  display: flex;
  flex-direction: column;
  gap: 6px;
  position: relative;
  padding-bottom: 60px; // 为手部轮廓和提示文字留出最小空间（减少空白）
  overflow: hidden; // 防止内容溢出导致滚动条
}

.guide-line-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
  z-index: 5;
  overflow: hidden; // 防止溢出导致滚动条
  // 确保 SVG 能覆盖到键盘和下方手部轮廓区域
  height: 100%;
}

  .finger-dot-container {
  pointer-events: none;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 8px;
  // 位置由 getHandOutlineStyle() 动态计算，放在键盘容器内的空白区域
  
  .finger-label-badge {
    order: -1; // 标签在手部轮廓上方
    margin-bottom: 5px;
  }
  
  .hand-outline-svg {
    background: transparent;
    padding: 0;
    filter: drop-shadow(0 2px 8px rgba(0, 0, 0, 0.2));
  }
  
  .hand-outline {
    .finger-group-active {
      animation: fingerGroupPulse 1.5s ease-in-out infinite;
      
      line {
        stroke-width: 3.5 !important;
        opacity: 0.9 !important;
      }
      
      circle {
        animation: fingerCirclePulse 1.5s ease-in-out infinite;
      }
    }
  }
  
  .finger-label-badge {
    color: #fff;
    padding: 8px 16px;
    border-radius: 20px;
    font-size: 15px;
    font-weight: 700;
    box-shadow: 0 4px 16px rgba(0, 0, 0, 0.3), 0 0 0 2px rgba(255, 255, 255, 0.6);
    white-space: nowrap;
    animation: badgePulse 1.5s ease-in-out infinite;
    backdrop-filter: blur(8px);
    border: 2px solid rgba(255, 255, 255, 0.4);
    text-shadow: 0 1px 3px rgba(0, 0, 0, 0.5);
    pointer-events: none; // 确保不阻挡交互
  }
  
  .finger-dot {
    width: 12px;
    height: 12px;
    border-radius: 50%;
    background: #42a5f5;
    border: 2px solid #fff;
    box-shadow: 0 2px 8px rgba(66, 165, 245, 0.6);
    animation: fingerDotPulse 1.5s ease-in-out infinite;
  }
}

@keyframes fingerGroupPulse {
  0%, 100% {
    opacity: 0.9;
    transform: scale(1);
  }
  50% {
    opacity: 1;
    transform: scale(1.05);
  }
}

@keyframes fingerCirclePulse {
  0%, 100% {
    r: 4;
  }
  50% {
    r: 5.5;
  }
}

@keyframes badgePulse {
  0%, 100% {
    transform: scale(1);
    box-shadow: 0 2px 8px rgba(66, 165, 245, 0.4);
  }
  50% {
    transform: scale(1.05);
    box-shadow: 0 4px 12px rgba(66, 165, 245, 0.6);
  }
}


@keyframes fingerDotPulse {
  0%, 100% {
    transform: scale(1);
    box-shadow: 0 2px 8px rgba(66, 165, 245, 0.6);
  }
  50% {
    transform: scale(1.2);
    box-shadow: 0 4px 12px rgba(66, 165, 245, 0.8);
  }
}

.finger-tip-dot {
  animation: fingerTipPulse 1.5s ease-in-out infinite;
  filter: drop-shadow(0 0 4px currentColor);
}

@keyframes fingerPulse {
  0%, 100% {
    opacity: 0.6;
    stroke-width: 3;
  }
  50% {
    opacity: 1;
    stroke-width: 3.5;
  }
}

@keyframes fingerTipPulse {
  0%, 100% {
    r: 6;
    opacity: 0.8;
  }
  50% {
    r: 8;
    opacity: 1;
  }
}

.guide-path {
  filter: drop-shadow(0 2px 4px rgba(0, 0, 0, 0.2));
  animation: guideLinePulse 2s ease-in-out infinite;
}

@keyframes guideLinePulse {
  0%, 100% {
    opacity: 0.8;
  }
  50% {
    opacity: 1;
  }
}

.keyboard-row {
  display: flex;
  gap: 4px;
  justify-content: center;
}

.key {
  min-width: 36px;
  height: 36px;
  display: flex;
  align-items: center;
  justify-content: center;
  border: 2px solid #ddd;
  border-radius: 5px;
  font-size: 11px;
  font-weight: 500;
  transition: all 0.1s ease;
  position: relative;
  cursor: default;
  user-select: none;

  .key-label {
    z-index: 2;
    position: relative;
  }

  &.key-highlighted {
    transform: scale(1.15);
    box-shadow: 0 4px 16px rgba(76, 175, 80, 0.6);
    z-index: 10;
    animation: pulse 1.5s ease-in-out infinite;
    border-width: 3px;
  }

  &.key-pressed {
    transform: scale(0.95);
    box-shadow: 0 2px 8px rgba(255, 152, 0, 0.6);
    z-index: 15;
    animation: press 0.2s ease-out;
  }
}

@keyframes press {
  0% {
    transform: scale(1);
  }
  50% {
    transform: scale(0.9);
  }
  100% {
    transform: scale(0.95);
  }
}

@keyframes pulse {
  0%, 100% {
    box-shadow: 0 4px 16px rgba(76, 175, 80, 0.6);
    background-color: #4caf50;
  }
  50% {
    box-shadow: 0 4px 24px rgba(76, 175, 80, 0.8);
    background-color: #66bb6a;
  }
}


.finger-indicators {
  margin-top: 20px;
  padding: 15px;
  background: rgba(0, 0, 0, 0.02);
  border-radius: 8px;
  position: relative;
  z-index: 1;
}

.finger-label {
  text-align: center;
  font-size: 14px;
  color: var(--color-font-2, #666);
  margin-bottom: 15px;
  font-weight: 500;
}

.finger-hands {
  display: flex;
  justify-content: space-around;
  align-items: flex-end;
  gap: 40px;
}

.hand {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 8px;
}

.left-hand {
  flex-direction: row-reverse;
}

.right-hand {
  flex-direction: row;
}

.finger {
  width: 20px;
  height: 60px;
  border: 2px solid #ddd;
  border-radius: 10px;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: flex-end;
  padding-bottom: 4px;
  transition: all 0.3s ease;
  position: relative;

  &.active {
    border-color: currentColor;
    transform: scale(1.1);
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.2);
  }

  .finger-tip {
    width: 12px;
    height: 12px;
    border-radius: 50%;
    background: rgba(0, 0, 0, 0.1);
  }

  &.active .finger-tip {
    background: rgba(255, 255, 255, 0.8);
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
  }
}

.thumb {
  width: 24px;
  height: 40px;
  margin-top: 20px;
}

// 响应式设计
@media (max-width: 768px) {
  .virtual-keyboard {
    padding: 10px;
  }

  .key {
    min-width: 32px;
    height: 32px;
    font-size: 10px;
  }

  .finger {
    width: 16px;
    height: 50px;
  }
}
</style>

