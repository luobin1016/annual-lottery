<script setup lang="ts">
import type { IPersonConfig } from '@/types/storeType'
import type { Material } from 'three'
import StarsBackground from '@/components/StarsBackground/index.vue'
import MobileJoinLottery from '@/components/MobileJoinLottery/index.vue'
import { useElementPosition, useElementStyle } from '@/hooks/useElement'
import i18n from '@/locales/i18n'
import useStore from '@/store'
import { filterData, selectCard } from '@/utils'
import { rgba } from '@/utils/color'
import * as TWEEN from '@tweenjs/tween.js'
import confetti from 'canvas-confetti'
import { storeToRefs } from 'pinia'
import { Object3D, PerspectiveCamera, Scene, Vector3 } from 'three'
import { CSS3DObject, CSS3DRenderer } from 'three-css3d'
import { TrackballControls } from 'three/examples/jsm/controls/TrackballControls.js'
import { computed, nextTick, onMounted, onUnmounted, ref, watch } from 'vue'
import { useI18n } from 'vue-i18n'
import { useRouter } from 'vue-router'
import { useToast } from 'vue-toast-notification'
import PrizeList from './PrizeList.vue'
import JoinLottery from '@/components/JoinLottery/index.vue'
import QRCode from 'qrcode'
import 'vue-toast-notification/dist/theme-sugar.css'

// 二维码弹窗
const showQrModal = ref(false)
const qrCodeDataUrl = ref('')

// 检测是否为移动端
const isMobile = computed(() => {
  const ua = navigator.userAgent
  return /Android|webOS|iPhone|iPad|iPod|BlackBerry|IEMobile|Opera Mini/i.test(ua)
})

const { t } = useI18n()
const toast = useToast()
const router = useRouter()
const personConfig = useStore().personConfig
const globalConfig = useStore().globalConfig
const prizeConfig = useStore().prizeConfig
const themeStore = useStore().themeStore

// 复制分享链接并显示二维码
async function copyShareLink() {
  const themeId = themeStore.currentThemeId
  if (!themeId) {
    console.warn('[copyShareLink] No theme ID')
    return
  }
  
  const baseUrl = window.location.origin
  const shareUrl = `${baseUrl}/t/${themeId}`
  
  // 生成二维码
  try {
    qrCodeDataUrl.value = await QRCode.toDataURL(shareUrl, {
      width: 200,
      margin: 2,
      color: { dark: '#1a1a2e', light: '#ffffff' }
    })
    showQrModal.value = true
  } catch (err) {
    console.error('[QRCode] Generate failed:', err)
  }
  
  // 降级方案：创建临时输入框（兼容非 HTTPS 环境）
  const fallbackCopy = () => {
    const input = document.createElement('input')
    input.value = shareUrl
    input.style.position = 'fixed'
    input.style.left = '-9999px'
    document.body.appendChild(input)
    input.select()
    input.setSelectionRange(0, 99999)
    try {
      document.execCommand('copy')
      toast.open({
        message: t('entry.linkCopied'),
        type: 'success',
        position: 'top-right',
        duration: 3000,
      })
    } catch (e) {
      console.error('[copyShareLink] execCommand failed:', e)
    }
    document.body.removeChild(input)
  }
  
  // 优先使用 clipboard API
  if (navigator.clipboard && window.isSecureContext) {
    navigator.clipboard.writeText(shareUrl).then(() => {
      toast.open({
        message: t('entry.linkCopied'),
        type: 'success',
        position: 'top-right',
        duration: 3000,
      })
    }).catch(() => {
      fallbackCopy()
    })
  } else {
    fallbackCopy()
  }
}

// 关闭二维码弹窗
function closeQrModal() {
  showQrModal.value = false
}

const { getAllPersonList: allPersonList, getNotPersonList: notPersonList, getNotThisPrizePersonList: notThisPrizePersonList,
} = storeToRefs(personConfig)
const { getCurrentPrize: currentPrize } = storeToRefs(prizeConfig)
const { getTopTitle: topTitle, getCardColor: cardColor, getPatterColor: patternColor, getPatternList: patternList, getTextColor: textColor, getLuckyColor: luckyColor, getCardSize: cardSize, getTextSize: textSize, getRowCount: rowCount, getBackground: homeBackground, getIsShowAvatar: isShowAvatar } = storeToRefs(globalConfig)
const tableData = ref<any[]>([])
const currentStatus = ref(0) // 0为初始状态， 1为抽奖准备状态，2为抽奖中状态，3为抽奖结束状态
const ballRotationY = ref(0)
const containerRef = ref<HTMLElement>()
const canOperate = ref(true)
const cameraZ = ref(3000)

const scene = ref()
const camera = ref()
const renderer = ref()
const controls = ref()
const objects = ref<any[]>([])
interface TargetType {
  grid: any[]
  helix: any[]
  table: any[]
  sphere: any[]
}
const targets: TargetType = {
  grid: [],
  helix: [],
  table: [],
  sphere: [],
}

const luckyTargets = ref<any[]>([])
const luckyCardList = ref<number[]>([])
const luckyCardMap = ref<Map<number, number>>(new Map()) // 中奖人员ID -> 卡片索引的映射
const luckyCount = ref(0)
const displayPage = ref(0) // 当前显示的页码（用于超过10人时滚动）
const maxDisplayCount = 10 // 一页最多显示10人
const autoScrollTimer = ref<any>(null)
const personPool = ref<IPersonConfig[]>([])

