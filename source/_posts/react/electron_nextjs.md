---
title: NextJs集成Electron
category: 
  - frontend
---

>Electron 简单来说就是一个浏览器套壳打包方案，相当于你开发了一个网页后，通过浏览器能访问这个网页；然后如果你想把网页打包成一个单独的应用的话，可以使用Electron，它内置了 chromium 和 node.js 运行环境，使用 Electron 相当于构建了一个只显示你的网页的浏览器程序。而且Electron可以通过node.js提供一些浏览器不能提供的功能。
>
>Electron 支持 **URL 加载**和**文件加载**两种方式。
>
>* URL加载是传入一个URL，显示URL内容。
>* 文件加载是传入HTML文件，Electron通过文件渲染内容。
>
>理论上来说，只要能进行静态导出，就能兼容各种前端框架。同时也提供了 Next.js 动态路由的加载的方式。

## 创建Nextjs工程文件

```bash
npx create-next-app@latest electron_nextjs
```

创建参数全部使用默认即可（使用Typescript，使用Tailwind CSS，使用App Router）

## 打包Nextjs工程

在`package.json`的scripts中已经有了build。

Nextjs打包有两种方式（需要配置`next.config.js`）：

* 静态模式

  ```js
  //next.config.js
  import type {NextConfig} from "next";
  const nextConfig: NextConfig = {
      output: 'export',
  };
  export default nextConfig;
  ```

* 动态模式

  ```js
  //next.config.js
  import type {NextConfig} from "next";
  const nextConfig: NextConfig = {
      output: 'standalone',
  };
  export default nextConfig;
  ```

**静态模式**是将工程压缩到一个HTML中，Electron只需要使用文件加载加载该HTML即可。但静态模式无法进行网络请求所以只用在一些仅展示的页面中，比如博客，公司简介等。

静态导出会将内容导出到`out/`目录下的`index.html`中，后续Electron只需要加载该文件即可（本文不讨论）：

```js
// main.js
mainWindow.loadFile(path.join(__dirname, '../out/index.html'))
```

本文采用**动态模式**，即Nextjs将以standalone模式进行打包，文件被导出到`.next/standalone`文件夹下，其中有`server.js`用于启动该项目。

```bash
node ./server.js
```

为了将公共资源包也整合进`.next/standalone`，需要重定义package.json中的`next build`

```json
//package.json
"scripts": {
    "build:next": "next build && cp -r public .next/standalone/ && cp -r .next/static .next/standalone/.next/",
  },
```

定义`build:next`脚本，执行`next build`后将`public`和`.next/static`导入到 `.next/standalone`中。

执行`npm run build:next` 即可打包Nextjs项目，然后进入`.next/standalone`使用`node ./server.js`启动Nextjs项目。

至此，Nextjs的打包过程就完成了。

## 配置Electron

