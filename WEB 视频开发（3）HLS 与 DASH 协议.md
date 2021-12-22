上篇文章介绍了 MSE 来播放流媒体，但是 WEB 视频开发并不只依靠 MSE。这篇文章就来介绍主流的两种协议 HLS 和 DASH，以及如何制作并使用支持这些协议开源的客户端库来播放视频。

---

[[_TOC_]]

## HLS

HLS (HTTP Live Streaming) 是苹果公司开发的流媒体传输协议，它使用 HTTP 来传输视频，可以防止被防火墙屏蔽。现在大部分视频网站都在使用，比如优酷、腾讯视频。

> 它的工作原理是把整个流分成一个个小的基于 HTTP 的文件来下载，每次只下载一些。当媒体流正在播放时，客户端可以选择从许多不同的备用源中以不同的速率下载同样的资源，允许流媒体会话适应不同的数据速率。

![](https://pubimg.xingren.com/a496aee8-7502-446a-b1fc-5628b6a35f76.jpg)

它会生成一个 `.m3u8` 文件，其中除了包含一些元数据，还记录被分割视频的存放位置。分割的视频是 `.ts` 结尾的文件，是 `MPEG-2 Transport Stream` 容器，不过现在 HLS 也[支持 fmp4](https://developer.apple.com/videos/play/wwdc2016/504/)。

```
#EXTM3U
#EXT-X-TARGETDURATION:10
#EXT-X-VERSION:4
#EXT-X-MEDIA-SEQUENCE:0
#EXTINF:10.0,
ad0.ts
#EXTINF:8.0,
ad1.ts
#EXT-X-DISCONTINUITY
#EXTINF:10.0,
movieA.ts
#EXTINF:10.0,
movieB.ts
```

一个 `.m3u8` 文件大概长上面那样。文件中以 `#` 开头的字符串要么是注释，要么就是标签，标签以 `#EXT` 开头，大小写敏感。

- `EXTM3U` M3U8 文件必须包含的标签，并且必须在文件的第一行
- `EXT-X-VERSION` M3U8 文件的版本，常见的是 3（目前最高版本应该是7），版本更高支持的标签就越多
- `EXT-X-TARGETDURATION` 指定了单个媒体文件持续时间的最大值
- `EXT-X-MEDIA-SEQUENCE` 播放列表第一个 URL 片段文件的序列号，默认序列号从 0 开始
- `EXTINF` 其后 URL 指定的媒体片段时长（秒）
- `EXT-X-DISCONTINUITY` 一般用于视频流中插入广告，表示前面的片段与后面不一样，让客户端做好准备

#### 制作

去网上随便下载[一个视频](https://github.com/bower-media-samples/big-buck-bunny-1080p-30s/raw/master/video.mp4)，用 Bento4 中的 mp4info 看一下文件信息，如下：

```php
mp4info ./video.mp4
...
Track 1:
  flags:        3 ENABLED IN-MOVIE
  id:           1
  type:         Video
  duration: 30000 ms
  language: und
  media:
    sample count: 720
    timescale:    12288
    duration:     368640 (media timescale units)
    duration:     30000 (ms)
    bitrate (computed): 5860.270 Kbps
  display width:  1920.000000
  display height: 1080.000000
  frame rate (computed): 24.000
  Sample Description 0
    Coding:      avc1 (H.264)
    Width:       1920
    Height:      1080
    Depth:       24
    AVC Profile:          100 (High)
    AVC Profile Compat:   0
    AVC Level:            40
    AVC NALU Length Size: 4
    AVC SPS: [67640028acd940780227e5c044000003000400000300c03c60c658]
    AVC PPS: [68ebe3cb22c0]
    Codecs String: avc1.640028
Track 2:
  flags:        3 ENABLED IN-MOVIE
  id:           2
  type:         Audio
  duration: 30022 ms
  language: und
  media:
    sample count: 1408
    timescale:    48000
    duration:     1441024 (media timescale units)
    duration:     30021 (ms)
    bitrate (computed): 192.583 Kbps
  Sample Description 0
    Coding:      mp4a (MPEG-4 Audio)
    Stream Type: Audio
    Object Type: MPEG-4 Audio
    Max Bitrate: 192580
    Avg Bitrate: 192580
    Buffer Size: 0
    Codecs String: mp4a.40.2
    MPEG-4 Audio Object Type: 2 (AAC Low Complexity)
    MPEG-4 Audio Decoder Config:
      Sampling Frequency: 48000
      Channels: 6
    Sample Rate: 48000
    Sample Size: 16
    Channels:    2
```

可以看到这个文件为 1080p，24 fps，5860 的码率。

```php
ffmpeg -i ./in.mp4 \
    -vf scale=w=1280:h=720:force_original_aspect_ratio=decrease,yadif \
    -c:a aac -b:a 128k -ar 44100 -ac 2 \
    -c:v libx264 -b:v 2500k -maxrate 2675k -bufsize 3000k \
    -pix_fmt yuv420p -level 4.1 \
    -profile:v high -preset veryfast -crf 20 \
    -g 120 -keyint_min 120 \
    -sc_threshold 0 \
    -threads 0 -muxpreload 0 -muxdelay 0 \
    -hls_time 10 -hls_playlist_type vod -hls_list_size 0 \
    -f hls -hls_segment_filename '720p_%03d.ts' 720p.m3u8
```

运行上面命令就可以将 `mp4` 转换成 `m3u8` 格式了。

![](https://pubimg.xingren.com/5e355432-0647-4e32-b38e-6236b6d67062.jpg)

```php
fmpeg -hide_banner -i ./720p_000.ts # 使用 ffmepg 查看一下切片信息，可以看到信息和上面命令指定的一样
Input #0, mpegts, from './720p_000.ts':
  Duration: 00:00:10.02, start: 0.060111, bitrate: 2095 kb/s
  Program 1
    Metadata:
      service_name    : Service01
      service_provider: FFmpeg
    Stream #0:0[0x100]: Video: h264 (Main) ([27][0][0][0] / 0x001B), yuv420p(progressive), 1280x720 [SAR 1:1 DAR 16:9], 24 fps, 24 tbr, 90k tbn, 48 tbc
    Stream #0:1[0x101](und): Audio: aac (LC) ([15][0][0][0] / 0x000F), 44100 Hz, 5.1, fltp, 134 kb/s
```

| 参数 | 描述|
| --- | --- |
| -vf | 后面是过滤器，`scale` 控制分辨率，这里让它变成保持原始比例的 720p 视频，yadif 让视频使用逐行扫描 |
| -c:a -b:a -ar -ac -c:v | 音频编码，音频码率，音频采样频率，音频通道数，视频码率 |
| -b:v -maxrate -bufsize | 码率，最大码率，缓存大小。设置 `maxrate` 和 `-b:v` 不一样可以启动 VBR，设置 `bufsize` 是为了让码率分布更均匀，设置的越小检查的频率越高，画质越差。一般设置 `maxrate` 的一到两倍之间  |
| pix_fmt | 改变像素格式 |
| level | 编码等级，B站投稿最高等级是 4.1，再高有些设备可能播放会卡顿 |
| profile:v | 编码档次，有 `baseline` `main` `high`。`high` 质量更好，`main` 兼容性更好 |
| preset | `h.264` 预设，因为 `h.264` 配置项太多，所有这里提供了几个预设值 `ultrafast` `superfast` `veryfast` `faster` `fast` `medium` `slow` `slower` `veryslow` 设置的越高质量越好，但是编码速度就越慢，默认是 `medium` |
| crf | 视频目标质量，0 表示无损，默认是 23，越低质量越好，最好不要设置 30 以上 |
| g keyint_min | 控制关键帧的生成，因为视频要分段，所以关键字生成时间要小于分段时长，因为 24fps，设置 120 表示大约每隔 5 秒生成一个关键帧 |
| sc_threshold 0 | 转换场景时不自动生成关键帧 |
| threads 0 | 自动根据计算机核心启用多线程 |
| muxpreload muxdelay | 设置初始和最大解复用延迟，这里设置为 0 为了片段时长严格根据设置的时长 |
| hls_time | 片段时长，点播 hls 推荐为 10 秒 |
| hls_playlist_type | hls 类型设置为点播类型 |
| hls_list_size 0 | 不限制播放列表长度 |
| -f hls -hls_segment_filename | 输出类型，片段名 |

hls 支持自动适应码率，根据当前网络状态自动切换清晰度，我们可以制作多种不同码率的视频来让 hls 自动切换。

```php
ffmpeg -threads 0 -vsync 1 -i .\video.mp4 \
    -lavfi '[0] scale=854:480[ed],[0] scale=1280:720[hd],[0] scale=1920:1080[fhd]' \
    -c:v libx264 -c:a aac -b:v:0 1400k -b:a:0 128k -b:v:1 2800k -b:a:1 128k -b:v:2 5000k -b:a:2 192k \
    -map '[ed]' -map 0:a -map '[hd]' -map 0:a -map '[fhd]' -map 0:a \
    -f hls -var_stream_map 'v:0,a:0,name:480p v:1,a:1,name:720p v:2,a:2,name:1080p' \
    -master_pl_name master.m3u8 \
    -hls_time 10 -hls_playlist_type vod -hls_list_size 0 \
    -hls_segment_filename '%v_%03d.ts' %v.m3u8
```

为了简化，一些参数就没配置了，运行上面命令可以生成 3 种不同清晰度的 `m3u8` 文件，还有一个将它们合并在一起的 `m3u8` 文件，hls 通过两层 `m3u8` 来实现自适应码率。

![](https://pubimg.xingren.com/ab685ef6-2967-45bd-a070-3c001a27f84a.jpg)

```
--- 文件：master.m3u8
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-STREAM-INF:BANDWIDTH=1680800,RESOLUTION=854x480,CODECS="avc1.64001e,mp4a.40.2"
480p.m3u8

#EXT-X-STREAM-INF:BANDWIDTH=3220800,RESOLUTION=1280x720,CODECS="avc1.64001f,mp4a.40.2"
720p.m3u8

#EXT-X-STREAM-INF:BANDWIDTH=5711200,RESOLUTION=1920x1080,CODECS="avc1.640028,mp4a.40.2"
1080p.m3u8
```

| 参数 | 描述 |
| --- | --- |
| -vsync 1 | 保持恒定 fps，少了复制，多了丢弃 |
| -lavfi | 复杂 filter，可以多输入和多输出，这里的方括号用来选择流和打标签 |
| -map | 用于手动选择流，这里使用 label 选择对于流 |
| -var_stream_map | 自定义流组合，最终输出文件需要包含 `%v`，这里用 `name` 给它赋有意义的值 |
| -master_pl_name | 主 `m3u8` 名 |

下面是不同分辨率的推荐码率。

| Quality | Resolution | bitrate - low motion |	bitrate - high motion | audio bitrate |
| --- | --- | --- | --- | --- |
| 240p | 426x240 | 400k | 600k | 64k |
| 360p | 640x360 | 700k | 900k | 96k |
| 480p | 854x480 | 1250k | 1600k | 128k |
| HD 720p | 1280x720 | 2500k | 3200k | 128k |
| HD 720p 60fps | 1280x720 | 3500k | 4400k | 128k |
| Full HD 1080p | 1920x1080 | 4500k | 5300k | 192k |
| Full HD 1080p 60fps | 1920x1080 | 5800k | 7400k | 192k |
| 4k | 3840x2160 | 14000k | 18200 | 192k |
| 4k 60fps | 3840x2160 | 23000k | 29500k | 192k |

下面是 Youtube 和 B 站上传视频推荐设置

![](https://pubimg.xingren.com/13942d57-f952-4f56-94d0-32cc3a7ff005.jpg)

![](https://pubimg.xingren.com/093d6a14-12ce-44a3-a5e1-c32fc11e6788.jpg)

### 音视频分离

一般视频网站都会把音频和视频分离，这样做的好处非常多，比如：

1. 如果视频有多个不同语言的版本，那么就可以实现实时切换视频语言。
2. 更加节约空间，比如多个不同码率的视频使用相同码率的音频。
3. 更好的兼容性，有些设备播放包含视频和音频的文件会出现一些问题，比如没声音。

![](https://pubimg.xingren.com/678c5ac1-50b8-4602-a715-a47809246327.jpg)

但是分量音视频也大大提高了复杂性，比如如何选择适合码率的音频和视频，还有播放时的音视频同步。

> 视频有 DTS（解码时间戳，诉播放器该在什么时候解码这一帧的数据）、PTS（显示时间戳，告诉播放器该在什么时候显示这一帧的数据） 。音频的播放也有 DTS、PTS 的概念，但是音频没有类似视频中 B 帧，不需要双向预测，所以音频帧的 DTS、PTS 顺序是一致的。所以需要控制视频和音频的播放，不然就会发生声画不一致。

```php
ffmpeg -threads 0 -vsync 1 -i .\video.mp4 \
    -lavfi '[0] scale=1280:720[hd],[0] scale=1920:1080[fhd]' \
    -c:v libx264 -c:a aac -b:v:0 2800k -b:a:0 128k -b:v:1 5000k -b:a:1 192k \
    -map '[hd]' -map 0:a -map '[fhd]' -map 0:a \
    -var_stream_map 'v:0,agroup:hd,name:video_hd a:0,agroup:hd,name:audio_hd v:1,agroup:fhd,name:video_fhd a:1,agroup:fhd,name:audio_fhd' \
    -f hls -master_pl_name master.m3u8 \
    -ar 44100 -ac 2 \
    -g 120 -keyint_min 120 -sc_threshold 0 -muxpreload 0 -muxdelay 0 \
    -hls_time 10 -hls_flags single_file -hls_playlist_type vod -hls_list_size 0 \
    -hls_segment_type fmp4 -hls_segment_filename '%v.mp4' %v.m3u8
```

上面命令将制作音视频分离的 HLS 文件。

![](https://pubimg.xingren.com/4c33f5ba-dc20-45f7-a368-c987fe8b4827.jpg)

```
--- 文件：master.m3u8
#EXTM3U
#EXT-X-VERSION:7
#EXT-X-MEDIA:TYPE=AUDIO,GROUP-ID="group_hd",NAME="audio_1",DEFAULT=YES,URI="audio_hd.m3u8"
#EXT-X-MEDIA:TYPE=AUDIO,GROUP-ID="group_fhd",NAME="audio_3",DEFAULT=YES,URI="audio_fhd.m3u8"
#EXT-X-STREAM-INF:BANDWIDTH=3220800,RESOLUTION=1280x720,CODECS="avc1.64001f,mp4a.40.2",AUDIO="group_hd"
video_hd.m3u8

#EXT-X-STREAM-INF:BANDWIDTH=140800,CODECS="mp4a.40.2",AUDIO="group_hd"
audio_hd.m3u8

#EXT-X-STREAM-INF:BANDWIDTH=5711200,RESOLUTION=1920x1080,CODECS="avc1.640028,mp4a.40.2",AUDIO="group_fhd"
video_fhd.m3u8

#EXT-X-STREAM-INF:BANDWIDTH=211200,CODECS="mp4a.40.2",AUDIO="group_fhd"
audio_fhd.m3u8

--- 文件：video_hd.m3u8
#EXTM3U
#EXT-X-VERSION:7
#EXT-X-TARGETDURATION:10
#EXT-X-MEDIA-SEQUENCE:0
#EXT-X-PLAYLIST-TYPE:VOD
#EXT-X-MAP:URI="video_hd.mp4",BYTERANGE="827@0"
#EXTINF:10.000000,
#EXT-X-BYTERANGE:4341047@827
video_hd.mp4
#EXTINF:10.000000,
#EXT-X-BYTERANGE:2573385@4341874
video_hd.mp4
#EXTINF:10.000000,
#EXT-X-BYTERANGE:4398334@6915259
video_hd.mp4
#EXT-X-ENDLIST
```

上面用 `-hls_flags single_file` 让 hls 使用 HTTP Range 来请求分段数据，而无需将视频切成一段段的，`-hls_segment_type fmp4` 使用 `fmp4` 而不是 `ts`。

### hls.js

现在我们制作好了 hls 视频，就可以在视频播放器中播放了，苹果的设备都支持 hls，所以直接设置 `video` 的 `src` 为 `m3u8` 文件就可以了。但是对于其他设备并不支持 hls 协议，这时候就可以使用 [hls.js](https://github.com/video-dev/hls.js)。

hls.js 是将 `ts` 容器转换成 `fmp4`，它需要 HTML 5 Video 和 MSE 来播放视频。

```
npm i -S hls.js # 安装
```

安装好后，还需要一个静态资源服务器来处理视频资源。

```
npm i -g http-server
# 安装好后在视频资源目录下 执行下面命令
http-server --cors -p 8001
```

最后在 js 文件加上如下代码。

```javascript
import Hls from 'hls.js'

const video = document.querySelector('video')
const url = 'http://127.0.0.1:8001/master.m3u8'
if (Hls.isSupported()) {
    const hls = new Hls();
    hls.loadSource(url)
    hls.attachMedia(video);
    hls.on(Hls.Events.MANIFEST_PARSED, () => {
        video.play();
    });
} else if (video.canPlayType('application/vnd.apple.mpegurl')) {
    video.src = url
    video.addEventListener('loadedmetadata', () => {
        video.play()
    })
}
```

在不支持 MSE 的情况下，就检测是否原生支持 hls，大概率是 IOS 的 Safari（没错它还不支持 MSE）。

![](https://pubimg.xingren.com/39e39407-e14c-4978-b283-cfd1bfa26e3c.jpg)

可以看到默认请求 hd，但是发现网速很快后就动态的请求 fhd 片段。另外 hls.js 对于 `fmp4` 还是测试阶段，可以使用更通用的 `ts` 格式取代。

文件的 base url 可以通过 `hls_base_url` 参数指定，默认播放文件可以通过 `var_stream_map` 的 `default:yes` 设置。

上面的例子很简单，更多关于 hls.js 可以查看 [官方文档](https://github.com/video-dev/hls.js/blob/master/docs/API.md)。

#### 使用 RPlayer

当然我们也可以使用第一篇文章里面制作的[视频播放器](https://github.com/woopen/RPlayer)。

```javascript
const video = document.createElement('video'); // 创建视频元素
const videoSrc = 'http://127.0.0.1:8001/master.m3u8';
const hls = new Hls();
hls.loadSource(videoSrc);
hls.attachMedia(video);

const player = new RPlayer({
  media: video,
  thumbnail: { // 上篇文章制作的预览缩略图
    images: [
      'http://127.0.0.1:8001/M1.jpg',
      'http://127.0.0.1:8001/M2.jpg',
      'http://127.0.0.1:8001/M3.jpg',
    ]
  }
})

player.mount() // 不传参数，默认挂载到 body
```

![](https://pubimg.xingren.com/60094914-5134-4b66-a93c-02490b39e547.jpg)

可以看到视频 seek 和视频 buffer 都没有问题，就和使用普通视频文件一样正常播放。

## DASH

> 基于HTTP的动态自适应流（Dynamic Adaptive Streaming over HTTP，缩写DASH，也称MPEG-DASH）是一种自适应比特率流技术，使高质量流媒体可以通过传统的HTTP网络服务器以互联网传递。

DASH 和 HLS 非常相似都是使用 `manifest` 描述视频信息和播放列表，然后通过 HTTP 自适应的请求合适的片段。

与 HLS 不同的是 DASH 是 [国际标准](https://standards.iso.org/ittf/PubliclyAvailableStandards/index.html)，而 HLS 属于苹果公司。并且 DASH 支持任何编码，它就可以用 `vp9` 编码的 `webm` 格式视频。目前有很多大视频网站都在使用 DASH，比如 youtube、netflix、bilibili。bilibili 也写了一篇文章 [为什么用 DASH](https://www.bilibili.com/read/cv855111)。

![](https://pubimg.xingren.com/29a2f105-f5d7-4c13-a3f1-70a846f69fa3.jpg)

| 字段 | 描述 |
| --- | --- |
| Period | 代表一个场景或一段歌曲，表示某一个时间段，可以在这里穿插广告 |
| AdaptationSet | 描述媒体流的信息，比如是音频流还是视频流 |
| Representation | 用来表示不同屏幕大小或码率，DASH 可以来选择合适文件。<br> Representation 的 Segments 一般都采用 1 个Init Segment 和多个普通 Segment 的方式，<br> 还有一种形式没有单独的 Init Segment，初始化信息包括在了各个 Segment 中 |
| SegmentBase | 实际的音频或视频 |

DASH 的索引文件是 `.mpd`（Media Presentation Description） 结尾的 `XML` 文件，具体文件内容如下。

```xml
<?xml version="1.0"?>
<MPD xmlns="urn:mpeg:dash:schema:mpd:2011" profiles="urn:mpeg:dash:profile:full:2011"
     minBufferTime="PT1.5S">
    <!-- Ad -->
    <Period duration="PT30S">
        <BaseURL>ad/</BaseURL>
        <AdaptationSet mimeType="video/mp2t">
            <Representation id="720p" bandwidth="3200000" width="1280" height="720">
                <BaseURL>720p.ts</BaseURL>
                <SegmentBase>
                    <RepresentationIndex sourceURL="720p.sidx"/>
                </SegmentBase>
            </Representation>
            <Representation id="1080p" bandwidth="6800000" width="1920"
                            height="1080">
                <BaseURL>1080p.ts</BaseURL>
                <SegmentBase>
                    <RepresentationIndex sourceURL="1080p.sidx"/>
                </SegmentBase>
            </Representation>
        </AdaptationSet>
    </Period>
    <!-- Normal Content -->
    <Period duration="PT10M">
        <BaseURL>main/</BaseURL>
        <AdaptationSet mimeType="video/mp2t">
            <BaseURL>video/</BaseURL>
            <Representation id="720p" bandwidth="3200000" width="1280" height="720">
                <BaseURL>720p/</BaseURL>
                <SegmentList timescale="90000" duration="5400000">
                    <RepresentationIndex sourceURL="representation-index.sidx"/>
                    <SegmentURL media="segment-1.ts"/>
                    <SegmentURL media="segment-2.ts"/>
                    <!-- 省略 -->
                </SegmentList>
            </Representation>
            <Representation id="1080p" bandwidth="6800000" width="1920"
                            height="1080">
                <BaseURL>1080/</BaseURL>
                <SegmentTemplate media="segment-$Number$.ts" timescale="90000">
                    <RepresentationIndex sourceURL="representation-index.sidx"/>
                    <SegmentTimeline>
                        <S t="0" r="9" d="5400000"/>
                    </SegmentTimeline>
                </SegmentTemplate>
            </Representation>
        </AdaptationSet>
        <AdaptationSet mimeType="audio/mp2t">
            <BaseURL>audio/</BaseURL>
            <Representation id="audio" bandwidth="128000">
                <SegmentTemplate media="segment-$Number$.ts" timescale="90000">
                    <RepresentationIndex sourceURL="representation-index.sidx"/>
                    <SegmentTimeline>
                        <S t="0" r="9" d="5400000"/>
                    </SegmentTimeline>
                </SegmentTemplate>
            </Representation>
        </AdaptationSet>
    </Period>
</MPD>
```

| MPD 属性 | 描述 |
| --- | --- |
| profiles | 有点类似 HLS 的版本，这些客户端实现了 profile 所需的功能，详情请[参考这个](https://dashif.org/identifiers/profiles/) |
| mediaPresentationDuration | 视频时长 |
| minBufferTime | 最小缓冲时间 |
| type | static 点播，dynamic 直播 |
| minimumUpdatePeriod | 直播专属，至少每隔这么长时间，MPD 就会更新 |

可以看到 mpd 比 m3u8 复杂多了，更多内容请[查看这里](https://dashif-documents.azurewebsites.net/DASH-IF-IOP/master/DASH-IF-IOP.html)。

### 制作

```php
 ffmpeg -i .\video.mp4 \
 -lavfi '[0] scale=1280:720[hd],[0] scale=1920:1080[fhd]' \
 -c:a aac -c:v libx264 -b:v:0 2800k -b:a:0 128k -b:v:1 5000k -b:a:1 192k \
 -map '[hd]' -map 0:a -map '[fhd]' -map 0:a \
 -use_timeline 1 -use_template 1 -single_file 1 \
 -single_file_name '$Bandwidth$_$RepresentationID$.$ext$' \
 -adaptation_sets "id=0,streams=v id=1,streams=a" -f dash out.mpd
```

![](https://pubimg.xingren.com/07c7e43c-42fb-4dd4-aade-b81af73ef885.jpg)

```xml
<?xml version="1.0" encoding="utf-8"?>
<MPD xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns="urn:mpeg:dash:schema:mpd:2011"
    xmlns:xlink="http://www.w3.org/1999/xlink"
    xsi:schemaLocation="urn:mpeg:DASH:schema:MPD:2011 http://standards.iso.org/ittf/PubliclyAvailableStandards/MPEG-DASH_schema_files/DASH-MPD.xsd"
    profiles="urn:mpeg:dash:profile:isoff-live:2011"
    type="static"
    mediaPresentationDuration="PT30.0S"
    minBufferTime="PT14.5S">
    <ProgramInformation>
    </ProgramInformation>
    <Period id="0" start="PT0.0S">
    	<AdaptationSet id="0" contentType="video" segmentAlignment="true" bitstreamSwitching="true" frameRate="24/1" maxWidth="1920" maxHeight="1080" par="16:9">
    		<Representation id="0" mimeType="video/mp4" codecs="avc1.64001f" bandwidth="2800000" width="1280" height="720" sar="1:1">
    			<BaseURL>2800000_0.mp4</BaseURL>
    			<SegmentList timescale="1000000" duration="5000000" startNumber="1">
    				<Initialization range="0-814" />
    				<SegmentURL mediaRange="815-4481448" indexRange="815-866" />
    				<!-- 省略 -->
    			</SegmentList>
    		</Representation>
    		<Representation id="2" mimeType="video/mp4" codecs="avc1.640028" bandwidth="5000000" width="1920" height="1080" sar="1:1">
    			<BaseURL>5000000_2.mp4</BaseURL>
    			<SegmentList timescale="1000000" duration="5000000" startNumber="1">
    				<Initialization range="0-815" />
    				<SegmentURL mediaRange="816-8928627" indexRange="816-867" />
    				<!-- 省略 -->
    			</SegmentList>
    		</Representation>
    	</AdaptationSet>
    	<AdaptationSet id="1" contentType="audio" segmentAlignment="true" bitstreamSwitching="true" lang="und">
    		<Representation id="1" mimeType="audio/mp4" codecs="mp4a.40.2" bandwidth="128000" audioSamplingRate="48000">
    			<AudioChannelConfiguration schemeIdUri="urn:mpeg:dash:23003:3:audio_channel_configuration:2011" value="6" />
    			<BaseURL>128000_1.mp4</BaseURL>
    			<SegmentList timescale="1000000" duration="5000000" startNumber="1">
    				<Initialization range="0-744" />
    				<SegmentURL mediaRange="745-83275" indexRange="745-796" />
    				<!-- 省略 -->
    			</SegmentList>
    		</Representation>
    		<Representation id="3" mimeType="audio/mp4" codecs="mp4a.40.2" bandwidth="192000" audioSamplingRate="48000">
    			<AudioChannelConfiguration schemeIdUri="urn:mpeg:dash:23003:3:audio_channel_configuration:2011" value="6" />
    			<BaseURL>192000_3.mp4</BaseURL>
    			<SegmentList timescale="1000000" duration="5000000" startNumber="1">
    				<Initialization range="0-744" />
    				<SegmentURL mediaRange="745-125638" indexRange="745-796" />
    				<!-- 省略 -->
    			</SegmentList>
    		</Representation>
    	</AdaptationSet>
    </Period>
</MPD>
```

| 参数 | 描述 |
| --- | --- |
| -use_timeline 1 | SegmentTemplate 中使用 SegmentTimeline |
| -use_template 1 | 使用 SegmentTemplate 而不是 SegmentList |
| -adaptation_sets | 分多个 AdaptationSet，这里设置它的 id 和使用那个流 |

### dash.js

在浏览器中播放可以使用 [dash.js](https://github.com/Dash-Industry-Forum/dash.js)。它同样基于 MSE。

和 HLS 一样，安装 dashjs 和启动静态资源服务器。

```php
npm i -S dashjs # 注意不是 .js
# 在资源文件夹下，执行下面命令
http-server --cors -p 8001
```

```javascript
import dash from 'dashjs'

dash
  .MediaPlayer()
  .create()
  .initialize(
    document.querySelector('video'),
    'http://127.0.0.1:8001/out.mpd',
    true // 自动播放
  )
```

![](https://pubimg.xingren.com/d6056ebb-d2a9-420e-bbd9-aafe80a46e42.jpg)

可以看到同样在发现网络环境不错的情况下，自动请求了高码率的片段。更多关于 dash.js 请参考 [官方文档](https://github.com/Dash-Industry-Forum/dash.js/wiki)。

## 总结

这篇文章介绍了 WEB 视频播放的两种主流的协议。但因为 HLS 出现的更早，更简单，有苹果公司支持等原因，现在比 DASH 更加常用，而且它们都基于 MSE，而 MSE 不支持 IE 10及以下。所以低版本浏览器可以需要降级到直接使用普通的 mp4 视频文件或使用 `flash` 播放。当然也有很多网站提示浏览器版本太低。

## 参考

- [HTTP Live Streaming](https://developer.apple.com/streaming/)
- [Guidelines for Implementation: DASH-IF Interoperability Points](https://dashif-documents.azurewebsites.net/DASH-IF-IOP/master/DASH-IF-IOP.html)
- [m3u8 文件格式详解](https://www.jianshu.com/p/e97f6555a070)
- [The structure of an MPEG-DASH MPD](https://www.brendanlong.com/the-structure-of-an-mpeg-dash-mpd.html)
- [Streaming Separate Audio and Video](https://developer.att.com/video-optimizer/docs/best-practices/separate-audio-video)
- [ffmpeg Documentation](https://www.ffmpeg.org/ffmpeg-all.html)