const intervalTimer = ref<any>(null)
const syncTimer = ref<any>(null)
const lastPersonCount = ref(0)
// 填充数据，填满七行
function initTableData() {
  if (allPersonList.value.length <= 0) {
    return
  }
  const totalCount = rowCount.value * 7
  const originPersonData = JSON.parse(JSON.stringify(allPersonList.value))
  const originPersonLength = originPersonData.length
  if (originPersonLength < totalCount) {
    const repeatCount = Math.ceil(totalCount / originPersonLength)
    // 复制数据
    for (let i = 0; i < repeatCount; i++) {
      tableData.value = tableData.value.concat(JSON.parse(JSON.stringify(originPersonData)))
    }
  }
  else {
    tableData.value = originPersonData.slice(0, totalCount)
  }
  tableData.value = filterData(tableData.value.slice(0, totalCount), rowCount.value)
}
function init() {
  const felidView = 40
  const width = window.innerWidth
  const height = window.innerHeight
  const aspect = width / height
  const nearPlane = 1
  const farPlane = 10000
  const WebGLoutput = containerRef.value

  scene.value = new Scene()
  camera.value = new PerspectiveCamera(felidView, aspect, nearPlane, farPlane)
  camera.value.position.z = cameraZ.value
  renderer.value = new CSS3DRenderer()
  renderer.value.setSize(width, height * 0.9)
  renderer.value.domElement.style.position = 'absolute'
  // 垂直居中
  renderer.value.domElement.style.paddingTop = '50px'
  renderer.value.domElement.style.top = '50%'
  renderer.value.domElement.style.left = '50%'
  renderer.value.domElement.style.transform = 'translate(-50%, -50%)'
  WebGLoutput!.appendChild(renderer.value.domElement)

  controls.value = new TrackballControls(camera.value, renderer.value.domElement)
  controls.value.rotateSpeed = 1
  controls.value.staticMoving = true
  controls.value.minDistance = 500
  controls.value.maxDistance = 6000
  controls.value.addEventListener('change', render)

  const tableLen = tableData.value.length
  for (let i = 0; i < tableLen; i++) {
    let element = document.createElement('div')
    element.className = 'element-card'

    const number = document.createElement('div')
    number.className = 'card-id'
    number.textContent = tableData.value[i].uid
    if(isShowAvatar.value) number.style.display = 'none'
    element.appendChild(number)

    const symbol = document.createElement('div')
    symbol.className = 'card-name'
    symbol.textContent = tableData.value[i].name
    if(isShowAvatar.value) symbol.className = 'card-name card-avatar-name'
    element.appendChild(symbol)

    const detail = document.createElement('div')
    detail.className = 'card-detail'
    detail.innerHTML = `${tableData.value[i].department}<br/>${tableData.value[i].identity}`
    if(isShowAvatar.value) detail.style.display = 'none'
    element.appendChild(detail)

    const avatar = document.createElement('img');
    avatar.className = 'card-avatar';
    avatar.src = tableData.value[i].avatar;
    avatar.alt = 'avatar';
    avatar.style.width = 'auto';
    avatar.style.height = 'auto';
    if(!isShowAvatar.value) avatar.style.display = 'none'
    element.appendChild(avatar);

    element = useElementStyle(element, tableData.value[i], i, patternList.value, patternColor.value, cardColor.value, cardSize.value, textSize.value, 'default', 'add', textColor.value)
    const object = new CSS3DObject(element)
    object.position.x = Math.random() * 4000 - 2000
    object.position.y = Math.random() * 4000 - 2000
    object.position.z = Math.random() * 4000 - 2000
    scene.value.add(object)

    objects.value.push(object)
  }

  createTableVertices()
  createSphereVertices()
  createHelixVertices()

  function createTableVertices() {
    const tableLen = tableData.value.length

    for (let i = 0; i < tableLen; i++) {
      const object = new Object3D()

      object.position.x = tableData.value[i].x * (cardSize.value.width + 40) - rowCount.value * 90
      object.position.y = -tableData.value[i].y * (cardSize.value.height + 20) + 1000
      object.position.z = 0

      targets.table.push(object)
    }
  }

  function createSphereVertices() {
    let i = 0
    const objLength = objects.value.length
    const vector = new Vector3()

    for (; i < objLength; ++i) {
      const phi = Math.acos(-1 + (2 * i) / objLength)
      const theta = Math.sqrt(objLength * Math.PI) * phi
      const object = new Object3D()

      object.position.x = 800 * Math.cos(theta) * Math.sin(phi)
      object.position.y = 800 * Math.sin(theta) * Math.sin(phi)
      object.position.z = -800 * Math.cos(phi)

      // rotation object

      vector.copy(object.position).multiplyScalar(2)
      object.lookAt(vector)
      targets.sphere.push(object)
    }
  }
  function createHelixVertices() {
    let i = 0
    const vector = new Vector3()
    const objLength = objects.value.length
    for (; i < objLength; ++i) {
      const phi = i * 0.213 + Math.PI

      const object = new Object3D()

      object.position.x = 800 * Math.sin(phi)
      object.position.y = -(i * 8) + 450
      object.position.z = 800 * Math.cos(phi + Math.PI)

      object.scale.set(1.1, 1.1, 1.1)

      vector.x = object.position.x * 2
      vector.y = object.position.y
      vector.z = object.position.z * 2

      object.lookAt(vector)

      targets.helix.push(object)
    }
  }
  window.addEventListener('resize', onWindowResize, false)
  transform(targets.table, 1000)
  render()
}

function transform(targets: any[], duration: number) {
  TWEEN.removeAll()
  if (intervalTimer.value) {
    clearInterval(intervalTimer.value)
    intervalTimer.value = null
    randomBallData('sphere')
  }

  return new Promise((resolve) => {
    const objLength = objects.value.length
    for (let i = 0; i < objLength; ++i) {
      const object = objects.value[i]
      const target = targets[i]
      new TWEEN.Tween(object.position)
        .to({ x: target.position.x, y: target.position.y, z: target.position.z }, Math.random() * duration + duration)
        .easing(TWEEN.Easing.Exponential.InOut)
        .start()

      new TWEEN.Tween(object.rotation)
        .to({ x: target.rotation.x, y: target.rotation.y, z: target.rotation.z }, Math.random() * duration + duration)
        .easing(TWEEN.Easing.Exponential.InOut)
        .start()
        .onComplete(() => {
          if (luckyCardList.value.length) {
            luckyCardList.value.forEach((cardIndex: any) => {
              const item = objects.value[cardIndex]
              useElementStyle(item.element, {} as any, i, patternList.value, patternColor.value, cardColor.value, cardSize.value, textSize.value, 'sphere', 'change', textColor.value)
            })
          }
          luckyTargets.value = []
          luckyCardList.value = []

          canOperate.value = true
        })
    }

    // 这个补间用来在位置与旋转补间同步执行，通过onUpdate在每次更新数据后渲染scene和camera
    new TWEEN.Tween({})
      .to({}, duration * 2)
      .onUpdate(render)
      .start()
      .onComplete(() => {
        canOperate.value = true
        resolve('')
      })
  })
}
function onWindowResize() {
  camera.value.aspect = window.innerWidth / window.innerHeight
  camera.value.updateProjectionMatrix()

  renderer.value.setSize(window.innerWidth, window.innerHeight)
  render()
}

/**
 * [animation update all tween && controls]
 */
function animation() {
  TWEEN.update()
  if (controls.value) {
    controls.value.update()
  }
  // 设置自动旋转
  // 设置相机位置
  requestAnimationFrame(animation)
}

// // 旋转的动画
function rollBall(rotateY: number, duration: number) {
  TWEEN.removeAll()

  return new Promise((resolve) => {
    scene.value.rotation.y = 0
    ballRotationY.value = Math.PI * rotateY * 1000
    const rotateObj = new TWEEN.Tween(scene.value.rotation)
    rotateObj
      .to(
        {
          // x: Math.PI * rotateX * 1000,
          x: 0,
          y: ballRotationY.value,
          // z: Math.PI * rotateZ * 1000
          z: 0,
        },
        duration * 1000,
      )
      .onUpdate(render)
      .start()
      .onStop(() => {
        resolve('')
      })
      .onComplete(() => {
        resolve('')
      })
  })
}
// 将视野转回正面
function resetCamera() {
  new TWEEN.Tween(camera.value.position)
    .to(
      {
        x: 0,
        y: 0,
        z: 3000,
      },
      1000,
    )
    .onUpdate(render)
    .start()
    .onComplete(() => {
      new TWEEN.Tween(camera.value.rotation)
        .to(
          {
            x: 0,
            y: 0,
            z: 0,
          },
          1000,
        )
        .onUpdate(render)
        .start()
        .onComplete(() => {
          canOperate.value = true
          // camera.value.lookAt(scene.value.position)
          camera.value.position.y = 0
          camera.value.position.x = 0
          camera.value.position.z = 3000
          camera.value.rotation.x = 0
          camera.value.rotation.y = 0
          camera.value.rotation.z = -0
          controls.value.reset()
        })
    })
}