根据[Electron官网](https://www.electronjs.org/docs/latest/tutorial/tutorial-first-app#initializing-your-npm-project)，将Electron依赖加入到项目中：

```bash
npm install electron --save-dev
```

为了能同时启动Nextjs和Electron，安装`concurrently`依赖包

```bash
npm install concurrently --save-dev
```

创建`main/main.js`文件作为Electron的启动入口，以及`main/preload.js`作为主进程与渲染进程的桥梁。

添加包依赖，需要在`main/main.js`中使用，`electron-is-dev`用于判断是否处于dev开发过程，`get-port-please`是用于获取一个未被使用的端口，用于启动nextjs的standalone。

```bash
npm install electron-is-dev get-port-please
```

`main/main.js`

主要功能实现（前两个功能在后续Nextjs中会用到，目前没用）：

* 关闭系统自带状态栏，启用自定义按钮。（需要在前端css设置 `app-region: drag`，提供可拖拽区域）。
* 在Electron主进程中监听系统theme并发送给渲染进程。
* loadURL前进行轮训，直至nextjs服务启动，保证Electron加载URL时，Nextjs已经启动。

```JS
// main/main.js
import {app, BrowserWindow, ipcMain, nativeTheme} from "electron";

import isDev from "electron-is-dev";
import http from "http";
import {fileURLToPath} from 'url';
import {dirname} from 'path';
import path from "node:path";
import {getPort} from "get-port-please";
import {startServer} from "next/dist/server/lib/start-server.js";

const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);

let win = null;
let systemTheme = nativeTheme.shouldUseDarkColors ? 'dark' : 'light'

// 判断Nextjs是否启动成功，每一秒检查一次
const waitForNextJs = (bg_url) => {
    return new Promise((resolve, reject) => {
        const interval = setInterval(() => {
            http.get(bg_url, (res) => {
                if (res.statusCode === 200) {
                    clearInterval(interval);
                    resolve(); // Next.js 启动完成
                }
            }).on('error', (err) => {
                console.log('Next.js not yet started, retrying...');
            });
        }, 1000);
    });
}

// 启动Nextjs standalone包
const startNextJSServer = async () => {
    try {
        const nextJSPort = await getPort({portRange: [3000, 8000]});
        const webDir = path.join(__dirname, "../.next/standalone/");
        console.log("path: " + webDir);

        await startServer({
            dir: webDir,
            isDev: false,
            hostname: "localhost",
            port: nextJSPort,
            customServer: true,
            allowRetry: false,
            keepAliveTimeout: 5000,
            minimalMode: false,
        });

        return nextJSPort;
    } catch (error) {
        console.error("Error starting Next.js server:", error);
        throw error;
    }
};

// BrowserWindow使用自定义状态栏
const createWindow = () => {
    win = new BrowserWindow({
        width: 1000,
        height: 800,
        minWidth: 600,
        minHeight: 400,
        titleBarStyle: 'hidden',   //关闭系统自带状态栏
        titleBarOverlay: {
            color: "rgba(255, 255, 255, 0)",
            symbolColor: 'rgba(255, 255, 255, 0)',
        },

        webPreferences: {
            preload: path.join(__dirname, 'preload.js'),
        }
    });
    // 首先加载Loading页面，等待Nextjs的启动
    win.loadFile(path.join(__dirname, '../public/loading.html'));
    const loadURL = async () => {
        if (isDev) {
            const bg_url = "http://localhost:3000";
            waitForNextJs(bg_url).then(() => {
                win.setTitleBarOverlay({
                    color: "rgba(255, 255, 255, 0)",
                    symbolColor: '#74b1be',
                    height: 40
                })
                win.loadURL(bg_url);
            })
        } else {
            try {
                // 启动 nextjs 打包后的 standalone 并获取启动 端口
                const port = await startNextJSServer();
                console.log("Next.js server started on port:", port);
                win.setTitleBarOverlay({
                    color: "rgba(255, 255, 255, 0)",
                    symbolColor: '#74b1be',
                    height: 40
                })
                win.loadURL(`http://localhost:${port}`);
            } catch (error) {
                console.error("Error starting Next.js server:", error);
            }
        }
    };
    loadURL();

    // Electron启动后主动发送系统主题到渲染进程
    win.webContents.on('did-finish-load', () => {
        win.webContents.send('system-theme-changed', systemTheme)
    })
    // win.webContents.openDevTools();
    // win.webContents.on("did-fail-load", (e, code, desc) => {
    //     win.webContents.reloadIgnoringCache();
    // });
}

app.disableHardwareAcceleration();
app.whenReady().then(createWindow)
app.on("window-all-closed", () => {
    if (process.platform !== "darwin") {
        app.quit();
    }
});

// 当系统主题发生变化时，Electron主进程发送系统主题到渲染进程
nativeTheme.on('updated', () => {
    systemTheme = nativeTheme.shouldUseDarkColors ? 'dark' : 'light'
    BrowserWindow.getAllWindows().forEach(win => {
        win.webContents.send('system-theme-changed', systemTheme)
    })
})
// 主进程接口 用于渲染进程获取
ipcMain.handle('get-system-theme', () => systemTheme)
```

`main/preload.js`

主要功能（渲染进程的接口，供Nextjs使用，目前没用）：

* 提供获取系统主题的接口
* 监听系统主题切换

```js
// main/preload.js
const {contextBridge, ipcRenderer} = require('electron')

