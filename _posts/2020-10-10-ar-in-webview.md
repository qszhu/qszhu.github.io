---
layout: post
title: WebView中的增强现实应用
date: 2020-10-12 22:00:00 +0800
tags: [ar, android, webview, webgl]
---

### 1. 背景

一个AR（增强现实）应用通常由以下几部分组成：

* 一个输入视频流，用于环境感知和渲染；
* 一个识别模块，用于识别和跟踪画面中感兴趣的对象；
* 一个渲染模块，用来同时展示现实和虚拟的内容。

对于某些应用，纯视觉的信息可能还不够，因此识别模块还需要获取运行设备上其他传感器的信息（如加速度传感器，陀螺仪等），不过这不在本文的讨论范围内。

根据是否需要对画面中的内容进行跟踪，我把AR应用划分为两类：

一类是HUD应用。此类应用不需要对画面中的内容进行跟踪，在发现画面中存在感兴趣的内容后，就直接在画面上叠加显示部分虚拟信息。由于不存在跟踪，虚拟信息的位置相对于屏幕的位置固定，这也是我将其称作HUD的原因。

另一类是传统意义上的AR应用。此类应用能对画面中感兴趣的内容进行跟踪，并在其上叠加虚拟的信息。对于3D的虚拟物体，还能体现观察视角的变化。

在渲染的实现上，也有两种方式：

一种是将视频流和虚拟内容交由不同的层渲染：

![渲染方式1](/assets/images/2020-10-10/render1.jpg)

这样视频画面和虚拟内容就是是分开渲染的。优点是渲染虚拟内容的卡顿不会影响到视频画面。缺点是当需要跟踪且发生卡顿时，虚拟内容所对应的位姿可能落后于视频画面，造成跟踪效果的滞后。所以这种实现方式只适用于上述的HUD类型的应用。

另一种是在虚拟内容层渲染视频画面：

![渲染方式2](/assets/images/2020-10-10/render2.jpg)

这样即使发生了卡顿，至少可以保证虚拟内容的跟踪效果和视频画面是同步的。由于虚拟内容的渲染通常是使用一些3D引擎来完成的，同时渲染虚拟内容和视频画面需要一些trick，有机会可以再说说。

某司的一个项目需要将原本运行在浏览器中的HUD应用移植到原生应用中，并加入跟踪功能，成为传统的AR应用。渲染时采用了上述第一种方式，例如在Android系统中，就使用了SurfaceView[1]来展示摄像头预览帧，并在其上覆盖WebView运行原有应用。在画面剧烈晃动时，就会看到虚拟内容落后于视频画面的现象。

### 2. 分析

解决方案是将上述的渲染方式1改为渲染方式2。在Android系统中，就意味着要将摄像头的预览帧传入WebView中，再通过某种方式渲染出来。由于渲染虚拟内容通常是通过3D引擎来完成的，就意味着要通过WebGL去渲染预览帧。而渲染的方式就是把预览帧作为纹理贴到一个撑满屏幕的平面上。

WebGL支持由一个`<img>`元素创建纹理[2]，所以一种可能的方案是，向WebView中传入预览帧的图像数据，经由一个`<img>`元素加载后创建成纹理。

由于Android系统中摄像头预览帧的默认格式为NV21[3]，无法直接被`<img>`元素加载，所以一个可行的方法是利用`YuvImage`[4]转换成jpeg后再加载。

此外，由于Android系统中WebView是通过JsBridge[5]与Java代码交换数据的，仅支持基本类型和字符串，因此需要将转换后的帧数据进行Base64编解码。

综上，整个流程是这样的：

![方案1](/assets/images/2020-10-10/method1.jpg)

### 3. 实现

实现的结果并不理想，因为整个管线太长，每个步骤消耗的时间累加起来就很可观了，导致最后帧率不够。

另一种可能的方案是，向WebView中直接传入yuv格式的帧数据，创建纹理后使用自定义的fragment shader进行渲染。这样就可以跳过yuv转jpeg，以及使用`<img>`元素加载图像的步骤：

![方案2](/assets/images/2020-10-10/method2.jpg)

实现下来依旧存在问题：

* shader中fragment的位置是用浮点数表示的，需找到帧数据中对应的yuv分量并转换成rgb。浮点数精度的问题可能导致转换出来的颜色略有偏差；
* 虽然比前一个方案省去了管线中的若干步骤，但帧率依旧没有提高。原因是yuv的压缩比不及jpeg。似乎透过JsBridge的数据传输是瓶颈。

可能的优化方案就是要减少透过JsBridge传输的数据量：

* 降低摄像头预览帧的分辨率。实测可以大幅提高帧率。可能带来的问题除了感观上的模糊影响体验以外，由于预览帧还被作为跟踪模块的输入，降低预览帧的分辨率可能会影响到跟踪的效果；
* 降低jpeg压缩的质量。这样跟踪模块和渲染模块可以使用不同质量的预览帧，相互不影响

### 4. 小结

另一种可能的绕过方式是彻底抛弃WebView，在原生层使用OpenGL实现对WebGL的调用。这本身就是一项大工程，更别提Web应用中包含的其他Web API调用了。

这个问题的根源在于不同的软件层级之间传输数据需要进行耗时的复制/序列化，这对需要实时处理视频流的系统来说是致命的。而如果是系统开发商的话就可以开一些能快速传输/映射数据的后门。这也从一个侧面说明了拥有一个自主可控的技术栈的重要性。

### 参考资料
* [1] [Camera API  \|  Android 开发者  \|  Android Developers](https://developer.android.com/guide/topics/media/camera#custom-camera)
* [2] [WebGLRenderingContext.texImage2D() - Web APIs \| MDN](https://developer.mozilla.org/en-US/docs/Web/API/WebGLRenderingContext/texImage2D)
* [3] [ImageFormat  \|  Android 开发者  \|  Android Developers](https://developer.android.com/reference/android/graphics/ImageFormat#NV21)
* [4] [YuvImage  \|  Android 开发者  \|  Android Developers](https://developer.android.com/reference/android/graphics/YuvImage)
* [5] [WebView  \|  Android 开发者  \|  Android Developers](https://developer.android.com/reference/android/webkit/WebView#addJavascriptInterface(java.lang.Object,%20java.lang.String))