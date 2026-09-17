---
title: Nextjs + HeroUI 工程
date: 2026-06-08 00:00:00 +0800
categories: [nextjs]
tags: [nextjs, heroui]
author: caohongchuan
pin: false
math: true
toc: true
comments: true
mermaid: true
---

> Nextjs 官方文档：https://nextjs.org/docs/app/getting-started/installation#quick-start
>
> HeroUI官方文档：https://heroui.com/en/docs/react/getting-started/quick-start

## 创建Nextjs工程

```bash
npx create-next-app@latest my-app --yes
cd my-app
npm run dev
```

## 添加HeroUI依赖

```bash
npm i @heroui/styles @heroui/react
```

在 `globals.css` 中导入 HeroUI 的CSS

```css
@import "tailwindcss";
@import "@heroui/styles"; 
```

引入测试

```typescript
import { Button } from '@heroui/react';

function App() {
  return (
    <Button>
      My Button
    </Button>
  );
}
```



```qml
import QtQuick
import QtQuick.Layouts
import QtQuick.Controls.Basic

ApplicationWindow {
    id: window
    width: 640
    height: 480
    minimumWidth: 200
    minimumHeight: 250
    visible: true
    flags: Qt.FramelessWindowHint | Qt.WindowStaysOnTopHint
    opacity: 0.7
    title: qsTr("Hello World")
    property bool lightMode: Application.styleHints.colorScheme === Qt.Light
    property color reallyDark: "#1f1f1f"
    property color dark: "#262626"
    property color reallyLight: "#e7e7e7"
    property color light: "#e0e0e0"

    Menu {
        id: windowMenu
        MenuItem {
            text: qsTr("关闭")
            onTriggered: window.close()
        }
    }

    TapHandler {
        acceptedButtons: Qt.RightButton
        onTapped: windowMenu.popup()
    }

    MouseArea {
        anchors.fill: parent
        acceptedButtons: Qt.LeftButton
        onPressed: mouse => {
            window.startSystemMove();
        }
        z: -1 // 放在最底层，避免遮挡按钮等控件
    }

    GridLayout {
        id: grid
        columns: width < 400 ? 1 : 2
        rowSpacing: 0
        columnSpacing: 0
        anchors.fill: parent

        Rectangle {
            id: rectangle1
            color: window.lightMode ? window.reallyLight : window.reallyDark
            Layout.fillHeight: true
            Layout.fillWidth: true

            ColumnLayout {
                anchors.fill: parent
                Layout.alignment: Qt.AlignHCenter | Qt.AlignTop

                Label {
                    id: text1
                    color: window.lightMode ? window.dark : window.light
                    font.pixelSize: 120
                    fontSizeMode: Text.Fit
                    text: qsTr("Hello World")
                    Layout.fillWidth: true
                    Layout.fillHeight: true
                    Layout.margins: 16
                    horizontalAlignment: Text.AlignHCenter
                    verticalAlignment: Text.AlignVCenter
                }
            }
        }

        Rectangle {
            id: rectangle2
            color: window.lightMode ? window.light : window.dark
            Layout.fillHeight: true
            Layout.fillWidth: true

            ColumnLayout {
                anchors.fill: parent
                Layout.alignment: Qt.AlignHCenter | Qt.AlignTop

                Button {
                    id: button1
                    text: window.lightMode ? qsTr("\u263D  Dark mode") : qsTr("\u263C  Light mode")
                    Layout.bottomMargin: 16
                    Layout.alignment: Qt.AlignHCenter | Qt.AlignBottom

                    contentItem: Text {
                        text: button1.text
                        color: window.lightMode ? window.light : window.dark
                        font: button1.font
                        horizontalAlignment: Text.AlignHCenter
                        verticalAlignment: Text.AlignVCenter
                    }

                    background: Rectangle {
                        implicitWidth: 120
                        implicitHeight: 36
                        radius: 8
                        color: window.lightMode ? window.dark : window.light
                    }

                    onClicked: window.lightMode = !window.lightMode
                }
            }
        }
    }
}

```

