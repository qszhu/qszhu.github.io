---
layout: post
title: ChatGPT教我开发VisionOS App
date: 2024-04-06 12:00:00 +0800
tags: [chatgpt, visionos]
---

## 背景

上回[1]提到, 在体验过Vision Pro之后, 我对其略显“荒芜”的App Store感到失望. 而作为一个曾经的移动应用开发人员, 自然会有“自己动手, 丰衣足食”的冲动. 不过理想是丰满的, 现实是骨感的. 我上次开发iOS应用可能还要追溯到将近10年前, 开发语言还是Objective-C, 搭界面用的还是XIB. 而现在似乎都是Swift+SwiftUI了. 更别提十年间框架的变动和新增, 让人难以在短期内全部消化.

然而时代变了! 现在我们有强大的号称能取代程序员的生成式AI, 能帮助我们更高效地完成日常工作的智能助理. 我也读到过有不懂编程的小白在GPT-4的帮助下做出iPhone应用的例子[2]. 10年的经验空缺加上新的目标平台, 应该让我的经验无限接近小白了. 别人能行的话我也能行...吧?

## 目标

假设我有一个基于OpenGL[3]的跨平台的UI库, 要怎么在VisionOS上使用呢? 不需要问生成式AI就可以知道[4], 苹果早就用自家的Metal取代了OpenGL, 即便是非苹果的平台(除了Web)也大多转向了Vulkan. 技术的更迭原本就是常态, 即使没有生成式AI也是一样.

于是目标就变成了, 假设我有一个C++的Metal渲染库, 要怎么在VisionOS上用起来?

## 第一次尝试

把问题直接抛给GhatGPT的话, 它就开始吐代码了:

![1](/assets/images/2024-04-06/1.png)

不过似乎它对问题的理解有偏差. 要求的是C++代码, 实际给出的却是Objective-C++. 我仿佛听到对方在说“你就说是不是C++吧🐶”. 行吧, 就当我问得不对.

## 第二次尝试

再问一次, 严格要求输出C++:

![2](/assets/images/2024-04-06/2.png)

虽然看上去有模有样, 但都是正确的废话. 不过似乎步入正轨了. 于是要求给出完整的工程代码:

![3](/assets/images/2024-04-06/3.png)

又回到objc了😓. 金鱼的记忆吗?

## 第三次尝试

那我先不强求C++了, objc的工程代码能跑起来吗? 复制粘贴以后发现不行. 因为这行:

```objc
#import <MetalKit/MetalKit.h>
```

问ChatGPT的话, 给了个用UIImage的方案, 而且语言又变成了Swift:

![4](/assets/images/2024-04-06/4.png)

思路是对的吧. 不过虽然知道不是最优的, 却又给不出其他的方案:

![5](/assets/images/2024-04-06/5.png)

于是只能人肉去搜索, 发现VisionOS的App是不能用MetalKit的. 可能see through的画面本身都是通过Metal渲染的, 要跟摄像头的画面做blending啥的, 整个OS界面就是一个Metal应用吧. 解决的办法可以是拿CAMetalLayer去给Metal渲染[5].

## 第四次尝试

有了具体选型之后再去提问, 似乎就靠谱很多:

![6](/assets/images/2024-04-06/6.png)

才怪! 又用到了MetalKit. 似乎限制选择的提示词(不要做什么/不要用什么)必须每次都提到, 不然就会在上下文中被忽略. 此外如果需要生成的内容太多, 可能质量也会降低. 于是要它一步步来:

![7](/assets/images/2024-04-06/7.png)

看上去似乎有戏!

## 第五次尝试

就这样step by step了4,5次以后得到了完整的工程代码, 复制粘贴以后不出意外地出了意外, 跑不起来🐶

于是开始帮AI找bug. 好在问题不大, 只是生成的代码里把UIView的大小指定为零, 导致CAMetalLayer大小也为零, 于是就没有Drawable可以用来渲染:

```swift
func makeUIView(context: Context) -> UIView {
    // Create a UIView to host the CAMetalLayer
    let metalView = UIView(frame: .zero)
    ...
```

不过生成的代码里的guard是一起判断的, 报了个错还不知道具体哪一行有问题, 得手动拆开来看才能定位:

```swift
func render() {
    guard let drawable = metalLayer?.nextDrawable(),
          let pipelineState = pipelineState,
          let commandQueue = commandQueue else {
        print("Unable to get drawable or pipeline state is not set up.")
        return
    }
```

修正以后就能跑通!

## 第六次尝试

即然Swift的工程能跑通了, C++的行不行呢? 继续尝试. 稳妥起见, 先让它把渲染相关的代码抽出来放到一个单独的Swift渲染类里:

