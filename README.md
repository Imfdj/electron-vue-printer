<h1 align="center">
   <img alt="Logo" width="166" src="https://cdn.pixabay.com/photo/2017/06/10/07/29/printer-2389244_960_720.png"/>
</h1>
<h1 align="center">
   electron-vue-printer
</h1>

<h3 align="center">
    electron-vue 打印图片插件。使用非常简单，可静默打印图片。
</h3>
<h3 align="center">
 	Simple and easy silent print pictures for electron-vue plugin.
</h3>

 <p align="center">
     <a href="https://choosealicense.com/licenses/mit/"><img
         alt="License MIT"
         src="https://img.shields.io/badge/licence-MIT-blue.svg?style=flat-square"></a>
     <img alt="No dependencies"
         src="https://img.shields.io/badge/dependencies-none-3387e0.svg?style=flat-square">
     <img alt="Current version"
             src="https://img.shields.io/badge/build-passing-brightgreen">
     <img alt="Current version"
             src="https://img.shields.io/badge/version-1.0.0-brightgreen">
 </p>

### Features
* Silent print
* Ultra small
* Simple usage
* No other dependencies

#### Install

``` bash
npm install electron-vue-printer -S
or
yarn add electron-vue-printer -S
```
#### Usage

``` bash
index.vue -->
<template>
  <div class="index">
    <electron-vue-printer
      ref="electronVuePrinter"
      :silent="false"
      @webview-ready="webviewReady"
    ></electron-vue-printer>
  </div>
</template>

<script>
  import electronVuePrinter from 'electron-vue-printer'

  export default {
    name: "index",
    components: {
      electronVuePrinter
    },
    methods: {
      webviewReady() {
        this.$refs.electronVuePrinter.print('https://cdn.pixabay.com/photo/2017/06/10/07/29/printer-2389244_960_720.png');
      },
    },
  }
</script>
```
| Configuration      | Description
| -------------- | ----------- |
| `options`  | `options.offsetX: number. Left the offset,default 0.` |
|            | `options.offsetY: number. Top the offset,default 0.` |
|            | `options.imageWidth: number. Width of picture.` |
|            | `options.imageHeight: number. Height of picture.` |
| `rotateType`  | `Image rotation direction. 0: normal;1: Clockwise;2: Counterclockwise;3: Upside down;default 0.` |
| `imgStyle`  | `Customize image styles.Width and height can override options.imageWidth and options.imageHeight.Transform will cause rotateType to become invalid;.` |
| `show`  | `Whether to display preview.default: false.` |
| `silent`  | `Whether to enable silent printing.default: false.` |


| Events            | Description
| -------------- | ----------- |
| `dom-ready(webview)`  | `The webview for the user to render the printed content is ready,And return the dom of the webview.` |
| `get-printer-list(data: Array)`  | `Return printer list data.Maybe setPrinterName method can use it.` |
| `webview-render-begin`  | `Print content starts to render.` |
| `webview-render-finish`  | `The printed content is rendered.` |
| `start-printing(state: Boolean)`  | `Start printing, and return whether to start printing normally.` |

### Methods
* print(`[src,base64]:String`) _- Set print picture src(The absolute path of the local picture or the address of the online picture) or base64 to trigger print automatically._
* setPrinterName(`name:String, callBack(err)`) _- Set the printer device name.If you don't want to use the default printer, you can use it.If err in callBack is not empty, it means the setting failed._

## License

[MIT](LICENSE)
