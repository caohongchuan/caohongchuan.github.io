---
title: video.js 基于forest主题的自定义播放器
category: frontend
---

# 安装

在nextjs框架中，通过npm下载video.js依赖包

```bash
npm install video.js
npm install @types/video.js
```

# 使用

将下面两个文件写入项目后，可以直接导入`VideoPlayer`来使用自定义的video.js

注：视频大小为16：9的格式

```tsx
// page.tsx
import VideoPlayer from "./components/VideoPlayer";

export default function Home() {
    const videoJsOptions = {
        autoplay: true,
        controls: true,
        sources: [
            {
                src: "http://192.168.101.67:9000/%E8%AF%9B%E4%BB%994K/52.mp4",
                type: "video/mp4",
            },
        ],
    };
    return (
        <div className="w-[800px] aspect-video">
            <VideoPlayer options={videoJsOptions} />
        </div>
    );
}

```

# TSX 文件

```tsx
"use client";
import { useEffect, useRef } from "react";
// 引入 video.js 及样式
import videojs from "video.js";
import "video.js/dist/video-js.css";
import "@videojs/themes/dist/forest/index.css";
import "./custom-videojs.css";

export interface VideoJSProps {
    options: {
        autoplay?: boolean;
        controls?: boolean;
        responsive?: boolean;
        sources: { src: string; type: string }[];
    };
    onReady?: (player: any) => void;
}

export const VideoPlayer: React.FC<VideoJSProps> = ({ options, onReady }) => {
    const videoRef = useRef<HTMLDivElement>(null);
    const playerRef = useRef<any>(null);

    useEffect(() => {
        if (!playerRef.current) {
            // 创建 video 元素并插入到容器中
            const videoElement = document.createElement("video-js");
            videoElement.classList.add(
                "vjs-fluid",
                "vjs-responsive",
                "vjs-theme-forest",
                "vjs-16-9"
            );
            videoRef.current?.appendChild(videoElement);

            // 注册回退 10 秒按钮
            class Rewind10Button extends videojs.getComponent("Button") {
                buildCSSClass() {
                    return "vjs-control vjs-button vjs-rewind-10-button";
                }

                handleClick() {
                    const player = this.player_;
                    if (player && !player.isDisposed()) {
                        player.currentTime(
                            Math.max(player.currentTime() - 10, 0)
                        );
                    }
                }
            }
            // 注册快进 10 秒按钮
            class Forward10Button extends videojs.getComponent("Button") {
                buildCSSClass() {
                    return "vjs-control vjs-button vjs-forward-10-button"; // 自定义类名
                }

                handleClick() {
                    const player = this.player_;
                    if (player && !player.isDisposed()) {
                        const duration = player.duration();
                        player.currentTime(
                            Math.min(player.currentTime() + 10, duration)
                        );
                    }
                }
            }
            // 注册 play 按钮
            class PlayButton extends videojs.getComponent("Button") {
                constructor(player: any, options: any) {
                    super(player, options);
                    // 初始化按钮状态
                    this.updateClasses();
                    // 监听播放器事件以更新按钮状态
                    this.player_.on("play", () => this.updateClasses());
                    this.player_.on("pause", () => this.updateClasses());
                }

                buildCSSClass() {
                    // 确保按钮包含默认的播放控制类和自定义类
                    return "vjs-control vjs-button vjs-play-button";
                }

                handleClick() {
                    const player = this.player_;
                    if (player && !player.isDisposed()) {
                        if (player.paused()) {
                            player.play();
                        } else {
                            player.pause();
                        }
                    }
                }

                updateClasses() {
                    // 直接使用 Video.js 默认的类切换逻辑
                    if (this.player_ && !this.player_.isDisposed()) {
                        this.removeClass("vjs-playing");
                        this.removeClass("vjs-paused");
                        if (this.player_.paused()) {
                            this.addClass("vjs-paused");
                        } else {
                            this.addClass("vjs-playing");
                        }
                    }
                }
            }
            // 注册音量按钮
            class VolumeButton extends videojs.getComponent("Button") {
                // 声明类属性
                private sliderContainer: HTMLElement | null = null;
                private volumeSlider: HTMLElement | null = null;
                private volumeBar: HTMLElement | null = null;

                private defaultVolume = 0.3;

                constructor(player: any, options: any) {
                    super(player, options);
                    this.initializePlayer();
                    // 初始化按钮状态
                    this.updateClasses();
                    // 监听音量变化和静音状态变化
                    this.player_.on("volumechange", () => this.updateClasses());
                    // 创建音量滑块
                    this.createVolumeSlider();
                    // 监听鼠标悬停事件
                    this.on("mouseenter", () => this.showVolumeSlider());
                    this.on("mouseleave", () => this.hideVolumeSlider());
                }

                buildCSSClass() {
                    return "vjs-control vjs-button vjs-volume-button";
                }

                handleClick() {
                    const player = this.player_;
                    if (player && !player.isDisposed()) {
                        // 切换静音状态
                        if (player.muted()) {
                            player.muted(false);
                            const lastVolume = player.lastVolume_() || 0.3;
                            player.volume(lastVolume);
                        } else {
                            console.log("click muted handle");
                            player.muted(true);
                        }
                    }
                }

                // 初始化播放器状态
                initializePlayer() {
                    if (this.player_ && !this.player_.isDisposed()) {
                        // 设置初始音量为30%
                        this.player_.volume(this.defaultVolume);
                        // 设置为静音状态
                        this.player_.muted(true);
                        // 保存默认音量到 lastVolume
                        this.player_.lastVolume_(this.defaultVolume);
                    }
                }

                updateClasses() {
                    if (this.player_ && !this.player_.isDisposed()) {
                        this.removeClass("vjs-vol-0");
                        this.removeClass("vjs-vol-1");
                        this.removeClass("vjs-vol-2");
                        this.removeClass("vjs-vol-3");
                        const vol = this.player_.volume();
                        const muted = this.player_.muted();
                        let level = 3;
                        if (muted || vol === 0) {
                            level = 0;
                        } else if (vol < 0.33) {
                            level = 1;
                        } else if (vol < 0.67) {
                            level = 2;
                        }
                        this.addClass(`vjs-vol-${level}`);
                    }
                }

                createVolumeSlider() {
                    // 创建音量滑块容器
                    this.sliderContainer = document.createElement("div");
                    this.sliderContainer.className =
                        "vjs-volume-slider-container vjs-hidden";
                    this.el().appendChild(this.sliderContainer);

                    // 创建音量滑块
                    this.volumeSlider = videojs.dom.createEl("div", {
                        className: "vjs-volume-slider-down",
                    }) as HTMLElement;

                    // 创建滑块条
                    this.volumeBar = videojs.dom.createEl("div", {
                        className: "vjs-volume-bar-down",
                    }) as HTMLElement;

                    this.volumeSlider.appendChild(this.volumeBar);
                    this.sliderContainer.appendChild(this.volumeSlider);

                    // 初始化滑块
                    this.updateSlider();

                    // 绑定滑块事件
                    this.volumeSlider.addEventListener("mousedown", (e) =>
                        this.handleSliderInteraction(e)
                    );
                    this.volumeSlider.addEventListener("click", (e) =>
                        e.stopPropagation()
                    );
                    this.player_.on("volumechange", () => this.updateSlider());
                }

                updateSlider() {
                    if (
                        this.player_ &&
                        !this.player_.isDisposed() &&
                        this.volumeBar
                    ) {
                        const volume = this.player_.muted()
                            ? 0
                            : this.player_.volume();
                        // 更新滑块条的高度（垂直滑块）
                        this.volumeBar.style.height = `${volume * 100}%`;
                    }
                }

                handleSliderInteraction(e: MouseEvent) {
                    e.stopPropagation();

                    if (
                        this.volumeSlider &&
                        this.player_ &&
                        !this.player_.isDisposed()
                    ) {
                        const rect = this.volumeSlider.getBoundingClientRect();
                        const updateVolume = (event: MouseEvent) => {
                            const y = event.clientY - rect.top;
                            const height = rect.height;
                            let newVolume = Math.max(
                                0,
                                Math.min(1, 1 - y / height)
                            );
                            this.player_.muted(false);
                            this.player_.volume(newVolume);
                        };
                        updateVolume(e);
                        const onMouseMove = (event: MouseEvent) =>
                            updateVolume(event);
                        const onMouseUp = () => {
                            document.removeEventListener(
                                "mousemove",
                                onMouseMove
                            );
                            document.removeEventListener("mouseup", onMouseUp);
                        };
                        document.addEventListener("mousemove", onMouseMove);
                        document.addEventListener("mouseup", onMouseUp);
                    }
                }

                showVolumeSlider() {
                    if (this.sliderContainer) {
                        this.sliderContainer.classList.remove("vjs-hidden");
                    }
                }

                hideVolumeSlider() {
                    if (this.sliderContainer) {
                        this.sliderContainer.classList.add("vjs-hidden");
                    }
                }

                dispose() {
                    if (this.volumeSlider) {
                        this.volumeSlider.removeEventListener(
                            "mousedown",
                            this.handleSliderInteraction
                        );
                        this.volumeSlider.removeEventListener("click", (e) =>
                            e.stopPropagation()
                        );
                    }
                    super.dispose();
                }
            }
            // 注册设置按钮
            class settingButton extends videojs.getComponent("Button") {
                private selectContainer: HTMLElement | null = null;
                private mainContainer: HTMLElement | null = null;
                private speedTitleElement: HTMLElement | null = null;
                private qualityTitleElement: HTMLElement | null = null;
                private speedContainer: HTMLElement | null = null;
                private speedElement: HTMLElement | null = null;
                private qualityContainer: HTMLElement | null = null;
                private qualityElement: HTMLElement | null = null;

                constructor(player: any, options: any) {
                    super(player, options);
                    this.createSelectContainer(); // 在构造函数中调用
                }

                buildCSSClass() {
                    return "vjs-control vjs-button vjs-setting-button";
                }

                handleClick() {
                    const player = this.player_;
                    if (player && !player.isDisposed()) {
                        if (this.selectContainer) {
                            this.selectContainer.classList.toggle("vjs-hidden");
                            this.initSettingView();
                        }
                    }
                }

                // 设置点击外部关闭的处理函数
                documentClickHandler = (event: Event) => {
                    if (
                        this.selectContainer &&
                        !this.selectContainer.contains(event.target as Node) &&
                        !this.el().contains(event.target as Node)
                    ) {
                        this.selectContainer.classList.add("vjs-hidden");
                    }
                };

                createSelectContainer() {
                    this.selectContainer = document.createElement("div");
                    this.selectContainer.className =
                        "vjs-setting-container vjs-hidden";
                    this.el().appendChild(this.selectContainer);

                    this.createMainContainer();
                    this.createSpeedContainer();
                    this.createQualityContainer();

                    this.selectContainer.appendChild(this.mainContainer!);
                    this.selectContainer.appendChild(this.speedContainer!);
                    this.selectContainer.appendChild(this.qualityContainer!);

                    this.initSettingView();

                    // 点击外部关闭选择容器
                    document.addEventListener(
                        "click",
                        this.documentClickHandler
                    );
                }

                createMainContainer() {
                    this.mainContainer = videojs.dom.createEl("div", {
                        className: "vjs-setting-main-container",
                    }) as HTMLElement;

                    // setting 按钮的点击事列表
                    // Quality
                    this.qualityTitleElement = videojs.dom.createEl("div", {
                        className: "vjs-setting-title",
                        innerHTML: `清晰度 1080P`,
                    }) as HTMLElement;
                    this.qualityTitleElement.addEventListener(
                        "click",
                        (e: Event) => {
                            e.stopPropagation();
                            if (this.mainContainer && this.qualityContainer) {
                                this.mainContainer.style.display = "none";
                                this.qualityContainer.style.display = "block";
                            }
                        }
                    );
                    this.mainContainer.appendChild(this.qualityTitleElement);

                    // speed
                    this.speedTitleElement = videojs.dom.createEl("div", {
                        className: "vjs-setting-title",
                        innerHTML: `倍速 ${this.player_.playbackRate()}x`,
                    }) as HTMLElement;
                    this.speedTitleElement.addEventListener(
                        "click",
                        (e: Event) => {
                            e.stopPropagation();
                            if (this.mainContainer && this.speedContainer) {
                                this.mainContainer.style.display = "none";
                                this.speedContainer.style.display = "block";
                            }
                        }
                    );
                    this.mainContainer.appendChild(this.speedTitleElement);
                }

                createSpeedContainer() {
                    // 倍速设置
                    this.speedContainer = videojs.dom.createEl("div", {
                        className: "vjs-setting-speed-container",
                    }) as HTMLElement;
                    this.speedElement = videojs.dom.createEl("div", {
                        className: "vjs-setting-select-element",
                    }) as HTMLElement;

                    const rates = [0.5, 1, 1.5, 2, 3];
                    const currentRate = this.player_.playbackRate();
                    rates.forEach((rate) => {
                        const option = videojs.dom.createEl("div", {
                            className: "vjs-rate-option",
                            innerHTML: `${rate}x`,
                        }) as HTMLElement;

                        option.addEventListener("click", (e: Event) => {
                            e.stopPropagation();
                            this.setPlaybackRate(rate);
                        });
                        this.speedElement?.appendChild(option);
                    });
                    this.setPlaybackRate(currentRate);

                    this.speedContainer.appendChild(this.speedElement);

                    this.bindPlayerEvents(); // 绑定播放器事件
                }

                createQualityContainer() {
                    // 清晰度设置
                    this.qualityContainer = videojs.dom.createEl("div", {
                        className: "vjs-setting-quality-container",
                    }) as HTMLElement;
                    this.qualityElement = videojs.dom.createEl("div", {
                        className: "vjs-setting-select-element",
                    }) as HTMLElement;
                    const qualities = [
                        "4k",
                        "2K",
                        "1080P",
                        "720P",
                        "480P",
                        "360P",
                    ];
                    // const currentQuality = this.player_.qualityLevel();
                    qualities.forEach((quality) => {
                        const option = videojs.dom.createEl("div", {
                            className: "vjs-rate-option",
                            innerHTML: quality,
                        }) as HTMLElement;
                        this.qualityElement?.appendChild(option);
                    });
                    this.qualityContainer.appendChild(this.qualityElement);
                }

                initSettingView() {
                    if (this.mainContainer)
                        this.mainContainer.style.display = "block";
                    if (this.speedContainer)
                        this.speedContainer.style.display = "none";
                    if (this.qualityContainer)
                        this.qualityContainer.style.display = "none";
                }

                setPlaybackRate(rate: number) {
                    // 更新播放器倍速
                    this.player_.playbackRate(rate);
                    // 更新选中状态
                    this.updateSelectedOption(rate);
                    // 更新主容器中的倍速显示文本
                    this.updateSpeedTitle(rate);
                    this.initSettingView();
                }

                // 新增方法：更新主容器中的倍速显示文本
                updateSpeedTitle(rate: number) {
                    if (this.speedTitleElement) {
                        this.speedTitleElement.innerHTML = `倍速 ${rate}x`;
                    }
                }

                updateSelectedOption(selectedRate: number) {
                    if (this.speedElement) {
                        const options =
                            this.speedElement.querySelectorAll(
                                ".vjs-rate-option"
                            );
                        options.forEach((option) => {
                            const rate = parseFloat(
                                (option as HTMLElement).innerText.replace(
                                    "x",
                                    ""
                                )
                            );
                            if (rate === selectedRate) {
                                option.classList.add("vjs-rate-selected");
                            } else {
                                option.classList.remove("vjs-rate-selected");
                            }
                        });
                    }
                }

                bindPlayerEvents() {
                    // 监听播放器倍速变化事件（可能由其他方式改变）
                    this.player_.on("ratechange", () => {
                        const currentRate = this.player_.playbackRate();
                        this.updateSelectedOption(currentRate);
                    });
                }

                dispose() {
                    // 清理事件监听器
                    if (this.documentClickHandler) {
                        document.removeEventListener(
                            "click",
                            this.documentClickHandler
                        );
                    }

                    // 调用父类的 dispose 方法
                    super.dispose();
                }
            }

            videojs.registerComponent("Rewind10Button", Rewind10Button);
            videojs.registerComponent("Forward10Button", Forward10Button);
            videojs.registerComponent("PlayButton", PlayButton);
            videojs.registerComponent("VolumeButton", VolumeButton);
            videojs.registerComponent("SettingButton", settingButton);

            const player = videojs(
                videoElement,
                {
                    ...options,
                    controlBar: {
                        children: [
                            "rewind10Button",
                            "playButton",
                            "forward10Button",
                            "progressControl",
                            "currentTimeDisplay",
                            "volumeButton",
                            "settingButton",
                            "pictureInPictureToggle",
                            "fullscreenToggle",
                        ],
                    },
                },
                () => {
                    console.log("player is ready");
                    playerRef.current = player;
                    onReady && onReady(player);
                }
            );

            playerRef.current = player;
        } else {
            const player = playerRef.current;
            player.autoplay(options.autoplay || false);
            player.src(options.sources);
        }
    }, [options, videoRef, onReady]);

    // 组件卸载时销毁播放器
    useEffect(() => {
        return () => {
            if (playerRef.current && !playerRef.current.isDisposed()) {
                playerRef.current.dispose();
                playerRef.current = null;
            }
        };
    }, []);

    return (
        <div data-vjs-player className="rounded-2xl overflow-hidden">
            <div ref={videoRef} />
        </div>
    );
};

export default VideoPlayer;

```

