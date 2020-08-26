<template>
  <div v-show="show">
    <webview
      ref="printWebview"
      style="width: 100vw;height: 100vh;overflow: hidden;"
      :src="fullPath" nodeintegration/>
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
  import { remote } from 'electron';

  export default {
    name: "ElectronImagePrinter",
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
        inited: false,
        // fullPath: process.env.NODE_ENV === 'production' ? path.resolve(__dirname, './node_modules/electron-vue-printer/render.html') : path.join(__dirname, 'render.html'),
        fullPath: '',
        deviceNamePrint: '',
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
      this.getPrinterList();
    },
    mounted() {
      // this.webviewInit()
    },
    methods: {
      /**
       * 初始化webview
       */
      webviewInit (version) {
        this.inited = true
        return new Promise((resolve) => {
          const webview = this.$refs.printWebview
          webview.addEventListener('dom-ready', () => {
            // webview.openDevTools()
            resolve()
            this.$emit('webview-ready', webview)
          })
          webview.addEventListener('ipc-message', (event) => {
            if (event.channel === 'webview-render-finish') {
              this.$emit('webview-render-finish')
              // 如果electron版本号为v6及以下，则可以使用回调参数
              if (/^[1-6]{1}\./.test(version)) {
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
            this.deviceNamePrint = v.name;
          }
        });
      },
      /**
       * 设置打印图片src,自动触发打印
       * @param srcData
       */
      print(srcData, version, NODE_ENV) {
        if(this.inited) {
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
        }else {
          if(NODE_ENV === 'production') {
            this.fullPath = path.resolve(__dirname, './resources/app/node_modules/electron-vue-printer/render.html')
          }else {
            this.fullPath = path.resolve(__dirname, './node_modules/electron-vue-printer/render.html')
          }
          this.$nextTick(() => {
            this.webviewInit(version).then(() => {
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
            })
          })
        }
      },
      /**
       * 设置打印机对象
       * @param name callBack
       */
      setPrinterName(name, callBack) {
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

