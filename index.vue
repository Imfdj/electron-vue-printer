<template>
  <div v-show="show">
    <webview
      ref="printWebview"
      style="width: 100vw;height: 100vh;overflow: hidden;"
      :src="webviewRenderPath" nodeintegration/>
    <div class="wrap-content" v-show="false">
      <div ref="wrapContentRef" :style="`transform: translate(${options.offsetX}px, ${options.offsetY}px);`">
        <img
          ref="imgRef"
          id="imgId"
          :src="srcData"
          :width="options.imageWidth"
          :height="options.imageHeight"
          :style="`${rotateTypeData[rotateType]}${imgStyle}`"
        >
      </div>
    </div>
  </div>
</template>

<script>
  import path from 'path'
  import fs from 'fs'
  import { remote } from 'electron'
  const packageJosn = require('./package.json')

  export default {
    name: 'ElectronImagePrinter',
    props: {
      options: {
        type: Object,
        required: false,
        default: function () {
          return {
            offsetX: 0, // 左偏移
            offsetY: 0, // 上偏移
            imageWidth: -1, // 图片宽度
            imageHeight: -1, // 图片高度
          }
        },
        validator: function (value) {
          return Object.values(value).every((e) => typeof e === 'number')
        }
      },
      // 图片旋转方向 0：正常；1：顺时针；2：逆时针；3：上下颠倒
      rotateType: {
        type: Number,
        required: false,
        default: 0,
        validator: function (value) {
          return [0, 1, 2, 3].includes(value)
        }
      },
      // 自定义图片样式,权重高于options中的imageWidth和imageHeight
      imgStyle: {
        type: String,
        required: false,
        default: '',
      },
      show: {
        type: Boolean,
        required: false,
        default: false,
      },
      silent: {
        type: Boolean,
        required: false,
        default: false,
      },
    },
    data() {
      return {
        webviewRenderPath: '',
        deviceNamePrint: '',
        electronVersion: process.versions.electron,
        srcData: '',
        printerList: [],
        rotateTypeData: [
          '',
          'transform: rotate(90deg) translate(0%,-100%);transform-origin: left top;',
          'transform: rotate(-90deg) translate(-100%,0%);transform-origin: left top;',
          'transform: rotate(180deg);',
        ],
      }
    },
    created() {
      this.getPrinterList()
      this.initWebviewRenderPath()
    },
    mounted() {
      this.webviewInit()
    },
    methods: {
      /**
       * 初始化webview src render html文件路径
       */
      initWebviewRenderPath() {
        const devPath = path.resolve(__dirname, `./node_modules/${packageJosn.name}/electron-vue-printer-render.html`)
        const proNoAsarPath = path.resolve(__dirname, `./resources/app/node_modules/${packageJosn.name}/electron-vue-printer-render.html`)
        const rootPath = path.resolve(__dirname, 'resources/electron-vue-printer-render.html')
        if (fs.existsSync(devPath)) {
          this.webviewRenderPath = devPath
        } else if (fs.existsSync(proNoAsarPath)) {
          this.webviewRenderPath = proNoAsarPath
        } else {
          if (!fs.existsSync(rootPath)) {
            fs.writeFileSync(rootPath, require('./electron-vue-printer-render.html'))
          }
          this.webviewRenderPath = rootPath
        }
      },

      /**
       * 初始化webview
       */
      webviewInit() {
        if (/^4\./.test(this.electronVersion)) {
          throw new Error('Does not support electron v4')
        }
        const webview = this.$refs.printWebview
        webview.addEventListener('dom-ready', () => {
          // webview.openDevTools()
          this.$emit('webview-ready', webview)
        })
        webview.addEventListener('ipc-message', (event) => {
          if (event.channel === 'webview-render-finish') {
            this.$emit('webview-render-finish')
            // 如果electron版本号为v6及以下，则可以使用回调参数
            if (/^[1-4]{1}\./.test(this.electronVersion)) {
              webview.print(
                {
                  silent: this.silent,
                  printBackground: true,
                  deviceName: this.deviceNamePrint
                },
                (state) => {
                  this.$emit('start-printing', state)
                },
              )
            } else if (/^5\./.test(this.electronVersion)) {
              webview.print(
                {
                  silent: this.silent,
                  printBackground: true,
                  deviceName: ''
                },
                (state) => {
                  this.$emit('start-printing', state)
                },
              )
            } else if (/^6\./.test(this.electronVersion)) {
              webview.print(
                {
                  silent: this.silent,
                  printBackground: true,
                  deviceName: ''
                },
              )
              this.$emit('start-printing', true)
            } else {
              webview.print(
                {
                  silent: this.silent,
                  printBackground: true,
                  deviceName: this.deviceNamePrint
                },
              ).then(() => {
                this.$emit('start-printing', true)
              }).catch((err) => {
                this.$emit('start-printing', false)
                throw err
              })
            }
          }
        })
      },

      /**
       * 获取打印机数据
       */
      getPrinterList() {
        // 获取打印机列表
        this.printerList = remote.getCurrentWindow().webContents.getPrinters()
        this.$emit('get-printer-list', this.printerList)
        this.printerList.forEach((v) => {
          if (v.isDefault) {
            this.deviceNamePrint = v.name
          }
        })
      },

      /**
       * 设置打印图片src,自动触发打印
       * @param srcData
       */
      print(srcData) {
        this.srcData = srcData
        this.$nextTick(() => {
          if (this.$refs.imgRef.complete) {
            this.$refs.printWebview.send('webview-render', this.$refs.wrapContentRef.outerHTML)
            this.$emit('webview-render-begin')
          } else {
            this.$refs.imgRef.onload = () => {
              this.$refs.printWebview.send('webview-render', this.$refs.wrapContentRef.outerHTML)
            }
          }
        })
      },

      /**
       * 设置打印机对象
       * @param name callBack
       */
      setPrinterName(name, callBack) {
        if (/^[5-6]{1}\./.test(this.electronVersion)) {
          throw new Error('Electron v5 or v6 does not support setting the printer name.Only use the default name')
        }
        const names = this.printerList.map((e) => {
          return e.name
        })
        if (names.includes(name)) {
          callBack()
          this.deviceNamePrint = name
        } else {
          callBack(new Error('inexistent printer'))
          throw new Error('inexistent printer')
        }
      }
    },
  }
</script>

