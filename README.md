# 一笔画小游戏（hackthon）

一笔画是<u>图论</u><sup>[科普](https://zh.wikipedia.org/wiki/%E5%9B%BE%E8%AE%BA)</sup>中一个著名的问题，它起源于<u>柯尼斯堡七桥问题</u><sup>[科普](https://zh.wikipedia.org/wiki/%E6%9F%AF%E5%B0%BC%E6%96%AF%E5%A0%A1%E4%B8%83%E6%A1%A5%E9%97%AE%E9%A2%98)</sup>。数学家欧拉在他 1736 年发表的论文《柯尼斯堡的七桥》中不仅解决了七桥问题，也提出了一笔画定理，顺带解决了一笔画问题。用图论的术语来说，对于一个给定的<u>连通图</u><sup>[科普](https://zh.wikipedia.org/wiki/%E8%BF%9E%E9%80%9A%E5%9B%BE)</sup>存在一条恰好包含所有线段并且没有重复的路径，这条路径就是「一笔画」。

寻找连通图这条路径的过程就是「一笔画」的游戏过程，如下：

## 游戏的实现

「一笔画」的实现不复杂，笔者把实现过程分成两步：

1. 底图绘制
2. 交互绘制

「底图绘制」把连通图以「点线」的形式显示在画布上，是游戏最容易实现的部分；「交互绘制」是用户绘制解题路径的过程，这个过程会主要是处理点与点动态成线的逻辑。

### 底图绘制

「一笔画」是多关卡的游戏模式，笔者决定把关卡（连通图）的定制以一个配置接口的形式对外暴露。对外暴露关卡接口需要有一套描述连通图形状的规范，而在笔者面前有两个选项：

- 点记法
- 线记法

举个连通图 ------ 五角星为例来说一下这两个选项。

点记法如下：

```javascript
levels: [
	// 当前关卡
	{
		name: "五角星",
		coords: [
			{x: Ax, y: Ay},
			{x: Bx, y: By},
			{x: Cx, y: Cy},
			{x: Dx, y: Dy},
			{x: Ex, y: Ey},
			{x: Ax, y: Ay}
		]
	}
	...
]
```

线记法如下：

```javascript
levels: [
  // 当前关卡
  {
    name: "五角星",
    lines: [
      { x1: Ax, y1: Ay, x2: Bx, y2: By },
      { x1: Bx, y1: By, x2: Cx, y2: Cy },
      { x1: Cx, y1: Cy, x2: Dx, y2: Dy },
      { x1: Dx, y1: Dy, x2: Ex, y2: Ey },
      { x1: Ex, y1: Ey, x2: Ax, y2: Ay }
    ]
  }
];
```

「点记法」记录关卡通关的一个答案，即端点要按一定的顺序存放到数组 `coords`中，它是有序性的记录。「线记法」通过两点描述连通图的线段，它是无序的记录。「点记法」最大的优势是表现更简洁，但它必须记录一个通关答案，笔者只是关卡的搬运工不是关卡创造者，所以笔者最终选择了「线记法」。：）

### 交互绘制

在画布上绘制路径，从视觉上说是「选择或连接连通图端点」的过程，这个过程需要解决 2 个问题：

- 手指下是否有端点
- 选中点到待选中点之间能否成线

收集连通图端点的坐标，再监听手指滑过的坐标可以知道「手指下是否有点」。以下伪代码是收集端点坐标：

```javascript
// 端点坐标信息
let coords = [];
lines.forEach(({ x1, y1, x2, y2 }) => {
  // (x1, y1) 在 coords 数组不存在
  if (!isExist(x1, y1)) coords.push([x1, y1]);
  // (x2, y2) 在 coords 数组不存在
  if (!isExist(x2, y2)) coords.push([x2, y2]);
});
```

以下伪代码是监听手指滑动：

```javascript
easel.addEventListener("touchmove", e => {
  let x0 = e.targetTouches[0].pageX,
    y0 = e.targetTouches[0].pageY;
  // 端点半径 ------ 取连通图端点半径的2倍，提升移动端体验
  let r = radius * 2;
  for (let [x, y] of coords) {
    if (Math.sqrt(Math.pow(x - x0, 2) + Math.pow(y - y0), 2) <= r) {
      // 手指下有端点，判断能否连线
      if (canConnect(x, y)) {
        // todo
      }
      break;
    }
  }
});
```

在未绘制任何线段或端点之前，手指滑过的任意端点都会被视作「一笔画」的起始点；在绘制了线段（或有选中点）后，手指滑过的端点能否与选中点串连成线段需要依据现有条件进行判断。

把「可以与指定端点连接成线段的端点称作**有效连接点**」。连通图端点的有效连接点从连通图的线段中提取：

```javascript
coords.forEach(coord => {
  // 有效连接点（坐标）挂载在端点坐标下
  coord.validCoords = [];
  lines.forEach(({ x1, y1, x2, y2 }) => {
    // 坐标是当前线段的起点
    if (coord.x === x1 && coord.y === y1) {
      coord.validCoords.push([x2, y2]);
    }
    // 坐标是当前线段的终点
    else if (coord.x === x2 && coord.y === y2) {
      coord.validCoords.push([x1, y1]);
    }
  });
});
```

But...有效连接点只能判断两个点是否为底图的线段，这只是一个静态的参考，在实际的「交互绘制」中，AB 已串连成线段，当前选中点 B 的有效连接点是 A 与 C。AB 已经连接成线，如果 BA 也串连成线段，那么线段就重复了，所以此时 BA 不能成线，只有 AC 才能成线。

对选中点而言，它的有效连接点有两种：

- 与选中点「成线的有效连接点」
- 与选中点「未成线的有效连接点」

其中「未成线的有效连接点」才能参与「交互绘制」，并且它是动态的。

回头本节内容开头提的两个问题「手指下是否有端点」 与 「选中点到待选中点之间能否成线」，其实可合并为一个问题：**手指下是否存在「未成线的有效连接点」**。只须把监听手指滑动遍历的数组由连通图所有的端点坐标 `coords` 替换为当前选中点的「未成线的有效连接点」即可。

至此「一笔画」的主要功能已经实现。可以抢先体验一下：

![demo](http://7xv39r.com1.z0.glb.clouddn.com/2017-10-31-qr.png?v=2)

[https://leeenx.github.io/OneStroke/src/onestroke.html](https://leeenx.github.io/OneStroke/src/onestroke.html)
