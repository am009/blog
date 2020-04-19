---
title: 树莓派electron-ssr
date: 2019/12/7 20:46:25
categories:
- 树莓派
tags:
- raspberrypi
- raspbian
---


# 树莓派electron-ssr

<!-- more -->


## 第一部分 electron-ssr代码解读

本来这篇文章只有第二部分的，无奈我忘了把我编译好的electron-ssr带回来，只好假装自己想学习ssr的使用，好在这一部分也非常简单。
https://github.com/ssrarchive/shadowsocks-rss/wiki/Python-client-setup-(Mult-language)
```
python local.py -s server_ip -p 443 -k password -m aes-256-cfb -o http_simple -O auth_chain_a
#说明：-p 端口 -k 密码  -m 加密方式 -o 混淆插件 -O 协议插件
```
而查看electron-ssr源码，其中最关心的部分便是调用python版ssr的部分，(只要看看electron-ssr日志就知道它只是执行简单的命令调用)已经关于订阅的部分。
```
export async function run (appConfig) {
  const listenHost = appConfig.shareOverLan ? '0.0.0.0' : '127.0.0.1'
  // 先结束之前的
  await stop()
  try {
    await isHostPortValid(listenHost, appConfig.localPort || 1080)
  } catch (e) {
    logger.error(e)
    dialog.showMessageBox({
      type: 'warning',
      title: '警告',
      message: `端口 ${appConfig.localPort} 被占用`
    })
  }
  const config = appConfig.configs[appConfig.index]
  // 参数
  const params = [path.join(appConfig.ssrPath, 'local.py')]
  params.push('-s')
  params.push(config.server)
  params.push('-p')
  params.push(config.server_port)
  params.push('-k')
  params.push(config.password)
  params.push('-m')
  params.push(config.method)
  params.push('-O')
  params.push(config.protocol)
  if (config.protocolparam) {
    params.push('-G')
    params.push(config.protocolparam)
  }
  if (config.obfs) {
    params.push('-o')
    params.push(config.obfs)
  }
  if (config.obfsparam) {
    params.push('-g')
    params.push(config.obfsparam)
  }
  params.push('-b')
  params.push(listenHost)
  params.push('-l')
  params.push(appConfig.localPort || 1080)
  if (config.timeout) {
    params.push('-t')
    params.push(config.timeout)
  }
  runCommand('python', params)
}
```

然而我喜欢用python，如果我用nodejs，这几行直接复制出来就可以组成一个命令行界面去调用了。
其实我也算是熟悉过nodejs，稍微学过一点。。。但是既然ssr都是python版起家的，我为什么不写个订阅处理完善一下呢。
好吧我想多了，在不断寻找下找到了https://github.com/noahziheng/ssr-helper
这个就是我想要的，啥也不说了，nodejs走起
另外注意，想要监听所有的地址，把127.0.0.1改成0.0.0.0，但是我发现想要监听ipv6就要改成监听::,这样修改后ipv4和ipv6都可以监听了！

----
## 第二部分 build树莓派上的electron-ssr

https://github.com/shadowsocksrr/electron-ssr
https://github.com/qingshuisiyuan/electron-ssr-backup/blob/master/issue.md
下载源码

树莓派
apt install nodejs npm
然后加上sudo安装electron
解压源码，sudo npm install下载半天
出现的问题：

目测使用了webpack加上electron-builder
Build error: Error: Unsupported arch arm

安装库sudo apt-get install libgconf-2-4
听说：
```
https://github.com/electron-userland/electron-builder/issues/831
./node_modules/.bin/build --help

Use --armv7l to build for ARM.
```
就使用了./node_modules/.bin/build --armv7
这样没有生效，后面报错
Error: Application entry file "dist/electron/main.js" in the "/home/pi/electron-ssr-0.2.7/dist/linux-armv7l-unpacked/resources/app.asar" does not exist. Seems like a wrong configuration.
所以还需要研究