function render() {
  if (renderer.value) {
    renderer.value.render(scene.value, camera.value)
  }
}
async function enterLottery() {
  if (!canOperate.value) {
    return
  }
  if (!intervalTimer.value) {
    randomBallData()
  }
  if (patternList.value.length) {
    for (let i = 0; i < patternList.value.length; i++) {
      if (i < rowCount.value * 7) {
        objects.value[patternList.value[i] - 1].element.style.backgroundColor = rgba(cardColor.value, Math.random() * 0.5 + 0.25)
      }
    }
  }
  canOperate.value = false
  await transform(targets.sphere, 600)
  currentStatus.value = 1
  rollBall(0.1, 1000)
}

// 返回初始状态
async function backToInitial() {
  if (!canOperate.value) {
    return
  }
  canOperate.value = false
  TWEEN.removeAll()
  if (intervalTimer.value) {
    clearInterval(intervalTimer.value)
    intervalTimer.value = null
  }
  
  // 重置场景旋转
  scene.value.rotation.x = 0
  scene.value.rotation.y = 0
  scene.value.rotation.z = 0
  
  // 重置相机位置
  camera.value.position.set(0, 0, cameraZ.value)
  camera.value.rotation.set(0, 0, 0)
  controls.value.reset()
  
  await transform(targets.table, 600)
  currentStatus.value = 0
  canOperate.value = true
}
// 开始抽奖
function startLottery() {
  if (!canOperate.value) {
    return
  }
  // 验证是否已抽完全部奖项
  if (currentPrize.value.isUsed || !currentPrize.value) {
    toast.open({
      message: i18n.global.t('error.personIsAllDone'),
      type: 'warning',
      position: 'top-right',
      duration: 10000,
    })

    return
  }
  personPool.value = currentPrize.value.isAll ? notThisPrizePersonList.value : notPersonList.value
  // 验证抽奖人数是否还够
  if (personPool.value.length < currentPrize.value.count - currentPrize.value.isUsedCount) {
    toast.open({
      message: i18n.global.t('error.personNotEnough'),
      type: 'warning',
      position: 'top-right',
      duration: 10000,
    })

    return
  }
  // 自定义抽奖个数 - 不设上限

  let leftover = currentPrize.value.count - currentPrize.value.isUsedCount
  const customCount = currentPrize.value.separateCount
  if (customCount && customCount.enable && customCount.countList.length > 0) {
    for (let i = 0; i < customCount.countList.length; i++) {
      if (customCount.countList[i].isUsedCount < customCount.countList[i].count) {
        leftover = customCount.countList[i].count - customCount.countList[i].isUsedCount
        break
      }
    }
  }
  luckyCount.value = leftover
  for (let i = 0; i < luckyCount.value; i++) {
    if (personPool.value.length > 0) {
      // 解决随机元素概率过于不均等问题
      const randomIndex = Math.floor(Math.random() * (personPool.value.length - 1))
      luckyTargets.value.push(personPool.value[randomIndex])
      personPool.value.splice(randomIndex, 1)
    }
  }

  toast.open({
    // message: `现在抽取${currentPrize.value.name} ${leftover}人`,
    message: i18n.global.t('error.startDraw', { count: currentPrize.value.name, leftover }),
    type: 'default',
    position: 'top-right',
    duration: 8000,
  })
  currentStatus.value = 2
  rollBall(10, 3000)
}

async function stopLottery() {
  if (!canOperate.value) {
    return
  }
  canOperate.value = false
  rollBall(0, 1)

  displayPage.value = 0
  const totalPages = Math.ceil(luckyTargets.value.length / maxDisplayCount)
  
  // 显示第一页中奖人员
  showLuckyPage(0, true)
  
  // 如果超过10人，启动自动滚动（延迟启动，等第一页动画完成）
  if (totalPages > 1) {
    setTimeout(() => {
      startAutoScroll()
    }, 2000)
  }
}

// 启动自动滚动
function startAutoScroll() {
  // 清除之前的定时器
  if (autoScrollTimer.value) {
    clearInterval(autoScrollTimer.value)
  }
  const totalPages = Math.ceil(luckyTargets.value.length / maxDisplayCount)
  autoScrollTimer.value = setInterval(() => {
    const oldPage = displayPage.value
    displayPage.value = (displayPage.value + 1) % totalPages
    showLuckyPage(displayPage.value, false, oldPage)
  }, 4000)
}

// 手动切换到上一页
function prevPage() {
  if (displayPage.value > 0) {
    // 停止自动滚动
    if (autoScrollTimer.value) {
      clearInterval(autoScrollTimer.value)
      autoScrollTimer.value = null
    }
    const oldPage = displayPage.value
    displayPage.value--
    showLuckyPage(displayPage.value, false, oldPage)
    // 手动操作后重新启动自动滚动
    setTimeout(() => startAutoScroll(), 4000)
  }
}

// 手动切换到下一页
function nextPage() {
  const totalPages = Math.ceil(luckyTargets.value.length / maxDisplayCount)
  if (displayPage.value < totalPages - 1) {
    // 停止自动滚动
    if (autoScrollTimer.value) {
      clearInterval(autoScrollTimer.value)
      autoScrollTimer.value = null
    }
    const oldPage = displayPage.value
    displayPage.value++
    showLuckyPage(displayPage.value, false, oldPage)
    // 手动操作后重新启动自动滚动
    setTimeout(() => startAutoScroll(), 4000)
  }
}

// 手动跳转到指定页
function goToPage(page: number) {
  if (page === displayPage.value) return
  // 停止自动滚动
  if (autoScrollTimer.value) {
    clearInterval(autoScrollTimer.value)
    autoScrollTimer.value = null
  }
  const oldPage = displayPage.value
  displayPage.value = page
  showLuckyPage(page, false, oldPage)
  // 手动操作后重新启动自动滚动
  setTimeout(() => startAutoScroll(), 4000)
}

