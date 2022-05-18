<template>
  <div id="tap" class="tap"></div>
</template>

<script setup>
import { onMounted } from 'vue'
import gsap from 'gsap'
import * as PIXI from 'pixi.js'
import 'pixi-sound'
import '@pixi/graphics-extras'
import { colors } from '@/assets/tap/colors'
import { audios } from '@/assets/tap/audios'

window.PIXI = PIXI
require('pixi-layers')

const animations = []

let uiGroup = null
let tapGroup = null
let animationGroup = null
let bgGroup = null
let uiLayer = null
let tapLayer = null
let animationLayer = null
let bgLayer = null

let taps = []
let bg = null
let colorIndex = 0
let nowPlay = ''
let isPlaying = false
let isTouch = false
let layout = 0
let bgmRes = null
// let isBgm = true

const app = new PIXI.Application({
  autoDensity: true,
  resolution: devicePixelRatio
})
let wrapper = null

// 格子行列数
let row = 4
let col = 8

const init = () => {
  wrapper.appendChild(app.view)

  app.stage = new PIXI.display.Stage()
  app.stage.sortableChildren = true

  uiGroup = new PIXI.display.Group(3, false)
  tapGroup = new PIXI.display.Group(2, false)
  animationGroup = new PIXI.display.Group(1, false)
  bgGroup = new PIXI.display.Group(-1, false)

  uiLayer = new PIXI.display.Layer(uiGroup)
  tapLayer = new PIXI.display.Layer(tapGroup)
  animationLayer = new PIXI.display.Layer(animationGroup)
  bgLayer = new PIXI.display.Layer(bgGroup)

  app.stage.addChild(uiLayer)
  app.stage.addChild(tapLayer)
  app.stage.addChild(animationLayer)
  app.stage.addChild(bgLayer)

  wrapper.addEventListener('pointerdown', tapStart)
  wrapper.addEventListener('pointermove', tapMove)
  wrapper.addEventListener('pointerup', tapEnd)
}

const tapStart = (e) => {
  isTouch = true
  click(e)
}
const tapMove = (e) => {
  click(e)
}
const tapEnd = () => {
  isTouch = false
  nowPlay = ''
}

const playSound = (id) => {
  isPlaying = true
  PIXI.Loader.shared.resources[id].sound.play()
  setTimeout(() => {
    isPlaying = false
  }, 150)
}

const click = (e) => {
  if (!isTouch) return
  e.stopPropagation()

  const c = Math.floor(e.offsetX / (window.innerWidth / col))
  const r = Math.floor(e.offsetY / (window.innerHeight / row))

  const id = 'sound' + (r * col + c + 1)
  if (nowPlay === id) return
  nowPlay = id

  gsap.to(taps[r][c], 0.05, { alpha: 1 })
  gsap.to(taps[r][c], 0.6, { delay: 0.05, alpha: 0 })

  if (!isPlaying) {
    playSound(id)
  }

  drawBackground()

  colorIndex++
  if (colorIndex > colors.length - 1) colorIndex = colorIndex % colors.length
  if (animations.length > 0) {
    animations[(r + c) % animations.length]({
      app: app,
      group: animationGroup,
      colors: colors,
      colorIndex: colorIndex
    })
  }
}

const resize = () => {
  app.renderer.resize(
    wrapper.clientWidth,
    wrapper.clientHeight
  )

  app.renderer.view.style.width = wrapper.clientWidth + 'px'
  app.renderer.view.style.height = wrapper.clientHeight + 'px'

  if (wrapper.clientWidth < wrapper.clientHeight) {
    layout = 0
  } else {
    layout = 1
  }

  if (bgmRes) {
    initBackground()
    for (const r of taps) {
      for (const tapper of r) {
        app.stage.removeChild(tapper)
      }
    }
    taps = []
    initTaps()
  }
}