看来原理首先是目录下的package.json，里面的script是npm run build之后的运行的命令
这个命令调用的是.electron-vue这个隐藏文件夹里面的命令。
这个隐藏文件夹才是关键！！！
里面的release文件夹就是build时传的命令，看来传的是json
```
  return builder.build({
    targets: targets,
    config: {
      productName: 'electron-ssr',
      appId: 'me.erguotou.ssr',
      artifactName: '${productName}-${version}.${ext}',
      compression: 'maximum',
      copyright: 'erguotou525@gmail.com',
      files,
      extraFiles: extraFiles,
      directories: {
        output: 'build'
      },
      publish: {
        provider: 'github'
      },
      dmg: {
        contents: [
          {
            x: 410,
            y: 150,
            type: 'link',
            path: '/Applications'
          },
          {
            x: 130,
            y: 150,
            type: 'file'
          }
        ]
      },
      mac: {
        icon: 'build/icons/icon.icns',
        category: 'public.app-category.developer-tools',
        target: [
          'zip',
          'dmg'
        ],
        extendInfo: {
          LSUIElement: 'YES'
        }
      },
      win: {
        icon: 'build/icons/icon.ico',
        target: [
          {
            target: 'nsis',
            arch: ['ia32']
          }
        ]
      },
      nsis: {
        license: 'LICENSE',
        oneClick: false,
        perMachine: true,
        allowToChangeInstallationDirectory: true
      },
      linux: {
        icon: 'build/icons',
        category: 'Development',
        synopsis: pkg.description,
        target: [
          'deb',
          'rpm',
          'tar.gz',
          'pacman',
          'appImage'
        ],
        desktop: {
          Name: 'electron-ssr',
          Encoding: 'UTF-8',
          Type: 'Application',
          Comment: pkg.description,
          StartupWMClass: 'electron-ssr'
        }
      }
      // appImage: {
      //   license: 'LICENSE'
      // }
    }
  }).then(() => {
    console.log(`${BLUE}Done${END}`)
  }).catch(error => {
    console.error(`${YELLOW}Build error: ${error}${END}`)
  })
}

```
所有在这里想办法加上--armv7l参数
真的是这样吗