contextBridge.exposeInMainWorld('electronAPI', {
    onSystemThemeChanged: (callback) => {
        ipcRenderer.on('system-theme-changed', (_, theme) => callback(theme))
    },
    getSystemTheme: () => ipcRenderer.invoke('get-system-theme')
})

```

在`public/`目录下创建`loading.html`

```html
<!--public/loading.html-->
<html>
<body>
</body>
</html>

<style>
    body {
        margin: 0;
        height: 100vh;
        width: 100vw;
        background: linear-gradient(to right, red, blue);
    }
</style>
```

设置`package.json`

```json
{
 "name": "nextjs_electron",
  "version": "0.1.0",
  "private": true,
  "main": "main/main.js",
  "type": "module",
  "scripts": {
    "dev:next": "next dev",
    "dev:electron": "electron ./main/main.js",
    "dev": "concurrently --kill-others  \"npm run dev:next\" \"npm run dev:electron\"",
    "build:next": "next build && cp -r public .next/standalone/ && cp -r .next/static .next/standalone/.next/",
  },
    ...
}
```

执行 `npm run dev` 同时启动Nextjs和Electron。（目前窗口时不能拖动的，后续会在Nextjs中添加）

## Electron Forge打包Electron

根据[Electron Forge](https://www.electronforge.io/import-existing-project#using-the-import-script)官网教程安装打包工具：

```bash
npm install --save-dev @electron-forge/cli
npm exec --package=@electron-forge/cli -c "electron-forge import"
```

然后在`package.json`中配置需要打包的类型和平台并更新scripts

`package.json`

```json
{
  "name": "nextjs_electron",
  "version": "0.1.0",
  "private": true,
  "main": "main/main.js",
  "type": "module",
  "scripts": {
    "dev:next": "next dev",
    "dev:electron": "electron ./main/main.js",
    "dev": "concurrently --kill-others  \"npm run dev:next\" \"npm run dev:electron\"",
    "build:next": "next build && cp -r public .next/standalone/ && cp -r .next/static .next/standalone/.next/",
    "build:electron": "electron-forge package",
    "build": "npm run build:next && npm run build:electron",
    "make": "electron-forge make",
    "dist": "npm run build && npm run make",
    "start": "electron-forge start",
    "lint": "next lint"
  },
  "config": {
    "forge": {
      "packagerConfig": {},
      "makers": [
        {
          "name": "@electron-forge/maker-deb",
          "platforms": ["linux"],
          "config": {
            "icon": "public/chat.png",
            "description": "ichat for deb",
            "productDescription": "ichat for deb"
          }
        }
      ]
    }
  },
  ... ...
}
```

注：打包过程中，本文设置了`makers.config.icon`，所以需要添加图片`public/chat.png`，不想设置icon可以直接在`package.json`中删除`makers.config.icon`。

执行`npm run dist`命令，即执行了*Nextjs的build*，*election-forge的package* 和 *election-forge的make*。最终的生成文件在`/out/make/`内。

注：本文生成的时Linux的deb文件，执行命令`sudo apt install ./nextjs_electron_0.1.0_amd64.deb`安装。

## Nextjs可拖拽状态栏 和 更改主题颜色（可选）

> 本Nextjs项目整合
>
> * **HeroUI**前端UI框架，具体方法参照[HeroUI官方文档](https://www.heroui.com/docs/frameworks/nextjs#manual-installation)的手动导入
> * **[Next Theme](https://www.heroui.com/docs/customization/dark-mode#using-next-themes)**主题切换，但要记得[在html标签中添加](https://github.com/pacocoursey/next-themes?tab=readme-ov-file#with-app)`suppressHydrationWarning`

添加上述两个依赖后，需要在`layout.tsx`中自定义状态栏，将状态栏所在的`div`的CSS添加`style={{"app-region": 'drag'}}`。该CSS特性是Electron专属，可以让模块进行拖拽。

`src/app/layout.tsx`

```tsx
import "./globals.css";
import {Providers} from "./providers";
import CustomBar from "@/app/component/bar/CustomBar";