const loading = () => {
  let loaded = 0
  const totalResourcesNum = Object.keys(audios).length + 1
  const step = wrapper.clientWidth / totalResourcesNum
  const progressBar = new PIXI.Graphics()
    .beginFill(0xffffff, 1)
    .drawRect(
      -wrapper.clientWidth,
      wrapper.clientHeight / 2,
      wrapper.clientWidth,
      10)

  const style = new PIXI.TextStyle({
    fontWeight: 'lighter',
    fill: '#fff',
    fontSize: 24
  })
  const text = new PIXI.Text('Loading', style)
  text.x = wrapper.clientWidth / 2 - 40
  text.y = wrapper.clientHeight / 2 - 40

  gsap.to([text, progressBar], 0.5, {
    alpha: 0.4,
    yoyo: true,
    repeat: -1,
    delay: 0.4
  })

  progressBar.parentGroup = uiGroup
  text.parentGroup = uiGroup

  app.stage.addChild(progressBar)
  app.stage.addChild(text)

  PIXI.Loader.shared.onProgress.add(async () => {
    loaded++
    const length = step * loaded
    gsap.to(progressBar, 0.5, {
      x: length
    })

    if (loaded === totalResourcesNum) {
      gsap.to([progressBar, text], 0.5, {
        alpha: 0,
        onComplete: () => {
          app.stage.removeChild(progressBar)
          app.stage.removeChild(text)
        }
      })
    }
  })
}

const initBackground = () => {
  if (bg) {
    app.stage.removeChild(bg)
  }

  bg = new PIXI.Graphics()
    .beginFill(colors[0], 1)
    .drawRegularPolygon(0, 0, 2 * Math.max(app.screen.width, app.screen.height), 4, 0)

  bg.displayGroup = bgGroup
  app.stage.addChild(bg)
}

const initTaps = () => {
  taps = []

  if (layout === 0) {
    const temp = row
    row = col
    col = temp
  }

  const width = wrapper.clientWidth / col
  const height = wrapper.clientHeight / row

  for (let r = 0; r < row; r++) {
    taps[r] = []
    for (let c = 0; c < col; c++) {
      const tapper = new PIXI.Graphics()
        .beginFill(0xffffff, 1)
        .drawRect(width * c, height * r, width, height)
      gsap.to(tapper, 0, { alpha: 0 })
      tapper.parentGroup = tapGroup
      tapper.interactive = true
      tapper.buttonMode = true
      tapper.hitArea = new PIXI.Rectangle(width * c, height * r, width, height)

      taps[r].push(tapper)
      app.stage.addChild(tapper)
    }
  }
}

const drawBackground = () => {
  const seed = Math.floor(Math.random() * colors.length)

  if (Math.floor(Math.random() * 10) !== 0) return
  const heading = Math.random()
  const radius = Math.max(
    wrapper.clientWidth,
    wrapper.clientHeight
  )
  const sides = 4
  const rotate = Math.random() * 360
  const x = heading >= 0.5 ? -2 * radius : 3 * radius
  const y = heading >= 0.5 ? -2 * radius : 3 * radius

  const bg = new PIXI.Graphics()
    .beginFill(colors[seed], 1)
    .drawRegularPolygon(x, y, radius * 2, sides, rotate)

  bg.parentGroup = animationGroup

  gsap.to(bg, 1, {
    duration: 1,
    x: heading >= 0.5 ? 2 * radius : -2 * radius,
    y: heading >= 0.5 ? 2 * radius : -2 * radius,
    onComplete: () => {
      bg
        .beginFill(colors[seed], 1)
        .drawRegularPolygon(0, 0, 2 * Math.max(app.screen.width, app.screen.height), 4, 0)
      app.stage.removeChild(bg)
    }
  })

  app.stage.addChild(bg)
}

const initAudios = async () => {
  PIXI.Loader.shared.add('bgm', 'https://fastly.jsdelivr.net/gh/blacktunes/hiironya@master/src/assets/tap/audios/%E8%96%84%E7%BE%A4%E9%9D%92.mp3')
  for (const name in audios) {
    PIXI.Loader.shared.add(name, audios[name])
  }

  PIXI.Loader.shared.load((loader, resources) => {
    resources.bgm.sound.loop = true
    resources.bgm.sound.volume = 0.5

    resources.bgm.sound.play()
    bgmRes = resources.bgm
    initTaps()
  })
}

onMounted(() => {
  wrapper = document.getElementById('tap')

  init()
  resize()
  window.addEventListener('resize', resize)
  loading()
  initAudios()
  initBackground()
  document.onvisibilitychange = () => {
    if (bgmRes) {
      if (document.visibilityState === 'hidden') {
        bgmRes.sound.stop()
      } else {
        bgmRes.sound.play()
      }
    }
  }
})

</script>

<style scoped lang="stylus">
.tap
  height 100vh
  width 100vw
</style>