https://github.com/qingshuisiyuan/electron-ssr-backup/blob/master/issue.md
sudo apt install libsodium libsodium-dev
看到可能要安装，就先安装了
然后不断看这个文档
https://www.electron.build/cli#targetconfiguration
修改后的完整配置文件：
```
const builder = require('electron-builder')
const os = require('os')
const pkg = require('../package.json')

const platform = os.platform()
const Platform = builder.Platform
const YELLOW = '\x1b[33m'
const BLUE = '\x1b[34m'
const END = '\x1b[0m'

let targets
const extraFiles = []

function release () {
  let files = [
    'dist/electron/**/*',
    '!dist/electron/imgs/ionicons--fonts.svg',
    '!dist/electron/fonts/ionicons--fonts.eot',
    '!dist/electron/fonts/ionicons--fonts.ttf',
    '!dist/electron/static/plane.svg',
    '!node_modules/{babel-runtime,batch-processor,core-js,deepmerge,element-resize-detector,erguotou-iview,mousetrap,rxjs,popper.js,qr-image,vue*}${/*}',
    '!node_modules/unbzip2-stream/dist${/*}',
    'node_modules/mousetrap/{mousetrap.js,package.json}',
    '!**/*.{md,markdown,MD,txt}',
    '!**/{test.js,license,LICENSE,.jscsrc}',
    '!**/sample?(s)${/*}'
  ]
  const macImages = [
    '!dist/electron/static/enabled@(Template|Highlight)?(@2x).png',
    '!dist/electron/static/pac@(Template|Highlight)?(@2x).png',
    '!dist/electron/static/global@(Template|Highlight)?(@2x).png'
  ]
  const winImages = [
    '!dist/electron/static/enabled?(@2x).png',
    '!dist/electron/static/pac?(@2x).png',
    '!dist/electron/static/global?(@2x).png'
  ]
  switch (platform) {
    case 'darwin':
      targets = Platform.MAC.createTarget()
      extraFiles.push({ from: 'src/lib/proxy_conf_helper', to: './' })
      files = files.concat(winImages)
      break
    case 'win32':
      targets = Platform.WINDOWS.createTarget()
      extraFiles.push({ from: 'src/lib/sysproxy.exe', to: './' })
      files = files.concat(macImages)
      break
    case 'linux':
      targets = Platform.LINUX.createTarget()
      files = files.concat(macImages)
  }
  return builder.build({
    targets: targets,
    config: {
      productName: 'electron-ssr',
      appId: 'me.erguotou.ssr',
      artifactName: '${productName}-${version}.${ext}',
      compression: 'maximum',
      copyright: 'erguotou525@gmail.com',
      files,
      extraFiles: extraFiles,
      directories: {
        output: 'build'
      },
      publish: {
        provider: 'github'
      },
      dmg: {
        contents: [
          {
            x: 410,
            y: 150,
            type: 'link',
            path: '/Applications'
          },
          {
            x: 130,
            y: 150,
            type: 'file'
          }
        ]
      },
      mac: {
        icon: 'build/icons/icon.icns',
        category: 'public.app-category.developer-tools',
        target: [
          'zip',
          'dmg'
        ],
        extendInfo: {
          LSUIElement: 'YES'
        }
      },
      win: {
        icon: 'build/icons/icon.ico',
        target: [
          {
            target: 'nsis',
            arch: ['ia32']
          }
        ]
      },
      nsis: {
        license: 'LICENSE',
        oneClick: false,
        perMachine: true,
        allowToChangeInstallationDirectory: true
      },
      linux: {
        icon: 'build/icons',
        category: 'Development',
        synopsis: pkg.description,
        target: [
          {
            target: 'deb',
            arch: ['armv7l']
          },
          {
            target: 'rpm',
            arch: ['armv7l']
          },
          {
            target: 'tar.gz',
            arch: ['armv7l']
          },
          {
            target: 'pacman',
            arch: ['armv7l']
          },
          {
            target: 'appImage',
            arch: ['armv7l']
          }
        ],
        desktop: {
          Name: 'electron-ssr',
          Encoding: 'UTF-8',
          Type: 'Application',
          Comment: pkg.description,
          StartupWMClass: 'electron-ssr'
        }
      }
      // appImage: {
      //   license: 'LICENSE'
      // }
    }
  }).then(() => {
    console.log(`${BLUE}Done${END}`)
  }).catch(error => {
    console.error(`build arch: ${builder.Arch}`)
    console.error(`${YELLOW}Build error: ${error}${END}`)
  })
}

module.exports = release

```
出现一次下载错误之后
Build error: Error: Exit code: ENOENT. spawn /root/.cache/electron-builder/appimage/appimage-9.0.5/linux-arm/appimagetool ENOENT
它居然就不下载了
我还要手动下载解压。7zip解压移动
好吧可能不是这个问题

https://github.com/electron-userland/electron-builder/issues/2348

That's because it is searching for linux-arm directory that is not provided for some reason in the 9.0.5 version. Then also, once compiled and uploaded, you will find a latest-linux-armv7l.yml near the AppImage bundle and to make electron-updater work you have to rename it in latest-linux-arm.yml.

In this way I'm able to publish and autoupdate on armv7l.

I'm trying to figure it out in the electron-builder project, but I still have some difficulties to make the all thing build, sorry 😕
所以这个版本的工具包居然没有linux-arm！！
我的系统是32位的好像！
I am experiencing the same issue as @ffalcinelli.
When trying to run the electron-builder for armv7l 32 bit architecture, the AppImage downloaded does not contain a folder linux-arm.

I have not tried on a 64 bit architecture yet as the requirement is for a 32 bit.

Any pointers would be greatly appreciated.
哭了
我去掉appimage编译吧
看看会编译什么
不过好像unpacked可以用？？

最终的结果，打包deb的时候又下载了别的架构的东西
所以
__appImage-armv7l      electron-ssr-0.2.7.tar.gz  linux-armv7l-unpacked
electron-builder.yaml  icons


