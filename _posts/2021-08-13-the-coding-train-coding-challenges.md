---
layout: post
title: 一些有意思的 The Coding Train 编码挑战
date: 2021-08-17 12:00:00 +0800
tags: [coding_train, p5.js]
---

***警告：多图杀猫***

## 背景

随着小朋友日夜长大，好奇心也不断增长。我作为家长自然希望能稍加引导，以免他长成个熊孩子。于是就去油管上找了下有啥面向初学者的编程课程，结果很有缘分地找到了一个叫`The Coding Train`[1]的频道。主播名叫`Daniel Shiffman`，订阅粉丝超百万。为什么说有缘分呢，因为Dan之前写过本书叫`"The Nature of Code"`，国内也引进过，译作《代码本色》，并且在我的书架上积灰了多年：

![代码本色](/assets/images/2021-08-13/thenatureofcode.jpg)

书中使用`Processing`模拟了一些自然系统。`Processing`基于Java，包含了自己的IDE。而在油管频道中Dan则是大量使用了`p5.js`。`p5.js`继承了大部分的`Processing`的API，并且可以直接在浏览器里运行，对于初学者来说会更友好。

频道的官网上有一块叫`Coding Challenges`[2]，是由频道粉丝提出一些小的idea，然后由Dan在直播中尝试用`p5.js`或`Processing`来实现，同时解释一些基础的编程知识。虽然中途时不时会翻车，但最终的实现效果通常都很不错。比如这个[SuperShape3D](https://qszhu.github.io/thecodingtrain_coding_challenges/26/index.html)[3]：

![SuperShape3D](/assets/images/2021-08-13/26.gif)

我家小朋友小时候就挺喜欢玩：

![SuperShape3D](/assets/images/2021-08-13/play.gif)

又比如这个[Phyllotaxis](https://qszhu.github.io/thecodingtrain_coding_challenges/30/index.html)[4]：

![Phyllotaxis](/assets/images/2021-08-13/30.gif)

可以看出Dan真的很喜欢彩虹色。

不过有些东西就比较抽象，可能我比较喜欢，但小朋友就没啥兴趣。比如这个[Kaleidoscope Snowflake](https://qszhu.github.io/thecodingtrain_coding_challenges/155/index.html)[5]：

![Kaleidoscope Snowflake](/assets/images/2021-08-13/155.gif)

为了找到能让小朋友满意的玩意儿，我利用业余时间把频道里的这一块视频都撸了一遍[6]，前后断断续续花了将近一年的时间。要不是因为疫情隔离原因Dan停更了一段时间，以及现在Dan好像又在写`The Nature of Code 2`的样子，导致后期这部分视频数量急剧减少，不然可能会花上更多的时间。

遗憾的是，撸完了现有的视频也没找到更多能让小朋友满意的东西。不过有些东西让我觉得挺有意思的，就在这里记录下。

## [10PRINT](https://qszhu.github.io/thecodingtrain_coding_challenges/76/index.html)

提问：生成下图需要多少代码？

![10PRINT](/assets/images/2021-08-13/76.png)

答案是一行C64 Basic代码[7]：

```
10 PRINT CHR$(205.5+RND(1)); : GOTO 10
```

当然用`p5.js`实现的话不只一行[8]。信息时代早期人们的创造力着实让我着迷。

## Perlin噪声

似乎IT行业中，除了游戏行业就很少会用到Perlin噪声。而在游戏行业里，最常用到Perlin噪声的地方就是[程序化生成地形](https://qszhu.github.io/thecodingtrain_coding_challenges/11/index.html)[9]：

![Terrain](/assets/images/2021-08-13/11.gif)

大名鼎鼎的`Minecraft`在生成地形时就大量使用了Perlin噪声[10]。此外Dan还展示了如何用多维的Perlin噪声实现那些看似随机，实则[无限循环的动图](https://qszhu.github.io/thecodingtrain_coding_challenges/136/index.html)[11]。

## 给图标磨个圆角

据说某米今年花了200万给自家的图标磨了个圆角[12]，甚至还给了个公式，让人不明觉厉。然而早在5年前的2016年[13]，Dan就已经手撸过[同一个公式](https://qszhu.github.io/thecodingtrain_coding_challenges/19/index.html)了[14]：

![SuperEllipse](/assets/images/2021-08-13/19.gif)

Dan作为纽约大学艺术学院的副教授，不知道在设计界有多大的名气？“东瀛罗永浩”的设计团队会是在Dan这里得到的灵感吗？

## 傅立叶变换

这是我个人比较喜欢的一个主题。首先，一系列大小不同的相切圆环的运动轨迹居然可以形成某种[分形](https://qszhu.github.io/thecodingtrain_coding_challenges/61/index.html)[15]：

![FractalSpirograph](/assets/images/2021-08-13/61.gif)

然后如果都在同一个方向上叠加的话就可以重现[傅立叶变换](https://qszhu.github.io/thecodingtrain_coding_challenges/125/index.html)[16]：

![Fourier Series](/assets/images/2021-08-13/125.gif)

依稀记得当年学数字信号处理的时候，老师让我们自己手绘不同周期的正弦波并叠加，以此来直观感受对方波的傅立叶变换。而现在的小朋友就可以很方便地在浏览器里手撸一个出来。我看着他们，满怀羡慕。

上面是一维的情况，简单扩展到二维的话可以画出一些美妙的[图形](https://qszhu.github.io/thecodingtrain_coding_challenges/116/index.html)[17]：

![Lissajous](/assets/images/2021-08-13/116.gif)

而对任意的平面点坐标序列做傅立叶变换的话，就可以[绘制任意图形](https://qszhu.github.io/thecodingtrain_coding_challenges/130/index.html)[18]：

![Fourier Transform Drawing](/assets/images/2021-08-13/130.gif)

## 3D

`p5.js`不但可以绘制2D图形，绘制[简单的3D图形](https://qszhu.github.io/thecodingtrain_coding_challenges/86/index.html)也很方便[19]：

![Bees and Bombs](/assets/images/2021-08-13/86.gif)

不过毕竟只是个lib，连个场景图都没有，做[略微复杂点的场景](https://qszhu.github.io/thecodingtrain_coding_challenges/142/index.html)就很痛苦[20]：

![Rubik's Cube](/assets/images/2021-08-13/142.gif)

Dan在做这个魔方的时候也是屡屡翻车。

## [Pi Collisions](https://qszhu.github.io/thecodingtrain_coding_challenges/139/index.html)

这个我个人也非常喜欢。谁能想到两个滑块之间的完全弹性碰撞次数居然跟`π`有联系[21]：

![Pi Collisions](/assets/images/2021-08-13/139.gif)

著名的3b1b对此也有解释[22]。另一篇解释[23]中还牵涉到了量子计算，其中提到了把大滑块看成了相同质量的小滑块的组合，而这正是Dan在实现时用到的方法！

## Ray casting

最近某来步了特斯拉自动驾驶“杀人”的后尘引起热议[24]。跟特斯拉一样，某来也没有激光雷达。Musk很早就说过傻瓜才用激光雷达[25]。那么如果有完美的激光雷达会怎样？借助Dan的[小项目](https://qszhu.github.io/thecodingtrain_coding_challenges/145/index.html)告诉你[26]：

![Ray Casting](/assets/images/2021-08-13/145.gif)

如果在发出的每条射线的方向上都可以精确得知自己到障碍物的距离，那么就可以渲染出一个[伪3D的场景](https://qszhu.github.io/thecodingtrain_coding_challenges/146/index.html)[27]了：

![Rendering Ray Casting](/assets/images/2021-08-13/146.gif)

早期的FPS游戏如Wolf3D和Doom等都用了这样的方法。相比纯视觉的方案，实现简单可靠，还有完美的可解释性，确实是连傻瓜都会用的方法。

## [MetaBalls](https://qszhu.github.io/thecodingtrain_coding_challenges/28/index.html)

![MetaBalls](/assets/images/2021-08-13/28.gif)

这本来是个平平无奇的小项目[28]，然而我的实现却跟Dan的实现在运行效率上差了不只一点半点。仔细比对以后发现差异就在一个destructure操作上：

```javascript
const [dx, dy] = [x - bx, y - by];
```

这个操作是在双重循环里的，改成单独赋值的话效率明显提升：

```javascript
const dx = x - bx
const dy = y - by
```

尝试做了一下benchmark，发现在Firefox里destructure操作有明显的性能问题[29]：

![benchmark](/assets/images/2021-08-13/benchmark.jpg)

Chrome里结果更接近一些，但destructure操作的性能还是不如直接赋值。所以说即便浏览器普遍都原生支持了ES6语法，使用一个ES5的transpiler还是很必要的。

先就这样吧，如果Dan以后又更新了哪些有意思的东西我再来更新一波。

## 参考资料
* [1] [The Coding Train](https://www.youtube.com/c/TheCodingTrain)
* [2] [Coding Challenges · The Coding Train](https://thecodingtrain.com/CodingChallenges/)
* [3] [3D Supershapes - Coding Challenge #26 · The Coding Train](https://thecodingtrain.com/CodingChallenges/026-supershape3d.html)
* [4] [Phyllotaxis - Coding Challenge #30 · The Coding Train](https://thecodingtrain.com/CodingChallenges/030-phyllotaxis.html)
* [5] [Kaleidoscope Snowflake #SupportP5 - Coding Challenge #155 · The Coding Train](https://thecodingtrain.com/CodingChallenges/155-kaleidoscope-snowflake.html)
* [6] [TheCodingTrain Coding Challenges](https://qszhu.github.io/thecodingtrain_coding_challenges/)
* [7] [10 PRINT CHR$(205.5+RND(1)); : GOTO 10](https://10print.org/)
* [8] [10PRINT in p5.js - Coding Challenge #76 · The Coding Train](https://thecodingtrain.com/CodingChallenges/076-10print.html)
* [9] [3D Terrain Generation with Perlin Noise in Processing - Coding Challenge #11 · The Coding Train](https://thecodingtrain.com/CodingChallenges/011-perlinnoiseterrain.html)
* [10] [Noise generator – Official Minecraft Wiki](https://minecraft.fandom.com/wiki/Noise_generator)
* [11] [Perlin Noise GIF Loops - Coding Challenge #136.2 · The Coding Train](https://thecodingtrain.com/CodingChallenges/136.2-perlin-noise-gif-loops)
* [12] [价值200万的小米新LOGO 你上你也行？](https://www.cnbeta.com/articles/tech/1109067.htm)
* [13] [Coding Challenge #19: Superellipse](https://www.youtube.com/watch?v=z86cx2A4_3E)
* [14] [Superellipse - Coding Challenge #19 · The Coding Train](https://thecodingtrain.com/CodingChallenges/019-superellipse.html)
* [15] [Fractal Spirograph - Coding Challenge #61 · The Coding Train](https://thecodingtrain.com/CodingChallenges/061-fractal-spirograph.html)
* [16] [Fourier Series - Coding Challenge #125 · The Coding Train](https://thecodingtrain.com/CodingChallenges/125-fourier-series.html)
* [17] [Lissajous Curve Table - Coding Challenge #116 · The Coding Train](https://thecodingtrain.com/CodingChallenges/116-lissajous.html)
* [18] [Drawing with Fourier Transform and Epicycles - Coding Challenge #130.1 · The Coding Train](https://thecodingtrain.com/CodingChallenges/130.1-fourier-transform-drawing.html)
* [19] [Cube Wave by Bees and Bombs - Coding Challenge #86 · The Coding Train](https://thecodingtrain.com/CodingChallenges/086-beesandbombs.html)
* [20] [Rubik’s Cube Part 3 - Coding Challenge #142.3 · The Coding Train](https://thecodingtrain.com/CodingChallenges/142.3-rubiks-cube.html)
* [21] [Calculating Digits of Pi with Collisions - Coding Challenge #139 · The Coding Train](https://thecodingtrain.com/CodingChallenges/139-pi-collisions.html)
* [22] [Why do colliding blocks compute pi?](https://www.youtube.com/watch?v=jsYwFizhncE)
* [23] [这一切要从碰撞的滑块和量子搜索讲起。。。](https://mp.weixin.qq.com/s/eabgu2X-7z0wamiZSEFYaA)
* [24] [蔚来车祸之殇 乃新势力造车全行业之痛 _ 东方财富网](https://stock.eastmoney.com/a/202108172049305587.html)
* [25] [特斯拉全自动驾驶硬件发布！马斯克明年推RoboTaxi：傻瓜才用激光雷达](https://mp.weixin.qq.com/s/NTGa1UAK4GcYDRw1R7J-Xg)
* [26] [Ray Casting 2D - Coding Challenge #145 · The Coding Train](https://thecodingtrain.com/CodingChallenges/145-2d-ray-casting.html)
* [27] [Rendering Ray Casting - Coding Challenge #146 · The Coding Train](https://thecodingtrain.com/CodingChallenges/146-rendering-ray-casting.html)
* [28] [Metaballs - Coding Challenge #28 · The Coding Train](https://thecodingtrain.com/CodingChallenges/028-metaballs.html)
* [29] [Benchmark: Destructure vs Assignments - MeasureThat.net](https://www.measurethat.net/Benchmarks/Show/14364/0/destructure-vs-assignments)