// 显示指定页的中奖人员 - 优化版本
function showLuckyPage(page: number, isFirstShow = false, prevPage: number = -1) {
  const windowSize = { width: window.innerWidth, height: window.innerHeight }
  const startIndex = page * maxDisplayCount
  const endIndex = Math.min(startIndex + maxDisplayCount, luckyTargets.value.length)
  const pagePersons = luckyTargets.value.slice(startIndex, endIndex)
  
  // 清除所有正在进行的TWEEN动画
  TWEEN.removeAll()
  
  // 隐藏之前显示的中奖卡片
  if (!isFirstShow && prevPage >= 0) {
    const prevStartIndex = prevPage * maxDisplayCount
    const prevEndIndex = Math.min(prevStartIndex + maxDisplayCount, luckyTargets.value.length)
    for (let i = prevStartIndex; i < prevEndIndex; i++) {
      const person = luckyTargets.value[i]
      const cardIndex = luckyCardMap.value.get(person.id)
      if (cardIndex !== undefined) {
        const item = objects.value[cardIndex]
        if (item) {
          item.position.z = -2000
        }
      }
    }
  }
  
  // 第一页且首次显示时清空映射
  if (page === 0 && isFirstShow) {
    luckyCardList.value = []
    luckyCardMap.value.clear()
    
    // 为所有中奖人员预先分配卡片索引
    luckyTargets.value.forEach((person: IPersonConfig) => {
      const cardIndex = selectCard(luckyCardList.value, tableData.value.length, person.id)
      luckyCardList.value.push(cardIndex)
      luckyCardMap.value.set(person.id, cardIndex)
    })
  }
  
  // 显示当前页的卡片
  pagePersons.forEach((person: IPersonConfig, index: number) => {
    const cardIndex = luckyCardMap.value.get(person.id)!
    const totalLuckyCount = pagePersons.length
    const item = objects.value[cardIndex]
    const { xTable, yTable } = useElementPosition(item, rowCount.value, totalLuckyCount, { width: cardSize.value.width * 2, height: cardSize.value.height * 2 }, windowSize, index)
    
    // 设置卡片样式
    item.element = useElementStyle(item.element, person, cardIndex, patternList.value, patternColor.value, luckyColor.value, { width: cardSize.value.width * 2, height: cardSize.value.height * 2 }, textSize.value * 2, 'lucky', 'change', textColor.value)
    
    if (isFirstShow) {
      // 第一次显示用动画
      new TWEEN.Tween(item.position)
        .to({ x: xTable, y: yTable, z: 1000 }, 800)
        .easing(TWEEN.Easing.Quadratic.Out)
        .start()
        .onComplete(() => {
          canOperate.value = true
          currentStatus.value = 3
        })
      new TWEEN.Tween(item.rotation)
        .to({ x: 0, y: 0, z: 0 }, 600)
        .easing(TWEEN.Easing.Quadratic.Out)
        .start()
        .onComplete(() => {
          if (index === 0) {
            confettiFire()
          }
          resetCamera()
        })
    } else {
      // 切换页面时直接设置位置，不用动画
      item.position.x = xTable
      item.position.y = yTable
      item.position.z = 1000
      item.rotation.x = 0
      item.rotation.y = 0
      item.rotation.z = 0
    }
  })
  
  // 手动渲染一次
  render()
}
// 继续
async function continueLottery() {
  if (!canOperate.value) {
    return
  }
  
  // 清理自动滚动定时器
  if (autoScrollTimer.value) {
    clearInterval(autoScrollTimer.value)
    autoScrollTimer.value = null
  }

  const customCount = currentPrize.value.separateCount
  if (customCount && customCount.enable && customCount.countList.length > 0) {
    for (let i = 0; i < customCount.countList.length; i++) {
      if (customCount.countList[i].isUsedCount < customCount.countList[i].count) {
        customCount.countList[i].isUsedCount += luckyCount.value
        break
      }
    }
  }
  currentPrize.value.isUsedCount += luckyCount.value
  luckyCount.value = 0
  if (currentPrize.value.isUsedCount >= currentPrize.value.count) {
    currentPrize.value.isUsed = true
    currentPrize.value.isUsedCount = currentPrize.value.count
  }
  personConfig.addAlreadyPersonList(luckyTargets.value, currentPrize.value)
  prizeConfig.updatePrizeConfig(currentPrize.value)
  await enterLottery()
}
function quitLottery() {
  // 清理自动滚动定时器
  if (autoScrollTimer.value) {
    clearInterval(autoScrollTimer.value)
    autoScrollTimer.value = null
  }
  enterLottery()
  currentStatus.value = 0
}
// 庆祝动画
function confettiFire() {
  const duration = 3 * 1000
  const end = Date.now() + duration;
  (function frame() {
    // launch a few confetti from the left edge
    confetti({
      particleCount: 2,
      angle: 60,
      spread: 55,
      origin: { x: 0 },
    })
    // and launch a few from the right edge
    confetti({
      particleCount: 2,
      angle: 120,
      spread: 55,
      origin: { x: 1 },
    })

    // keep going until we are out of time
    if (Date.now() < end) {
      requestAnimationFrame(frame)
    }
  }())
  centerFire(0.25, {
    spread: 26,
    startVelocity: 55,
  })
  centerFire(0.2, {
    spread: 60,
  })
  centerFire(0.35, {
    spread: 100,
    decay: 0.91,
    scalar: 0.8,
  })
  centerFire(0.1, {
    spread: 120,
    startVelocity: 25,
    decay: 0.92,
    scalar: 1.2,
  })
  centerFire(0.1, {
    spread: 120,
    startVelocity: 45,
  })
}
function centerFire(particleRatio: number, opts: any) {
  const count = 200
  confetti({
    origin: { y: 0.7 },
    ...opts,
    particleCount: Math.floor(count * particleRatio),
  })
}

async function setDefaultPersonList() {
  await personConfig.setDefaultPersonList()
  // 刷新页面
  window.location.reload()
}
// 随机替换数据
function randomBallData(mod: 'default' | 'lucky' | 'sphere' = 'default') {
  // 两秒执行一次
  intervalTimer.value = setInterval(() => {
    // 产生随机数数组
    const indexLength = 4
    const cardRandomIndexArr: number[] = []
    const personRandomIndexArr: number[] = []
    for (let i = 0; i < indexLength; i++) {
      // 解决随机元素概率过于不均等问题
      const randomCardIndex = Math.floor(Math.random() * (tableData.value.length - 1))
      const randomPersonIndex = Math.floor(Math.random() * (allPersonList.value.length - 1))
      if (luckyCardList.value.includes(randomCardIndex)) {
        continue
      }
      cardRandomIndexArr.push(randomCardIndex)
      personRandomIndexArr.push(randomPersonIndex)
    }
    for (let i = 0; i < cardRandomIndexArr.length; i++) {
      if (!objects.value[cardRandomIndexArr[i]]) {
        continue
      }
      objects.value[cardRandomIndexArr[i]].element = useElementStyle(objects.value[cardRandomIndexArr[i]].element, allPersonList.value[personRandomIndexArr[i]], cardRandomIndexArr[i], patternList.value, patternColor.value, cardColor.value, { width: cardSize.value.width, height: cardSize.value.height }, textSize.value, mod, 'change', textColor.value)
    }
  }, 200)
}
// 监听键盘
function listenKeyboard(e: KeyboardEvent) {
  // 阻止空格键的默认滚动行为
  if (e.keyCode === 32) {
    e.preventDefault()
  }
  
  if ((e.keyCode !== 32 && e.keyCode !== 27) && !canOperate.value) {
    return
  }
  
  // ESC 键处理
  if (e.keyCode === 27) {
    if (currentStatus.value === 1) {
      backToInitial()
    } else if (currentStatus.value === 3) {
      quitLottery()
    }
    return
  }
  
  // 空格键处理
  if (e.keyCode !== 32) {
    return
  }
  switch (currentStatus.value) {
    case 0:
      enterLottery()
      break
    case 1:
      startLottery()
      break
    case 2:
      stopLottery()
      break
    case 3:
      continueLottery()
      break
    default:
      break
  }
}