```
i@raspberrypi:~/electron-ssr-0.2.7 $ sudo npm run build 

> electron-ssr@0.2.7 build /home/pi/electron-ssr-0.2.7
> node .electron-vue/build.js

 ___              __                       
/\_ \       __   /\ \__     ____           
\//\ \    / ,.`\ \ \ ,_\   / ,__\  _______ 
  \_\ \_ /\  __/  \ \ \/  /\__, `\/\______\
  /\____\\ \____\  \ \ \_ \/\____/\/______/
  \/____/ \/____/   \ \__\ \/___/          
                     \/__/                 

 __                       ___       __    
/\ \       __  __   __   /\_ \     /\ \   
\ \ \____ /\ \/\ \ /\_\  \//\ \    \_\ \  
 \ \  ,. \\ \ \_\ \\/\ \   \_\ \_ /\ ,. \ 
  \ \____/ \ \____/ \ \ \  /\____\\ \____\
   \/___/   \/___/   \/_/  \/____/ \/___ /
                                          

  - building main process
  - building renderer process
  ✔ building main process
  ✔ building renderer process



Hash: b19245d59dd4d60d07c2
Version: webpack 4.41.2
Time: 46678ms
Built at: 2019/12/08 上午12:03:28
  Asset      Size  Chunks             Chunk Names
main.js  83.2 KiB       0  [emitted]  main
Entrypoint main = main.js
  [0] external "electron" 42 bytes {0} [built]
  [1] ./node_modules/babel-runtime/regenerator/index.js 49 bytes {0} [built]
  [2] ./node_modules/babel-runtime/core-js/promise.js 88 bytes {0} [built]
  [3] external "fs-extra" 42 bytes {0} [built]
  [4] ./node_modules/babel-runtime/helpers/asyncToGenerator.js 906 bytes {0} [built]
  [5] ./node_modules/babel-runtime/helpers/slicedToArray.js 1.18 KiB {0} [built]
  [6] external "path" 42 bytes {0} [built]
 [10] ./node_modules/babel-runtime/core-js/object/keys.js 92 bytes {0} [built]
 [11] external "child_process" 42 bytes {0} [built]
 [16] external "electron-log" 42 bytes {0} [built]
 [23] external "electron-updater" 42 bytes {0} [built]
 [30] ./node_modules/rxjs/Observable.js 12.9 KiB {0} [built]
 [31] external "urlsafe-base64" 42 bytes {0} [built]
 [80] external "auto-launch" 42 bytes {0} [built]
[156] ./src/main/index.js + 25 modules 83.2 KiB {0} [built]
      | ./src/main/index.js 3.19 KiB [built]
      | ./src/main/logger.js 874 bytes [built]
      | ./src/shared/env.js 1.02 KiB [built]
      | ./src/main/bootstrap.js 4.82 KiB [built]
      | ./src/main/window.js 3.62 KiB [built]
      | ./src/shared/ssr.js 5.89 KiB [built]
      | ./src/main/data.js 4.06 KiB [built]
      | ./src/main/subscribe.js 4.61 KiB [built]
      | ./src/main/pac.js 6.17 KiB [built]
      | ./src/main/proxy.js 3.42 KiB [built]
      | ./src/main/updater.js 2.72 KiB [built]
      | ./src/main/tray.js 6.18 KiB [built]
      | ./src/main/menu.js 2.92 KiB [built]
      | ./src/main/ipc.js 2 KiB [built]
      | ./src/main/http-proxy.js 3.19 KiB [built]
      |     + 11 hidden modules
    + 142 hidden modules

Hash: 79531d9b6fdfc0037990
Version: webpack 4.41.2
Time: 105652ms
Built at: 2019/12/08 上午12:04:27
                         Asset       Size  Chunks             Chunk Names
     fonts/ionicons--fonts.eot    118 KiB          [emitted]  
     fonts/ionicons--fonts.ttf    184 KiB          [emitted]  
    fonts/ionicons--fonts.woff   66.3 KiB          [emitted]  
      imgs/ionicons--fonts.svg    326 KiB          [emitted]  
                    index.html  317 bytes          [emitted]  
                   renderer.js    654 KiB       0  [emitted]  renderer
           static/disabled.png  357 bytes          [emitted]  
        static/disabled@2x.png  666 bytes          [emitted]  
            static/enabled.png  362 bytes          [emitted]  
         static/enabled@2x.png  659 bytes          [emitted]  
   static/enabledHighlight.png  277 bytes          [emitted]  
static/enabledHighlight@2x.png  412 bytes          [emitted]  
    static/enabledTemplate.png  266 bytes          [emitted]  
 static/enabledTemplate@2x.png  419 bytes          [emitted]  
             static/global.png  357 bytes          [emitted]  
          static/global@2x.png  642 bytes          [emitted]  
    static/globalHighlight.png  328 bytes          [emitted]  
 static/globalHighlight@2x.png  564 bytes          [emitted]  
     static/globalTemplate.png  318 bytes          [emitted]  
  static/globalTemplate@2x.png  544 bytes          [emitted]  
       static/notification.png  694 bytes          [emitted]  
    static/notification@2x.png   1.22 KiB          [emitted]  
                static/pac.png  389 bytes          [emitted]  
             static/pac@2x.png  706 bytes          [emitted]  
       static/pacHighlight.png  319 bytes          [emitted]  
    static/pacHighlight@2x.png  525 bytes          [emitted]  
        static/pacTemplate.png  307 bytes          [emitted]  
     static/pacTemplate@2x.png  526 bytes          [emitted]  
              static/plane.svg  463 bytes          [emitted]  
Entrypoint renderer = renderer.js
  [0] ./node_modules/babel-runtime/helpers/extends.js 544 bytes {0} [built]
  [1] external "electron" 42 bytes {0} [built]
  [4] ./node_modules/babel-runtime/core-js/object/keys.js 92 bytes {0} [built]
 [15] ./node_modules/babel-runtime/helpers/slicedToArray.js 1.18 KiB {0} [built]
 [21] ./node_modules/babel-runtime/core-js/json/stringify.js 95 bytes {0} [built]
 [34] ./node_modules/babel-runtime/core-js/promise.js 88 bytes {0} [built]
 [35] ./node_modules/babel-runtime/helpers/toConsumableArray.js 466 bytes {0} [built]
 [36] external "urlsafe-base64" 42 bytes {0} [built]
 [50] ./node_modules/vue-style-loader!./node_modules/css-loader!./node_modules/vue-loader/lib/loaders/stylePostLoader.js!./node_modules/stylus-loader!./node_modules/vue-loader/lib??vue-loader-options!./src/renderer/App.vue?vue&type=style&index=0&lang=stylus& 653 bytes {0} [built]
 [52] external "path" 42 bytes {0} [built]
 [53] ./node_modules/babel-runtime/helpers/typeof.js 1.04 KiB {0} [built]
 [54] external "mousetrap" 42 bytes {0} [built]
 [74] ./node_modules/babel-runtime/core-js/array/from.js 91 bytes {0} [built]
[227] ./src/renderer/App.vue?vue&type=style&index=0&lang=stylus& 650 bytes {0} [built]
[234] ./src/renderer/main.js + 316 modules 739 KiB {0} [built]
      | ./src/renderer/main.js 395 bytes [built]
      | ./node_modules/vue/dist/vue.esm.js 318 KiB [built]
      | ./src/renderer/components/index.js 2.15 KiB [built]
      | ./src/renderer/store/index.js 9.46 KiB [built]
      | ./src/renderer/ipc.js 2.44 KiB [built]
      | ./src/renderer/shortcut.js 570 bytes [built]
      | ./src/renderer/App.vue 513 bytes [built]
      | ./node_modules/vue-loader/lib/runtime/componentNormalizer.js 2.63 KiB [built]
      | ./node_modules/erguotou-iview/src/components/spin/index.js 649 bytes [built]
      | ./node_modules/erguotou-iview/src/components/icon/index.js 51 bytes [built]
      | ./node_modules/erguotou-iview/src/components/poptip/index.js 58 bytes [built]
      | ./node_modules/erguotou-iview/src/components/tooltip/index.js 61 bytes [built]
      | ./node_modules/erguotou-iview/src/components/tree/index.js 51 bytes [built]
      | ./node_modules/erguotou-iview/src/components/tag/index.js 48 bytes [built]
      | ./node_modules/erguotou-iview/src/components/table/index.js 54 bytes [built]
      |     + 302 hidden modules
    + 220 hidden modules
Child html-webpack-plugin for "index.html":
         Asset     Size  Chunks  Chunk Names
    index.html  533 KiB       0  
    Entrypoint undefined = index.html
    [0] ./node_modules/html-webpack-plugin/lib/loader.js!./src/index.ejs 1.08 KiB {0} [built]
    [1] ./node_modules/lodash/lodash.js 528 KiB {0} [built]
    [2] (webpack)/buildin/module.js 497 bytes {0} [built]


 OKAY  take it away `electron-builder`

  • electron-builder version=19.56.2
  • writing effective config file=build/electron-builder.yaml
  • no native production dependencies
  • packaging       platform=linux arch=armv7l electron=1.8.8 appOutDir=build/linux-armv7l-unpacked
  • building        target=tar.gz arch=armv7l file=build/electron-ssr-0.2.7.tar.gz
  • building        target=deb arch=armv7l file=build/electron-ssr-0.2.7.deb
  • downloading     path=/root/.cache/electron-builder/fpm/fpm-1.9.3-2.3.1-linux-x86_64 url=https://github.com/electron-userland/electron-builder-binaries/releases/download/fpm-1.9.3-2.3.1-linux-x86_64/fpm-1.9.3-2.3.1-linux-x86_64.7z
build arch: {"0":"ia32","1":"x64","2":"armv7l","3":"arm64","ia32":0,"x64":1,"armv7l":2,"arm64":3}
Build error: Error: Exit code: 1. Command failed: /root/.cache/electron-builder/fpm/fpm-1.9.3-2.3.1-linux-x86_64/fpm -s dir -t deb --architecture armv7l --name electron-ssr --force --after-install /tmp/t-dw4Fwv/0-after-install --after-remove /tmp/t-dw4Fwv/1-after-remove --description Cross platform ShadowsocksR GUI client built with electron
 Cross platform ShadowsocksR GUI client built with electron --version 0.2.7 --package /home/pi/electron-ssr-0.2.7/build/electron-ssr-0.2.7.deb --maintainer erguotou <erguotou525@gmail.com> --url https://github.com/shadowsocksrr/electron-ssr/ --vendor erguotou <erguotou525@gmail.com> --deb-compression xz --depends gconf2 --depends gconf-service --depends libnotify4 --depends libappindicator1 --depends libxtst6 --depends libnss3 --depends libxss1 --license MIT /home/pi/electron-ssr-0.2.7/build/linux-armv7l-unpacked/=/opt/electron-ssr /home/pi/electron-ssr-0.2.7/build/icons/16x16.png=/usr/share/icons/hicolor/16x16/apps/electron-ssr.png /home/pi/electron-ssr-0.2.7/build/icons/24x24.png=/usr/share/icons/hicolor/24x24/apps/electron-ssr.png /home/pi/electron-ssr-0.2.7/build/icons/32x32.png=/usr/share/icons/hicolor/32x32/apps/electron-ssr.png /home/pi/electron-ssr-0.2.7/build/icons/48x48.png=/usr/share/icons/hicolor/48x48/apps/electron-ssr.png /home/pi/electron-ssr-0.2.7/build/icons/64x64.png=/usr/share/icons/hicolor/64x64/apps/electron-ssr.png /home/pi/electron-ssr-0.2.7/build/icons/96x96.png=/usr/share/icons/hicolor/96x96/apps/electron-ssr.png /home/pi/electron-ssr-0.2.7/build/icons/128x128.png=/usr/share/icons/hicolor/128x128/apps/electron-ssr.png /home/pi/electron-ssr-0.2.7/build/icons/256x256.png=/usr/share/icons/hicolor/256x256/apps/electron-ssr.png /tmp/t-dw4Fwv/7-electron-ssr.desktop=/usr/share/applications/electron-ssr.desktop
/root/.cache/electron-builder/fpm/fpm-1.9.3-2.3.1-linux-x86_64/lib/ruby/bin/ruby:行6: /root/.cache/electron-builder/fpm/fpm-1.9.3-2.3.1-linux-x86_64/lib/ruby/bin.real/ruby：无法执行二进制文件: 可执行文件格式错误
/root/.cache/electron-builder/fpm/fpm-1.9.3-2.3.1-linux-x86_64/lib/ruby/bin/ruby:行6: /root/.cache/electron-builder/fpm/fpm-1.9.3-2.3.1-linux-x86_64/lib/ruby/bin.real/ruby: 成功

/root/.cache/electron-builder/fpm/fpm-1.9.3-2.3.1-linux-x86_64/lib/ruby/bin/ruby:行6: /root/.cache/electron-builder/fpm/fpm-1.9.3-2.3.1-linux-x86_64/lib/ruby/bin.real/ruby：无法执行二进制文件: 可执行文件格式错误
/root/.cache/electron-builder/fpm/fpm-1.9.3-2.3.1-linux-x86_64/lib/ruby/bin/ruby:行6: /root/.cache/electron-builder/fpm/fpm-1.9.3-2.3.1-linux-x86_64/lib/ruby/bin.real/ruby: 成功

```
最后获得了打包的tar.gz了
但是运行unpack的虽然能成功运行，但是窗口是全白的空的
不过搜索 raspbian electron blank有不少答案
难道我说不定可以不看界面却依然能用ssr哈哈
事实证明果然还是这样。我把我ubuntu下正常运行的electron-ssr的配置文件拿过来，运行。
发现报错，说ssr目录不存在。这个路径在配置文件一起的~/.config/shadowsocksr/（路径不记得了，不过是一个python文件）。
心想，看来electron调用ssr也只是一个通过启动命令行去调用shadowsocksr的python版，既然这样，那我复制我ubuntu下的ssr过去应该也行吧，既然python是跨平台的。
所以就把上面的shadowsocksr文件夹也移动过去了，修改配置文件里面的路径，最后得到了没有窗口只有（任务栏吗）顶部的图标和菜单的"headless"- electron-ssr.
而且如果服务器地址太多，菜单过长，可能导致在列表靠下的地方的服务器选择不到。。。