# CSS 文件

```css
/* custom-videojs.css */

.vjs-theme-forest {
    --custom-theme-color: #34495e;
}

/* 确保 Video.js 字体加载 */
@font-face {
    font-family: "VideoJS";
    src: url("https://vjs.zencdn.net/font/VideoJS.woff") format("woff");
    font-weight: normal;
    font-style: normal;
}

/* ********************************************************** 按钮图标 ********************************************************** */

/* 后退 10 秒按钮图标 */
.vjs-theme-forest .vjs-rewind-10-button .vjs-icon-placeholder::before {
    content: "\f11d"; /* Video.js 图标字体中的后退图标 */
    font-family: "VideoJS";
    font-size: 1.5em;
    line-height: inherit;
    display: inline-flex;
    align-items: center;
    justify-content: center;
}

/* 快进 10 秒按钮图标 */
.vjs-theme-forest .vjs-forward-10-button .vjs-icon-placeholder::before {
    content: "\f120"; /* Video.js 图标字体中的快进图标 */
    font-family: "VideoJS";
    font-size: 1.5em;
    line-height: inherit;
    display: inline-flex;
    align-items: center;
    justify-content: center;
}

/* 播放按钮 默认样式（暂停状态显示播放图标） */
.vjs-theme-forest .vjs-play-button.vjs-paused .vjs-icon-placeholder::before {
    content: "\f101"; /* Video.js 播放图标 */
    font-family: "VideoJS";
    font-size: 2em;
    line-height: inherit;
    display: inline-flex;
    align-items: center;
    justify-content: center;
}

/* 播放按钮 暂停图标 */
.vjs-theme-forest .vjs-play-button.vjs-playing .vjs-icon-placeholder::before {
    content: "\f103"; /* Video.js 暂停图标 */
    font-family: "VideoJS";
    font-size: 2em;
    line-height: inherit;
    display: inline-flex;
    align-items: center;
    justify-content: center;
}

/* 音量按钮 - 3级音量图标 */
.vjs-theme-forest .vjs-volume-button.vjs-vol-3 .vjs-icon-placeholder::before {
    content: "\f107"; /* Video.js 音量图标 */
    font-family: "VideoJS";
    font-size: 1.7em;
    line-height: inherit;
    display: inline-flex;
    align-items: center;
    justify-content: center;
}
/* 音量按钮 - 2级音量图标 */
.vjs-theme-forest .vjs-volume-button.vjs-vol-2 .vjs-icon-placeholder::before {
    content: "\f106"; /* Video.js 音量图标 */
    font-family: "VideoJS";
    font-size: 1.7em;
    line-height: inherit;
    display: inline-flex;
    align-items: center;
    justify-content: center;
}
/* 音量按钮 - 1级音量图标 */
.vjs-theme-forest .vjs-volume-button.vjs-vol-1 .vjs-icon-placeholder::before {
    content: "\f105"; /* Video.js 音量图标 */
    font-family: "VideoJS";
    font-size: 1.7em;
    line-height: inherit;
    display: inline-flex;
    align-items: center;
    justify-content: center;
}
/* 音量按钮 - 静音图标 */
.vjs-theme-forest .vjs-volume-button.vjs-vol-0 .vjs-icon-placeholder::before {
    content: "\f104"; /* Video.js 静音图标 */
    font-family: "VideoJS";
    font-size: 1.7em;
    line-height: inherit;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    position: relative;
}

/* 音量按钮 - 3级音量图标 */
.vjs-theme-forest
    .vjs-picture-in-picture-control
    .vjs-icon-placeholder::before {
    content: "\f127"; /* Video.js 图标字体中的画中画图标 */
    font-family: "VideoJS";
    font-size: 1.5em;
    line-height: inherit;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    position: relative;
}

/* 全屏按钮 */
.vjs-theme-forest .vjs-fullscreen-control .vjs-icon-placeholder::before {
    content: "\f108"; /* Video.js 全屏图标 */
    font-family: "VideoJS";
    font-size: 1.5em;
    line-height: inherit;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    position: relative;
}

/* 设置按钮 */
.vjs-theme-forest .vjs-setting-button:before {
    content: "\f114";
    font-family: "VideoJS";
    font-size: 1.5em;
    line-height: inherit;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    position: relative;
}

/* ********************************************************** 设置样式 ********************************************************** */

.vjs-theme-forest .vjs-setting-container {
    position: absolute;
    bottom: 100%;
    left: 50%;
    transform: translateX(-50%);
    width: 60px;
    background: transparent; /* 白色半透明背景，透明度 0.2 */
    /* 应用磨砂效果 */
    backdrop-filter: blur(10px); /* 模糊半径为 10px */
    border-radius: 4px;
    padding: 0;
    z-index: 100;
}
.vjs-theme-forest .vjs-setting-main-container {
    position: relative;
    bottom: 5px;
    width: 100%;
    height: 100%;
    background: rgba(255, 255, 255, 0.2); /* 白色半透明背景，透明度 0.2 */
    /* 应用磨砂效果 */
    backdrop-filter: blur(10px); /* 模糊半径为 10px */
    border-radius: 4px;
}
.vjs-theme-forest .vjs-setting-title {
    color: #ecf0f1; /* 浅色标题 */
    font-size: 1em;
    text-align: center;
    padding: 7px 0;
    width: 100%;
}
.vjs-theme-forest .vjs-setting-title:hover {
    background: var(--custom-theme-color);
    border-radius: 4px;
}

.vjs-theme-forest .vjs-setting-speed-container {
    position: relative;
    left: 20%;
    bottom: 5px;
    width: 60%;
    height: 100%;
    background: rgba(255, 255, 255, 0.2); /* 白色半透明背景，透明度 0.2 */
    /* 应用磨砂效果 */
    backdrop-filter: blur(10px); /* 模糊半径为 10px */
    border-radius: 1px;
    cursor: pointer;
    border-radius: 4px;
}

.vjs-theme-forest .vjs-setting-container:not(.vjs-hidden) {
    display: block;
}
.vjs-theme-forest .vjs-rate-option {
    padding: 5px 0;
    cursor: pointer;
    text-align: center;
}
.vjs-theme-forest .vjs-rate-option:hover {
    background: var(--custom-theme-color);
    border-radius: 4px;
}

.vjs-theme-forest .vjs-rate-selected {
    background: var(--custom-theme-color);
    border-radius: 4px;
}

/* ********************************************************** 音量容器 ********************************************************** */

/* 音量滑块容器 */
.vjs-theme-forest .vjs-volume-slider-container {
    position: absolute;
    bottom: 100%;
    left: 50%;
    transform: translateX(-50%);
    width: 30px;
    height: 100px;
    background: transparent;
    border-radius: 4px;
    padding: 0;
    display: none;
    z-index: 100;
}

/* 显示滑块 */
.vjs-theme-forest .vjs-volume-slider-container:not(.vjs-hidden) {
    display: block;
}

/* 音量滑块 */
.vjs-theme-forest .vjs-volume-slider-down {
    position: relative;
    bottom: 5px;
    width: 100%;
    height: 100%;
    background: rgba(255, 255, 255, 0.2); /* 白色半透明背景，透明度 0.2 */
    /* 应用磨砂效果 */
    backdrop-filter: blur(10px); /* 模糊半径为 10px */
    border-radius: 1px;
    cursor: pointer;
    border-radius: 4px;
}

/* 音量滑块条 */
.vjs-theme-forest .vjs-volume-bar-down {
    position: absolute;
    bottom: 0;
    width: 100%;
    background: var(--custom-theme-color);
    border-radius: 1px;
    transition: height 0.1s;
    border-radius: 4px;
}

/* ********************************************************** 通用设置 ********************************************************** */

/* 按钮通用样式，适用于所有控制栏按钮 */
.vjs-theme-forest .vjs-rewind-10-button,
.vjs-theme-forest .vjs-forward-10-button,
.vjs-theme-forest .vjs-play-button,
.vjs-theme-forest .vjs-volume-button,
.vjs-theme-forest .vjs-setting-button,
.vjs-theme-forest .vjs-picture-in-picture-control,
.vjs-theme-forest .vjs-fullscreen-control {
    background-color: transparent !important;
    color: #ecf0f1 !important; /* 浅色图标 */
    width: 30px;
    height: 100%;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    cursor: pointer;
    padding: 0;
}

/* 按钮悬停时的样式 */
.vjs-theme-forest .vjs-play-button:hover,
.vjs-theme-forest .vjs-rewind-10-button:hover,
.vjs-theme-forest .vjs-forward-10-button:hover,
.vjs-theme-forest .vjs-volume-button:hover,
.vjs-theme-forest .vjs-setting-button:hover,
.vjs-theme-forest .vjs-picture-in-picture-control:hover,
.vjs-theme-forest .vjs-fullscreen-control:hover {
    background-color: var(--custom-theme-color) !important; /* 悬停时的颜色 */
    border-radius: 4px; /* 圆角 */
}

/* 防止点击按钮时出现光标 */
.vjs-theme-forest .vjs-play-button:focus,
.vjs-theme-forest .vjs-play-button:focus-visible,
.vjs-theme-forest .vjs-rewind-10-button:focus,
.vjs-theme-forest .vjs-rewind-10-button:focus-visible,
.vjs-theme-forest .vjs-forward-10-button:focus,
.vjs-theme-forest .vjs-forward-10-button:focus-visible,
.vjs-theme-forest .vjs-volume-button:focus,
.vjs-theme-forest .vjs-volume-button:focus-visible,
.vjs-theme-forest .vjs-picture-in-picture-control:focus,
.vjs-theme-forest .vjs-picture-in-picture-control:focus-visible,
.vjs-theme-forest .vjs-fullscreen-control:focus,
.vjs-theme-forest .vjs-fullscreen-control:focus-visible,
.vjs-theme-forest.vjs-fullscreen .vjs-play-button:focus,
.vjs-theme-forest.vjs-fullscreen .vjs-play-button:focus-visible,
.vjs-theme-forest.vjs-fullscreen .vjs-rewind-10-button:focus,
.vjs-theme-forest.vjs-fullscreen .vjs-rewind-10-button:focus-visible,
.vjs-theme-forest.vjs-fullscreen .vjs-forward-10-button:focus,
.vjs-theme-forest.vjs-fullscreen .vjs-forward-10-button:focus-visible,
.vjs-theme-forest.vjs-fullscreen .vjs-volume-button:focus,
.vjs-theme-forest.vjs-fullscreen .vjs-volume-button:focus-visible,
.vjs-theme-forest.vjs-fullscreen .vjs-picture-in-picture-control:focus,
.vjs-theme-forest.vjs-fullscreen .vjs-picture-in-picture-control:focus-visible,
.vjs-theme-forest.vjs-fullscreen .vjs-fullscreen-control:focus,
.vjs-theme-forest.vjs-fullscreen .vjs-fullscreen-control:focus-visible {
    outline: none;
    border: none; /* 防止可能的边框闪烁 */
    box-shadow: none; /* 防止可能的阴影效果 */
    user-select: none;
    cursor: default;
}

/* 时间标签 样式 */
.vjs-theme-forest .vjs-current-time,
.vjs-theme-forest .vjs-time-divider,
.vjs-theme-forest .vjs-duration {
    display: inline-block;
    padding: 0;
}

/* 确保 vjs-icon-placeholder 填充整个按钮并居中 */
.vjs-theme-forest .vjs-rewind-10-button .vjs-icon-placeholder,
.vjs-theme-forest .vjs-forward-10-button .vjs-icon-placeholder,
.vjs-theme-forest .vjs-play-button .vjs-icon-placeholder {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 100%;
    height: 100%;
    position: relative;
    top: 0;
    left: 0;
    margin: 0;
    padding: 0;
}

/* 控件顺序 */
.vjs-theme-forest .vjs-rewind-10-button {
    order: 0;
}
.vjs-theme-forest .vjs-play-button {
    order: 1;
}
.vjs-theme-forest .vjs-forward-10-button {
    order: 2;
}
.vjs-theme-forest .vjs-progress-control {
    order: 3;
}
.vjs-theme-forest .vjs-current-time {
    order: 4;
}
.vjs-theme-forest .vjs-time-divider {
    order: 5;
}
.vjs-theme-forest .vjs-duration {
    order: 6;
}
.vjs-theme-forest .vjs-volume-button {
    order: 7;
}
.vjs-theme-forest .vjs-setting-button {
    order: 8;
}
.vjs-theme-forest .vjs-picture-in-picture-control {
    width: 30px;
    order: 9;
}
.vjs-theme-forest .vjs-fullscreen-control {
    width: 30px;
    order: 10;
}

```