function cleanup() {
//   animationRunning.value = false
  clearInterval(intervalTimer.value)
  intervalTimer.value = null
  
  // 清理自动滚动定时器
  if (autoScrollTimer.value) {
    clearInterval(autoScrollTimer.value)
    autoScrollTimer.value = null
  }
  
  if (scene.value) {
    scene.value.traverse((object: Object3D) => {
      if ((object as any).material) {
        if (Array.isArray((object as any).material)) {
          (object as any).material.forEach((material: Material) => {
            material.dispose()
          })
        }
        else {
          (object as any).material.dispose()
        }
      }
      if ((object as any).geometry) {
        (object as any).geometry.dispose()
      }
      if ((object as any).texture) {
        (object as any).texture.dispose()
      }
    })
    scene.value.clear()
  }

  if (objects.value) {
    objects.value.forEach((object) => {
      if (object.element) {
        object.element.remove()
      }
    })
    objects.value = []
  }

  if (controls.value) {
    controls.value.removeEventListener('change')
    controls.value.dispose()
  }
  //   移除所有事件监听
  window.removeEventListener('resize', onWindowResize)
  scene.value = null
  camera.value = null
  renderer.value = null
  controls.value = null
}
// 重新初始化画布（不刷新页面）
function reinitCanvas() {
  // 清理现有的3D场景
  cleanup()
  
  // 清空targets
  targets.grid = []
  targets.helix = []
  targets.table = []
  targets.sphere = []
  
  // 重新初始化数据和场景
  tableData.value = []
  initTableData()
  
  if (tableData.value.length > 0) {
    init()
    randomBallData()
  }
  
  console.log('[Sync] Canvas reinitialized with new data')
}

// 同步数据 - 定期从服务器获取最新人员数据
async function syncDataFromServer(isFirstSync = false) {
  try {
    // 重新加载人员数据
    await personConfig.loadFromTheme()
    
    // 检查人员数量是否变化
    const currentCount = allPersonList.value.length
    
    // 如果是首次同步且有数据，或者数量发生变化，重新初始化画布
    if (isFirstSync) {
      // 首次同步：如果当前显示的数据和服务器数据不一致
      if (currentCount > 0 && tableData.value.length === 0) {
        console.log(`[Sync] First sync: found ${currentCount} persons, reinitializing canvas...`)
        reinitCanvas()
      }
    }
    else if (currentCount !== lastPersonCount.value) {
      // 后续同步：数量变化就重新初始化画布
      console.log(`[Sync] Person count changed: ${lastPersonCount.value} -> ${currentCount}, reinitializing canvas...`)
      reinitCanvas()
    }
    
    lastPersonCount.value = currentCount
  }
  catch (error) {
    console.error('[Sync] Failed to sync data:', error)
  }
}

// 启动数据同步
function startDataSync() {
  // 记录初始人员数量
  lastPersonCount.value = allPersonList.value.length
  
  // 首次同步（延迟1秒，等待页面初始化完成）
  setTimeout(() => {
    syncDataFromServer(true)
  }, 1000)
  
  // 每5秒同步一次数据
  syncTimer.value = setInterval(() => {
    syncDataFromServer(false)
  }, 5000)
}

// 停止数据同步
function stopDataSync() {
  if (syncTimer.value) {
    clearInterval(syncTimer.value)
    syncTimer.value = null
  }
}

onMounted(() => {
  // 移动端不初始化3D场景
  if (isMobile.value) {
    return
  }
  
  initTableData()
  init()
  animation()
  if (containerRef.value) {
    containerRef.value.style.color = `${textColor}`
  }
  randomBallData()
  window.addEventListener('keydown', listenKeyboard)
  
  // 启动数据同步
  startDataSync()
})
onUnmounted(() => {
  // 移动端不需要清理3D场景
  if (isMobile.value) {
    return
  }
  
  nextTick(() => {
    cleanup()
  })
  clearInterval(intervalTimer.value)
  intervalTimer.value = null
  
  // 停止数据同步
  stopDataSync()
  
  window.removeEventListener('keydown', listenKeyboard)
})
</script>