2. 看看怎么修复吧
dependencies里面写
 "@sentry/electron": "0.14.0"
这个版本对应的是哪个electron版本？？
好吧是electron 3.0.3
怎么后面发布的几个版本的里面写的electron全都是3.0.3？？

https://github.com/electron/electron/issues/12937
到最后它居然关闭了issue，那就更新吧。。。更新到4.2去
看看新版本行不行。。

npm install @sentry/electron@0.17.4 --save
https://github.com/npm/npm/issues/17268
sudo npm install -g electron@3.0.3 --unsafe-perm=true --allow-root

我可能是瞎了，"electron": "^2.0.0"才是真正的electron，什么的sentry的可能根本就不是electron。
不对，这应该是不知道什么时候冒出来的。
Build error: Error: Package "electron" is only allowed in "devDependencies". Please remove it from the "dependencies" section in your package.json.
所以还是不能有electron
npm remove electron --save
更新了sentry/electron之后连弹窗都不弹了。。。
还是没有界面
第二次build还是不行吗

但是santry/electron的最新的beta版本用的是5.0.12,下载下来试试。


```
其他没有用的办法

https://github.com/electron/electron/issues/12850

--disable-gpu

试试

https://github.com/electron/electron/issues/12329

好像可以

apt install --no-install-recommends libgl1-mesa-dri

安装electron@2.0.0
但是好像怎么还是不行。。。
sudo npm install -g electron@1.6.11 --unsafe-perm=true --allow-root
看看这样行不行？算了，版本也太旧了

```
