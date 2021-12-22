这篇文章来介绍 web 中如何使用 `video`，并制作一个简单的视频播放器。

---

[[_TOC_]]

## 介绍

以前想在网站放播放视频，就需要安装 `flash` 插件，但是 `flash` 占用系统资源高。而且它是 `Adobe` 一项封闭的商业应用，内置 `flash` 有可能引入相关的安全漏洞，苹果更是大力反对 `falsh`。

现在视频网站几乎都用 `html 5` 播放视频，它占用资源小更省电、省流量，是一项完全免费并且开放的新标准。无需安装任何插件直接使用 `video` 标签就行，而且它的兼容性也非常好，所有主流浏览器都支持。

![](https://pubimg.xingren.com/5fa79d3d-00a2-491d-8d6e-8a02d6874831.jpg)

## video 标签

```html
<video controls poster="/poster.jpg">
    <source
        src="https://interactive-examples.mdn.mozilla.net/media/examples/friday.mp4"
        type="video/mp4"
    >
    <source src="/other.webm" type="video/webm">
    <track 
        src="https://interactive-examples.mdn.mozilla.net/media/examples/friday.vtt"
        kind="captions" 
        srclang="en"
        label="显示在这"
        default
    >
    浏览器不支持 video 标签
</video>
```

### video

video 除了上面展示的两个还有很多属性。

| 事件 | 描述 |
| --- | --- |
| controls | 使用浏览器默认的视频控制器 |
| controlslist | 当浏览器显示自己的控件集，控制浏览器的控件，比如不显示下载按钮 |
| poster | 一个海报帧的 `URL`，用于在用户播放或者跳帧之前展示 |
| src | 视频的地址，也可以使用 `source` 代替 |
| muted | 静音 |
| autoplay | 自动播放，但是有些浏览器考虑用户体验，需要与 `muted` 一起使用，或[一些其他方式](https://developers.google.com/web/updates/2017/09/autoplay-policy-changes#new-behaviors) |
| loop | 循环播放 |
| width/height | 设置视频的宽高 |
| crossorigin | 是否必须用到跨域请求，`anonymous` 值为执行跨域请求但不发送 `cookie` 等凭证，`use-credentials` 发送凭证 |
| preload | 预加载，它有 3 个值，默认值由浏览器决定，有些浏览器会在特殊情况忽略此属性，如 3G 网下|

`payload` 3 个值如下：

        none      不进行预加载
        metadata  预加载视频元数据
        auto      预加载整个视频

### source

> 上面 `video` 标签下的 `source` 是用来指定视频的地址，如果浏览器不支持这个格式它就会查看下一个 `source`，也可以简单的使用 `video` 的 `src` 属性。

`<video src="http://video.mp4">当浏览器不支持 video 会显示</video>`

> 使用 `Media Fragments API` 可以为视频添加开始和结束时间。

`<source src="video.mp4#t=2,5" type="video/mp4">`

视频将在 2 秒播放，5 秒结束。它的格式为 `#t=[start_time][,end_time]`，需要确保服务器支持 `Range Requests`。

### track

`track` 元素使用 [WebVTT](https://developer.mozilla.org/zh-CN/docs/Web/API/WebVTT_API) 格式来显示字幕。一个媒体元素的任意两个 `track` 子元素不能有相同的 `kind`, `srclang`, 和 `label`属性。

- `default` 指定 `track` 默认启用
- `label` 给浏览器使用的 `text track` 的标题，这种标题是用户可读的

![](https://pubimg.xingren.com/2937ab31-989f-4273-b30b-98695b7d2f05.jpg)

- `src` 字幕地址
- `srclang` 文本数据的语言，中文是 `zh`，如果 `kind` 属性被设为 `subtitles`, 那么必须定义此属性。
- `kind` 定义 `text track` 应该如何使用。如果省略，默认就是 `subtitles`，它有以下属性值：

        subtitles     字幕给观影者看不懂的内容提供解释
        captions      隐藏式字幕提供了音频的转录甚至是翻译
        descriptions  视频内容的文本描述
        chapters      章节标题用于用户浏览媒体资源的时候
        metadata      脚本使用的 track 用户不可见

## JS 中的 video

在 `js` 中，通过 `document.querySelector('video')` 等方式获取 `video` 元素，就可以操作视频行为了，下面介绍 `video` 常用的事件、属性和方法。

### 事件

> 加载相关

|  事件   | 描述 |
|  ---  | ---  |
| loadstart  | 在媒体开始加载时触发 |
| loadedmetadata  | 媒体的元数据已经加载完毕，比如视频的宽高，长度等信息 |
| loadeddata  | 媒体的第一帧已经加载完毕 |
| canplay  | 在媒体数据已经有足够的数据可供播放时触发 |
| canplaythrough  | 媒体可以在保持当前的下载速度的情况下不被中断地播放完毕时触发 |
| progress  | 告知媒体相关部分的下载进度时周期性地触发 |
| durationchange  | 元信息已载入或已改变，表明媒体长度发生改变。比如媒体已被加载足够的长度从而得知总长度时 |从而得知总长度时

> 错误或特殊情况

| 事件 | 描述 |
| --- | --- |
| error | 在发生错误时触发 |
| stalled | 在尝试获取媒体数据，但数据不可用时触发 |
| suspend | 媒体资源加载终止时触发，这可能是因为下载已完成或因为其他原因 |

> 播放时

| 事件 | 描述 |
| --- | --- |
| playing | 在媒体开始播放时触发可能是初次播放、暂停后恢复或结束后重新开始 |
| play | 在媒体回放被暂停后再次开始时触发 |
| pause | 播放暂停时触发 |
| timeupdate | 元素的 `currentTime` 属性表示的时间已经改变，播放时周期触发 |
| ended | 播放结束时触发 |
| abort | 在播放被终止时触发，比如当播放中的视频重新开始播放时 |
| waiting | 操作延迟，等待另一操作完成时触发的事件 |
| ratechange | 播放速率改变 |
| volumechange | 音频音量改变时触发，也可能是 `muted` 属性改变 |
| seeking | 跳跃操作开始时触发 |
| seeked | 跳跃操作完成时触发 |

### 属性

通过 `video` 元素，我们可以获取上面提到的属性，也可以改变它来操作视频，比如设置 `video.muted=true` 设置静音。

| 属性 | 描述 |
| --- | --- |
| videoWidth / videoHeight | 返回视频的宽高（`width/height` 属性可能被 css 影响） |
| currentSrc | 返回浏览器当前使用来源的 `src` |
| currentTime | 获取或设置视频播放位置 |
| volume | 获取或设置音量 |
| playbackRate | 获取或设置播放速率，可以设置为负数 |
| paused | 是否已暂停 |
| defaultMuted | 获取或设置是否默认静音 |
| defaultPlaybackRate | 获取或设置默认速率 |
| error | 获取错误信息 |
| duration | 返回视频长度 |
| seeking | 返回是否在跳跃中 |
| buffered | [TimeRanges](https://developer.mozilla.org/zh-CN/docs/Web/API/TimeRanges) 对象，读取到哪段时间范围内的媒体被缓存 |
| played | [TimeRanges](https://developer.mozilla.org/zh-CN/docs/Web/API/TimeRanges) 对象，表示视频一播放的范围 |
| textTracks | 包含 video 字幕的数组 |

`video` 元素上还有 `readyState` 属性，表示视频当前的状态，它的值 `0` 到 `4` 的数字，`video` 上有相关描述的常量属性。

| video 常量属性 | 描述 |
| --- | --- |
| HAVE_NOTHING | 0 没有信息，视频未准备好 |
| HAVE_METADATA | 1 视频元数据已准备 |
| HAVE_CURRENT_DATA | 2 视频当前位置数据可用，但是下一帧数据没有 |
| HAVE_FUTURE_DATA | 3 当前和至少下一帧数据可用 |
| HAVE_ENOUGH_DATA | 4 有足够的数据可以播放 |

还有一个 `networkState` 属性表示当前网络状况。

| video 常量属性 | 描述 |
| --- | --- |
| NETWORK_EMPTY | 0 还没初始化 |
| NETWORK_IDLE | 1 处于活跃状态，但还没使用网络 |
| NETWORK_LOADING | 2 浏览器在下载数据 |
| NETWORK_NO_SOURCE | 3 没有找到数据源 |

### 方法

| video 方法 | 描述 |
| --- | --- |
| load() | 在没有开始播放的情况下加载或重新加载视频来源，比如修改 `src` |
| play() | 开始播放 |
| pause() | 暂停播放 |
| canPlayType(type) | 检查浏览器支持的视频格式 |
| addTextTrack(kind,label,lang) | 创建和返回 [TextTrack](https://developer.mozilla.org/zh-CN/docs/Web/API/TextTrack) 对象|

其中 `canPlayType` 方法参数接收 `mime-type` 字符串或在加上可选的编解码器，返回如下 3 个值。

| canPlayType 返回值 | 描述 |
| --- | --- |
| ''（空字符串） | 容器和（或编解码器）不受支持 |
| maybe | 容器和编解码器可能受支持，但是浏览器需要下载部分视频才能确认 |
| probably | 格式似乎受支持 |

它的参数可能是：

- video/ogg
- video/mp4
- video/webm
- video/ogg; codecs="theora, vorbis"
- video/mp4; codecs="avc1.4D401E, mp4a.40.2"
- video/webm; codecs="vp8.0, vorbis"

## 视频播放器

```html
<div class="player player-loading">
    <video
      playsinline
      x5-playsinline
      src="https://interactive-examples.mdn.mozilla.net/media/examples/flower.webm"
    />
    <div class="loading"></div>
    <div class="controls">
      <div class="bar">
        <div class="bar_buffered"></div>
        <div class="bar_played"></div>
      </div>
      <div class="control_items">
        <button class="play_btn"></button>
        <div class="time">0 / 0</div>
        <button class="fullscreen"></button>
      </div>
    </div>
</div>
```

其中 `video` 设置了 `playsinline` 属性，是为 `IOS` 视频播放时不自动进入全屏。`x5-playsinline` 是让腾讯 `x5`浏览器内核不自动进入全屏。`X5` 是腾讯基于 `Webkit` 开发的浏览器内核，应用于 `Android` 端的微信、`QQ` 等应用。更多关于 `x5 video` 属性[参考这里](https://x5.tencent.com/tbs/guide/video.html)。

```css
.player-loading .loading { opacity: 1; }
.player-playing .play_btn::after { content: '暂停'; }
.player-controls-hide { cursor: none; }
.player-controls-hide .controls { opacity: 0; }
.player-fullscreen .fullscreen:after { content: '退出全屏'; }
```

加点简单的 CSS，这里主要关注使用 JS 实现功能的核心代码，样式部分就省略了。

```javascript
const player = document.querySelector('.player')
const video = document.querySelector('video')
```

### 控制器的显示和隐藏

关于控制器显示/隐藏需要注意几点：

1. 当视频没有播放时控制器要显示出来
2. 当视频播放时需要等一会儿再将控制器隐藏
3. 当视频播放时点击鼠标或移动鼠标需要将控制器显示
4. 当视频播放结束时控制器显示出来

```javascript
let controlsTimer = null
function showControls() {
    clearTimeout(controlsTimer)
    player.classList.remove('player-controls-hide')
    updatePlayedBarAndTime() // 更新进度条，查看下一小节
}
function delayHideControls() {
    showControls();
    controlsTimer = setTimeout(() => {
        if (video.played.length && !video.paused) {
            player.classList.add('player-controls-hide')
        }
    }, 3000)
}
player.addEventListener('click'， delayHideControls)
player.addEventListener('mousemove'， delayHideControls)
video.addEventListener('play', delayHideControls)
video.addEventListener('pause'， showControls)
```

这里主要是通过判断 `video.played.length && !video.paused` 来判断是否隐藏控制器，也就是视频播放过并且视频正在播放，这里没有监听 `ended` 事件，因为播放完毕也会触发 `pause` 事件。

### 进度条和时间显示

```javascript
const playedBar = document.querySelector('.bar_played')
const bufferedBar = document.querySelector('.bar_played')
const time = document.querySelector('.time')

function updatePlayedBarAndTime(currentTime, percentage) {
    currentTime = currentTime == null ? Math.round(video.currentTime) : currentTime
    percentage = percentage == null ? Math.min(video.currentTime / video.duration, 1) : percentage
    time.textContent = currentTime + ' / ' + Math.round(video.duration)
    playedBar.style.transform = 'scaleX('+ percentage +')'
}

video.addEventListener('durationchange', updatePlayedBarAndTime)
video.addEventListener('timeupdate', () => {
  if (!player.classList.contains('player-controls-hide')) {
    updatePlayedBarAndTime()
  }
})
video.addEventListener('progress', () => {
  const percentage = video.buffered.length ? video.buffered.end(video.buffered.length - 1) / video.duration : 0;
  bufferedBar.style.transform = 'scaleX('+ Math.min(percentage, 1) +')'
})
```

通过监听 `timeupdate` 事件在控制器显示的情况下更新 DOM，`progress` 事件更新视频缓存进度条 `video.buffered.end(video.buffered.length - 1)` 可以获取最后一段 `TimeRange` 的结束时间。

### 播放控制

```javascript
const playPauseBtn = document.querySelector('.play_btn')

playPauseButton.addEventListener('click', evt => {
  evt.stopPropagation() // 为了防止父级处理事件，比如视频控件
  if (video.paused) {
    video.play()
  } else {
    video.pause()
  }
})
video.addEventListener('play', () => {
  player.classList.add('player-playing')
})
video.addEventListener('pause', () => {
  player.classList.remove('player-playing')
})
video.addEventListener('ended', () => {
  player.classList.remove('player-playing')
  video.currentTime = 0
})
```

这里没有在按钮点击事件中处理视频播放暂停 UI 变化而是在 `video` 事件中处理，是为了让 UI 更精准，不止有这个按钮会控制视频播放和暂停。这里没有展示控制视频播放速率，控制播放速率直接设置 `video.playbackRate` 就行。

### 控制进度条

```javascript
const bar = document.querySelector('.bar')
const { width: barWidth, x: barLeft } = bar.getBoundingClientRect()
let barPending = false
let barLastX = 0
let videoSeekTime = -1

function onBarPointerStart(evt) {
    evt.preventDefault()
    bar.setPointerCapture(ev.pointerId)
    bar.addEventListener('pointermove', this.onPointerMove, true)
    video.pause()
    barLastX = evt.pageX
    const [percentage, seekTime] = calcPercentageAndSeekTime(evt.pageX)
    updatePlayedBarAndTime(seekTime, percentage)
}
function onBarPointerMove(evt) {
    evt.preventDefault()
    barLastX = evt.pageX
    if (barPending) return
    barPending = true
    requestAnimationFrame(handleBarMove)
}
function onBarPointerEnd(evt) {
    evt.preventDefault()
    bar.releasePointerCapture(evt.pointerId)
    bar.removeEventListener('pointermove', onBarPointerMove, true)
    if (videoSeekTime > 0) video.curentTime = videoSeekTime
    video.play()
}
function calcPercentageAndSeekTime() {
    const percentage = Math.max(Math.min((barLastX - barLeft) / barWidth, 1), 0)
    videoSeekTime = percentage * video.duration
    return [percentage, videoSeekTime]
}
function handleBarMove() {
    if (!barPending) return
    const [percentage, seekTime] = calcPercentageAndSeekTime()
    updatePlayedBarAndTime(seekTime, percentage)
    barPending = false
}

bar.addEventListener('pointerdown', onBarPointerStart, true)
bar.addEventListener('pointerup', onBarPointerEnd, true)
bar.addEventListener('pointercancel', onBarPointerEnd, true)
```

这里使用 `PointerEvent` 来实现监听控制条的拖拽，它的好处是兼容 PC 的鼠标拖拽和移动的手势拖拽，结束时通过设置 `video.curentTime` 来跳到指定时间点。控制音量与这个相似。

### 全屏

```javascript
const isIos = /(iPad|iPhone|iPod)/gi.test(navigator.platform)
const fullscreenBtn = document.querySelector('.fullscreen')
fullscreenBtn.addEventListener('click', evt => {
    evt.stopPropagation()
    if (document.fullscreenElement) {
        if (isIos) {
            video.webkitExitFullscreen()
        } else {
            document.exitFullscreen()
        }
    } else {
        if (isIos) {
            video.webkitEnterFullscreen()
        } else {
            player.requestFullscreen()
        }
    }
})
document.addEventListener('fullscreenchange', () => {
  player.classList.toggle('player-fullscreen', document.fullscreenElement)
})
```

这里需要注意对 IOS 的兼容。对于老浏览器请求、退出和全局全屏元素都需要添加浏览器前缀。想要跨浏览器兼容的全屏 API 可以使用 [screenfull.js](https://github.com/sindresorhus/screenfull.js)。

### Loading 处理

```javascript
video.addEventListener('canplay', () => {
    player.classList.remove('player-loading')
})
video.addEventListener('waiting', () => {
    player.classList.add('player-loading')
    
    const startWaitingTime = video.currentTime
    const checkCanPlay = () => {
      if (startWaitingTime !== video.currentTime) {
        player.classList.remove('player-loading')
        video.removeEventListener('timeupdate', checkCanPlay)
      }
    }
    video.addEventListener('timeupdate', checkCanPlay)
})
```

并不是所有浏览器在 `waiting` 事件触发后，当可播放时还会触发 `canplay` 事件。所以这里通过 `timeupdate` 事件来比对时间，确认已经可以播放视频了。

不过并不是所有浏览器能正确触发 `waiting` 事件，所以我们需要自己检测是否停住等待加载视频。

```javascript
let prevCurrentTime = 0
let playerLoading = false
let loadingTimer = setInterval(() => {
    const currentTime = video.currentTime
    if (!playerLoading && prevCurrentTime === currentTime && !video.paused) {
        player.classList.add('player-loading')
        playerLoading = true
    } else if (playerLoading && currentTime !== prevCurrentTime) {
        player.classList.remove('player-loading')
        playerLoading = false
    }
    prevCurrentTime = currentTime
}, 100)
```

这里实现比较简单主要是通过定时器去不断获取视频 `currentTime` 通过比对它来确定视频是否卡住等待播放。还可以将上面监听 `progress` 事件获取到的 `buffered` 时间，比对 `currentTime` 来决定是否去除 `player-loading`。

### 字幕

```javascript
if (video.textTracks && video.textTracks.length) {
    const defaultLang = navigator.language?.split('-')[0] || 'zh'
    let saw = false
    video.textTracks.forEach(track => {
        if (track.language === defaultLang) {
            track.mode = 'showing'
            saw = true
        } else {
            track.mode = 'hidden'
        }
    })
    if (!saw) video.textTracks[0].mode = 'showing'
    // 这里只是默认显示一个 textTrack
    // 应该有个菜单可以让用户选择字幕，这里就省略了
}
```

```css
::cue {
  color:#fff;
  background: transparent;
  text-shadow: 2px 3px 5px rgba(0, 0, 0, .5);
}
```

上面我们通过控制 `textTrack` 的 `mode` 属性控制显示哪个 `textTrack`，通过 `::cue` 伪类设置字幕样式，但是如果要更精准的控制字幕，我们就需要自己使用 DOM 元素来显示字幕。

```javascript
if (video.textTracks && video.textTracks.length) {
    const container = document.querySelector('.player_subtitle') // 字幕容器
    const track = video.textTracks[0]
    track.mode = 'hidden'
    track.oncuechange = () => {
        const cue = track.activeCues[0] // cue 代表一条字幕
        container.innerHTML = ''
        if (cue) {
            const subtitle = document
                                .createElement('div')
                                .appendChild(cue.getCueAsHTML())
                                .innerHTML
            // 因为 getCueAsHTML 返回的是 document-fragment
            
            container.innerHTML = subtitle.split(/\r\n|\n|\r/).map(t => `<p>${t}</p>`).join('')
        }
    }
}
```

通过监听 `oncuechange` 获取当前 `cue`，然后获取它的内容，然后加入到自定义字幕容器中。 `cue` 对象长下面这样。

![](https://pubimg.xingren.com/0d68eec4-b9c4-4679-9e38-9ac8708b11fa.jpg)

### 画中画

Picture-in-Picture（画中画）可以让视频弹出来小屏播放，就算最小化浏览器或者切换其他 tab 页也可以播放。

![](https://pubimg.xingren.com/a3688f5c-884d-42b5-8939-3d8bd371406a.jpg)

```javascript
if (document.pictureInPictureEnabled) { // 是否启用画中画
    const pip = document.querySelector('.pip')
    pip.style.display = 'block'
    pip.addEventListener('click', () => {
        if (document.pictureInPictureElement) { // 当前画中画的元素
            video.exitPictureInPicture()
        } else {
            video.requestPictureInPicture() // 返回 Promise，里面是 pipWindow
        }
    })

    video.addEventListener('enterpictureinpicture', pipWindow => {
        console.log(pipWindow.width, pipWindow.height)
        player.classList.add('player-pip')
    })

    video.addEventListener('leavepictureinpicture', () => {
       player.classList.remove('player-pip')
    })
}
```

### 截图

视频截图是通过在 `canvas` 渲染视频实现的。

```javascript
const screenshotBtn = document.querySelector('.screenshot_btn')

screenshotBtn.addEventListener('click', () => {
    const canvas = document.createElement('canvas')
    canvas.width = video.videoWidth;
    canvas.height = video.videoHeight
    canvas.getContext('2d').drawImage(video, 0, 0, canvas.width, canvas.height)

    const fileName = video.currentTime + '.png'
    canvas.toBlob(blob => {
        const url = URL.createObjectURL(blob)
        const a = document.createElement('a')
        a.href = url
        a.download = fileName
        a.style.display = 'none'
        document.body.appendChild(a)
        a.click()
        document.body.removeChild(a)
        URL.revokeObjectURL(url)
    }, 'image/png')
})
```

## 源码

上面的功能只展示实现相关功能的核心思想，忽略了一些细节、样式和浏览器兼容问题。只要再添加一点细节和 CSS 最终播放器就长这样。

![](https://pubimg.xingren.com/4e036999-1838-447f-b2a7-a92dcdb31872.jpg)

完整的视频播放器源码：[https://github.com/woopen/RPlayer](https://github.com/woopen/RPlayer)（现在播放器功能还比较简单，还在不断完善中）

## 总结

这篇文章介绍几乎所有的 video 的 API（需要实现功能时可以直接参考是否有相关 API）。然后用这些 API 来展示如何实现一些播放器的核心功能。

## 参考

- [The Video Embed element](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/video)
- [HTML Audio/Video DOM Reference](http://www.w3schools.com/tags/ref_av_dom.asp)
- [Mobile Web Video Playback](https://developers.google.com/web/fundamentals/media/mobile-web-video-playback)
