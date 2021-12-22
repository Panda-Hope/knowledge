> 弾幕（だんまく/danmaku）、barrage 是显示在影片上的评论，大量吐槽评论从屏幕飘过时效果看上去像是飞行射击游戏里的弹幕。弹幕视频系统源自日本弹幕视频分享网站（niconico动画），国内首先引进为 AcFun 以及后来的 bilibili。

这篇文章将介绍 3 种实现方法，并找出可以兼容多个浏览器并且流畅播放的方案。

![](https://pubimg.xingren.com/206d080e-90de-4863-bca4-950d4d5f5e3f.jpg)

[[_TOC_]]

## 思路

视频弹幕可以分为两种，一种是静止显示在视频的顶部或底部，另一种是从右到左滚动显示。静止弹幕比较简单，这里主要介绍滚动弹幕。

视频播放时会有很大量弹幕从右边移动到左边，为了方便观看会限制单次导入的弹幕数量，并且让弹幕之间不发生重叠。如果用户开启无限弹幕模式，那么就无需限制弹幕数量和弹幕之间是否重叠。

![](https://pubimg.xingren.com/87954b96-fcdc-4eb9-bf78-be8d5bdace20.jpg)

可以将视频分割成一行行的隧道，每次插入弹幕时选择当前最短的一个隧道插入。

### 速度

因为每个弹幕的显示时间固定，所以长弹幕的速度比短弹幕的速度快。但是长弹幕的速度也不能太快，这样会让它覆盖到前面比较短的弹幕。

为了让弹幕显示时间不要太长，那么就需要后一个弹幕到达最左端时，它前一个弹幕刚好消失。也就是后一个弹幕的速度比前一快，但是又不能太快，它的速度可以让它到达最左端时才追上前一个弹幕。

为了计算后一个弹幕的速度，我们可以设后一个弹幕的速度为 `s`，经过时间 `t` 后，后一个弹幕追上前一个弹幕。那么 `t` 就等于前一个弹幕要走的路程除以前一个弹幕的速度。知道了时间 `t` 那么 `s` 就等于后一个弹幕的要走的路程除以时间 `t`。

```math
s = \frac {current.x} {({prev.x} + {prev.width}) \div prev.speed}
```

## 实现

方式 | 描述
--- | ---
`requestAnimationFrame` 和 `transform` | 使用 js 控制弹幕的 `transform` 来实现滚动弹幕
`requestAnimationFrame` 和 `canvas` | 与第一种相似，但是不使用的 css，而是用 canvas 渲染
`transform` 和 `transition` | 不自己控制动画，而是用 css3 `transition` 属性实现

这三种方法在 `chrome` 上都非常流畅。一般最先想到的方法就是第一种方法，它实现起来非常的简单，但是第一种方法在 `IE` 上有明显卡顿。

由于第一和第二种实现方式差不多，只是最终渲染的时候一个是操作 DOM，一个是操作 `canvas`，所以这里就只演示比较流畅的 `canvas` 版本。

### 通用接口

这篇文章只关注弹幕的实现，一些通用代码就用接口表示了。

下面是一条弹幕

```typescript
interface Item {
  text: string; // 评论
  time: number; // 显示的时间
  color?: string; // 颜色
}
```

播放器对象

```typescript
interface Player {
    width: number; // 播放器宽度
    height: number; // 播放器高度
    currentTime: number; // 当前播放时间
    on: (event: string, callback: Function) => void; // 监听 video 元素事件
    appendChild: (dom: Element) => void; // 添加元素到播放器
}
```

### canvas 版本

canvas 版本，实现比较直接。这里使用两个类 `Danmaku` 控制所有弹幕，`Bullet` 滚动的单个弹幕。

```javascript
private update = (): void => {
    this.draw();
    this.timer = requestAnimationFrame(this.update);
};
```

我们使用 `requestAnimationFrame` 来循环执行 `draw` 方法。

`draw` 方法在没一帧中执行下面 4 个步骤：

1. 加载当前时间点的弹幕
2. 把加载的弹幕填入到合适的隧道中
3. 更新弹幕位置
4. 清理超出界限的弹幕

```javascript
private draw(): void {
    const items = this.load(); // 加载弹幕，返回的是 Item
    if (items) {
      for (let i = 0, l = items.length; i < l; i++) {
        if (!this.fill(items[i], i)) break; // 如果 fill 方法返回 false 代表，所有隧道已填满，放弃剩下的弹幕
      }
    }
    this.ctx.clearRect(0, 0, this.dom.width, this.dom.height); // 清理 canvas
    this.bullets.forEach((bullet) => bullet.display()); // 移动弹幕位置
    this.clearBullets(); // 清理超出边界弹幕，放入弹幕池中，下次可以直接使用
}
```

其中比较复杂是 `fill` 方法。它会找出当前最短的隧道，然后判断是否可以插入弹幕。

```javascript
private fill(item: Item, i = 0, force = false): boolean {
    const [tunnel, prevBullet] = this.getShortestTunnel(); // 获取最短隧道
    if (!prevBullet || prevBullet.length < this.width + 200) { 
      // 这里限制弹幕数量的策略是，每个隧道长度不超过视频宽度加 200
      this.scroll[tunnel] = this.getBullet(item, tunnel, prevBullet);
      // scroll 记录每个隧道的最后一个弹幕
      this.bullets.push(this.scroll[tunnel]);
      item = undefined;
    }
    if (!item) return true;
    if (force) { // 如果是无限模式，就不关心是否会重叠弹幕
      this.scroll.push(this.getBullet(item, i % this.tunnel));
      return true;
    }
    return false;
  }
}
```

对于每个弹幕 `Item`，都会有个 `Bullet` 与它对应，下面是 `Bullet` 类完整代码。

```javascript
class Bullet {
  private readonly ctx: CanvasRenderingContext2D;
  private readonly danmaku: Danmaku;
  item: Item;
  prevBullet: Bullet;
  width = 0;
  x = 0;
  y = 0;
  speed = 0;
  tunnel = 0;

  constructor(
    danmaku: Danmaku,
    item: Item,
    tunnel: number,
    prevBullet?: Bullet
  ) {
    this.danmaku = danmaku;
    this.ctx = danmaku.ctx;
    this.reset(item, tunnel, prevBullet);
  }

  get length(): number { // 当前弹幕的长度
    return this.x + this.width;
  }

  get canRecycle(): boolean {
    return -this.x > this.width; // 是否可回收，当弹幕从屏幕上消失就可以回收它
  }

  reset(item: Item, tunnel: number, prevBullet?: Bullet): this {
    this.item = item;
    this.tunnel = tunnel;
    this.width = this.ctx.measureText(item.text).width;
    this.prevBullet = prevBullet;
    this.x = Math.max(prevBullet?.length ?? 0, this.danmaku.width);
    this.updateSpeed();
    this.updateY();
    return this;
  }

  recycle(): this {
    this.prevBullet = null;
    return this;
  }

  updateSpeed(): void {
    if (this.prevBullet && this.prevBullet.length > this.danmaku.width) {
      this.speed = this.prevBullet.speed;
    } else {
      this.speed = this.length / 500; // 这里为了简单换成具体的时间，弹幕只显示 500 帧
      if (this.prevBullet) {
        const maxSpeed = (this.x * this.prevBullet.speed) / this.prevBullet.length; // 上面提到的速度公式
        if (this.speed > maxSpeed) this.speed = maxSpeed;
      }
    }
  }

  updateY(): void {
    this.y = this.tunnel * this.danmaku.tunnelHeight;
  }

  display(): void {
    this.x -= this.actualSpeed;
    if (this.x > this.danmaku.width) return; // 只有出现在屏幕上才渲染
    this.ctx.fillStyle = this.item.color || '#fff';
    this.ctx.fillText(this.item.text, this.x, this.y);
  }
}
```

下面是 `Danmaku` 的完整代码。

```javascript
class Danmaku {
  readonly player: RPlayer;
  private running = false;
  private timer: number;
  private remain: Item[] = []; // 剩余弹幕
  private prevCurrentTime = -1; // 上次加载弹幕时的视频时间
  private bullets: Bullet[] = []; // 当前正在显示的弹幕
  private scroll: Bullet[] = []; // 保存每个隧道最后一个弹幕
  private pool: Bullet[] = []; // 回收的弹幕池
  tunnel = 0; // 一共有多少隧道
  tunnelHeight = 0; // 隧道高度
  readonly ctx: CanvasRenderingContext2D;
  readonly dom: HTMLCanvasElement;

  constructor(player: Player, items: Item[]) {
    this.player = player;
    this.dom = document.createElement('canvas');
    this.ctx = this.dom.getContext('2d');
    this.remain = [...items];
    
    player.appendChild(this.dom);
    player.on('pause', this.stop);
    player.on('play', this.start);
    player.on('ended', this.stop);

    this.resizeTunnelHeight();
    this.resizeTunnel();
    this.start();
  }

  get font(): string {
    return `bold 24px/1.1 SimHei, "Microsoft JhengHei", Arial, Helvetica, sans-serif`;
  }

  get width(): number {
    return this.player.width;
  }

  get height(): number {
    return this.player.height;
  }
  
  private initCanvas(): void {
    this.ctx.font = this.font;
  }
 
  private resizeTunnel(): void {
    this.tunnel = (this.height / this.tunnelHeight) | 0;
  }
  
  private resizeTunnelHeight(): void { // 这里使用 DOM 的方式获取隧道高度
    const text = document.createElement('span');
    text.innerText = '中'
    text.style.font = this.font;
    text.style.position = 'absolute';
    text.style.opacity = '0';
    document.body.appendChild(text);
    const height = text.scrollHeight;
    document.body.removeChild(text);
    this.tunnelHeight = height + 1;
 }

  private start = (): void => {
    if (this.running) return;
    this.running = true;
    this.initCanvas();
    this.update();
  };
  
  private stop = (): void => {
    this.running = false;
    cancelAnimationFrame(this.timer);
  };

  private update = (): void => {
    this.draw();
    this.timer = requestAnimationFrame(this.update);
  };

  private draw(): void {
    const items = this.load();
    if (items) {
      for (let i = 0, l = items.length; i < l; i++) {
        if (!this.fill(items[i], i)) break;
      }
    }
    this.ctx.clearRect(0, 0, this.dom.width, this.dom.height);
    this.bullets.forEach((bullet) => bullet.display());
    this.clearBullets();
  }

  private recycleBullet(b: Bullet): void {
    if (this.pool.length < 20) { // 默认弹幕池大小小于 20
      this.pool.push(b.recycle());
    }
  }

  private clearBullets(): void { // 清理已经消失在屏幕上的弹幕
    const bullets: Bullet[] = [];
    this.bullets.forEach((b) => {
      if (b.canRecycle) {
        this.recycleBullet(b);
      } else {
        bullets.push(b);
      }
    });
    this.bullets = bullets;
    for (let i = 0; i < this.tunnel; i++) {
      if (this.scroll[i] && this.scroll[i].canRecycle) {
        this.scroll[i] = undefined;
      }
    }
  }

  private getBullet(item: Item, tunnel: number, prevBullet?: Bullet): Bullet {
    const bullet = this.pool.pop();
    if (bullet) return bullet.reset(item, tunnel, prevBullet);
    return new Bullet(this, item, tunnel, prevBullet);
  }

  private getShortestTunnel(): [number, Bullet] { // 获取最短的隧道
    let length = Infinity;
    let tunnel = -1;
    let bullet: Bullet = null;
    for (let i = 0; i < this.tunnel; i++) {
      if (!this.scroll[i] || this.scroll[i].canRecycle) return [i, null];
      const l = this.scroll[i].length;
      if (l < length) {
        length = l;
        tunnel = i;
        bullet = this.scroll[i];
      }
    }
    return [tunnel, bullet];
  }
  
  private load(): void | Item[] {
    if (!this.remain.length) return;
    const time = this.player.currentTime | 0;
    if (this.prevCurrentTime === time) return;
    this.prevCurrentTime = time;
    const remain: Item[] = [];
    let toShow: Item[] = [];

    for (let i = 0, l = this.remain.length; i < l; i++) {
      const item = this.remain[i];
      if (item.time === time) {
        toShow.push(item);
      } else if (item.time > time) {
        remain.push(item);
      }
    }
    this.remain = remain;
    if (!toShow.length) return;
    return toShow;
  }

  private fill(item: Item, i = 0, force = false): boolean {
    const [tunnel, prevBullet] = this.getShortestTunnel();
    if (!prevBullet || prevBullet.length < this.width + 200) { 
      // 这里限制弹幕数量的策略是，每个隧道长度不超过视频宽度加 200
      this.scroll[tunnel] = this.getBullet(item, tunnel, prevBullet);
      this.bullets.push(this.scroll[tunnel]);
      item = undefined;
    }
    if (!item) return true;
    if (force) { // 如果是无限模式，就不关是否会重叠弹幕
      this.scroll.push(this.getBullet(item, i % this.tunnel));
      return true;
    }
    return false;
  }
}
```

#### 性能

下面使用 chrome 开发者工具检测的截图，可以看到弹幕可以稳定到 60 fps。

![](https://pubimg.xingren.com/60ed86ef-8559-40bc-a4d2-6982de2eae11.jpg)

![](https://pubimg.xingren.com/d65aa7d5-8200-4cc8-98ec-0ee821bdbcba.jpg)

`canvas` 实现起来比较简单，在现代浏览器上也比较流畅。

如果这么简单就找到了这么流畅的方法，那也太小瞧 `IE` 了，在 `IE` `Edge` 浏览器上这个版本会有一点卡顿，虽然没有第一种方法那么严重，但还是影响观影。

为了在 `IE` 上也能流畅的发射弹幕，就需要使用下面这个方法。

> 完整代码 [@rplayer/danmaku](https://github.com/woopen/RPlayer/blob/1497ed6b0d09ff3cc414682c637a85a66c460624/packages/rplayer-danmaku/src/ts/danmaku.ts)

### transform 和 transition 版本

这个版本很接近纯 CSS 的方式，将弹幕的滚动交给 `transition`。这个版本与 `canvas` 版本有比较大的区别，而且比 `canvas` 版本复杂的多。

#### 原理

这个版本并不是利用 `requestAnimationFrame`，而是使用 `video` 元素的 `timeupdate` 事件，在该事件的回调函数中执行与 `canvas` 版本相同的步骤。

每个弹幕都有一个开始时间和结束时间，是播放视频的具体时间点。当视频播放到弹幕的开始时间时，就给弹幕设置 `transition` 属性，时长等于弹幕的结束时间减去开始时间。这样浏览器就自动帮我们执行弹幕滚动动画。并且监听弹幕的 `transitionend` 事件，当它触发时回收弹幕。

使用这种方法也让弹幕与视频时间轴绑定在一起。

#### 实现

首先来看看 `timeupdate` 的回调函数，它与 `canvas` 的 `requestAnimationFrame` 非常类型。

```javascript
private onTimeUpdate = (): void => {
    if (this.player.paused) return;
    const time = this.player.currentTime;
    const items = this.load(time);
    if (!items && !this.bullets.length) return;
    
    if (items) {
      for (let i = 0, l = items.length; i < l; i++) {
        if (!this.insert(items[i], time, i)) break;
      }
    }
    
    const bullets: Bullet[] = [];
    let bullet: Bullet;
    for (let i = 0, l = this.bullets.length; i < l; i++) {
      bullet = this.bullets[i];
      if (bullet.update(time)) { // update 方法返回 true 代表可以回收
        this.recycleBullet(bullet);
      } else {
        bullets.push(bullet);
      }
    }
    this.bullets = bullets;
};
```

为了弹幕之间不重叠，弹幕的结束时间要根据它前一个弹幕的结束时间来计算。通过上面的提到的公式，弹幕的时长的就等于

```javascript
let duration = (player.width + this.width) / (player.width / (prevBullet.endTime - this.startTime))
if (duration < 5) duration = 5 // 每个弹幕最少显示 5 秒
this.endTime = this.startTime + duration
```

为了知道下一个弹幕的开始时间，我们需要知道弹幕完全展示的时间点，也就是弹幕的右侧刚好与播放器的右侧接触的时间点。只有前一个弹幕完全展示出来，后一个弹幕才能开始。

弹幕再加个 `showTime` 属性代表它完全展示的时间。

```javascript
this.showTime = this.startTime + (this.width * duration) / (player.width + this.width) + 0.2;
```

通过 `showTime` 就可以知道下一个弹幕的开始时间，后面加 `0.2` 秒，为了让每个弹幕之间有点间隙。

```javascript
private insert(item: Item, time: number, i = 0, force?: boolean): boolean {
    const [tunnel, prevBullet] = this.getShortestTunnel();
    if (!prevBullet || prevBullet.showTime <= time + 2) {
        this.scroll[tunnel] = this.getBullet(item, tunnel, time, prevBullet);
        this.bullets.push(this.scroll[tunnel]);
        item = undefined;
    }
    if (!item) return true;
    if (this.opts.unlimited || force) {
      this.bullets.push(this.getBullet(item, i % this.tunnel, time));
      return true;
    }
    return false;
}
```

`getShortestTunnel` 获取 `showTime` 最短的一个隧道，这里设置只要这个隧道在未来两秒可以完全展示，就可以新增弹幕在这个隧道。

了解了 `insert` 方法下面来看，弹幕的 `update` 方法。

```javascript
update(time: number): boolean {
    if (this.canRecycle) return true;
    if (this.running || this.startTime > time) return false;
    this.startTime = time;
    this.setTransition(this.endTime - time);
    this.dom.offsetTop;
    this.setTransform(player.width + this.width);
    this.running = true;
}
```

如果还没到开始时间或已经在运行，就直接返回，否则就设置 `transition` 和 `transform` 来发射弹幕。

上面的代码就已经可以顺利运行弹幕了。但是如果视频暂停了呢？这个版本并不可以取消定时器来暂停动画。

```javascript
player.on('pause', () => {
    const time = player.currentTime;
    this.bullets.forEach((b) => b.pause(time));
});
```

在 `Danmaku` 类中监听暂停时间，并执行所有运行弹幕的 `pause` 方法，在 `pause` 方法中我们需要计算当前的弹幕已经走了对少距离，并设置 `transform`，然后把 `transition` 设置为 `0` 来停止动画。

```javascript
pause(time: number): void {
    if (time <= this.startTime || this.endTime <= time) {
      return;
    }
    const x =
      (this.length / (this.endTime - this.startTime)) *
        (time - this.startTime) + this.lastX;
    // length 初始值为 player.width + this.width
    this.setTransform(x);
    this.setTransition(0);
    this.length = player.width - x + this.width;
    this.lastX = x;
    this.running = false;
}
```

当视频恢复播放时会触发 `timeupdate` 事件，所以无需监听 `play` 事件。

#### 性能

如果直接用上面代码在 `IE` 浏览器中运行，会发现有非常明显的卡顿现象，这是因为少了几个 CSS 属性。

`will-change: transform;` 告知浏览器该元素会有哪些变化的方法，这样浏览器可以在元素属性真正发生变化之前提前做好对应的优化准备工作。因为弹幕一出生就会设置 `transform` 属性，所以这个属性非常适合它。

为了启用硬件加速设置 `transform` 时，设置 `x,y,z` 三个值。而不是 `translateX` 这样并不能启用硬件加速。我们需要设置成 `translateX(${player.width + this.width}px) translateY(0) translateZ(0)`。还可以设置 `perspective` 和 `backface-visibility` 防止可能出现的动画闪动。

下面是 chrome 开发者工具的截图

![](https://pubimg.xingren.com/6753dcda-1236-40e4-885b-cdb2158adfa2.jpg)

![](https://pubimg.xingren.com/3ecd1bf3-3e1e-4c23-af1e-ad450fef8b0c.jpg)

这种方法可以在 `IE` `Edge` 上流畅的运行，但是到了 `firefox` 上有时可能会出现一点点小卡顿，影响并不大。`canvas` 版本在 `firefox` 上表现要比这种方法好一点点。

> 完整代码 [@rplayer/danmaku](https://github.com/woopen/RPlayer/tree/master/packages/rplayer-danmaku)

## 比较

方法 | 描述
--- | ---
canvas 版本 | 实现简单，可以流畅在主流的现代浏览器。缺点就是在 `IE` `Edge` 上有点卡顿
transform 和 transition 版本 | 实现复杂，可以流畅大部分浏览器包括 `IE` `Edge`，但在 `firefox` 有时会不如 canvas 版本流畅

`transform, transition` 版本是比 canvas 更好的选择，它在大部分浏览器上都可以流畅运行，而且对于弹幕种添加图片或一些其他特效的情况下 css 实现也更加简单。

## 总结

这里介绍了三种方法，第一种方法虽然有 `canvas` 版本的实现简单和 `css` 灵活性，但是非常卡顿。如果不嫌麻烦的话 `canvas` 版本和 `transform,transition` 版本也可以同时实现，`firefox` 中使用 `canvas` 版本，否则使用 `transform,transition` 版本。

## 参考

- [弹幕](https://baike.baidu.com/item/%E5%BC%B9%E5%B9%95/6278416)