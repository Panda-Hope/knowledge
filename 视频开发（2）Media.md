如果检查主流视频网站的视频，就会发现网站的 `video` 元素的 `src` 属性都是 `blob` 开头的字符串。

![](https://pubimg.xingren.com/572a2fc5-3169-45b2-88d7-0267f8d0cbea.jpg)

为什么视频链接前面会有 `blob` 前缀？这是因为视频网站使用了这篇文章要讲的 MSE 来播放视频。

---

[[_TOC_]]

## Media Source Extensions

虽然 `video` 很强大，但是还有很多功能 `video` 并不支持，比如直播，即时切换视频清晰度，动态更新音频语言等功能。

MSE（Media Source Extensions）就来解决这些问题，它是 W3C 的一种规范，如今大部分浏览器都支持。

![](https://pubimg.xingren.com/283e3448-3f42-4cc1-b321-25c1e2f29ff3.jpg)

它使用 `video` 标签加 JS 来实现这些复杂的功能。它将 `video` 的 `src` 设置为 `MediaSource` 对象，然后通过 HTTP 请求获取数据，然后传给 `MeidaSource` 中的 `SourceBuffer` 来实现视频播放。

![](https://pubimg.xingren.com/e18fcb8c-d864-4ad1-b5a3-efa59e526638.jpg)

如何将 `MediaSource` 和 `video` 元素连接呢？这就需要用到 `URL.createObjectURL` 它会创建一个 `DOMString` 表示指定的 `File` 对象或 `Blob`（二进制大对象） 对象。这个 URL 的生命周期和创建它的窗口中的 `document` 绑定。

```javascript
const video = document.querySelector('video')
const mediaSource = new MediaSource()

mediaSource.addEventListener('sourceopen', ({ target }) => {
    URL.revokeObjectURL(video.src)
    const mime = 'video/webm; codecs="vorbis, vp8"'
    const sourceBuffer = target.addSourceBuffer(mime) // target 就是 mediaSource
    fetch('/static/media/flower.webm')
        .then(response => response.arrayBuffer())
        .then(arrayBuffer => {
            sourceBuffer.addEventListener('updateend', () => {
                if (!sourceBuffer.updating && target.readyState === 'open') {
                    target.endOfStream()
                    video.play()
                }
            })
            sourceBuffer.appendBuffer(arrayBuffer)
        })
})

video.src = URL.createObjectURL(mediaSource)
```

`addSourceBuffer` 方法会根据给定的 MIME 类型创建一个新的 `SourceBuffer` 对象，然后会将它追加到 `MediaSource` 的 `SourceBuffers` 列表中。

我们需要传入相关具体的编解码器（codecs）字符串，这里第一个是音频（vorbis），第二个是视频（vp8），两个位置也可以互换，知道了具体的编解码器浏览器就无需下载具体数据就知道当前类型是否支持，如果不支持该方法就会抛出 `NotSupportedError` 错误。更多关于媒体类型 MIME 编解码器可以参考 [RFC 4281](https://tools.ietf.org/html/rfc4281)。

这里还在一开始就调用了 `revokeObjectURL`。这并不会破坏任何对象，可以在 `MediaSource` 连接到 `video` 后随时调用。 它允许浏览器在适当的时候进行垃圾回收。

视频并没有直接推送到 `MediaSource` 中，而是 `SourceBuffer`，一个 `MeidaSource` 中有一个或多个 `SourceBuffer`。每个都与一种内容类型关联，可能是视频、音频、视频和音频等。

![](https://pubimg.xingren.com/38187141-0900-4b87-8d49-33d71c58fd89.jpg)

## 视频格式

HTML5 标准指定时，想指定一种视频格式作为标准的一部分，所有浏览器都必须实现。但是对于 `H.264` 视频编码各个厂商产生的争论，主要是 `H.264` 非常强大（高画质、高压缩比、成熟的编解码器...），但是它也要高昂的授权费。Mozilla 这类免费浏览器，并没有从其开发的浏览器上获得直接收入，但是让 `H.264` 加入标准，它就要支付相应的授权费，所有认为是不可接受的。于是后来放弃了视频格式指定的统一，浏览器厂商可以自由选择支持的格式。

不过现在所有主流浏览器都支持 `H.264` 编码格式的视频，所有选择视频编码时优先选择 `H.264` 编码。

## MediaSource API

### `MediaSource` 属性

| 属性 | 描述 |
| --- | --- |
| sourceBuffers | 返回包含 `MediaSource` 所有 `SourceBuffer` 的 [SourceBufferList](https://developer.mozilla.org/en-US/docs/Web/API/SourceBufferList) 对象 |
| activeSourceBuffers | 上个属性的子集，返回当前选择的视频轨道，启用的音频规定和显示或隐藏的文本轨道 |
| duration  | 获取或者设置当前媒体展示的时长，负数或 `NaN` 时抛出 `InvalidAccessError`，<br>`readyState` 不为 `open` 或 `SourceBuffer.updating` 属性为 `true` 时抛出 `InvalidStateError` |
| readyState | 表示 `MediaSource` 的当前状态 |

`readyState` 有以下值：

- `closed` 未附着到一个 `media` 元素上
- `open` 已附着到一个 `media` 元素并准备好接收 `SourceBuffer` 对象
- `ended` 已附着到一个 `media` 元素，但流已被 `MediaSource.endOfStream()` 结束

### `MediaSource` 方法

| 方法 | 描述 |
| --- | --- |
| addSourceBuffer(mime) | 根据给定 MIME 类型创建一个新的 SourceBuffer 对象，将它追加到 MediaSource 的 SourceBuffers 列表中 |
| removeSourceBuffer(sourceBuffer) | 移除 MediaSource 中指定的 SourceBuffer。如果不存在则抛出 NotFoundError 异常 |
| endOfStream(endOfStreamError) | 向 MediaSource 发送结束信号，接收一个 DOMString 参数，表示到达流末尾时将抛出的错误 |
| setLiveSeekableRange(start, end) | 设置用户可以跳跃的视频范围，参数是以秒单位的 `Double` 类型，负数等非法参数会抛出异常 |
| clearLiveSeekableRange | 清除上次设置的 `LiveSeekableRange` |

其中 `addSourceBuffer` 可能会抛出一下错误：

| 错误 | 描述 |
| --- | --- |
| InvalidAccessError | 提交的 mimeType 是一个空字符串 |
| InvalidStateError | 	MediaSource.readyState 的值不等于 open |
| NotSupportedError | 当前浏览器不支持的 mimeType |
| QuotaExceededError | 浏览器不能再处理 SourceBuffer 对象 |

`endOfStream` 的参数可能是如下两种字符串：

- `network` 终止播放并发出网络错误信号
- `decode` 终止播放并发出解码错误信号

当 `MediaSource.readyState` 不等于 `open` 或有 `SourceBuffer.updating` 等于 `true`，调用 `endOfStream` 会抛出 `InvalidStateError` 异常。

它还有一个静态方法

| 静态方法 | 描述 |
| --- | --- |
| isTypeSupported(mime) | 是否支持指定的 mime 类型，返回 `true` 表示可能支持并不能保证 |

### `MediaSource` 事件

| 错误 | 描述 |
| --- | --- |
| sourceopen | `readyState` 从 `closed` 或 `ended` 到 `open` |
| sourceended | `readyState` 从 `open` 到 `ended` |
| sourceclose | `readyState` 从 `open` 或 `ended` 到 `closed` |

## SourceBuffer API

`SourceBuffer` 通过 `MediaSource` 将一块块媒体数据传递给 `meida` 元素播放。

### `SourceBuffer` 属性

| 属性 | 描述 |
| --- | --- |
| mode | 控制处理媒体片段序列，`segments` 片段时间戳决定播放顺序，`sequence` 添加顺序决定播放顺序，<br>在 `MediaSource.addSourceBuffer()` 中设置初始值，如果媒体片段有时间戳设置为 `segments`，否则 `sequence`。<br>自己设置时只能从 `segments` 设置为 `sequence`，不能反过来。 |
| updating | 是否正在更新，比如 `appendBuffer()` 或 `remove()` 方法还在处理中 |
| buffered | 返回当前缓冲的 [TimeRanges](https://developer.mozilla.org/en-US/docs/Web/API/TimeRanges) 对象|
| timestampOffset | 控制媒体片段的时间戳偏移量，默认是 0 |
| audioTracks | 返回当前包含的 AudioTrack 的 [AudioTrackList](https://developer.mozilla.org/en-US/docs/Web/API/AudioTrackList) 对象|
| videoTracks | 返回当前包含的 VideoTrack 的 [VideoTrackList](https://developer.mozilla.org/en-US/docs/Web/API/VideoTrackList) 对象 |
| textTracks | 返回当前包含的 TextTrack 的 [TextTrackList](https://developer.mozilla.org/en-US/docs/Web/API/TextTrackList) 对象 |
| appendWindowStart | 设置或获取 append window 的开始时间戳 |
| appendWindowEnd | 设置或获取 append window 的结束时间戳 |

> append window 是一个时间戳范围来过滤 `append` 的编码帧。在范围内的编码编码帧允许添加到 `SourceBuffer`，之外的会被过滤。

### `SourceBuffer` 方法

| 方法 | 描述 |
| --- | --- |
| appendBuffer(source) | 添加媒体数据片段（`ArrayBuffer` 或 `ArrayBufferView`）到 `SourceBuffer` |
| abort | 中断当前片段，重置段解析器，可以让 `updating` 变成 `false` |
| remove(start, end) | 移除指定范围的媒体数据 |

### `SourceBuffer` 事件

| 方法 | 描述 |
| --- | --- |
| updatestart | `updating` 从 `false` 变为 `true` 时 |
| update | `append` 或 `remove` 已经成功完成，`updating` 从 `true` 变为 `false` |
| updateend | `append` 或 `remove` 已经结束，在 `update` 之后触发 |
| error | `append` 时发生了错误，`updating` 从 `true` 变为 `false` |
| abort | `append` 或 `remove` 被 `abort()` 方法中断，`updating` 从 `true` 变为 `false` |

## 注意事项

1. **几乎所有 `MediaSource` 方法调用或设置属性，在 `MediaSource.readyState` 不是 `open` 时会抛出 `InvalidStateError` 错误，应该在调用方法或设置属性前查看当前状态，即使是在事件回调中，因为可能在回调执行之前改变了状态。此外 `endOfStream` 方法还会因为 `SourceBuffer` 的 `updating` 为 `true` 时也抛出该异常**
2. **在调用 `SourceBuffer` 方法或设置属性时，应该检查 `SourceBuffer` 是否为 `false`**
3. **当 `MediaSource.readyState` 的值是 `ended` 时，调用 `appendBuffer()` 和 `remove()` 或设置 `mode` 和 `timestampOffset` 时，将会让 `readyState` 变为 `open`，并触发 `sourceopen` 事件，所有应该要有处理多个 `sourceopen` 事件准备**

## Initialization Segment

如果随便找一个 `mp4` 文件，使用上面那个例子播放，就会发现播放不了。这是因为 `SourceBuffer` 接收两种类型的数据：

1. `Initialization Segment` 视频的初始片段，其中包含媒体片段序列进行解码所需的所有初始化信息。
2. `Media Segment` 包含一部分媒体时间轴的打包和带时间戳的媒体数据。

MSE 需要使用 `fmp4 (fragmented MP4)` 格式，`MP4` 文件使用面向对象格式其中包含 `Boxes (或叫 Atoms)`，可以使用 [这个网站](http://mp4parser.com/) 查看 `Mp4` 文件信息。

![](https://pubimg.xingren.com/09e5dc37-03b7-4963-8995-79cc8b14b9c3.jpg)

这是一个普通的 MP4 文件，可以看到它有一个很大的 `mdat` （电影数据）`box`。

![](https://pubimg.xingren.com/6e8ac66a-4b28-4561-aee9-94353db0bc11.jpg)

这是 fragmented MP4 的截图，ISO BMFF 初始化段定义为单个文件类型框（File Type Box `ftyp`）后跟单个电影标题框（Movie Header Box `moov`），更多信息可以查看 [ISO BMFF Byte Stream Format](https://www.w3.org/TR/mse-byte-stream-format-isobmff/)。

moov 只包含一些视频基础的信息（类型，编码器等），moof 存放样本位置和大小，moof 框后都有一个 mdat，其中包含如前面的 moof 框中所述的样本。

要查看当前视频是不是 fmp4，就可以看 ftyp 后面是不是跟着 moov，然后是 moof mdat 对就行了。

要将普通 MP4 转换成 FMP4 可以下载 [Bento4](https://www.bento4.com/downloads/)。然后用命令行输入如下命令就可以了。

```bash
mp4fragment ./friday.mp4 ./friday.fmp4
```

Bento4 `/bin` 目录中有非常多好用的 mp4 工具，`/utils` 目录中都是 `python` 实用脚本。

## 工具

除了上面介绍的 Bento4，还有很多其他好用的工具。有了下面的工具，就可以快速制作 MSE 实践的视频素材了。

### chrome://media-internals/

`chrome://media-internals/` 是 chrome 浏览器用来调试多媒体的工具，直接在地址栏输入该值就可以。

![](https://pubimg.xingren.com/6f47e5c1-60d8-4504-8696-a65c3c9b2324.jpg)

### Shaka Packager

[Shaka Packager](https://github.com/google/shaka-packager/releases) 是 Google 出的一个小巧视频工具，它只有 5M 左右，它可以用来查看视频信息，分离音频和视频，还支持 HLS 和 DASH。

它的命令行使用格式如下：

```
packager stream_descriptor[,stream_descriptor] [flags]
```

比如用它查看视频信息：

```
packager in=friday.mp4 --dump_stream_info # 查看视频信息

File "friday.mp4":
Found 2 stream(s).
Stream [0] type: Audio
 codec_string: mp4a.40.2 # 音频的编码器字符串
 time_scale: 44100
 duration: 271360 (6.2 seconds)
 is_encrypted: false
 codec: AAC
 sample_bits: 16
 num_channels: 2
 sampling_frequency: 44100
 language: und

Stream [1] type: Video
 codec_string: avc1.4d401e # 视频的编码器字符串
 time_scale: 3000
 duration: 18500 (6.2 seconds)
 is_encrypted: false
 codec: H264
 width: 640
 height: 480
 pixel_aspect_ratio: 1:1
 trick_play_factor: 0
 nalu_length_size: 4
```

更多请查看[官方文档](https://google.github.io/shaka-packager/html/tutorials/tutorials.html)。

### FFmpeg

[FFmpeg](https://www.ffmpeg.org/download.html) 是功能非常强大的视频处理开源软件，很多视频播放器就是使用它来做为内核。后面文章的实例都会使用这个工具。

比如上面将普通 MP4 转换为 FMP4，可以使用如下命令：

```
ffmpeg -i ./friday.mp4 -movflags empty_moov+frag_keyframe+default_base_moof f.mp4
```

它的命令行格式如下：

```
ffmpeg [global_options] {[input_file_options] -i input_url} ... {[output_file_options] output_url} ...
```

它可以有不限数量的输入和输出文件，`-i` 后面是输入 url，后面不能解析为参数的为输出文件。

```
 _______              ______________
|       |            |              |
| input |  demuxer   | encoded data |   decoder
| file  | ---------> | packets      | -----+
|_______|            |______________|      |
                                           v
                                       _________
                                      |         |
                                      | decoded |
                                      | frames  |
                                      |_________|
 ________             ______________       |
|        |           |              |      |
| output | <-------- | encoded data | <----+
| file   |   muxer   | packets      |   encoder
|________|           |______________|
```

它将输入文件的容器解开，然后对里面的数据进行解码，然后按照指定的格式进行编码，然后使用指定的容器进行封装生成输出文件。在 `decoded frames` 后 FFmpeg 可以使用 filter 进行处理，比如添加滤镜、旋转、锐化等操作，filter 分为简单和复杂，复杂可以处理多个输入流。

如果我们只是想改变视频的容器，那么就可以省略解码和编码过程，来提升速度。

```
ffmpeg -i input.avi -c copy output.mp4
```

`-c` 是指定编码器，`-c copy` 表示直接复制编码，`-c:v` 表示视频编码，`-c:a` 表示音频编码，比如 `-c:v libx264` 表示使用 CPU 将视频编码为 `h.264`，`-c:v h264_nvenc` 则是使用 N卡，这样速度更快。`-c:a copy` 表示直接复制音频编码。

```
ffmpeg -y -i myvideo.mp4 -c:v copy -an myvideo_video.mp4
ffmpeg -y -i myvideo.mp4 -c:a copy -vn myvideo_audio.m4a
```

`-an` 去除音频流 `-vn` 去除视频流。`-y` 是不经过确认，输出时直接覆盖同名文件。

```
ffmpeg -help #查看帮助
ffmpeg -i input.mp4 # 查看视频信息
ffmpeg -formats # 查看支持的容器
ffmpeg -codecs # 查看支持的编码格式
ffmpeg -encoders # 查看内置的编码器
```

更多关于 FFmpeg 请查看[官方文档](https://www.ffmpeg.org/ffmpeg-all.html)。

## 视频缩略图预览

了解了上面好用的工具，就来用 FFmpeg 来实现一个视频播放器小功能吧。

现在视频网站，当鼠标放到进度条上时就会出现，一个小缩略图来预览这个时间点内容。

```
ffmpeg -i ./test.webm -vf 'fps=1/10:round=zero:start_time=-9,scale=160x90,tile=5x5' M%d.jpg
```

我们可以通过上面这个命令生成一个雪碧图，由 25 张 160x90 预览图组成。

`-vf` 参数后面跟着过滤器，多个过滤器用 `,` 分开，一个过滤器多个参数使用 `:` 分开。

`fps=1/10` 表示 10 秒输出一张图，`fps=1/60` 为一分钟一张，`round=zero` 时间戳向 0 取整，`start_time=-9` 是因为 `fps` 是每多少秒生成一张，并不是从 0 秒开始 `-9` 是让它从 1 秒开始截取，忽略掉 0 秒的黑屏帧。

`scale=160x90` 设置输出图像分辨率大小，`tile=5x5` 将小图用 5x5 的方式组合在一起，`M%d.jpg` 表示输出为 jpg，而且文件是 `M1.jpg M2.jpg...` 这样递增。

> 如果想用 NodeJS，可以用 node-fluent-ffmpeg 的 [thumbnails](https://github.com/fluent-ffmpeg/node-fluent-ffmpeg#screenshotsoptions-dirname-generate-thumbnails) 方法来生成。

![](https://pubimg.xingren.com/7f245f22-b6af-405e-ba47-b4426a46688e.jpg)

有了雪碧图，我们就在上篇文章实现的播放器的基础上在加个视频缩略图功能。主要通过 css 的 `background` 来实现。

```javascript
const thumb = document.querySelector('.thumb')
const gapSec = 10 // 一张图片显示几秒
const images = [...] // 图片
const row = 5, col = 5 // 一张图有几行几列
const width = 160, height = 90; // 缩略图的宽高
const thumbQuantityPerImg = col * row

function updateThumbnail(seconds) { // 传入要显示缩略图的秒数
    const thumbNum = (seconds / gapSec) | 0; // 当前是第几张缩略图
    const url = images[(thumbNum / thumbQuantityPerImg) | 0];
    const x = (thumbNum % col) * width; // x 的偏移量
    const y = ~~((thumbNum % thumbQuantityPerImg) / row) * height; // y 的偏移量

    thumb.style.backgroundImage = `url(${url})`;
    thumb.style.backgroundPosition = `-${x}px -${y}px`;
}
```

效果如下

![](https://pubimg.xingren.com/524438b0-8983-4055-b235-3514bbcbbdb9.jpg)

这里只展示缩略图更新的逻辑，忽略了时间显示，事件处理和样式等代码。完整的代码请参考 [RPlayer](https://github.com/woopen/RPlayer)。

## 视频切片

有了 MSE 我们就可以将一个视频分割成多个小视频，然后可以自己控制缓存进度来节省流量，还可以将视频压缩成不同的分辨率，在用户网不好的情况动态加载码率低的分段，分成多段还可以实现插入广告，动态切换音频语言等功能。

```
./audio/
  ├── ./128kbps/
  |     ├── segment0.mp4
  |     ├── segment1.mp4
  |     └── segment2.mp4
  └── ./320kbps/
        ├── segment0.mp4
        ├── segment1.mp4
        └── segment2.mp4
./video/
  ├── ./240p/
  |     ├── segment0.mp4
  |     ├── segment1.mp4
  |     └── segment2.mp4
  └── ./720p/
        ├── segment0.mp4
        ├── segment1.mp4
        └── segment2.mp4
```

> 当然也可以不切割视频，而使用 HTTP Range 来范围请求数据。

```c
ffmpeg -i ./friday.mp4 -f segment -segment_time 2 -segment_format_options movflags=dash ff%04d.mp4
```

我们使用上面命令将一个视频切成 2 秒的 fmp4 视频片段。

```javascript
window.MediaSource = window.MediaSource || window.WebKitMediaSource
const video = document.querySelector('video')
const mime = 'video/mp4; codecs="avc1.4d401e, mp4a.40.2"'
const segments = [
    '/static/media/ff0000.5b66d30e.mp4', 
    '/static/media/ff0001.89895c46.mp4', 
    '/static/media/ff0002.44cfe1e4.mp4'
]
const segmentData = []
let currentSegment = 0

if ('MediaSource' in window && MediaSource.isTypeSupported(mime)) {
    const ms = new MediaSource()
    video.src = URL.createObjectURL(ms)
    mediaSource.addEventListener('sourceopen', sourceOpen)
}

function sourceOpen({ target }) {
    URL.revokeObjectURL(video.src)
    target.removeEventListener('sourceopen', sourceOpen)
    const sb = target.addSourceBuffer(mime)
    fetchSegment(ab => sb.appendBuffer(ab))
    sb.addEventListener('updateend', () => {
        if (!sb.updating && segmentData.length ) {
            sb.appendBuffer(segmentData.shift())
        }
    })
    video.addEventListener('timeupdate', function timeupdate() {
        if (
            currentSegment > segments.length &&
            !sb.updating &&
            target.readyState === 'open'
        ) { // 所有片段全部加载完成
            target.endOfStream()
            video.removeEventListener('timeupdate', timeupdate)
        } else if (
            video.currentTime > (currentSegment * 2 * 0.8) 
            // 一个片段时长 2 秒
            // 在一个片段播放到 80% 的时候才去请求下一个片段
        ) {
            fetchSegment(ab => {
                if (sb.updating) {
                    segmentData.push(ab)
                } else {
                    sb.appendBuffer(ab)
                }
            })
        }
    })
    video.addEventListener('canplay', () => {
        video.play()
    })
}

function fetchSegment(cb) {
    fetch(segments[currentSegment])
        .then(response => response.arrayBuffer())
        .then(arrayBuffer => {
            currentSegment += 1
            cb(arrayBuffer)
        })
}
```

这个实例非常简单，并没有处理 seek，自适应码率等复杂功能。

## 总结

现在视频网站几乎全部都在使用 MSE 来播放视频。使用 MSE 有提供更好的用户体验，更加节约成本等好处。虽然视频播放一般使用 `hls`  `dash` 等协议的开源客户端来播放视频，我们自己不会使用到 MSE，但这些客户端底层都是使用 MSE，了解 MSE 才更了解这些客户端。

## 参考

- [MediaSource](https://developer.mozilla.org/en-US/docs/Web/API/MediaSource)
- [Media Source Extensions™](https://www.w3.org/TR/media-source/)
- [Media Source Extensions](https://developers.google.com/web/fundamentals/media/mse/basics)
- [Streaming media on demand with Media Source Extensions](https://hacks.mozilla.org/2015/07/streaming-media-on-demand-with-media-source-extensions/)
- [Streaming a video with Media Source Extensions](https://axel.isouard.fr/blog/2016/05/24/streaming-webm-video-over-html5-with-media-source)