export default function RootLayout({
                                       children,
                                   }: Readonly<{
    children: React.ReactNode;
}>) {
    return (
        <html lang="en" suppressHydrationWarning>
        <body>
        <Providers>
            <div className="h-screen w-screen flex flex-col">
                <div className="h-[40px] w-full bg-[#E4E4E7] dark:bg-[#27272A]" style={{"app-region": 'drag'}}>
                    <CustomBar/>
                </div>
                <div className="flex-1 bg-[var(--background)]">
                    {children}
                </div>
            </div>
        </Providers>
        </body>
        </html>
    );
}

```

`src/app/globals.css`

```ts
@tailwind base;
@tailwind components;
@tailwind utilities;

body {
  font-family: Arial, Helvetica, sans-serif;
  margin: 0;
}
```

`src/app/provider.tsx`

```ts
'use client'

import {HeroUIProvider} from '@heroui/react'
import {ThemeProvider as NextThemesProvider} from "next-themes";

export function Providers({children}: { children: React.ReactNode }) {
    return (
        <HeroUIProvider>
            <NextThemesProvider attribute="class" defaultTheme="dark">
                {children}
            </NextThemesProvider>
        </HeroUIProvider>
    )
}
```

`src/app/component/bar/CustomBar.tsx`

```ts
"use client"

import {ThemeSwitcher} from "@/app/component/bar/ThemeSwitcher";
// import Settings from "@/app/component/bar/Settings";
// import UserInfo from "@/app/component/bar/UserInfo";

export default function CustomBar() {

    return (
        <div className="h-full w-full flex flex-end pr-[100px] gap-2">
            <div className="mr-auto">bar</div>
            <div style={{"app-region": 'no-drag'}}>
                <ThemeSwitcher/>
            </div>
            {/*<div style={{"app-region": 'no-drag'}}>*/}
            {/*    <Settings/>*/}
            {/*</div>*/}
            {/*<div className="flex items-center" style={{"app-region": 'no-drag'}}>*/}
            {/*    <UserInfo/>*/}
            {/*</div>*/}
        </div>
    )
}
```

`ThemeSwitcher.tsx`

使用`useTheme`来更改Nextjs的主题，使用HeroUI的`Dropdown`来提供更改按钮

```ts
"use client";

import {Button} from "@heroui/react";
import {useTheme} from "next-themes";
import React, {useEffect, useState} from "react";
import {Dropdown, DropdownItem, DropdownMenu, DropdownTrigger} from "@heroui/dropdown";

const DarkIcon = () => {
    return (
        <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" width="18" height="18" fill="currentColor">
            <path
                d="M10 7C10 10.866 13.134 14 17 14C18.9584 14 20.729 13.1957 21.9995 11.8995C22 11.933 22 11.9665 22 12C22 17.5228 17.5228 22 12 22C6.47715 22 2 17.5228 2 12C2 6.47715 6.47715 2 12 2C12.0335 2 12.067 2 12.1005 2.00049C10.8043 3.27098 10 5.04157 10 7ZM4 12C4 16.4183 7.58172 20 12 20C15.0583 20 17.7158 18.2839 19.062 15.7621C18.3945 15.9187 17.7035 16 17 16C12.0294 16 8 11.9706 8 7C8 6.29648 8.08133 5.60547 8.2379 4.938C5.71611 6.28423 4 8.9417 4 12Z"></path>
        </svg>
    )
}

