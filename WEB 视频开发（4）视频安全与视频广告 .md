这篇文章通过 HLS 的方式介绍如何保护视频和插入视频广告。

[[_TOC_]]

## 视频安全

有些视频是需要付费或者开通会员才能观看，那么怎么保护这些视频呢？最常用的方法就是服务器鉴权，HTTP 请求的时候带上一个签名，服务器判断当前用户是否有权观看这个视频。

这里还是使用上篇文章音视频分离中制作的 HLS 视频。通过上篇文章设置好后，就可以更改 js 代码，如下：

```javascript
import Hls from 'hls.js';

const video = document.querySelector('video')
const url = 'http://127.0.0.1:8001/master.m3u8'
const hls = new Hls({
  xhrSetup(xhr, url) {
    xhr.open('GET', url + '?sign=授权签名', true)
  },
});
hls.loadSource(url)
hls.attachMedia(video);
hls.on(Hls.Events.MANIFEST_PARSED, () => {
    video.play();
});
```

![](https://pubimg.xingren.com/586a5875-3936-4c63-a2e6-c168e58194ee.jpg)

这里只是给 hls.js 添加了 `xhrSetup` 参数来将签名加到 `url` 后面，让每个请求都带上这个签名。当然也可以服务器端返回的 `m3u8` 文件中就将签名加上。

hls.js 有两个 `loader` 一个是 `xhr-loader` 一个是 `fetch-loader`。不过所有请求都是通过 `xhr-loader` 发送的。要想自定义请求，可以添加 `loader` 参数，实现自己的 `loader`。

### 随机文件名

我们发现视频名称很有规律，很容易就知道另一个文件名。可以在生成分段名的时候加上一些随机字符。

```
-hls_segment_filename "%v_$(uuidgen).ts"
```

这里给每个文件名加上了 `uuid`。如果想用时间的做分段的名的话可以加上 `-strftime 1 -strftime_mkdir 1`，然后名字就可以使用 `%Y%m%d%H%M%S` 这些表达字符表示目录或文件名，注意这时候不能使用 `single_file` 这个 `flag`。

```
ffmpeg -i sample.mpeg \
   -f hls -hls_time 3 -hls_list_size 5 \
   -hls_flags second_level_segment_index+second_level_segment_size+second_level_segment_duration \
   -strftime 1 -strftime_mkdir 1 -hls_segment_filename "segment_%Y%m%d%H%M%S_%%04d_%%08s_%%013t.ts" stream.m3u8
```

### 视频加密

一般视频保护上面的方法就足够了，优酷的 VIP 视频就是使用的上面的方法。但是如果打开开发者工具选择一个 `ts` 请求，右键点击在新窗口打开就可以直接下载视频，并且可以使用本地播放器直接播放视频。

![](https://pubimg.xingren.com/140b63bd-63e5-4560-9f69-9f988cea0da5.jpg)

如果想让视频下载下来也不能观看的话可以对视频片段进行 `AES128` 加密，`AES128` 是 HLS 最常用的加密，并且 `hls.js` 也支持这种加密，它是对称加密（使用同一个密钥进行加密和解密）。

要进行加密首先要生成一个密钥。

```php
openssl rand 16 > file.key
# 用 openssl 生成一个密钥文件
```

使用 ffmpeg 对 HLS 视频加密，还需要一个 `keyinfo` 文件，文件格式如下：

```
http://www.www.com/path/file.key # hls 客户端获取密钥文件地址
file.key # ffmpeg 获取密钥文件地址
7c3cb56562d0a10827489996dead35eb # 可选的 16 进制初始化向量，不指定则使用段序列号加密
```

```
echo file.key > file.keyinfo
echo file.key >> file.keyinfo
echo $(openssl rand -hex 16) >> file.keyinfo
```

通过上面命令创建好 `keyinfo` 文件后，就可以使用 ffmpeg 生成加密的 HLS 视频了。

```php
ffmpeg -threads 0 -vsync 1 -i ./video.mp4 \
    -lavfi '[0] scale=1280:720[hd],[0] scale=1920:1080[fhd]' \
    -c:v libx264 -c:a aac -b:v:0 2800k -b:a:0 128k -b:v:1 5000k -b:a:1 192k \
    -map '[hd]' -map 0:a -map '[fhd]' -map 0:a \
    -var_stream_map 'v:0,agroup:hd,name:video_hd a:0,agroup:hd,name:audio_hd v:1,agroup:fhd,name:video_fhd a:1,agroup:fhd,name:audio_fhd' \
    -f hls -master_pl_name master.m3u8 \
    -ar 44100 -ac 2 -g 120 -keyint_min 120 -sc_threshold 0 -muxpreload 0 -muxdelay 0 \
    -hls_key_info_file ./file.keyinfo -hls_time 10 -hls_playlist_type vod -hls_list_size 0 \
    -hls_segment_filename "%v%03d.ts" %v.m3u8
```

通过 `-hls_key_info_file` 参数指定 `keyinfo` 文件。

![](https://pubimg.xingren.com/19f15390-a21d-4755-970a-2f588ac3289a.jpg)

如果这时候我们点击一个视频文件播放，就会提示无法播放。

![](https://pubimg.xingren.com/b57a101d-4dff-40f1-9be8-25eb8fa88593.jpg)

```php
ffmpeg -hide_banner -i ./video_hd000.ts
./video_hd000.ts: Invalid data found when processing input
```

```
--- video_fhd.m3u8
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-TARGETDURATION:10
#EXT-X-MEDIA-SEQUENCE:0
#EXT-X-PLAYLIST-TYPE:VOD
#EXT-X-KEY:METHOD=AES-128,URI="file.key",IV=0x7c3cb56562d0a10827489996dead35eb
#EXTINF:10.000000,
video_fhd000.ts
#EXTINF:10.000000,
video_fhd001.ts
#EXTINF:10.000000,
video_fhd002.ts
#EXT-X-ENDLIST
```

这时候刷新浏览器就可以看到对 `key` 文件的请求。

![](https://pubimg.xingren.com/10174c0b-b034-4736-a823-d74947d3cb23.jpg)

## 视频广告

### HLS 广告

视频中插入广告可以使用 HLS 的 `#EXT-X-DISCONTINUITY` 标签，因为广告和视频的分辨率，码率等信息不同，当从广告切换到视频会产生视频质量的下降，这个标签就可以通知客户端提前做好准备。

```php
ffmpeg -i ad.mp4 -c:v libx264 -c:a aac \
    -hls_init_time 0 -g 120 -keyint_min 120 -sc_threshold 0 -muxpreload 0 -muxdelay 0 \
    -hls_time 5 -hls_playlist_type vod -hls_list_size 0 \
    -hls_segment_filename "ad_%03d.ts" master.m3u8
```

```php
ffmpeg -i video.mp4 -c:v libx264 -c:a aac \
    -hls_init_time 0 -g 120 -keyint_min 120 -sc_threshold 0 -muxpreload 0 -muxdelay 0 \
    -hls_time 10 -hls_flags append_list -hls_playlist_type vod -hls_list_size 0 \
    -hls_segment_filename "video_%03d.ts" master.m3u8
```

`-hls_flags append_list` 告诉 ffmpeg 添加到最后，而不是替换原文件。

![](https://pubimg.xingren.com/e9c43caa-3bf3-46d8-b757-75019e8e9d5d.jpg)

```
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-TARGETDURATION:10
#EXT-X-MEDIA-SEQUENCE:0
#EXT-X-PLAYLIST-TYPE:VOD
#EXTINF:5.005333,
ad_000.ts
#EXTINF:5.005333,
ad_001.ts
#EXTINF:0.500533,
ad_002.ts
#EXT-X-DISCONTINUITY
#EXTINF:10.000000,
video_003.ts
#EXTINF:10.000000,
video_004.ts
#EXTINF:10.000000,
video_005.ts
#EXT-X-ENDLIST
```

### 动态获取广告

上面这种方法对于视频点播来说并不实用。我们可以保持 `m3u8` 只包含视频内容的信息，然后通过 HTTP 请求去获取广告。

视频广告主要分为 Liner 和 NonLiner 两种类型。

#### Liner 广告

Liner 广告就是我们常看到的视频广告。它会打断正片的播放，可以查播在正片之前，之后和中间。

要实现 Liner 广告，可以用一堆的 video 元素，每个 video 元素都是一个广告，多个广告组合成一个播放列表，一次播放完毕。用 CSS 覆盖正片的 video 的广告。

```javascript
const player = document.querySelector('.player')  // 正片视频
const linerContaniner = document.querySelector('.liner') // 视频广告容器
let prevPlayerCurrentTime = 0
let currentAd = null
let playlist = []

class Video {
    constructor() {
        this.dom = document.createElement('video')
        this.dom.addEventListener('ratechange', this.onRateChange)
        this.dom.addEventListener('timeupdate', this.onTimeUpdate)
        this.dom.addEventListener('seeking', this.onSeeking)
        this.dom.addEventListener('ended', this.onEnded)
        
        this.prevTime = 0
    }
    
    onRateChange = () => {
        this.dom.playbackRate = 1 // 禁止改变播放速度
    }
    
    onTimeUpdate = () => {
        if (!this.dom.seeking) this.prevTime = this.dom.currentTime
    }
    
    onSeeking = () => {
        const delta = this.dom.currentTime - this.prevTime
        if (Math.abs(delta) > 1) this.dom.currentTime = this.prevTime // 禁止 seek
    }
    
    onEnded = () => {
        this.dom.parentNode.removeChild(this.dom)
        playNext() // 播放下一个广告
    }
    
    play() {
        this.dom.play()
    }
}

function playLinerAds() {
    const ads = getAds() // 获取当前需要播放的广告
    playlist = ads.map(ad => ({ ad, video: new Video(ad) }))
    
    // 禁用当前正片
    prevPlayerCurrentTime = player.currentTime
    player.addEventListener('play', player.pause)
    player.addEventListener('timeupdate', player.pause)
    player.pause()
    
    playlist.forEach(ad => {
        linerContainer.appendChild(ad.video.dom)
    })
    linerContaniner.hidden = false // 显示视频广告
    
    playNext()
}

function playNext() {
    if (!playlist.length) return end()
    const ad = playlist.shift()
    currentAd = ad
    ad.video.play()
}

function end() { // 恢复播放
    player.removeEventListener('play', player.pause)
    player.removeEventListener('timeupdate', player.pause)
    player.currentTime = this.prevPlayerCurrentTime;
    player.play()
    linerContaniner.hidden = true
}
```

上面的代码省略了很多的细节，比如视频播放出错的处理，视频加载的处理，广告倒计时等问题。还有很多浏览器会禁止视频带声音的自动播放，所以当广告播放失败时，应该使用静音播放，再失败时再采用用户点击播放。

详细的代码可以参考 [@rplayer/ads](https://github.com/woopen/RPlayer/blob/master/packages/rplayer-ads/README.md)

![](https://pubimg.xingren.com/93970f57-4da3-443b-a4c5-00c972e27f27.jpg)

#### NonLiner 广告

NonLiner 广告并不会打断正片的播放，它可以是一张图片，视频或一个 `iframe`，下面来实现一个暂停的弹框广告。

```javascript
const player = document.querySelector('.player')  // 正片视频
const nonLiner = document.querySelector('.liner') // 弹框广告容器

const img = new Image()
img.src = getImgSrc() // 获取图片的链接
nonLiner.appendChild(img)

player.addEventListener('play', () => { nonLiner.hidden = false })
player.addEventListener('timeupdate', () => { nonLiner.hidden = true })
```

弹框广告相对比较简单，这里也省略很多细节。更多可以参考 [@rplayer/ads](https://github.com/woopen/RPlayer/blob/master/packages/rplayer-ads/README.md)。

![](https://pubimg.xingren.com/a0407864-9ae2-43ef-bd86-401e3c5daaee.jpg)

## 总结

这篇文章介绍了如何使用 HLS 来实现加密，对于 DASH 来说实现方式也类似。视频广告是使用多个 video 元素加载，当然也可以通过 MSE 的方式只使用一个 video 元素进行广告加载。

## 参考

- [ffmpeg Documentation](https://www.ffmpeg.org/ffmpeg-all.html)
- [Interactive Media Ads SDKs](https://developers.google.com/interactive-media-ads)