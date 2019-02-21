# 从零打造Echarts —— v1 ZRender和MVC
本篇开始进入正文。
## 写在前面
- 图形、元素、图形元素，都指的是`XElement`，看情况哪个顺口用哪个。
- `ts`可能会报警告，我只是想用代码提示功能而已，就不管辣么多了。
- 文内并没有贴出所有代码，且随着版本更迭，可能有修改不及时导致文内代码和源码不一致的情况，可以参考源码进行查看。
- 源码查看的方式，源码放在[这里](https://github.com/webbillion/xrender-src)，每一个版本都有对应的分支。
- 由于水平所限，以及后续设计的变更，无法在最开始的版本中就写出最优的代码，甚至可能还会存在一些问题，如果遇到你认为不应该这样写的代码请先不要着急。
## zrender
`zrender`是`echarts`使用的`2d`渲染器，意思是，对于`2d`图表，`echarts`更多的是对于数据的处理，将数据绘制到`canvas`上这一步是由`zrender`来完成的。

大概流程就是，使用者告诉`echarts`我要画条形图，有十条数据，`echarts`计算出条形图高度和坐标和使用`zrender`在画布上绘制坐标轴和十个矩形。它也是`echarts`唯一的依赖。它是一个轻量的二维绘图引擎，但是实现了很多功能。本文就从实现`zrender`开始作为实现`echarts`的第一步。
## 本篇目标
前文说到，打造`echarts`从打造一个`zrender`开始，但是`zrender`的功能同样很多，不可能一步到位，所以先从最基础的功能开始，而我们的库我给它命名为`XRender`，即无限可能的渲染器。本篇结束后它将实现`zrender`的以下功能。
```javascript
import * as xrender from '../xrender'

let xr = xrender.init('#app')
let circle = new xrender.Circle({
  shape: {
    cx: 40,
    cy: 40,
    r: 20
  }
})
xr.add(circle)
// 现在画布上有一个半径为20的圆了
```
## 正文
### 模式
首先明确一点，我们根据数据来实现视图。

然后看看我们需要哪些东西来实现我们要的功能。
- 要绘制的元素，如圆、长方形， 即`Element`，为了和`html`中区分，暂命名为`XElment`。
- 因为会有多个元素，我们需要对其进行增查删改等管理，类似于`3d`游戏开发中常见的`scene`（场景），这里叫做`Stage`，舞台。`zrender`中叫做`Storage`。都差不多。
- 需要将舞台上的元素绘制到画布上，叫做`Paniter`。
- 最终需要将上面的三者关联起来，即`XRender`。

也就是`MV`模式。

考虑到会有多种图形，所以`xrender`最终导出的是一个命名空间，遵循`zrender`的设计，并不向外暴露`XRender`类。那么接下来就可以开始写代码了。
### 环境搭建
为了方便，我使用了`vue-cli`搭建环境，你也可以用其它方式，只要能支持出现的语法就行。接着创建`xrender`目录。或者克隆仓库一键安装。根据上面列出的类，创建如下文件。
``` sh
index.js # 外部引用的入口
Painter.js 
Stage.js
XElement.js
XRender.js
```
但是需要做一点小小的修正，因为`XElement`应该是一个抽象类，它只代表一个元素，它本身不提供任何绘制方法，提供绘制方法的应该是继承它的圆`Circle`类。所以修改后的目录如下。
``` sh
│  index.js
│  Painter.js
│  Stage.js
│  XRender.js
│
└─xElements
        Circle.js
        XElement.js
```
接着在每个文件内创建对应的类，并让构造函数打印出当前类的名称，然后导出，以便搭建整体架构。如：
```javascript
class Stage {
  constructor () {
    console.log('Stage')
  }
}

export default Stage

```
然后编写`index.js`
```javascript
import XRedner from './XRender'
// 导出具体的元素类
export { default as Circle } from './xElements/Circle'
// 只暴露方法而不直接暴露`XRender`类
export function init () {
  return new XRedner()
}

```
在使用它之前我们还得为`XRender`类添加`add`方法，尽管现在它什么都没做。
```javascript
// 尽管没有使用，但是需要用它来做类型提示
// 用Flow和ts，或jsdoc等，都嫌麻烦
import XElement from "./xElements/XElement";

class XRender {
  /**
   * 
   * @param {XElement} xel 
   */
  add (xel) {
    console.log('add an el')
  }
}
```
接下来就可以在`App.vue`中写最开始的代码。如果一切顺利，应该能在控制台上看到
``` sh
XRender
Circle
add an el
```
## 细节填充
在下一步之前，我们可能需要一些辅助函数，比如我们经常会判断某个参数是不是字符串。为此我们创建`util`文件夹来存放辅助函数。
### XElement
图形元素，一个抽象类，它应该帮继承它的类如`Circle`处理好样式之类的选项，`Circle`只需要绘制即可。显然它的构造函数应该接受一个选项作为参数，包括这些：
```typescript
import { merge } from '../util'
/**
 * 目前什么都没有
 */
export interface XElementShape {
}
/**
 * 颜色
 */
type Color = String | CanvasGradient | CanvasPattern
export interface XElementStyle {
  // 先只设定描边颜色和填充
  /**
   * 填充
   */
  fill?: Color
  /**
   * 描边
   */
  stroke?: Color
}
/**
 * 元素选项接口
 */
interface XElementOptions {
  /**
   * 元素类型 
  */
  type?: string
  /**
   * 形状
   */
  shape?: XElementShape
  /**
   * 样式
   */
  style?: XElementStyle
}
```
接着是对类的设计，对于所有选项，它应该有一个默认值，然后在更新时被覆盖。
```typescript
class XElement {
  shape: XElementShape = {}
  style: XElementStyle = {}
  constructor (opt: XElementOptions) {
    this.options = opt
  }
  /**
   * 这一步不在构造函数内进行是因为放在构造函数内的话，会被子类的默认属性声明重写
   */
  updateOptions () {
    let opt = this.options
    if (opt.shape) {
      // 这个函数会覆盖第一个参数中原来的值
      merge(this.shape, opt.shape)
    }
    if (opt.style) {
      merge(this.style, opt.style)
    }
  }
}
```
对于一个元素，应该提供一个绘制方法，正如上面所提到的，这由它的子类提供。此外在绘制之前还需要对样式进行处理，绘制之后进行还原。而这就需要一个`canvas`的`context`。这里认为它由外部提供。涉及到的`api`请自行查阅。
```typescript
class XElement {
  /**
   * 绘制
   */
  render (ctx: CanvasRenderingContext2D) {

  }
  /**
   * 绘制之前进行样式的处理
   */
  beforeRender (ctx: CanvasRenderingContext2D) {
    this.updateOptions()
    let style = this.style
    ctx.save()
    ctx.fillStyle = style.fill
    ctx.strokeStyle = style.stroke
    ctx.beginPath()
  }
  /**
   * 绘制之后进行还原
   */
  afterRender (ctx: CanvasRenderingContext2D) {
    ctx.stroke()
    ctx.fill()
    ctx.restore()
  }
  /**
   * 刷新，这个方法由外部调用
   */
  refresh (ctx: CanvasRenderingContext2D) {
    this.beforeRender(ctx)
    this.render(ctx)
    this.afterRender(ctx)
  }
```
为什么不在创建它的时候传入`ctx`作为属性的一部分？实际上这完全可行。只是`zrender`这样设计，我也暂时先这么做。可能是为了解耦以及多种`ctx`的需要。
### Circle
基类`XElement`已经初步构造完毕，接下来就来构造`Circle`，我们只需声明它需要哪些配置，并提供绘制方法即可。也就是，如何绘制一个圆。
```typescript
import XElement, { XElementShape } from './XElement'

interface CircleShape extends XElementShape {
  /**
   * 圆心x坐标
   */
  cx: number
  /**
   * 圆心y坐标
   */
  cy: number
  /**
   * 半径
   */
  r: number
}
interface CircleOptions extends XElementOptions {
  shape: CircleShape
}

class Circle extends XElement {
  name ='circle'
  shape: CircleShape = {
    cx: 0,
    cy: 0,
    r: 100
  }
  constructor (opt: CircleOptions) {
    super(opt)
  }
  render (ctx: CanvasRenderingContext2D) {
    let shape = this.shape
    ctx.arc(shape.cx, shape.cy, shape.r, 0, Math.PI * 2, true)
  }
}

export default Circle
```
来验证一下吧，在`App.vue`中加入如下代码:
```typescript
mounted () {
  let canvas = document.querySelector('#canvas') as HTMLCanvasElement
  let ctx = canvas.getContext('2d') as CanvasRenderingContext2D
  circle.refresh(ctx)
}
```
查看页面，已经有了一个黑色的圆。
### Stage
需要它对元素进行增查删改，很容易写出这样的代码。
```typescript
class Stage {
  /**
   * 所有元素的集合
   */
  xelements: XElement[] = []
  constructor () {
    console.log('Stage')
  }
  /**
   * 添加元素
   * 显然可能会添加多个元素
   */
  add (...xelements: XElement[]) {
    this.xelements.push(...xelements)
  }
  /**
   * 删除指定元素
   */
  delete (xel: XElement) {
    let index = this.xelements.indexOf(xel)
    if (index > -1) {
      this.xelements.splice(index)
    }
  }
  /**
   * 获取所有元素
   */
  getAll () {
    return this.xelements
  }
}
```
### Painter
绘画控制器，它将舞台上的元素绘制到画布上，那么创建它时就需要提供一个`Stage`和画布——当然，库的通用做法是也可以提供一个容器，由库来创建画布。

```typescript
/**
 * 创建canvas
 */
function createCanvas (dom: string | HTMLCanvasElement | HTMLElement) {
  if (isString(dom)) {
    dom = document.querySelector(dom as string) as HTMLElement
  }
  if (dom instanceof HTMLCanvasElement) {
    return dom
  }
  let canvas = document.createElement('canvas');
  (<HTMLElement>dom).appendChild(canvas)

  return canvas
}

class Painter {
  canvas: HTMLCanvasElement
  stage: Stage
  ctx: CanvasRenderingContext2D
  constructor (dom: string | HTMLCanvasElement | HTMLElement, stage: Stage) {
    this.canvas = createCanvas(dom)
    this.stage = stage
    this.ctx = this.canvas.getContext('2d')
  }
}
```
它应该实现一个`render`方法，遍历`stage`中的元素进行绘制。
```typescript
render () {
    let xelements = this.stage.getAll()
    for (let i = 0; i < xelements.length; i += 1) {
      xelements[i].refresh(this.ctx)
    }
  }
```
### XRender
最后一步啦，创建`XRender`将它们关联起来。这很简单。
```typescript
import XElement from './xElements/XElement'
import Stage from './Stage'
import Painter from './Painter'

class XRender {
  stage: Stage
  painter: Painter
  constructor (dom: string | HTMLElement) {
    let stage = new Stage()
    this.stage = stage
    this.painter = new Painter(dom, stage)
  }
  add (...xelements: XElement[]) {
    this.stage.add(...xelements)
    this.render()
  }
  render () {
    this.painter.render()
  }
}
```
现在去掉之前试验`Circle`的代码，保存之后可以看见，仍然绘制出了一个圆，这说明成功啦！

让我们再多添加几个圆试一下，并传入不同的参数。
```typescript
let xr = xrender.init('#app')
    let circle = new xrender.Circle({
      shape: {
        cx: 40,
        cy: 40,
        r: 20
      }
    })
    let circle1 = new xrender.Circle({
      shape: {
        cx: 60,
        cy: 60,
        r: 20
      },
      style: {
        fill: '#00f'
      }
    })
    let circle2 = new xrender.Circle({
      shape: {
        cx: 100,
        cy: 100,
        r: 40
      },
      style: {
        fill: '#0ff',
        stroke: '#f00'
      }
    })
    xr.add(circle, circle1, circle2)
```
可以看到屏幕上出现了3个圆。接下来我们再尝试扩展一个矩形。
```typescript
interface RectShape extends XElementShape {
  /**
   * 左上角x
   */
  x: number
  /**
   * 左上角y
   */
  y: number
  width: number
  height: number
}
interface RectOptions extends XElementOptions {
  shape: RectShape
}

class Rect extends XElement {
  name ='rect'
  shape: RectShape = {
    x: 0,
    y: 0,
    width: 0,
    height: 0
  }
  constructor (opt: RectOptions) {
    super(opt)
  }
  render (ctx: CanvasRenderingContext2D) {
    let shape = this.shape
    ctx.rect(shape.x, shape.y, shape.width, shape.height)
  }
}
```
然后在`App.vue`中添加代码：
```typescript
let rect = new xrender.Rect({
      shape: {
        x: 120,
        y: 120,
        width: 40,
        height: 40
      },
      style: {
        fill: 'transparent'
      }
    })
    xr.add(rect)
```
可以看到矩形出现了。

## 小结
虽然还有很多问题，比如样式规则不完善，比如多次调用`add`会有不必要的重绘；实现添加圆和矩形这样的功能搞得如此复杂看起来也有点不必要。但是我们已经把基础的框架搭建好了，接下来相信可以逐步完善，最终达成我们想要的效果。
## V2预览
[下个版本](./Version2.md)中除了解决小结中出现的两个问题外，还将实现图形分层的功能，即指定图形的层叠顺序。