const LightIcon = () => {
    return (
        <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" width="18" height="18" fill="currentColor">
            <path
                d="M12 18C8.68629 18 6 15.3137 6 12C6 8.68629 8.68629 6 12 6C15.3137 6 18 8.68629 18 12C18 15.3137 15.3137 18 12 18ZM12 16C14.2091 16 16 14.2091 16 12C16 9.79086 14.2091 8 12 8C9.79086 8 8 9.79086 8 12C8 14.2091 9.79086 16 12 16ZM11 1H13V4H11V1ZM11 20H13V23H11V20ZM3.51472 4.92893L4.92893 3.51472L7.05025 5.63604L5.63604 7.05025L3.51472 4.92893ZM16.9497 18.364L18.364 16.9497L20.4853 19.0711L19.0711 20.4853L16.9497 18.364ZM19.0711 3.51472L20.4853 4.92893L18.364 7.05025L16.9497 5.63604L19.0711 3.51472ZM5.63604 16.9497L7.05025 18.364L4.92893 20.4853L3.51472 19.0711L5.63604 16.9497ZM23 11V13H20V11H23ZM4 11V13H1V11H4Z"></path>
        </svg>
    )
}

const SystemIcon = () => {
    return (
        <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" width="18" height="18" fill="currentColor">
            <path
                d="M4 16H20V5H4V16ZM13 18V20H17V22H7V20H11V18H2.9918C2.44405 18 2 17.5511 2 16.9925V4.00748C2 3.45107 2.45531 3 2.9918 3H21.0082C21.556 3 22 3.44892 22 4.00748V16.9925C22 17.5489 21.5447 18 21.0082 18H13Z"></path>
        </svg>
    )
}

export function ThemeSwitcher() {
    const [mounted, setMounted] = useState(false)
    const {theme, setTheme} = useTheme()
    useEffect(() => {
        setMounted(true)
        window.electronAPI.getSystemTheme().then(initialTheme => {
            setTheme(initialTheme)
        })
        window.electronAPI.onSystemThemeChanged(setTheme)
    }, [])
    if (!mounted) return null


    const toggleTheme = (key: React.Key) => {
        if (key === "system") {
            setTheme("system")
        } else if (key === "dark") {
            setTheme("dark")
        } else if (key === "light") {
            setTheme("light")
        } else {
            console.log("theme error")
        }
    }

    return (
        <Dropdown classNames={{
            content: "min-w-32 w-32"
        }}>
            <DropdownTrigger>
                <Button className="h-10 w-10 min-w-10 p-0" variant="light" isIconOnly>
                    {theme === "system" && <SystemIcon/>}
                    {theme === "light" && <LightIcon/>}
                    {theme === "dark" && <DarkIcon/>}
                </Button>
            </DropdownTrigger>
            <DropdownMenu onAction={toggleTheme}
                          disallowEmptySelection
            >
                <DropdownItem key="light" startContent={<LightIcon/>}>Light</DropdownItem>
                <DropdownItem key="dark" startContent={<DarkIcon/>}>Dark</DropdownItem>
                <DropdownItem key="system" startContent={<SystemIcon/>}>System</DropdownItem>
            </DropdownMenu>
        </Dropdown>
    )
};
```

为了布局美观，将原来的`src/app/page.tsx`内容删除，改为

```tsx
export default function Home() {
  return (
      <div> main </div>
  );
}
```

此时执行`npm run dev`就可以启动程序了，并且可以切换主题。

但执行`npm run dist`打包的时候会报错，CSS中没有属性`app-region`，因为该特性是Electron特有的，在Nextjs打包时候会报错，为了不让Nextjs进行类型认证，在`next.config.ts`中设置跳过类型认证：

```ts
import type {NextConfig} from "next";

const nextConfig: NextConfig = {
    output: 'standalone',

    eslint: {
        ignoreDuringBuilds: true, // 忽略 eslint 检查
    },
    typescript: {
        ignoreBuildErrors: true, // 忽略 TypeScript 检查
    }
};

export default nextConfig;

```

## 错误

1. 如果在使用Typescript时出现Nextjs报错无法使用`main/preloading.js`内的方法时，可能是因为Typescript的类型审查不过关，需要告诉Nextjs在使用`window.electronAPI.method()`时该方法的类型。

   在根目录下创建`electron-env.d.ts`（因为使用Typescript，需要定义函数类型）：

   ```ts
   // electron-env.d.ts
   export interface IElectronAPI {
       getSystemTheme: () => Promise<string>,
       onSystemThemeChanged: (fn: any) => void,
   }
   
   declare global {
       interface Window {
           electronAPI: IElectronAPI
       }
   }
   ```