<template>
  <!-- 移动端显示专用UI -->
  <MobileJoinLottery v-if="isMobile" />
  
  <!-- PC端显示完整抽奖页面 -->
  <div v-else class="home-page" :class="{ 'has-bg-image': (homeBackground as any)?.url }">
    <!-- 自定义背景图片 -->
    <div v-if="(homeBackground as any)?.url" class="custom-bg-image">
      <img :src="(homeBackground as any)?.url" alt="background" />
    </div>
    
    <!-- 背景动画 - 只在没有背景图片时显示 -->
    <div class="bg-animation" v-if="!(homeBackground as any)?.url">
      <div class="stars"></div>
      <div class="stars2"></div>
      <div class="stars3"></div>
    </div>
    
  <!-- 顶部标题区域 -->
  <div class="header-area">
    <div class="title-container">
      <h2
        class="main-title header-title"
        :style="{ fontSize: `${textSize * 1.5}px` }"
      >
        {{ topTitle }}
      </h2>
      <div class="title-decoration"></div>
    </div>
    
    <!-- 空数据提示 -->
    <div v-if="tableData.length <= 0" class="empty-actions">
      <button class="action-btn primary" :style="{ fontSize: `${textSize * 0.5}px` }" @click="router.push('/config')">
        <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 16v1a3 3 0 003 3h10a3 3 0 003-3v-1m-4-8l-4-4m0 0L8 8m4-4v12" />
        </svg>
        {{ t('button.noInfoAndImport') }}
      </button>
      <button class="action-btn secondary" :style="{ fontSize: `${textSize * 0.5}px` }" @click="setDefaultPersonList">
        <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M17 20h5v-2a3 3 0 00-5.356-1.857M17 20H7m10 0v-2c0-.656-.126-1.283-.356-1.857M7 20H2v-2a3 3 0 015.356-1.857M7 20v-2c0-.656.126-1.283.356-1.857m0 0a5.002 5.002 0 019.288 0M15 7a3 3 0 11-6 0 3 3 0 016 0zm6 3a2 2 0 11-4 0 2 2 0 014 0zM7 10a2 2 0 11-4 0 2 2 0 014 0z" />
        </svg>
        {{ t('button.useDefault') }}
      </button>
    </div>
  </div>

  <!-- 3D 容器 -->
  <div id="container" ref="containerRef" class="lottery-container">
    <!-- 底部操作菜单 -->
    <div id="menu" class="menu-container">
      <!-- 状态 0: 进入抽奖 -->
      <button v-if="currentStatus === 0 && tableData.length > 0" class="lottery-btn enter-btn" :style="{ fontSize: `${textSize * 0.6}px` }" @click="enterLottery">
        <span class="btn-icon">🎰</span>
        <span class="btn-text">{{ t('button.enterLottery') }}</span>
        <span class="btn-glow"></span>
      </button>

      <!-- 状态 1: 开始抽奖 -->
      <div v-if="currentStatus === 1" class="start">
        <button class="lottery-btn start-btn" :style="{ fontSize: `${textSize * 0.6}px` }" @click="startLottery">
          <span class="btn-icon">🎲</span>
          <span class="btn-text">{{ t('button.start') }}</span>
          <span class="btn-particles"></span>
        </button>
        <p class="status-hint" :style="{ fontSize: `${textSize * 0.45}px` }">{{ t('button.start') }} - Space</p>
      </div>

      <!-- 状态 2: 抽取幸运儿 -->
      <div v-if="currentStatus === 2" class="rolling-state">
        <button class="lottery-btn stop-btn" :style="{ fontSize: `${textSize * 0.6}px` }" @click="stopLottery">
          <span class="btn-icon spinning">🎯</span>
          <span class="btn-text">{{ t('button.selectLucky') }}</span>
        </button>
        <p class="status-hint rolling" :style="{ fontSize: `${textSize * 0.45}px` }">Rolling...</p>
      </div>

      <!-- 状态 3: 继续/取消 -->
      <div v-if="currentStatus === 3" class="result-state">
        <!-- 分页指示器 -->
        <div v-if="luckyTargets.length > maxDisplayCount" class="page-indicator">
          <span class="page-info" :style="{ fontSize: `${textSize * 0.45}px` }">
            {{ t('lottery.showingPage') || '第' }} {{ displayPage + 1 }} / {{ Math.ceil(luckyTargets.length / maxDisplayCount) }} {{ t('lottery.page') || '页' }}
            ({{ t('lottery.total') || '共' }} {{ luckyTargets.length }} {{ t('lottery.winners') || '人中奖' }})
          </span>
          <div class="page-controls">
            <button 
              class="page-btn" 
              :style="{ fontSize: `${textSize * 0.4}px` }"
              :disabled="displayPage === 0"
              @click="prevPage"
            >
              ◀ {{ t('entry.prevPage') || '上一页' }}
            </button>
            <div class="page-dots">
              <span 
                v-for="i in Math.ceil(luckyTargets.length / maxDisplayCount)" 
                :key="i" 
                class="page-dot"
                :class="{ active: displayPage === i - 1 }"
                @click="goToPage(i - 1)"
              ></span>
            </div>
            <button 
              class="page-btn" 
              :style="{ fontSize: `${textSize * 0.4}px` }"
              :disabled="displayPage >= Math.ceil(luckyTargets.length / maxDisplayCount) - 1"
              @click="nextPage"
            >
              {{ t('entry.nextPage') || '下一页' }} ▶
            </button>
          </div>
        </div>
        <div class="result-actions">
          <button class="lottery-btn continue-btn" :style="{ fontSize: `${textSize * 0.6}px` }" @click="continueLottery">
            <span class="btn-icon">🎉</span>
            <span class="btn-text">{{ t('button.continue') }}</span>
          </button>
          <button class="lottery-btn cancel-btn" :style="{ fontSize: `${textSize * 0.6}px` }" @click="quitLottery">
            <span class="btn-icon">↩️</span>
            <span class="btn-text">{{ t('button.cancel') }}</span>
          </button>
        </div>
      </div>
    </div>
  </div>
  <StarsBackground :home-background="homeBackground" />
  <PrizeList class="absolute left-0 top-32" :text-size="textSize" />
  <JoinLottery :text-size="textSize" />
  
  <!-- 返回主题选择按钮 -->
  <div class="back-btn-wrapper">
    <button class="back-btn" :style="{ fontSize: `${textSize * 0.45}px` }" @click="router.push('/entry')" :title="t('entry.backToThemes')">
      <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M10 19l-7-7m0 0l7-7m-7 7h18" />
      </svg>
      <span class="back-text">{{ t('entry.backToThemes') }}</span>
    </button>
  </div>
  
  <!-- 分享按钮 -->
  <div class="share-btn-wrapper">
    <button class="share-btn" :style="{ fontSize: `${textSize * 0.45}px` }" @click="copyShareLink" :title="t('entry.shareLink')">
      <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M8.684 13.342C8.886 12.938 9 12.482 9 12c0-.482-.114-.938-.316-1.342m0 2.684a3 3 0 110-2.684m0 2.684l6.632 3.316m-6.632-6l6.632-3.316m0 0a3 3 0 105.367-2.684 3 3 0 00-5.367 2.684zm0 9.316a3 3 0 105.368 2.684 3 3 0 00-5.368-2.684z" />
      </svg>
      <span class="share-text">{{ t('entry.shareLink') }}</span>
    </button>
  </div>
  
  <!-- 二维码弹窗 -->
  <Teleport to="body">
    <div v-if="showQrModal" class="qr-modal-overlay" @click="closeQrModal">
      <div class="qr-modal" @click.stop>
        <button class="qr-close-btn" @click="closeQrModal">✕</button>
        <h3 class="qr-title" :style="{ fontSize: `${textSize * 0.65}px` }">{{ t('entry.scanToJoin') }}</h3>
        <div class="qr-code-wrapper">
          <img :src="qrCodeDataUrl" alt="QR Code" class="qr-code-img" />
        </div>
        <p class="qr-hint" :style="{ fontSize: `${textSize * 0.45}px` }">{{ t('entry.linkCopied') }}</p>
      </div>
    </div>
  </Teleport>
  </div>
</template>