![8](/assets/images/2024-04-06/8.png)

再让它把Swift代码转换成C++的:

![9](/assets/images/2024-04-06/9.png)

这一步虽然它知道还需要一个objc wrapper来桥接Swift和C++, 但却无法给出具体的C++代码. 可能是因为metal-cpp没有在线文档的关系. 后续的提问[6]似乎验证了这个猜测, ChatGPT的回答都不得要领, 而问题的答案其实都在metal-cpp压缩包的README文件中.

## 开始的结束

最后跑起来的结果是这样的:

![10](/assets/images/2024-04-06/10.png)

最终实现用的Nim, 整条链路就成了Nim <-> C++ <-> Objc <-> Swift. 详见完整代码[7].

## 小结

整个过程虽然不是很丝滑, 但至少跑通了. 从中似乎可以得出以下结论:

* 对于缺少数据的信息, ChatGPT的回答质量不高. 比如VisionOS还比较新, 相关信息较少; 又比如metal-cpp的信息不是网页的形式, 可能就没有被索引到.
* 限制选择的提示词似乎需要不断被强调. 比如不要用objc, 不要用MetalKit等. 不强调的话似乎对话上下文多少会被无视掉, 回到更容易的选择上去(AI也会偷懒?).
* 生成内容比较多的话最好是一步步来, 否则一次性生成的结果中可能会忽略很多内容.

整个过程其实是个根据已有信息进行推理的过程. 比如这次的目标是通过C++调用Metal接口在VisionOS的App窗口中渲染. 直接跑是跑不起来的, 那么中间还需要哪些步骤呢?

* VisionOS中用不了MetalKit
  * -> 渲染到CAMetalLayer提供的Drawable上
    * -> 自定义符合UIViewRepresentable的UIView来绑定CAMetalLayer
* Swift无法直接调用C++
  * -> Swift调用ObjC的Wrapper
    * -> ObjC的Wrapper调用C++

以目前ChatGPT的能力, 一方面推理能力不足, 难以点到点地完成整条推理链路; 另一方面相关数据不足, 又限制了推理过程中每一步的可选项. 因此要让小白在生成式AI的辅助下完成一些前沿的工作可能还比较困难.

但要说生成式AI毫无是处也是不公平的. 对于有足够数据的内容, ChatGPT都能较好地回答. 比如问它VisionOS它说不清楚, 问它Core Animation就开始滔滔不绝. 此时从聊天框里获取信息就比从搜索引擎获取信息效率更高. 因此在AI能够完全自主推理之前, 由AI搜集和整合信息, 由人类补充信息和进行推理可能就是最优解. 生成式AI现在就像一个记忆力超强的小孩子, 就某个问题看过足够多的案例之后自己归纳总结出了一些规律, 但可能并未理解问题的本质, 因此在被要求复述过程的时候可能会出现一些偏差. 你可以批评它不够完美, 但并不意味着它不能被加以利用.

最后回到老问题上, 就是生成式AI到底能否取代程序员? 我觉得有一类程序员可能比较危险, 就是那些熟读了各类文档和规范, 可能被称为“活字典”或是“语言律师”[8]的人. 比如这次在没有读完所有文档的情况下, 我这个10年小白在生成式AI的辅助下跑通了一个App. 又比如此前报道GPT-4在律师考试中超过了人类水平[9], 虽然此“律师”非彼“律师”, 但名字的相似性暗示了工作内容的相似性🐶

## 参考

* [1] [Apple Vision Pro初体验](/2024/02/17/apple-vision-pro-first-impressions.html)
* [2] [A Coder Considers the Waning Days of the Craft](https://www.newyorker.com/magazine/2023/11/20/a-coder-considers-the-waning-days-of-the-craft)
* [3] [谈谈Nim与C/C++的互操作——以OpenGL为例](/2023/12/17/nim-for-opengl.html)
* [4] [OpenGL ES Programming Guide](https://developer.apple.com/library/archive/documentation/3DDrawing/Conceptual/OpenGLES_ProgrammingGuide/Introduction/Introduction.html)
* [5] [`MTKView` on visionOS](https://developer.apple.com/forums/thread/732355)
* [6] [完整聊天记录](https://chat.openai.com/share/83702172-62a8-4a61-82a8-3536daf2fb6f)
* [7] [完整代码](https://github.com/qszhu/visionos-nim)
* [8] [language lawyer](http://www.catb.org/jargon/html/L/language-lawyer.html)
* [9] [Latest version of ChatGPT aces bar exam with score nearing 90th percentile](https://www.abajournal.com/web/article/latest-version-of-chatgpt-aces-the-bar-exam-with-score-in-90th-percentile)