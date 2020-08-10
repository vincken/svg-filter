### 滤镜的工作原理
&emsp;&emsp;虽然SVG不是一种位图描述语言，佴它仍然允许我们使用一些相同的工具，比如滤镜。当SVG阅读器程序处理一个图形对象时，它会将对象呈现在位图输出设备上；在某一时刻，阅读器程 
序会把对象的描述信息转换为一组对应的像素，然后呈现在输出设备上。

&emsp;&emsp;例如我们用SVG的<filter>元素指定一组操作（也称作基元，primitive)，在对象的旁边显示一个模糊的投影，然后把这个滤镜附加给一个对象。由于图片在显示样式中用了滤镜，所以SVG不会将图片直接渲染为最终图形。相反，SVG会渲染图片的像素到临时位图中。由滤镜指定的操作会被应用到该临时区域，其结果会被渲染为最终图形。

```
<filter id="shadow" width="200%" height="200%">
    <feGaussianBlur in="SourceAlpha" stdDeviation="20" result="blur"></feGaussianBlur>
    <feOffset in="blur" dx="40" dy="40" result="offsetBlur"></feOffset>
    <feMerge>
        <feMergeNode in="offsetBlur"></feMergeNode>
        <feMergeNode in="SourceGraphic"></feMergeNode>
    </feMerge>
</filter>
```

原图：

![image](http://vincken.top/svg/filter/image/shadow_origin.png)

应用滤镜后：

![image](http://vincken.top/svg/filter/image/shadow_filter.png)

[demo](http://vincken.top/svg/filter/demo/demo1.svg)

起始和结束 <filter>标记之间就是执行我们想要的操作的滤镜基元，通常以fe开头，代表filter。

每个基元有一个或多个输入，但只有一个输出。一个输人可以是原始图形（被指定为SourceGraphic)、图形的阿尔法（不透明度）通道（被指定为SourceAlpha)，或者是前一个滤镜基元的输出。当只对图形的形状感兴趣而不管其颜色时，阿尔法源是有用的，它会避免阿尔法和颜色相互作用。

在以上代码中，

1. result属性指定当前基元的结果稍后可以通过blur名引用。这不同于XML id,给定的名称是一个局部名称，它只在包含该基元的<filter>中有效。
1. <feOffset>基元接受它的输人，在这里就是Gaussian blur的返回结果blur，它的偏移由dx和dy的值指定，然后将结果位图存储在offsetBlur名字下面。
1. <feMerge>基元包裹一个<feMergeNode>元素列表，其中每个元素都指定一个输人。这些输入按照它们出现的顺序一个堆叠在另一个上面。在这里我们希望offsetBlur在原始SourceGraphic 下面。

以上是一个简单的例子，简单介绍了feGaussianBlur,feOffset,feMerge这几种基元的使用。接下来我们来深入学习其它的滤镜基元。
### <feColorMatrix>
feColorMatrix元素允许我们以一种非常通用的方式改变颜色值。比如创建蓝绿色发光式投影：

```
<filter id="shadow" width="200%" height="200%">
    <feColorMatrix type="matrix"
    values="
        0 0 0 0 0
        0 0 0 0.9 0
        0 0 0 0.9 0
        0 0 0 1 0
    ">
    </feColorMatrix>
    <feGaussianBlur stdDeviation="2" result="blur"></feGaussianBlur>
    <feOffset in="blur" dx="40" dy="40" result="offsetBlur"></feOffset>
    <feMerge>
        <feMergeNode in="offsetBlur"></feMergeNode>
        <feMergeNode in="SourceGraphic"></feMergeNode>
    </feMerge>
</filter>
```

<feColorMatrlx>是一个通用的基元，允许我们修改任意像紊点的颜色或阿尔法值。当type属性等于matrix的时候，找们必须设置value为20个数字来描述变换信息。这20个数字按照4行5列编写时最好理解。毎行代表一个代数方程，定义了如何计算输出的R、G、B、A值（按行的顺序）。每行中的数字分别乘以输入像素的R、G、B、A的值和常量1<按照列的顺序>，然后加在一起得到输出值。要设置一个变换，将所有不透明区域绘制 
为相同的颜色，可以忽略输人颜色和常量，只要设置阿尔法列的值即可。这个矩阵模型看起来如下所示：

```
0 0 0 red 0
0 0 0 green 0
0 0 0 blue 0
0 0 0 1 0
```
这里red、green和blue的值通常是0到1之间的十进制数。

注意，这个例子中并没有一个用作这个基元输人的in属性，默认使用SourceGraphic。这个基元也没有result属性。这意味着这个颜色矩阵操作的输出只用于下一个滤镜基元的隐性输入。如果使用这种快捷方式，那么下一个滤镜基元一定不能有in属性。

[demo](http://vincken.top/svg/filter/demo/demo2.svg)

该例子使用的是最通用的一种颜色矩阵，type属性还有另外三个值，分别是hueRotate、saturate和luminaceToAlpha。每个内置的矩阵都完成一个特定的视觉任务，并且都有自己的指定values的方式，此处不详细介绍。

### <feComponentTransfer>
操作颜色除了用feColorMatrix以外，feComponentTransfer也提供了一种更方便，更灵活的方式来单独操作每个颜色分量。它还允许我们对毎个颜色分量作出不同的调整。

可以通过在<feComponentTransfer> 内配置<feFuncR>、<feFuncG>、<feFuncB> 和 <feFuncA>
元素，调整红、绿、蓝色和阿尔法的级别。每个子元索都可以单独指定一个type属件，说明应如何修改该通道。

最常用的type属性为linear，它会把当前颜色分量值C放到公式slope * C + intercept中，intercept为结果提供了一个基准值，slope是一个简单的比例因子。

[demo](http://vincken.top/svg/filter/demo/demo4.svg)

惯例，linear也只是最简单的一种线性类型，还有gamma、table、discrete这三种类型，用于实现更为复杂的颜色算法。
### <feImage>
到目前为止，我们只见过将原始图形或者阿尔法通道作为滤镜的输入源。SVG的<feimage>元素允许我们使用任意的JPG、 PNG、SVG文件，或者带有id属性的SVG元素或者SVG片段作为滤镜的输入源。

下面为利用SVG矩形的渐变色实现光晕的例子:

[demo](http://vincken.top/svg/filter/demo/demo5.svg)

### 总结
<filter>元素包含一系列滤镜基元，每个都接受一个或多个输入，同时提供了唯一的结果给其他滤镜使用，一系列滤镜中最后一个滤镜的结果会呈现到最终的图形上。我们用x、y、width、height属性指定应用滤镜的画布的尺寸。用filterUnits指定用来定义滤镜范围的单位，用primitiveUnits为滤镜基元中的各种长度值指定坐标系统。

![image](http://vincken.top/svg/filter/image/sum.png)
### 附录
各种常见的滤镜效果应用：

亮度：
[demo](http://vincken.top/svg/filter/demo/demo1.html)

对比度：
[demo](http://vincken.top/svg/filter/demo/demo2.html)

饱和度：
[demo](http://vincken.top/svg/filter/demo/demo3.html)

色彩：
[demo](http://vincken.top/svg/filter/demo/demo4.html)

模糊:
[demo](http://vincken.top/svg/filter/demo/demo9.html)

锐化：
[demo](http://vincken.top/svg/filter/demo/demo5.html)

x射线:
[demo](http://vincken.top/svg/filter/demo/demo6.html)

晕影:
[demo](http://vincken.top/svg/filter/demo/demo7.html)