<style scoped lang="scss">
// ==================== 页面容器和背景 ====================
.home-page {
  position: relative;
  width: 100vw;
  height: 100vh;
  overflow: hidden;
  background: linear-gradient(135deg, #1a1a2e 0%, #16213e 50%, #0f3460 100%);
  
  &.has-bg-image {
    background: transparent;
  }
}

// 自定义背景图片
.custom-bg-image {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  z-index: 0;
  
  img {
    width: 100%;
    height: 100%;
    object-fit: cover;
    opacity: 0.7;
  }
}

// 背景星星动画
.bg-animation {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
  z-index: 0;
}

.stars, .stars2, .stars3 {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: transparent;
}

.stars {
  background-image: radial-gradient(2px 2px at 20px 30px, #eee, transparent),
    radial-gradient(2px 2px at 40px 70px, rgba(255,255,255,0.8), transparent),
    radial-gradient(1px 1px at 90px 40px, #fff, transparent),
    radial-gradient(2px 2px at 160px 120px, rgba(255,255,255,0.9), transparent);
  background-size: 200px 200px;
  animation: twinkle 5s ease-in-out infinite;
}

.stars2 {
  background-image: radial-gradient(1px 1px at 50px 80px, #fff, transparent),
    radial-gradient(1px 1px at 100px 150px, rgba(255,255,255,0.7), transparent),
    radial-gradient(2px 2px at 180px 50px, #eee, transparent);
  background-size: 250px 250px;
  animation: twinkle 7s ease-in-out infinite;
  animation-delay: 1s;
}

.stars3 {
  background-image: radial-gradient(1px 1px at 30px 100px, rgba(255,255,255,0.6), transparent),
    radial-gradient(2px 2px at 120px 20px, #fff, transparent);
  background-size: 300px 300px;
  animation: twinkle 9s ease-in-out infinite;
  animation-delay: 2s;
}

@keyframes twinkle {
  0%, 100% { opacity: 0.5; }
  50% { opacity: 1; }
}

// ==================== 头部区域 ====================
.header-area {
  position: absolute;
  z-index: 10;
  left: 50%;
  transform: translateX(-50%);
  display: flex;
  flex-direction: column;
  align-items: center;
  padding-top: 40px;
}

.title-container {
  position: relative;
  text-align: center;
}

.main-title {
  margin: 0 0 8px;
  font-family: 'Arial', sans-serif;
  font-weight: 700;
  letter-spacing: 4px;
  background: linear-gradient(135deg, #fff 0%, #a78bfa 50%, #f472b6 100%);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
  text-shadow: none;
  filter: drop-shadow(0 4px 20px rgba(167, 139, 250, 0.5));
}

.title-decoration {
  width: 120px;
  height: 3px;
  margin: 0 auto;
  background: linear-gradient(90deg, transparent, #a78bfa, #f472b6, #a78bfa, transparent);
  border-radius: 2px;
  animation: shimmer 2s ease-in-out infinite;
}

@keyframes shimmer {
  0%, 100% { opacity: 0.5; transform: scaleX(0.8); }
  50% { opacity: 1; transform: scaleX(1); }
}

// 空数据操作按钮
.empty-actions {
  display: flex;
  gap: 16px;
  margin-top: 24px;
}

.action-btn {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 14px 28px;
  border: none;
  border-radius: 12px;
  font-size: 15px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s ease;
  
  &.primary {
    background: linear-gradient(135deg, #8b5cf6 0%, #6366f1 100%);
    color: white;
    box-shadow: 0 4px 20px rgba(139, 92, 246, 0.4);
    
    &:hover {
      transform: translateY(-3px);
      box-shadow: 0 8px 30px rgba(139, 92, 246, 0.5);
    }
  }
  
  &.secondary {
    background: rgba(255, 255, 255, 0.1);
    color: white;
    border: 1px solid rgba(255, 255, 255, 0.2);
    backdrop-filter: blur(10px);
    
    &:hover {
      background: rgba(255, 255, 255, 0.2);
      transform: translateY(-3px);
    }
  }
}

// ==================== 3D 容器 ====================
.lottery-container {
  position: relative;
  width: 100%;
  height: 100%;
}

// ==================== 底部菜单 ====================
.menu-container {
  position: absolute;
  z-index: 100;
  width: 100%;
  bottom: 60px;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 12px;
}

// 通用抽奖按钮样式
.lottery-btn {
  position: relative;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 12px;
  padding: 18px 48px;
  border: none;
  border-radius: 60px;
  font-size: 18px;
  font-weight: 700;
  cursor: pointer;
  overflow: hidden;
  transition: all 0.4s cubic-bezier(0.175, 0.885, 0.32, 1.275);
  
  .btn-icon {
    font-size: 24px;
    transition: transform 0.3s ease;
  }
  
  .btn-text {
    letter-spacing: 2px;
    text-transform: uppercase;
  }
  
  &:hover .btn-icon {
    transform: scale(1.2);
  }
}

// 进入抽奖按钮
.enter-btn {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 50%, #f093fb 100%);
  background-size: 200% 200%;
  color: white;
  box-shadow: 
    0 8px 32px rgba(102, 126, 234, 0.4),
    0 0 0 1px rgba(255, 255, 255, 0.1) inset;
  animation: gradient-flow 3s ease infinite, float 3s ease-in-out infinite;
  
  .btn-glow {
    position: absolute;
    top: 50%;
    left: 50%;
    width: 100%;
    height: 100%;
    background: radial-gradient(circle, rgba(255,255,255,0.3) 0%, transparent 70%);
    transform: translate(-50%, -50%);
    opacity: 0;
    transition: opacity 0.3s;
  }
  
  &:hover {
    transform: translateY(-5px) scale(1.05);
    box-shadow: 
      0 15px 40px rgba(102, 126, 234, 0.5),
      0 0 60px rgba(240, 147, 251, 0.3);
    
    .btn-glow {
      opacity: 1;
    }
  }
}

@keyframes gradient-flow {
  0% { background-position: 0% 50%; }
  50% { background-position: 100% 50%; }
  100% { background-position: 0% 50%; }
}

@keyframes float {
  0%, 100% { transform: translateY(0); }
  50% { transform: translateY(-8px); }
}

// 开始按钮
.start-btn {
  background: linear-gradient(135deg, #f093fb 0%, #f5576c 100%);
  color: white;
  box-shadow: 
    0 8px 32px rgba(245, 87, 108, 0.5),
    0 0 0 2px rgba(255, 255, 255, 0.2) inset;
  animation: pulse-glow 1.5s ease-in-out infinite;
  
  &:hover {
    transform: scale(1.1);
    box-shadow: 
      0 15px 50px rgba(245, 87, 108, 0.6),
      0 0 80px rgba(240, 147, 251, 0.4);
  }
}

@keyframes pulse-glow {
  0%, 100% { 
    box-shadow: 
      0 8px 32px rgba(245, 87, 108, 0.5),
      0 0 0 2px rgba(255, 255, 255, 0.2) inset;
  }
  50% { 
    box-shadow: 
      0 8px 50px rgba(245, 87, 108, 0.7),
      0 0 40px rgba(240, 147, 251, 0.5),
      0 0 0 2px rgba(255, 255, 255, 0.3) inset;
  }
}

// 停止按钮
.rolling-state {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 16px;
}

.stop-btn {
  background: linear-gradient(135deg, #fa709a 0%, #fee140 100%);
  color: #1a1a2e;
  box-shadow: 0 8px 32px rgba(250, 112, 154, 0.5);
  animation: shake 0.5s ease-in-out infinite;
  
  .spinning {
    animation: spin 1s linear infinite;
  }
  
  &:hover {
    transform: scale(1.1);
    animation: none;
  }
}

@keyframes shake {
  0%, 100% { transform: translateX(0); }
  25% { transform: translateX(-5px); }
  75% { transform: translateX(5px); }
}

@keyframes spin {
  from { transform: rotate(0deg); }
  to { transform: rotate(360deg); }
}

// 结果状态容器
.result-state {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 16px;
}

// 分页指示器
.page-indicator {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 8px;
  margin-bottom: 8px;
}

.page-info {
  color: rgba(255, 255, 255, 0.8);
  font-size: 14px;
  letter-spacing: 1px;
}

.page-controls {
  display: flex;
  align-items: center;
  gap: 16px;
}

.page-btn {
  padding: 8px 16px;
  background: rgba(255, 255, 255, 0.15);
  border: 1px solid rgba(255, 255, 255, 0.3);
  border-radius: 20px;
  color: white;
  font-size: 13px;
  cursor: pointer;
  transition: all 0.2s ease;
  
  &:hover:not(:disabled) {
    background: rgba(255, 255, 255, 0.25);
    transform: scale(1.05);
  }
  
  &:disabled {
    opacity: 0.4;
    cursor: not-allowed;
  }
}

.page-dots {
  display: flex;
  gap: 8px;
}

.page-dot {
  width: 10px;
  height: 10px;
  border-radius: 50%;
  background: rgba(255, 255, 255, 0.3);
  cursor: pointer;
  transition: all 0.2s ease;
  
  &:hover {
    background: rgba(255, 255, 255, 0.6);
  }
  
  &.active {
    background: linear-gradient(135deg, #f093fb 0%, #f5576c 100%);
    transform: scale(1.2);
  }
}

// 结果操作按钮
.result-actions {
  display: flex;
  gap: 24px;
}

.continue-btn {
  background: linear-gradient(135deg, #11998e 0%, #38ef7d 100%);
  color: white;
  box-shadow: 0 8px 32px rgba(56, 239, 125, 0.4);
  
  &:hover {
    transform: translateY(-5px) scale(1.05);
    box-shadow: 0 15px 40px rgba(56, 239, 125, 0.5);
  }
}

.cancel-btn {
  background: linear-gradient(135deg, #4a5568 0%, #2d3748 100%);
  color: white;
  box-shadow: 0 8px 32px rgba(45, 55, 72, 0.4);
  
  &:hover {
    transform: translateY(-5px) scale(1.05);
    box-shadow: 0 15px 40px rgba(45, 55, 72, 0.5);
  }
}

// 状态提示
.status-hint {
  color: rgba(255, 255, 255, 0.6);
  font-size: 14px;
  letter-spacing: 1px;
  margin: 0;
  
  &.rolling {
    color: #fee140;
    animation: blink 0.5s ease-in-out infinite;
  }
}

@keyframes blink {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.5; }
}

// ==================== 右侧浮动按钮 ====================
.back-btn-wrapper {
  position: fixed;
  right: 24px;
  bottom: 250px;
  z-index: 1000;
}

.back-btn {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 14px 24px;
  background: linear-gradient(135deg, #6366f1 0%, #8b5cf6 100%);
  border: none;
  border-radius: 50px;
  color: white;
  font-size: 14px;
  font-weight: 600;
  cursor: pointer;
  box-shadow: 
    0 4px 20px rgba(99, 102, 241, 0.4),
    0 0 0 1px rgba(255, 255, 255, 0.1) inset;
  transition: all 0.3s ease;
  
  &:hover {
    transform: translateY(-4px) scale(1.05);
    box-shadow: 
      0 10px 30px rgba(99, 102, 241, 0.5),
      0 0 0 1px rgba(255, 255, 255, 0.2) inset;
  }
  
  &:active {
    transform: translateY(-2px) scale(1.02);
  }
}

.back-text {
  white-space: nowrap;
}

.share-btn-wrapper {
  position: fixed;
  right: 24px;
  bottom: 190px;
  z-index: 1000;
}

.share-btn {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 14px 24px;
  background: linear-gradient(135deg, #10b981 0%, #059669 100%);
  border: none;
  border-radius: 50px;
  color: white;
  font-size: 14px;
  font-weight: 600;
  cursor: pointer;
  box-shadow: 
    0 4px 20px rgba(16, 185, 129, 0.4),
    0 0 0 1px rgba(255, 255, 255, 0.1) inset;
  transition: all 0.3s ease;
  
  &:hover {
    transform: translateY(-4px) scale(1.05);
    box-shadow: 
      0 10px 30px rgba(16, 185, 129, 0.5),
      0 0 0 1px rgba(255, 255, 255, 0.2) inset;
  }
  
  &:active {
    transform: translateY(-2px) scale(1.02);
  }
}

.share-text {
  white-space: nowrap;
}

// ==================== 标题动画 ====================
.header-title {
    -webkit-animation: tracking-in-expand-fwd 0.8s cubic-bezier(0.215, 0.610, 0.355, 1.000) both;
    animation: tracking-in-expand-fwd 0.8s cubic-bezier(0.215, 0.610, 0.355, 1.000) both;
}

.start {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 12px;
}

@-webkit-keyframes tracking-in-expand-fwd {
    0% {
        letter-spacing: -0.5em;
        -webkit-transform: translateZ(-700px);
        transform: translateZ(-700px);
        opacity: 0;
    }

    40% {
        opacity: 0.6;
    }

    100% {
        -webkit-transform: translateZ(0);
        transform: translateZ(0);
        opacity: 1;
    }
}

@keyframes tracking-in-expand-fwd {
    0% {
        letter-spacing: -0.5em;
        -webkit-transform: translateZ(-700px);
        transform: translateZ(-700px);
        opacity: 0;
    }

    40% {
        opacity: 0.6;
    }

    100% {
        -webkit-transform: translateZ(0);
        transform: translateZ(0);
        opacity: 1;
    }
}

// ==================== 二维码弹窗 ====================
.qr-modal-overlay {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.7);
  backdrop-filter: blur(4px);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 9999;
  animation: fadeIn 0.2s ease;
}

.qr-modal {
  position: relative;
  background: linear-gradient(145deg, #1e1e3f, #2a2a5a);
  border-radius: 20px;
  padding: 32px;
  text-align: center;
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.5);
  border: 1px solid rgba(255, 255, 255, 0.1);
  animation: scaleIn 0.3s ease;
}

.qr-close-btn {
  position: absolute;
  top: 12px;
  right: 12px;
  width: 32px;
  height: 32px;
  border: none;
  background: rgba(255, 255, 255, 0.1);
  color: white;
  border-radius: 50%;
  cursor: pointer;
  font-size: 16px;
  transition: all 0.2s;
  
  &:hover {
    background: rgba(255, 255, 255, 0.2);
    transform: scale(1.1);
  }
}

.qr-title {
  margin: 0 0 20px;
  font-size: 20px;
  font-weight: 600;
  color: white;
  letter-spacing: 1px;
}

.qr-code-wrapper {
  background: white;
  padding: 16px;
  border-radius: 12px;
  display: inline-block;
}

.qr-code-img {
  display: block;
  width: 200px;
  height: 200px;
}

.qr-hint {
  margin: 16px 0 0;
  color: #10b981;
  font-size: 14px;
}

@keyframes fadeIn {
  from { opacity: 0; }
  to { opacity: 1; }
}

@keyframes scaleIn {
  from { 
    opacity: 0;
    transform: scale(0.9);
  }
  to { 
    opacity: 1;
    transform: scale(1);
  }
}
</style>
