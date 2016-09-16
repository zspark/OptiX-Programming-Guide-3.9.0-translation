NVIDIA OptiX 射线追踪引擎

编程指南

Version 3.9

2015年2月12日

翻译目的：
进一步加强自己对OptiX的理解与记忆，翻译必然会存在理解上或者表达上的错误，一经发现，立即改正提交。

资源发布在Github：https://github.com/zspark/OptiX-Programming-Guide-3.9.0-translation.git


# 第一章 介绍
## 1.1 OptiX总揽
<<>>GPU对高纬度的并行处理非常在行，射线追踪非常符合它这一口味。然而，典型的射线追踪算法非常的参差不齐，它给与了任何人一探GPU计算本质的严重挑战。NVIDIA的OptiX射线追踪引擎与API集接受了这一挑战，并且提供了一个驾驭现在、乃至下一代来图形硬件架构的框架去整合射线追踪与交互式应用。借助OptiX与NVIDIA的CUDA架构，没有计算机图形学博士学位的开发者或者团队里的射线追踪工程师与交互式射线追踪技术的开发最终得以可行。

<<>>OptiX不是个渲染器。相反，它是个可伸缩的框架用于开发基于射线追踪的应用。OptiX引擎具有2个重要的内容：一个位于C端的API，定义了射线追踪必要的数据结构；一个基于CUDA
C++
的编程系统，可用来生成射线，射线与表面相交，反馈这些相交。这2部分提供了底层的“原始射线追踪”。这就允许应用程序使用射线追踪来开发图形图形，碰撞检测，声音传播，可见性检测等。

### 1.1.1 动机
<<>>真是因为抽线了常规射线追踪器的执行模型，OptiX才能够方便的集成射线追踪系统，分离用户编写的对象遍历，着色器派发、内存管理等算法。而且，返回系统可以承受未来GPU硬件结构的变化与SDK变化所带来的影响————与OpenGL、D3D一样为光栅化管线提供了一层抽象。
只要可行，OptiX引擎劲量避免具体射线追踪的行为，相反它提供一种执行用户提供的CUDA
C代码的机制去实现着色（包括
递归射线）、摄像机模型、甚至是颜色的呈现。总结来说的话，OptiX引擎可以用来执行Whitted风格的射线追踪、路径追踪、碰撞检测、原子映射，或者是其他基于射线追踪的算法。他被设计用来操作8个独立的或者与OpenGL、DirectX应用交互的射线追踪光栅化应用。

### 1.1.2 编程模型
<<>>OptiX的核心是个简单但是强大的对光线追踪器的抽象。这个光线追踪器利用用户提供的程序初始化光线，让光线与表面相交，给材质着色，引发新的光线（译注：比如透明物体需要在相交点再次发射光线以获取背后物体的信息）。光线携带的用户自定义数据描述了每一条射线各自的变量，比如颜色，迭代深度，重要程度，或者其它的一些属性。开发者利用基于CUDA
C提供的函数为这些属性提供具体的功能。因为光线追踪是个固有的递归算法，OptiX允许用户递归新的射线，内部执行机制管理着所有的递归堆栈的细节。OptiX提供了动态函数派发与高端的变量继承机制，让光线追踪系统能够被普遍与紧凑的编写。

## 1.2 射线追踪基础
<<>>“射线追踪”这个术语的含义需要依赖上下文。有时它指一条3D射线与一组3D对象（比如一个球体）之间交点的计算。有时它指一个专门的算法，比如Whitted的图像生成方法，或者是一个石油开采业中用来模拟地波扩展的算法。有时又用来表示Whitted算法与射线发射等算法的集合。OptiX是一个射线追踪引擎：它允许用户用来对3D物体相交，（待译），另外，OptiX提供给用户自己编写诸如射线生成，这些射线碰撞到物体的具体行为的功能。
<<>>图形学中，射线追踪最早由Arthur Appel与1968年为渲染固态对象而提出，Turner
Whitted提出了利用递归计算反射、折射的可行性进一步发展了这么技术。后续的发展加入了景深效果，互相反射下的漫反射，软阴影，模糊动画等等加强了渲染结果的准确性。同时，许多研究者利用新的算法为场景中物体建立索引增加了射线追踪的性能。基于射线追踪的真实渲染算法被用来准确的模拟光线的传输。一些这样的算法模拟了虚拟环境下光子的衍生。（待译：物理方面的，比较难）。一些算法还利用了双向方法。OptiX工作在这些算法之下，所以可以用来实现这些算法中的任何一个。
<<>>射线追踪经常被用在“非图像”的应用中。在计算机辅助设计中，射线追踪被用来预算复杂部件的体积（容积）：向这个部件发射一组平行的射线，部分碰撞到该部件的射线返回横截面的面积，光线的平均长度告诉我们平均的深度。射线追踪也可以用来判断复杂移动物体的街进行（包括碰撞）。通常采用从对象表面发射“试探”光线去判断什么东西在附近。射线还被鼠标用来选择物体，从而判断哪些对象在一个像素中是可见的，也用来在游戏中做投掷物的碰撞。OptiX可以在上面任何应用中胜任。
<<>>射线追踪算法的常规特性就是用来计算3D射线与一组表面（被称为“模型”或者“场景”）的交点。在着色应用中，射线与模型在视觉属性上的交点决定了这条射线会会发生什么（比如，它可能会被反射、折射或者吸收）。其他应用可能除了交点位置以外不关心其他信息，甚至不关心是否相交。这种需求的多样性意味着OptiX的支持多种射线场景查询，射线与场景相交的用户定义的行为是值得做的。
<<>>射线追踪一个点赞的特性就是它可以方便的支持任意几何对象与一条3D直线的相交。比如，它可以直接支持球体而不需要曲面细分。另一个点赞的特性就是射线追踪的执行普遍成次线性（sub-linear）————加倍场景中物体的数量，并不会加倍执行的时间。这是用组织物体形成了加速结构体（译注：一个数据结构，用于加速碰撞检测，就是粗检测）来完成的，该结构体可以快速的排除全部对象图元与一些射线相交的可能性。场景中静止的对象，这种结构体可以一直使用（不需要重新计算），动态的对象的话，OptiX支持在必要的时候重新计算这种结构。这个结构体仅仅需要查询它所包含的几何体的包围盒数据，所以新的图元类型的加入不会影响加速结构体的正常工作（也不需要变动数据），只要新的图元能提供包围盒数据。
<<>>图形图像的应用程序中，射线追踪要比光栅化好。一个好处是可以方便的支持普通的照相机模型；用户可以用任意方向关联屏幕上的点，没有要求起点一致。另一个优点是重要的视觉效果比如反射、折射能够被短短几行代码所支持，硬投影可以不借助阴影映射而生成，软投影也不是很难。
<<>>进一步说，射线追踪可以作为一个材质生成的管线与传统图形图像程序互溶。比如，可以用深度缓存中的点作为射线起点去计算镜面反射。还有很多使用z缓冲区与射线追踪技术的“混合算法”。
<<>>进一步了解射线追踪在图形图像方面的信息，参考如下：
* The classic and still relevant book is “An Introduction to Ray Tracing” (Edited by A. Glassner, Academic Press, 1989).
* A very detailed beginner book is "Ray Tracing from the Ground Up" (K. Suffern, AK Peters, 2007).
* A concise description of ray tracing is in “Fundamentals of Computer Graphics” (P. Shirley and S. Marschner, AK Peters, 2009).
* A general discussion of realistic batch rendering algorithms is in "Advanced Global Illumination" (P. Dutré, P. Bekaert, K. Bala, AK Peters, 2006).
* A great deal of detailed information on ray tracing and algorithms that use ray tracing is in "Physically Based Rendering" (M. Pharr and G.  Humphreys, Morgan Kaufmann, 2004).
* A detailed description of photon mapping is in “Realistic Image Synthesis Using Photon Mapping” (H. Jensen, AK Peters, 2001).
* A discussion of using ray tracing interactively for picking and collision detection, as well as a detailed discussion of shading and ray-primitive
intersection is in "Real-Time Rendering" (T. Akenine-Möller, E. Haines, N. Hoffman, AK Peters, 2008).


# 第二章 编程模型总揽
<<>>OptiX的编程模型由两部分组成：C端代码与GPU设备端的程序。这章介绍对象，程序（译注：G端程序），和在C端定义G端使用的变量。

## 2.1 对象模型
<<>>OptiX是一个基于对象的C
API，实现了一个简单的对象模型层级结构。执行在G端的程序增强了这种面向对象的接口。系统的主要对象如下：

* 上下文（Context）：一个运行着的OptiX引擎实例；
* 程式（Program）：一个CUDA
* C的函数，被编译为NVIDIA的PTX虚汇编语言；（译注：Program我在这里专门翻译为‘程式’，以区分其与正常C端程序的不同）
* 变量（Variable）：一个名称用来传递C端数据到OptiX的程式；
* 缓冲区（Buffer）：一个多维度的数组，可以绑定到变量；
* 纹理采样器（TextureSampler）：一个或者多个绑定了差值机制的缓冲区；
* 材质（Material）：在射线与最近图元相交时或者潜在相交时执行的一组程式；
* 几何实例（GeometryInstance）：一个绑定了几何体与材质的对象；
* 组（Group）：一个具有层级结构的一组对象；
* 几何组（GeometryGroup）：一组由几何实例组成的对象；
* 变换（Transform）：一个层级结构下节点，可以几何变换射线，以至于变化几何对象；
* 选择器（Selector）：一个可编程的层级结构节点，用来选择哪些子节点（children）被遍历；
* 加速器（Acceleration）：一个加速结构体对象，可以用来绑定到层级结构节点；
* 远端设备（RemoteDevice）：一个为了远程视觉渲染的网络连接；

<<>>这些对象的创建、销毁、修改与绑定都是通过C端API实现，进一步的细节参考第三章。OptiX的行为可以通过对这些对象的任意配置而控制。

## 2.2 程式
<<>>Optix提供的射线追踪管线包含一些可编程的组件（程式）。这些程式会由GPU在执行射线追踪算法的特定时刻而调用执行。一共有8中不同的程式：

* 射线生成（Ray Generation）：这是进入射线追踪管线的接口点，由系统并行的为每一个像素、采样或者其他用户定义的工作而调用执行；
* 异常（Exception）：异常句柄，在诸如堆栈溢出和其他一些错误的时候调用；
* 最近碰撞（Closest Hit）：跟踪的射线找到最近的相交点后调用，用于材质着色；
* 任意碰撞（Any Hit）：跟踪的射线找到一个全新的潜在最近相交点后调用，用于阴影计算；
* 相交（Intersection）：在遍历的时候调用，实现射线与图元的相交测试；
* 包围盒（Bounding Box）：计算图元在世界空间下的包围盒，在系统创建了一个加速结构体后调用；
* 丢失（Miss）：当射线与场景中所有几何体都不相交时调用；
* 访问（Visit）：当遍历选择节点的时候调用，用来决定哪些子节点（Children）会被便利到；（译注：选择节点没有加速结构体，也不是个实体对象，比如几何体。它的调用是在OptiX判断射线与其子节点相交的时候）；

<<>>这些程式的输入语言叫PTX。OptiX SDK同样为NVIDIA C编译器（nvcc）提供一组包装类与头文件，它提供了一种用CUDA C编写PTX的方式。这些程式的进一步细节见第四章。

## 2.3 变量
<<>>OptiX提供一种灵活的且强大的变量系统，用来将数据与程式连接起来。当OptiX程式引用一个变量的时候，一个良好定义的作用域集便会帮助它查询这个变量的定义。这允许为查找来的变量动态的为重写定义。
<<>>举例来说，一个最近碰撞程式引用了一个变量叫做color.这个程式可能被附加（attach）在好几个材质对象上，而这些材质对象依次又附加在几何实例对象上，最近碰撞程式上的变量首先查找该程式对象上的定义，然后依次查找几何实例，材质，最后查找上下文对象。这就可以在材质对象上定义color这个变量的默认值，但是在具体的几何实例对象上定义变量color的其它值从而重写材质对象上的color默认值。（译注：因为在最近碰撞程式上几何实例对象作用域查找要优先于材质对象，不同程式查找顺序不同。）参见3.4节以获取更多信息。

## 2.4 执行模型
<<>>当所有的这些对象，程式，变量被集成在可用的上下文（Context）后，射线生成程式可能需要被执行了（launched）。执行时候需要纬度信息和尺寸信息，并且会调用执行射线生成程式好多遍，具体次数视尺寸而定。
<<>>当射线生成程式被调用后，一个特殊的语义变量（semantic variable）可能会被查询用来定位运行时的射线生成程式。举例来说，一个常规的用法是执行（launch）一个二维程式，其高宽符合将要被渲染的图像的像素尺寸。参见4.3.2节获取更多信息。

# 第三章 客户端API
## 3.1 上下文
<<>>一个OptiX的上下文控制着射线追踪引擎的装配与后序的工作。上下文通过rtContextCreate函数创建。上下文对象封装了所有OptiX的资源————纹理，几何体，用户定义的程式等等，上下文的销毁通过调用rtContextDestroy函数，它会销毁所有资源并且将这些资源的句柄失效。
<<>>rtContextLaunch{1,2,3}D是射线引擎计算的入口函数。发射函数（rtContextLaunchXD）需要一个入口点作为参数（入口点3.1.1节讨论），1个，2个或者3个格子纬度参数。纬度建立了一个逻辑上的计算网格。rtContextLaunch执行的时候，一切必要的预处理过程需要先执行，然后执行射线生成程式，这个程式是绑定到对应的接口点的，并且会被计算网格中的每一个格子所调用。预处理包括状态的验证，必要的话还会执行加速结构体的创建（译注：可能是第一次全新创建，也可能是由于动态设置为dirty后的再次创建）与内核编译。发射函数的输出由OptiX缓冲区传递回来，缓冲区的纬度一般（非必须）是计算网格的纬度。

{% highlight c++%}
RTcontext context;
rtContextCreate( &context );
unsigned int entry_point = ...;
unsigned int width = ...;
unsigned int height = ...;
// Set up context state and scene description
...
rtContextLaunch2D( entry_point, width, height );
rtContextDestroy( context );
{% endhighlight %}

<<>>可以在有限的情况下激活好多上下文，但这没有必要，一个上下文对象可以支配多硬件设备。要使用的设备数量可以通过rtContextSetDevices函数设置。默认是最大的可支持OptiX的计算设备数量（译注：不确定这里的翻译）。下面的一些规则可以确定设备的兼容性。这些规则将来会改变。如果一个不兼容的设备被选择会由rtContextSetDevices抛出一个错误.

* All SM 3.5 devices can be run in multi-GPU configurations with other SM 3.5 devices.
* All SM 3.0 devices can be run in multi-GPU configurations with other SM 3.0 devices.
* All SM 2.X devices can be run in multi-GPU configurations with other SM 2.X devices.
* All SM 1.2 and 1.3 devices can be run in multi-GPU configuration with other SM 1.2 and 1.3 devices.
* All SM 1.1 and 1.0 devices can only be run in single-GPU configurations.

### 3.1.1 接入点
<<>>每一个上下文可能含有多个计算接入点。一个接入点被关联在一个的射线生成程式与一个（可选的）异常程式上。一个上下文可以通过rtContextSetEntryPointCount设置接入点的数量。接入点关联程式的设定与获取通过函数：
rtContext{Set|Get}RayGenerationProgram
rtContext{Set|Get}ExceptionProgram
每一个接入点在使用的时候必须关联一个射线生成程式，异常程式是可选的。多接入点的机制允许在不同的渲染算法之间切换，也方便在一个OptiX上下文上进行多通道渲染技术的实现。

{% highlight c++ %}
RTcontext context = ...;
rtContextSetEntryPointCount( context, 2 );
RTprogram pinhole_camera = ...;
RTprogram thin_lens_camera = ...;
RTprogram exception = ...;
rtContextSetRayGenerationProgram( context, 0,
pinhole_camera );
rtContextSetRayGenerationProgram( context, 1,
thin_lens_camera
);
rtContextSetExceptionProgram( context, 0, exception
);
rtContextSetExceptionProgram( context, 1, exception
);
{% endhighlight %}


### 3.1.2 射线类型（译注：注意区别8中不同的程式，完全两个目的，两个意思，不要混淆）
<<>>OptiX支持射线类型这么一个标记，主要用于在跟踪过程中区分不同的射线以达到不同的目的。比如，一个渲染器可能需要区别用于计算颜色的与用于判断是否与光源可见的不同射线（阴影射线）。合理的归类像这样不同定义的射线不仅有助于增强程式的模块性，也有利于OptiX更高效的运算。
<<>>射线类型与射线的行为完全由应用程序定义。射线类型的数量可以通过rtContextSetRayTypeCount函数设置。下面这些属性在不同的射线类型中可能是有却别的：

* 射线的夹带数据（译注：我喜欢将payload翻译成夹带，好比考试作弊夹带纸条一样，虽然与考试本身没有关系，但很重要，每个考生都可能有自己不同风格、需求的夹带:)）；
* 每一种材质的最近碰撞程式；
* 每一种材质的任意碰撞程式；
* 丢失程式；

<<>>夹带（数据）是一种用户定义的数据结构用于关联到各个射线。比如，通常的用法是保存一个返回的颜色，射线的递归深度，阴影的衰减系数等。它可以看成是射线追踪结束后的返回数据。它也可以用于在递归期间在不同的射线生成程式中保存、发布数据。
<<>>指派给材质的最近碰撞与任意碰撞程式好比传统渲染系统中的着色器：它们会在射线与几何图元发生相交时调用。因为这些程式指派给材质是区分射线类型的，所以并不是所有的射线类型都必须有这两种程式。参见4.5，4.6节以获取更多细节。
<<>>丢失程式是在追踪的光线确定没有与任何集合体相撞的时候调用的。丢失程式可以用来返回一个常量以表示天空的颜色或者返回从环境贴图中采样的颜色。
<<>>表格1 可以作为一个Whitted风格的递归射线追踪器的示例，目的是展示如何使用射线类型。

表格1
|Ray Type|Payload|Closest Hit|Any Hit|Miss|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
|Radiance|RadiancePL|计算颜色、继续追踪递归深度|n/a|环境贴图查询|
|Shadow|ShadowPL|n/a|计算阴影衰减，如果对象不透明则终止追踪|n/a|

上表中的夹带数据结构可能会是这样的：

{% highlight c++%}
// Payload for ray type 0: radiance rays
struct RadiancePL
{
float3 color;
int recursion_depth;
};
// Payload for ray type 1: shadow rays
struct ShadowPL
{
float attenuation;
};
{% endhighlight %}

对rtContextLaunch的调用，射线生成程式发射radiance射线到场景中，将传递的结果（结果来自夹带数据中的颜色字段）写入输出缓冲区以便显示。

{% highlight c++%}
RadiancePL payload;
payload.color = make_float3( 0.f, 0.f, 0.f );
payload.recursion_depth = 0; // initialize recursion depth
Ray ray = ... // some camera code creates the ray
ray.ray_type = 0; // make this a radiance ray
rtTrace( top_object, ray, payload );
// Write result to output buffer
writeOutput( payload.color );
{% endhighlight %}

图元与radiance射线相交后，会执行最近碰撞程式，用来计算射线的颜色，有时候需要发射阴影射线和反射射线。阴影射线的部分内容见下：

{% highlight c++%}
ShadowPL shadow_payload;
shadow_payload.attenuation = 1.0f; // initialize to visible
Ray shadow_ray = ... // create a ray to light source
shadow_ray.ray_type = 1; // make this a shadow ray
rtTrace( top_object, shadow_ray, shadow_payload );
// Attenuate incoming light (‘light’ is some user-defined
// variable describing the light source)
float3 rad = light.radiance * shadow_payload.attenuation;
// Add the contribution to the current radiance ray’s
// payload (assumed to be declared as ‘payload’)
payload.color += rad;
{% endhighlight %}

<<>>为了合理的衰减阴影射线，所有的材质使用任意碰撞程式来调整衰减度与终止射线的遍历。下面的代码将不透明的材质的衰减度设为0。

{% highlight c++%}
shadow_payload.attenuation = 0; // assume opaque material
rtTerminateRay(); // it won’t get any darker, so terminate
{% endhighlight %}

### 3.1.3 全局状态
<<>>除了射线类型与接入点之外，还有一些其他的全局设置封装在了OptiX的上下文中。每一个上下文持有一组属性，他们可以通过rtContext{Get|Set}Attribute函数设置。比如，OptiX上下文分配的内存数量可以通过传入RT_CONTEXT_ATTRIBUTE_SUED_HOST_MEMORY而查询到。

{% highlight c++%}
RTcontext context = ...;
RTsize used_host_memory;
rtContextGetAttribute( context, RT_CONTEXT_ATTRIBUTE_USED_HOST_MEMORY, sizeof(RTsize), &used_host_memory );
{% endhighlight %}

当前rtContextGetAttribute支持如下属性：
* RT_CONTEXT_ATTRIBUTE_MAX_TEXTURE_COUNT
* RT_CONTEXT_ATTRIBUTE_GPU_PAGING_FORCE_OFF
* RT_CONTEXT_ATTRIBUTE_GPU_PAGING_ACTIVE
* RT_CONTEXT_AVAILABLE_DEVICE_MEMORY
* RT_CONTEXT_USED_HOST_MEMORY
* RT_CONTEXT_CPU_NUM_THREADS

可以通过RT_CONTEXT_CPU_NUM_THREADS设置CPU线程数量，用于多种任务比如构建加速结构体（以后简称加速体），通过RT_CONTEXT_ATTRIBUTE_GPU_PAGING_FORCE_OFF禁用大内存分页，其他的属性是只可读的。
<<>>为了支持递归调用，OptiX使用一小块堆栈关联每一个线程的执行。rtContext{Get|Set}StackSize可以设置或者查询这块堆栈的大小。堆栈大小的设置一定要小心，过大的堆栈将会导致性能的降级而过小的堆栈将会导致堆栈溢出。堆栈溢出错误可以通过用户定义的异常程式经行处理。
<<>>可以通过像C风格下的printf函数一样的rtContextSetPrint\*函数集开启OptiX程式的打印功能，允许这些程式更加容易被调试。CUDA C函数rtContextSetPrintEnabled可以开启或者关闭打印全部（程式），而rtContextSetPrintLaunchIndex可以针对具体的计算网格进行打印。全局打印状态被关闭的时候，打印语句不会对性能有一点损害，默认状态是关闭的。
<<>>打印请求会被缓存在一个内部缓冲区中，缓冲区的大小可以通过rtContextSetPrintBufferSize设置。该缓冲区的溢出会导致输出流的截断（译注：应该不会报告溢出错误，而是直接抛弃最早的记录）。输出流是在所有的计算进行结束而rtContextLaunch函数返回之前打印到标准输出的。

{% highlight c++%}
RTcontext context = ...;
rtContextSetPrintEnabled( context, 1 );
rtContextSetPrintBufferSize( context, 4096 );
{% endhighlight %}

在OptiX程式中，rtPrintf函数像C语言下的printf一样工作。对rtPrintf的调用会自动将（数据）保存进输出缓冲区中。**但是同一个线程的分次调用或者不同线程的调用其输出会是相互交错的。（译注：多线程的输出交错能理解，单线程的多次调用怎么还会任意交错输出？）**

{% highlight c++%}
rtDeclareVariable(uint2, launch_idx ,rtLaunchIndex,
);
RT_PROGRAM void any_hit()
{
rtPrintf( "Hello from index %u, %u!\n",
launch_idx.x, launch_idx.y );
}
{% endhighlight %}

<<>>上下文也是OptiX最外层的变量作用域。通过rtContextDeclareVariable申明的变量是关联到给定上下文的所有对象上的。为了避免命名冲突，存在的变量可以通过rtContextQueryVariable（传变量名）或者rtContextGetVariable（传索引）查询，通过rtContextRemoveVariable删除。
<<>>可以在装配过程的任意时刻调用rtContextValidate来检测上下文状态与关联对象的有效性与合法性。也会一起检测已存在的必要程式（比如一个相交程式）的内部状态与关联的外部变量的合法性。合法性检测总会在上下文执行的时候隐性的被调用。
<<>>执行rtContextCompile会明确的请求计算内核与关联的上下文对象的编译。当场景参数或者程式变化后并不需要严格的调用rtContextCompile，因为在下次调用rtContextLaunch的时候会触发编译。rtContextCompile允许用户控制编译的时间，但是尽量在编译之前设置完毕所有的上下文内容，因为后续的改动会导致rtContextLaunch在调用时重新编译。
<<>>（译注：待译，原文：rtContextSetTimeoutCallback specifies a callback function of type RTtimeoutcallback that is called at a specified maximum frequency from OptiX API calls that can run long, such as acceleration structure builds, compilation, and kernel launches. ）这就允许应用程序更新它的接口或者执行其他任务。回调函数也可以要求OptiX停止当前的工作并将控制权返回给应用程序，这个请求是被立即执行的。（待译，原文： Output buffers expected to be written to by an rtContextLaunch are left in an undefined state, but otherwise OptiX tracks what tasks still need to be performed and resumes cleanly in subsequent API calls.） 

{% highlight c++%}
// Return 1 to ask for abort, 0 to continue.
// An RTtimeoutcallback.
int CBFunc()
{
update_gui();
return bored_yet();
}
…
// Call CBFunc at most once every 100 ms.
rtContextSetTimeoutCallback( context, CBFunc, 0.1 );
{% endhighlight %}

<<>>rtContextGetErrorString函数可以返回任何出现在上下文设置、合法性检测或者发射（发射射线）执行的失败描述。

## 3.2 缓冲区
<<>>OptiX利用缓冲区向C端与G端传递数据。缓冲区要在rtContextLaunch使用之前利用函数rtBufferCreate创建，这个函数可以为缓冲区设置类型与可选的标记（flags），他们是通过位运算或组合起来的。缓冲区的类型定义了数据流的走向。类型是个枚举值，定义在了RTbuffertype里面：

* RT_BUFFER_INPUT：只有C端可以写数据。数据从C端传送到G端，G端只可读；
* RT_BUFFER_OUTPUT：RT_BUFFER_INPUT的相反类型；
* RT_BUFFER_INPUT_OUTPUT：允许C、G两端都读与写；
* RT_BUFFER_PROGRESSIVE_STREAM：自动更新的发射输出流。可以在网络连接中有效流输出。

缓冲区的标志描述了它的特征，也是枚举类型见下：

* RT_BUFFER_GPU_LOCAL：（待译）；
* RT_BUFFER_LAYERED：当缓冲区是用来作为纹理缓冲区时，它的深度信息表示层的数量而非3D缓冲区中的深度；
* RT_BUFFER_CUBEMAP：深度作为立方体的面来使用，而不是3D缓冲区中的深度；

<<>>在使用一个缓冲区之前，它的大小、纬度、数据的格式必须先被指定。数据格式可以通过函数rtBuffer{Get|Set|}Fomart设置，格式是RTformat枚举类型的，是为C、CUDA C的数据类型而设计的，比如unsigned int 、float3等。可以使用RT_FORMAT_USER来指定任意类型的数据。通过rtBufferSetElementSize函数指定元素的大小。通过rtBufferSetSize{1,2,3}D来指定缓冲区的大小，同样他们也指定了缓冲区的纬度。通过调用rtBufferGetMipLevelSize{1,2,3}D函数并且传入mip层，可以获取到纹理缓冲区中mip层的大小。

{% highlight c++%}
RTcontext context = ...;
RTbuffer buffer;
typedef struct { float r; float g; float b; } rgb;
rtBufferCreate( context, RT_BUFFER_INPUT_OUTPUT, &buffer );
rtBufferSetFormat( RT_FORMAT_USER );
rtBufferSetElementSize( sizeof(rgb) );
rtBufferSetSize2D( buffer, 512, 512 );
{% endhighlight %}

<<>>C端获取到缓冲区中的数据是要通过rtBufferMap函数，这个函数返回一个一维的数组指针。所有的缓冲区必须通过rtBufferUnmap取消映射（unmap），这样上下文的验证才能成功。

{% highlight c++%}
// Using the buffer created above unsigned int width, height;
rtBufferGetSize2D( buffer, &width, &height );
void* data;
rtBufferMap( buffer, &data );
rgb* rgb_dat = (rgb*)data;
for( unsigned int i = 0; i < width*height; ++i ) {
	rgb_dat[i].r = rgb_dat[i].g = rgb_dat[i].b =0.0f;
}
rtBufferUnmap( buffer );
{% endhighlight %}

rtBufferMapEx与rtBufferUnmapEx设置mipmap的纹理如下：

{% highlight c++%}
// Using the buffer created above
unsigned int width, height;
rtBufferGetMipLevelSize2D( buffer, &width, &height, level+1 );
rgb *dL, *dNextL;
rtBufferMapEx( buffer, RT_BUFFER_MAP_READ_WRITE, level, 0, &dL );
rtBufferMapEx( buffer, RT_BUFFER_MAP_READ_WRITE, level+1, 0, &dNextL );
unsigned int width2 = width*2;
for( unsigned int y = 0; y < height; ++y ) {
	for( unsigned int x = 0; x < width; ++x ) {
		dNextL[x+width*y] = 0.25f *
		(dL[x*2+width2*y*2] +
		dL[x*2+1+width2*y*2] +
		dL[x*2+width2*(y*2+1)] +
		dL[x*2+1+width2*(y*2+1)]);
	}
}
rtBufferUnmapEx( buffer, level );
rtBufferUnmapEx( buffer, level+1 );
{% endhighlight %}

在OptiX程式中访问缓冲区需要一个简单的数组语法。下面模板的2个形参申明了元素类型一维度。

{% highlight c++%}
rtBuffer<rgb,2> buffer;
...
uint2 index=...;
floa r=buffer[index].r;
{% endhighlight %}

### 3.2.1 缓冲区与缓冲区ID
<<>>从OptiX3.5开始，缓冲区包含了缓冲区ID。在C端将输入缓冲区的格式申明为RT_FORMAT_BUFFER_ID的话，缓冲区就会填充缓冲区的ID（ID是通过rtBufferGetId或者optix::Buffer::getId获得）。一个特殊的id值为RT_BUFFER_ID_NULL，利用它可以判断哪些buffer是合理哪些不是。RT_BUFFER_ID_NULL永远不会以有效的缓冲区ID返回。下面的示例创建了2个输入缓冲区，第一个保存数据，第二个保存ID。

{% highlight c++%}
Buffer inputBuffer0 = context->createBuffer( RT_BUFFER_INPUT, RT_FORMAT_INT, 3 );
Buffer inputBuffers = context->createBuffer( RT_BUFFER_INPUT, RT_FORMAT_BUFFER_ID, 1);
int* buffers = static_cast<int*>(inputBuffers- >map());
buffers[0] = inputBuffer0->getId();
inputBuffers->unmap();
{% endhighlight %}

<<>>>在G端来的话，ID缓冲区使用模板形参为rtBufferId的参数进行申明。在缓冲区中的保存的标识符被隐式的转化为缓冲期句柄。下面的示例创建了一个一维缓冲区，它的元素就是它们自己————一个一维的存储了整型变量的缓冲区。（译注：bufferId与buffer在本质上相同的，参见OptiX_API_Reference_3.0.0.pdf P.98）

{% highlight c++%}
rtBuffer<rtBufferId<int,1>,1> input_buffers;
{% endhighlight %}

获取缓冲区id的方式与正常缓冲区一样：

{% highlight c++%}
// Grab the first element of the first buffer in 'input_buffers'
int value=intpu_buffer[buf_index][0];
{% endhighlight %}

也可以查询缓冲区的大小，以遍历内容：

{% highlight c++%}
for (size_t i=0;k<input_buffers.size();++i)
	result+=input_buffers[i];
{% endhighlight %}

缓冲区可以任意的嵌套，就是每一层嵌套被访问时会有额外内存开销。多个缓冲区的查找可以通过使用引用或者rtBufferId的副本来避免。

{% highlight c++%}
rtBuffer<rtBufferId<rtBufferId<int,1>,1>,1> input_buffers3;
...
rtBufferId<int,1>& buffer=input_buffers3[buf_index1][buf_index2];
size_t size =buffer.size();
for(size_t i=0;i<size;++i)
	value+=buffer[i];
{% endhighlight %}

<<>>当前只有非互相操作（non-interop）而且类型为RT_BUFFER_INPUT的缓冲区才能容纳缓冲区ID，并且这些ID对应的缓冲区必须在元素与维度上一致，大小可以不一样。可以通过rtCOntextGetBufferFromId函数获取与缓冲区ID关联的RTbuffer对象，也可以通过C++接口optix::Context::getBufferFromId获取。
<<>>除了将缓冲区ID保存在缓冲区中以外，你可以将它们保存在任意的结构体或者RTvariables中，也可以作为夹带数据的元素或者作为参数传递给可调用程序（callable programs）。一个rtBufferId对象可以使用缓冲区ID进行构建。

{% highlight c++%}
rtDeclareVariable(int, id,,);
rtDeclareVariable(int, index,,);
...
//译注：将id强转为rtBufferId<int,1>，然后取索引为index的值；
int value = rtBufferId<int,1>(id)[index];
{% endhighlight %}

一个传递给可调用程序的示例如下：

{% highlight c++%}
#include <optix_world.h>
using namespace optix;

struct BufInfo {
	int index;
	rtBufferId<int, 1> data;
};
rtCallableProgram(int, getValue, (BufInfo));
//译注：pdf下面getVal打错，应该为getValue，see P.21
RT_CALLABLE_PROGRAM int getVal( BufInfo bufInfo ) {
	return bufInfo.data[bufInfo.index];
}

rtBuffer<int,1> result;
rtDeclareVariable(BufInfo, buf_info, ,);
RT_PROGRAM void bindlessCall() {
	int value = getValue(buf_info);
	result[0] = value;
}
{% endhighlight %}

<<>>注意到rtCallableProgram与rtDeclareVariable（译注：原文有误：rtCallProgram）都是宏，因此应该使用类型定义、结构体代替模板类型，以便绕过C的预处理限制。

{% highlight c++%}
typedef rtBufferId<int,1> RTB;
rtDeclareVariable(RTB,buf,,);
{% endhighlight %}

<<>>在optixpp_namespace.h文件中有一个对G端rtBufferId的镜像定义，可以使用它来定义在两端都能使用了代码。下面是个使用BufInfo结构体的C端代码：

{% highlight c++%}
BufInfo buf_info;
buf_info.index = 0;
buf_info.data = rtBufferId<int,1>(inputBuf0-> getId() );
context["buf_info"]->setUserData(sizeof(buf_info), &buf_info);
{% endhighlight %}

## 3.3 纹理
<<>>OptiX的纹理支持普通的纹理映射功能，包括过滤，各种缠绕（wrap）方式，纹理采样。rtTextureSamplerCreate用来创建纹理对象。每一个纹理对象关联到一个或者几个存有纹理数据的缓冲区。缓冲区可以是1D、2D或者3D的，可以通过rtTextureSamplerSetBuffer函数设置。函数rtTextureSamplerSetFilteringModes可以用来设置放大缩小，mipmap的过滤方式。缠绕方式通过函数rtTextureSamplerSetWrapMode设置。给定纹理的最大各向异性通过函数rtTextureSamplerSetMaxAnisotropy函数进行，大于0的值会在该值的特定情况下开启各向异性。rtTextureSamplerSetReadMode函数可以用来将读取的纹理结果自动转换成单位化了的浮点值。

{% highlight c++%}
RTcontext context = ...;
RTbuffer tex_buffer = ...; // 2D buffer
RTtexturesampler tex_sampler;
rtTextureSamplerCreate( context, &tex_sampler );
rtTextureSamplerSetWrapMode( tex_sampler, 0, RT_WRAP_CLAMP_TO_EDGE);
rtTextureSamplerSetWrapMode( tex_sampler, 1, RT_WRAP_CLAMP_TO_EDGE);
rtTextureSamplerSetFilteringModes( tex_sampler,
	RT_FILTER_LINEAR,
	RT_FILTER_LINEAR,
	RT_FILTER_NONE );
rtTextureSamplerSetIndexingMode( tex_sampler, RT_TEXTURE_INDEX_NORMALIZED_COORDINATES );
rtTextureSamplerSetReadMode( tex_sampler, RT_TEXTURE_READ_NORMALIZED_FLOAT );
rtTextureSamplerSetMaxAnisotropy( tex_sampler, 1.0f );
rtTextureSamplerSetBuffer( tex_sampler, 0, 0, tex_buffer );
{% endhighlight %}

<<>>OptiX的3.9版本支持立方体、层级和mipmap纹理，分别通过rtBufferMapEx，rtBufferUnmapEx，rtBufferSetMipLevelCount设置。层级纹理与CUDA等级纹理、OpenGL的纹理数组是相同的。通过rtBufferCreate函数传递RT_BUFFER_LAYERED参数创建层级纹理。传递RT_BUFFER_CUBEMAP创建立方体纹理。这2种方式下的缓冲区深度分别表示层的数量与立方体的面数，并不是3D缓冲区的深度概念。OptiX程式可以通过CUDA C的内建函数tex1D、tex2D、tex3D获取纹理的数据。

{% highlight c++%}
rtTextureSampler<uchar4, 2, cudaReadModeNormalizedFloat> t;
...
float2 tex_coord = ...;
float4 value = tex2D( t, tex_coord.x, tex_coord.y );
{% endhighlight %}

OptiX3.0版本支持‘非绑定’纹理（bindless）。非绑定纹理允许OptiX程式直接引用纹理而不通过变量绑定，这是通过纹理ID来实现的。使用非绑定纹理可以动态的在多纹理之间切换而不需要明确的申明变量，也不需要手工实现切换代码。要切换的纹理可以拥有不同的属性，比如缠绕方式、纹理尺寸等，提供了比纹理数组更多的灵活性。为了从已存在纹理采样器中获取设备句柄，可以使用rtTextureSamplerGetId函数：

{% highlight c++%}
RTtexturesampler tex_sampler = ...;
int tex_id;
rtTextureSamplerGetId( tex_sampler, &tex_id );
{% endhighlight %}

<<>>一个纹理ID的值是不可变且有效的，除非销毁关联它的纹理采样器。让纹理ID在OptiX程式中可用，可以通过输入缓冲区或者OptiX的变量：

{% highlight c++%}
RTbuffer tex_id_buffer = ...; // 1D buffer
unsigned int index = ...;
void* tex_id_data;
rtBufferMap( tex_id_buffer, &tex_id_data );
((int*)tex_id_data)[index] = tex_id;
rtBufferUnmap( tex_id_buffer );
{% endhighlight %}

与CUDA C的纹理函数一样，OptiX程式可以通过rtTex1D<>，rtTex2D<>，rtTex3D<>函数访问非绑定纹理：

{% highlight c++%}
rtBuffer<int, 1> tex_id_buffer;
unsigned int index = ...;
int tex_id = tex_id_buffer[index];
float2 tex_coord = ...;
float4 value = rtTex2D<float4>( tex_id, tex_coord.x, tex_coord.y );
{% endhighlight %}

纹理也可以通过向其提供mipmap的LoD或各向异性过滤的渐变来进行采样。

{% highlight c++%}
float4 v;
if( mip_mode == MIP_DISABLE )
v = rtTex2DLayeredLod<float4>( tex, uv.x, uv.y, tex_layer );
else if( mip_mode == MIP_LEVEL )
v = rtTex2DLayeredLod<float4>( tex, uv.x, uv.y, tex_layer, lod );
else if( mip_mode == MIP_GRAD )
v = rtTex2DLayeredGrad<float4>( tex, uv.x, uv.y, tex_layer, dpdx, dpdy );
{% endhighlight %}

## 3.4 节点图（Graph Nodes）
<<>>当射线在程式中通过函数rtTrace发射的时候，需要给传递一个根节点代表节点图。C端应用通过组合OptiX API提供的节点类型创建这种图。这种图的基本结构是一个层级结构，底部是描述几何对象的节点，顶部是收集对象的节点。
<<>>这种图的结构并不是传统场景中的场景图（scene graph），相反，它提供了一种方式来将程式或者动作（action）绑定当场景中的一组对象上。由于trTrace的调用需要一个根节点，那么不同的树或者子树都可以被传入（译注：这就表示一组对象）。比如，具有阴影的对象或者可以反射光线的对象可能使用不同的表现————要么为了性能，要么为了艺术效果。
<<>>节点图中的节点通过调用rt\*Create函数创建（需要传入上下文作为一个参数）。由于这些节点对象是属于上下文的而非他们的父节点，所以调用rt\*Destroy函数可以删除这个对象变量，但是不会做任何引用计数或者自动释放它的子节点。
<<>>图1 展示了一个节点图可能的样子，接下来的小节会表述各个不同的节点类型。
<<P24>>
<<>>表2 说明了那些节点可以作为其他节点的子节点，包括与加速提节点的关联。

|Node type| Children nodes allowed|
|:---:|:---:|
|Geometry|-none-|
|Material|-none-|
|GeometryInstance|Geometry,Material|
|GeometryGroup|GeometryInstance,Acceleration|
|Group|GeometryGroup,Group,Seletor,Transform,Acceleration|
|Transform|GeometryGroup,Group,Selector,Transform|
|Selector|GeometryGroup,Group,Selector,Transform|
|Acceleration|-none-|

### 3.4.1 几何体
<<>>几何体节点是表述几何体对象（一个集合了用户定义的图元，用来与射线相交）的基础节点。几何体节点所包含的图元数量通过rtGeometrySetPrimitiveCount函数设置，为了定义图元，就需要通过函数rtGeometrySetIntersectionProgram指派给几何体相交程式（译注：图元的定义完全在于C端，OptiX完全不知道也没有枚举值定义各种图元的类型，所以与用户定义的图元是否相交完全在于用户定义的相交程式，所以这就是原文中图元的定义是通过相交程式给出的含义）。输入的参数一个是图元的索引（index），一个是射线，相交程式的工作就是去判断它两之间的相交情况。结合程式的变量这就提供了一种必要的机制去定义任意图元类型与射线相交。一个常用的示例是个三角网格，相交程式从传入的顶点数据缓冲区中读取必要的三角面数据然后进行射线-三角形相交。
<<>>为了给任意几何体构建加速体，让OptiX去查询每一个图元的边界（bounds）是必要的。出于这样的考虑，一个边界程式（bounds program）必须通过rtGeometrySetBoundingBoxProgram函数提供给几何体（译注：就是前文翻译的包围盒程式）。这个程式简单的计算出需求的图元的包围盒（译注：该程式在被OptiX调用的时候会传入2个参数，一个是图元在几何体中的索引，一个是存储该图元包围盒大小的指针。详细讨论见：sfsfsfs），然后OptiX用它作为基础来构架加速体。
<<>>几何节点可以完全是动态的，比如图元的数量或者在调用rtContextLaunch函数之间绑定到相交、包围盒程式中的变量值都是可变的。当这样的情况发生之后，包含那个修改了的几何体的加速体必须被告知（几何体变化了），这样的告知是通过函数rtGeometryMakDirty函数进行的。针对加速体重构建的更多信息参考章节3.5。
<<>>下面的示例展示了如何构建一个用球体作为单一图元的几何对象。相交与包围盒程式利用单个参数作为球体的半径。

{% highlight c++%}
RTgeometry geometry;
RTvariable variable;
// Set up geometry object.
rtGeometryCreate( context, &geometry );
rtGeometrySetPrimitiveCount( geometry, 1 );
rtGeometrySetIntersectionProgram( geometry, sphere_intersection);
rtGeometrySetBoundingBoxProgram( geometry, sphere_bounds );
// Declare and set the radius variable.
rtGeometryDeclareVariable( geometry, "radius", &variable );
rtVariableSet1f( variable, 10.0f );
{% endhighlight %}

## 3.4.2 材质
<<>>材质封装了行为，它会在射线与某个图元相交后被调用（材质必须先前关联该几何体）。这样的行为包括：计算反射颜色，发射额外的射线，忽视相交，终止射线等。通过申明程式变量可以为材质提供任意的参数。
<<>>两种类型的程式需要被指派给材质：最近碰撞程式与任意碰撞程式。这两种类型的不同之处是何时、什么频率被执行。最近碰撞程式与传统渲染系统相似，是为射线与场景中最近相交处调用最多一次（每条射线）。它典型的会去执行纹理查询、计算反射颜色、光源采样、递归射线追踪等动作，最后把结果保存在射线的夹带数据结构中。
<<>>任意碰撞程式是在遍历过程中每一个潜在最近相交的情况下调用的（译注：OptiX官方Demo中的cu程序一般都是将rtReportIntesection写在rtPotentialIntersection的true结果内，所以才会有前面这句话，其实哪怕不是潜在最近相交，我们也可以直接调用rtReportIntersection来触发任意碰撞程式的调用）。相交后对该类型程式的调用可能与射线碰撞图元的（视觉）先后顺序不一样，实际上射线与场景中图元的相交次数如果需求的话可以被枚举出来（靠每次相交后调用rtIgnoreIntersection函数）（译注：编程指南这里说的没问题，但对于初学OptiX的新手难免有点疑惑，OptiX内核有个机制就是如果在相交后调用了rtIgnoreIntersection函数，就会还原t区间，这样就不会丢弃对区间以外几何体的相交了）。使用任意碰撞程式的典型用法就是在阴影射线中尽早终止遍历（使用rtTerminateRay）或者binary transparency（译注：咋翻译？）。比如是否忽略相交是基于纹理查询的结果。
<<>>要说明的一个重点是：两种类型的程式是以射线类型指派给材质的，这就意味着每种材质可以持有多于1种的最近或者任意碰撞程式。应用程序能够区别具体类型的射线是什么行为是有用的，比如将阴影射线的类型分离出来，有助于仅仅用来判断场景中两点之间的可见性。这种情况的话，一个简单的任意碰撞程式（指定为只接收阴影射线类型）可以立即结束射线的遍历，而最近碰撞程式完全不会有触发（译注：因为最近碰撞程式接受的射线类型不是阴影射线定义的类型）。这个概念允许对每种类型的射线进行高效的特殊化编程。

### 3.4.3 几何实例
<<>>一个几何实例代表了一个几何体与一组材质。关联的几何体通过rtGeometryInstanceSetGeometry设置，关联的材质的数量通过rtGeometryInstanceSetMaterialCount函数设置。而关联的材质数量由材质的最大索引决定，这个索引是用来在相交程式中被报告时使用的（译注：就是调用rtReportIntersection函数的时候传递的参数，就是索引编号）。注意，多个几何实例可以引用一个几何体，允许几何体有不同的材质。同样，材质也可以在不同的几何实例之间重用。
<<>>这是示例配置了一个几何实例，它的第一个材质索引为mat_phong，第二个为mat_diffuse，两个都是rtMaterial对象并且指派了恰当的程式。几何实例引用了triangle_mesh这么一个rtGeometry对象。

{% highlight c++%}
RTgeometryinstance ginst;
rtGeometryInstanceCreate( context, &ginst );
rtGeometryInstanceSetGeometry( ginst, triangle_mesh);
rtGeometryInstanceSetMaterialCount( ginst, 2 );
rtGeometryInstanceSetMaterial( ginst, 0, mat_phong);
rtGeometryInstanceSetMaterial( ginst, 1, mat_diffuse);
{% endhighlight %}

### 3.4.4 几何组
<<>>几何组就是个可以存放任意数量几何实例的容器。存放的数量通过rtGeometryGroupSetChildCount设定，几何实例通过函数rtGeometryGroupSetChild指定。每一个几何组必须通过rtGeometryGroupSetAcceleration函数指定一个加速体（见3.5节）。最简单的使用方式就是指派一个几何实例：

{% highlight c++%}
RTgeometrygroup geomgroup;
rtGeometryGroupCreate( context, &geomgroup );
rtGeometryGroupSetChildCount( geomgroup, 1 );
rtGeometryGroupSetChild( geomgroup, 0, geometry_instance );
{% endhighlight %}

<<>>多个几何组之间可以共享孩子，这就意味着几何实例可以是好几个几何组的孩子。

### 3.4.5 组
<<>>组在节点图中代表一个集合其他对象的更高级节点。它们用来编译节点图结构，然后传递给rtTrace进行射线相交。
<<>>一个组可以容纳任意数量的子节点，这些子节点必须是rtGroup、rtGeometryGroup、trTransform与rtSelector对象。通过rtGroupSetChildCount函数指定子节点数量，通过rtGroupSetChild指派具体的子节点对象。每一个组必须通过rtGroupSetAcceleration函数指派一个加速体。
<<>>组的常规用法就是集合几个互相之间独立动态运动的几何组。位置、旋转以及缩放参数可以通过变换节点进行展现，所以在rtContextLaunch函数之间（译注：每次渲染就要调用这个函数）进行重构建的唯一加速节点就是顶层节点。这通常比更新整个场景的加速体要块的多。
<<>>注意到组中的子节点可以被其他组共享，那么没有子节点有可能也是另一个组的子节点。这允许非常灵活且轻量级实例的场景（的展示）。尤其是结合了共享加速体（见3.5节）。

### 3.4.6 变换
<<>>变换节点用来对其下子节点进行投影变换后的呈现。变换节点必须指派一个如下节点：rtGroup、rtGeometryGroup、trTransform、或者rtSelector。这就是说在变换节点下的节点要么就是几何组中普通的几何体，要么就是一整个场景中全新的子节点图。
<<>>变换其实就是通过rtTransormSetMatrix传递了一个4X4浮点矩阵（16个元素的一维数组）。它可以看成是这个矩阵变换了其下几何体。然而，这种变换的效果是通过变换射线而得到的。这就意味着变换没有让OptiX重构建任何加速体。
<<>>这个示例展示了一个只有平移变换的变换对象是如何创建的：

{% highlight c++%}
RTtransform transform;
const float x=10.0f, y=20.0f, z=30.0f;
// Matrices are row-major.
const float m[16] = 
	{ 1, 0, 0, x,
		0, 1, 0, y,
		0, 0, 1, z,
		0, 0, 0, 1 };
rtTransformCreate( context, &transform );
rtTransformSetMatrix( transform, 0, m, 0 );
{% endhighlight %}

### 3.4.7 选择器
<<>>选择器与组（节点）相似就是一个更高级别的节点图中的节点。其中的子节点数量通过rtSelectorSetChildCound设置，具体的子节点通过rtSelectorSetChild设置。有效的子节点类型为：rtGroup、rtGeometryGroup、trTransform、或者rtSelector。
<<>>选择器（节点）与组（节点）的主要区别在于选择器没有加速体。取而代之的是访问程式的绑定，通过rtSelectorSetVisitProgram绑定。这个程式是在射线遍历过程中每次遇到选择器节点后调用。程式说明了哪些子节点（选择器下的子节点）能够被射线继续遍历，调用rtInstersectChild函数后即表示该子节点需要被继续遍历。
<<>>选择器的典型用法就是动态的调节层次细节（Lod）：场景中的对象可能以多种几何节点形式存在，每一个包含了不同的Lod。包含这些层次细节的几何体的几何组们（译注：注意这里是几何组们，每一个几何组包含所有几何体的一种Lod）可以指派给一个选择器。访问程式根据标准（比如基于足印或者当前射线的长度为标准）选择哪个子节点（译注：就是哪个几何组）可以相交，剩下的就可以忽略掉。
<<>>选择器的子节点可以和其他场景图中的节点共享，以允许灵活实现。

## 3.5 射线追踪中的加速结构体
<<>>加速体在加速遍历与相交查询中是个重要的工具，尤其是对数据量大的场景。成功的加速体代表场景中几何体被瓦解之后的层级结构，这个层级结构之后被用来在射线相交时候快速排除空间区域。
<<>>有好多种不同类型的加速体，每一种都有他的优点与短板。不同的场景需要不同类型的加速体以达到性能优化的目的（比如静态与动态场景，普通的图元与三角形图元等）。通常的取舍就是构建速度与射线追踪的性能，极限的解决方案就好比光谱的两头，比如一个高质量的SBVH构建器需要话费数分钟去构建它的加速体，一旦完成，射线就可以比其他类型的加速体更加高效的进行追踪，其碰撞速度肯定也会更快（意译）。然而，见表4中的Trbvh类型。
<<>>没有任何一个加速体可以为所有的场景优化。为了让应用程序平衡取舍，OptiX会让你选择几个支持类型中的一个。你甚至可以在节点图中混合使用这些加速体。

### 3.5.1 节点图中的加速体对象
<<>>加速体是OptiX中独立的API对象，被称为rtAcceleration。当通过rtAccelerationCreate函数创建一个加速体对象后，它就会被指派给组（节点）或者几何组（节点）。节点图中的任何一个组或者几何组都要为射线与节点相交指派一个加速体对象。这个示例创建了一个几何组与一个加速体，然后连接了它们。

{% highlight c++%}
RTgeometrygroup geomgroup;
RTacceleration accel;
rtGeometryGroupCreate( context, &geomgroup );
rtAccelerationCreate( context, &accel );
rtGeometryGroupSetAcceleration( geomgroup, accel );
{% endhighlight %}

<<>>在节点图中的设置组与几何组的用途，应用程序有更高的控制权决定加速体如何构建场景中的几何体。考虑一个场景中有几个几何实例的情况，有好几种方式可以把他们按照应用程序的用途组织起来。比如，图2将所有的几何实例放在了一个几何组中，一个指派给该几何组的加速体将会构建组中所有几何实例的几何体的图元。这就允许OptiX去构建一个好像把这些几何体合并在一起（作为一个整体）的大几何体对象，这样构建出的加速体的性能与构建一个几何体之后的性能一样。一个不同的组织多几何实例的方式展现在图3中。每一个（几何）实例都放在自己独立的几何组中，也就是说每个（几何）实例拥有独立的加速体。几何组被收集在一个顶层的组（节点）中（该节点依然需要一个加速体）。加速体的构建是基于子节点的包围盒的。因为子节点通常处于节点图的底部，所以高层结构体更新速度会比较快。这种构建的优点就是如果一个几何实例发生了变化，其他几何实例的加速体必不进行重构建。然而，因为更高层的加速体是在其子节点的基础上粒度更大的构建了额外一个结构，所以图3中的节点图将会比图2中的要性能差点。再次说明下，这是一个需要应用程序平衡的地方，也就是说在这个例子中需要考虑独立的几何实例会不会频繁的变化。

<<P.30两张图>>

### 3.5.2 构建器与遍历器
<<>>一个rtAcceleration对象是由一个构建器（builder）与一个遍历器（traverser）组成的。构建器的责任就是收集输入的几何体（注意：在大多数情况下，这里的“几何体”是由包围盒程式生成的轴对齐几何体）并且计算加速体中的值以便遍历器加速射线场景的相交查询。构建器与遍历器不是应用程序定义的程式，而是需要应用从表3中选择恰当的构建器与遍历器。

（译注：具体内容待译）；
|Builder/Traverser|Description|
|:---:|:----|
|Trbvh/Bvh||
|Sbvh/Bvh or BvhCompact||
|Bvh/Bvh or BvhCompact||
|MedianBvh/Bvh or BvhCompact||
|Lvh/Bvh or BvhCompact||
|TriangleKdTree/KdTree||
|NoAccel/NoAccel|
表3 支持的构建器与遍历器

<<>>表3展示了OptiX目前支持的构建器与遍历器。构建器通过rtAccelerationSetBuilder设置，对应的遍历器（一定要兼容相应的构建器）通过rtAccelerationSetTraverser设置。构建器与遍历器可以随时改动；改变一个构建器会导致加速体被标记从而重新构建。这个示例展示了一个典型的加速对象的初始化；

{% highlight c++%}
RTacceleration accel;
rtAccelerationCreate( context, &accel );
rtAccelerationSetBuilder( accel, "Trbvh" );
rtAccelerationSetTraverser( accel, "Bvh" );
{% endhighlight %}

### 3.5.3 属性
<<>>针对不同情况精调加速体的构建过程是有好处的。为了这个目的，构建器暴露了一些变量（被称作属性）见表4：
|Property|Available in Builder|Description|
|:---:|:---:|:---:|
|refine|Bvh Lbvh MedianBvh|传递给BVH的改进次数，用以改善BVH的品质。默认“0”|
|refit|Bvh Lbvh MedianBvh|默认“0”|
|vertex_buffer_name|Sbvh Trbvh TriangleKdTree|默认“vertex_buffer”|
|vertex_buffer_stride|Sbvh Trbvh TriangleKdTree|默认“0”|
|index_buffer_name|Sbvh Trbvh TriangleKdTree|默认“index_buffer”|
|index_buffer_stride|Sbvh Trbvh TriangleKdTree|默认“0”|
|chunk_size|Trbvh||
|builde_type|Trbvh||
表4 加速体属性

<<>>属性通过rtAccelerationSetProperty函数设置。他们的值是通过字符串形式给出的（会被OptiX解析）。只有当加速体实际上重新构建的时候，属性才会起作用。设置或者改变属性并不会令加速体自动标志为“重新构建”，如果做见下节。不能被构建器识别的属性会静默的忽视。

{% highlight c++%}
// Enable fast refitting on a BVH acceleration.
rtAccelerationSetProperty( accel, "refit", "1");
{% endhighlight %}

### 3.5.4 加速体构建
<<>>在OptiX中，当加速体需要被重新构建的时候就会被标记（标记为“脏”）。在rtContextLaunch调用后，所有标记了的加速体需要被重新构建才能开始执行射线追踪。每一个新创建的加速体对象初始化时都被标记以便构建。
<<>>应用程序可以在任何时间明确的标记一个加速体让他重新构建。举个例子，如果几何组中的几何体改变了，关联在几何组上的加速体必须被重新构建。这一过程需要调用rtAccelerationMarkDirty函数。几何组中新添加几何实例或者移除几何实例也要求情加速体购新构建。
<<>>在组（节点）上的加速体也是一样的：添加或者移除子节点，改变组中的变换节点等都需要加速体被标记而后重新构建。一般来说，每一个引起几何体改变的操作，其所在的加速体都要被重新构建（对于组而言，几何体就是子节点的AABB）。然而，有些变化是不需要被重构建的，比如一些节点图中的节点转移到了节点图中更底的位置，而其并没有影响组中子节点的包围盒。
<<>>应用程序独立决定了每一个加速体是否需要被重新构建。OptiX将不会尝试自动检测改变，将一个加速体标记为“脏”后将不会衍生到其他加速体的标志位。要标记但是没有标记的话会导致不可预料的行为————通常就是丢失相交或者性能恶化。

### 3.5.5 缓存加速数据
<<>>加速体的构建可能会慢，这取决于构建器的选取与关联数据的复杂程度。OptiX提供了一种方式就是可以从此前构建的加速体中导出数据，这就允许应用程序可以保存加速体的数据以便此后复用。
<<>>加速体的数据可以通过函数rtAccelerationGetData从加速对象中获取。通过函数rtAccelerationSetData进行保存。下面的例子展示了如何从一个加速体中获取数据：

{% highlight c++%}
RTsize size;
void\* data;

rtAccelerationGetDataSize(accel,&size);
data=malloc(size);
rtAccelerationGetData(accel,data);
{% endhighlight %}

<<>>注意，数据的获取只能从没有被标记为“脏”的加速体中获取（意思是说加速体必须已经被构建），这样规定（要求）的目的是保证数据是有效的。因此，强制让加速体构建而不经过射线追踪是有用的（译注：在调用rtContextLaunch后才会进行“脏”加速体的构建，但此后会执行射线追踪）。可以通过调用rtContextLaunch函数（传递维度变量为0）后完成（强制构建）。

{% highlight c++%}
rtContextLaunch1D(context,0,0);
{% endhighlight %}

<<>>由rtAccelerationGetData返回的数据会包含诸如构建器、遍历器还有属性等信息。成功的调用了rtAccelerationSetData后，所有这些数据都被保存在加速体中，并且该加速体被标记为“不脏”，就好像执行了一次普通的构建过程。这意味着，调用rtAccelerationSetData函数后可能会改变加速体的构建器名。
<<>>注意，返回的数据仅包含重构建加速体的信息，而几何体的数据，组或者几何组节点图的信息没有被包含。应用程序应该保证恢复的数据匹配组（或者几何组）中的几何体，否则行为将不可定义。
<<>>良好编码出的应用程序也应该总是准备好rtAccelerationSetData失败的情况。调用该函数失败的原因可能是OptiX改版后内部数据格式的改变，或者不同平台的兼容性。对于这种情况要得到正确结果的最直接方式就是将失败的加速体标记为“脏”，这将会导致加速体在运行中被构建而不是从缓存的数据中。

{% highlight c++ %}
if(rtAcceleratinSetDAta(accel,data,size)!=RT_SUCCESS){
	rtAccelerationMarkDirty(accel);
}
{% endhighlight %}

### 3.5.6 共享加速体
<<>>可以将一个节点绑定到多个其他节点内作为其子节点的机制让节点图的组合变得灵活，（译注：待译，主要是instancing怎么翻译：and enable interesting instancing applications, Instancing can be seen as inexpensive reuse of scene objects or parts of the graph by referencing nodes multiple times instead of duplicating them.）。
<<>>OptiX在节点图之间解耦了加速体作为分开的对象。这样，加速体就能够自然的在组、结合组之间共享，只要这些组（即合租）内部的几何体是同一个。

{% highlight c++%}
// Attach one acceleration to multiple groups;
rtGroupSetAcceleration( group1,accel);
rtGroupSetAcceleration( group2,accel);
rtGroutSetAcceleration( grout3,accel);
{% endhighlight %}

<<>>每个应用程序必须保证共享的加速体能够匹配其对应的几何体。不这样做的话会导致未定义的行为。同样，加速体是不能够在组与几何组之间共享的。
<<>>共享加速体对最大化性能是个重要的概念，如图4所示。处于中间的加速体节点绑定了2个几何组，几何组引用了同一个几何体对象。这种对加速体的重用减少了内存的使用和构建时候的时间，新加的几何组也可以像这样添加。
<<P.37图>>
图4 2个几何组共享一个加速体与几何对象。

## 3.6 Rendering on VCA
（暂时不译）

# 第四章 程式
<<>>这张描述了OptiX中的程式。程式提供了对射线相交，渲染，还有通用的OptiX射线追踪内核计算的可编程控制。OptiX的程式由绑定点关联起来，在射线追踪计算过程中提供了不同的语义角色。与其他概念一样，OptiX将程式抽象为对象模型叫做程式对象（program objects）。

## 4.1 OptiX程式对象
<<>>OptiX API的核心主题是可编程。OptiX的程式由CUDA C编写，通过字符串或者关联到CUDA的PTX（并行线程执行虚拟汇编语言 parallel thread execution virtual assembly language）文件提供。发布于CUDA SDK的nvcc编译器结合OptiX的头文件用来创建PTX文件。
<<>>这些PTX文件然后通过C端API绑定到程式对象上。程式对象可以用在任何OptiX程式类型上，后面章节会讨论它们。

### 管理程式对象
<<>>OptiX提供了两种创建程式对象的入口点：rtProgramCraeteFromPTXString，与rtProgramCreateFromPTXFile。第一种从PTX源码组成的字符串中穿件程式对象。后者从磁盘的PTX文件中创建。

{% highlight c++%}
RTcontext context = ...;
const char *ptx_filename = ...;
const char *program_name = ...;
RTprogram program = ...;
rtProgramCreateFromPTXFile( context, ptx_filename, function_name, &program);
{% endhighlight %}

在这个例子中，ptx_filename是一个在磁盘上的PTX文件的名字，function_name是该文件中的一个特指的函数名。如果程式格式错误或者不能够编译，这些如何点返回错误编号。
<<>>可以通过rtProgramValidate函数检测程式的完整性，就像下面的示例一样。

{% highlight c++%}
if( rtProgramValidate(context, program)!=RT_SUCCESS) {
	printf( "Program is not complete." );
}
{% endhighlight %}

<<>>从rtProgramValidate返回的错误码表示了程式对象或者其他绑定到其上的错误。最后，rtProgramGetContext函数返回拥有这段程式的上下文，而rtProgramDestroy函数表示释放自己以及关联的资源。

### 4.1.2 通过变量通信
<<>>OptiX的程式对象通过变量（variables）与C端程序通信。在OptiX中定义的变量是通过rtDeclareVariable宏申明的：

{% highlight c++%}
rtDeclareVariable(float, x, ,);
{% endhighlight %}

<<>>这个申明创建了一个类型为float的x变量，它可以通过OptiX的变量对象API对C端程序可见，也可以通过C语言的语义与G端程序通信。注意示例中后两个参数为空，但逗号依然要保留。讨论G端变量的地址是不被支持的。这表示示例中对x变量的指针或者引用是不被允许的。如果，你需要将x传递给需要float\*类型参数的的函数，你需要先拷贝x到栈中的一个变量，然后将局部的变量传递进去：

{% highlight c++%}
void my_func( float\* my_float) {…}
RT_PROGRAM call_my_func()
{
	my_func(&x); // not allowed
	float local_x = x;
	my_func(&local_x); // allowed
}
{% endhighlight %}

<<>>这样申明的变量可以通过rtVariableGet\*与rtVariableSet\*家族函数在C端读或者写。这样定义的变量隐式的在G端成为常变量（const-qualified）。如果从G端到C端的通信是必须的，那就使用rtBuffer对象代替变量对象。
<<>>OptiX2.0时代，为了避免命名冲突变量会被在任意嵌套的命名空间中定义。C端变量若要引用OptiX的变量需要指定全路径。
<<>>程式中的变量可以用语义（semantics）形式定义（译注：sematics咋翻译？）。用语义形式定义的变量将会被绑定给一个特殊的对象，OptiX在整个射线追踪内核的生命期内部管理该对象。举例来说，用rtCurrentRay语音形式定义的变量将会是个只读的程式变量，该变量会是个当前正在追踪射线的镜像：


{% highlight c++%}
rtDeclareVariable( optix::Ray,ray,rtCurrentRay,);
{% endhighlight %}

<<>>通过内建语音形式定义的变量只会在射线追踪的内核运行期存在，而且不能被C端修改或者查询。不同与普通的变量，一些用语义形式定义的变量可以被G端程式修改。
<<>>定义变量的时候关联一个只读的注解（annotation）可以被C端程序翻译成人类可读的变量描述字符串。

{% highlight c++%}
rtDeclareVariable( float, shininess, , “The shininess of the sphere” );
{% endhighlight %}

<<>>变量注解是rtDeclareVariable函数的第四个参数，紧接着变量的语音形式可选参数。C端程序可以通过rtVariableGetAnnotation函数查询变量的注释。

### 4.1.3 内部提供的语义形式

{% highlight c++%}
{% endhighlight %}

{% highlight c++%}
{% endhighlight %}

{% highlight c++%}
{% endhighlight %}
### 4.8.2 报告相交
{% highlight c++%}
{% endhighlight %}
<<>>射线在遍历的过程中，当其与几何体的图元相交的时候就需要调用相交程式（译注：这里应该是射线与几何体的包围盒相交的时候就会调用相交程式）。相交程式的职责就是计算射线与图元是否真实相交，并且向后续调用提供相交的t值（t-value)。另外，相交程式也有责任计算相交处的一些细节，比如表面法线，并通过属性变量（attribute variables）提供给后续调用。
<<>>一旦相交程式找到了t值，它必须通过调用OptiX的两个函数从而向后续步骤报告，函数是：rtProtectialIntersection与rtReportIntersection

{% highlight c++%}
__device__ bool rtPotentialIntersection(float t);
__device__ bool rtReportIntersection(unsigned int material);
{% endhighlight %}

rtPotentialIntersection需要相交的t值作为参数。如果在当前的遍历中t值能够成为潜在的最近碰撞点的话，函数就会缩小t区间（t-interval）并且返回true。如果t值在区间外函数就会返回false，随后相交程式就会结束（译注：官方在这里的表达过于具体化，调用了rtPotentialIntersection后返回false，用户在程序里面不返回继续做些其他什么逻辑也是可以的）。
<<>>如果rtPotentialIntersection返回了true，相交程式就可以设置属性变量（attribute variables）的值并且调用rtReportIntersection函数。rtReportIntersection函数需要给传入一个代表材质索引的unsigned int值（材质必须与最近碰撞与任意碰撞程式关联）。材质索引可以用来支持单个几何体的不同材质需求。管线马上调用对应的任意碰撞程式。如果任意碰撞程式视本次碰撞为无效可调用rtIgnoreIntersection函数，如此一来rtReportIntercetion就会返回false，相反的话就是true。
<<>>属性变量值的修改必须在rtPotentialIntersection与rtReportIntersection函数调用之间。在之外对属性变量的修改被视为无定义的（译注：但不报错）。通过这种方式修改的属性变量，其值可以在任意与最近碰撞程式中用到。
<<>>如果任意碰撞程式调用了rtIgnoreIntersection函数，那么任何属性变量的值（包括t值）将会被还原成此前的值，如果当前射线与图元没有相交，相交程式只需要返回即可。

### 4.8.3 包围盒的说明
<<>>加速体使用包围盒来包围场景中的图元以便加速射线遍历的性能。包围盒程式的主要职责是描述给定图元（第一个参数）的aabb的最小三维尺度，并将该尺寸保存在第二个参数里面。包围盒总是在对象空间下计算的，所以用户不应该对其有变换的操作。
为了得到正确的结果，包围盒必须仅仅包含图元。为了最好的性能，包围盒应该尽可能的紧凑。

### 4.8.4 相交程式与包围盒程式示例
<<>>下面的代码展示了如何将相交程式与包围盒程式联合起来描述几何图元。球体是一个简单的图形，它有着大家非常熟悉的相交算法。在下面的示例中，float4 变量sphere代表了中心与半径。

{% highlight c++%}
rtDeclareVariable( float4, sphere, , );
rtDeclareVariable( optix::Ray, ray, rtCurrentRay, );
rtDeclareVariable( float3, normal, attribute normal);

RT_PROGRAM void intersect_sphere( int prim_index ) {
	float3 center = make_float3( sphere.x, sphere.y, sphere.z );
	float radius = sphere.w;
	float3 O = ray.origin - center;
	float b = dot( O, ray.direction );
	float c = dot( O, O ) - radius*radius;
	float disc = b*b - c;
	if( disc > 0.0f ) {
		float sdisc = sqrtf( disc );
		float root1 = (-b - sdisc);
		bool check_second = true;
		if( rtPotentialIntersection( root1 ) ) {
			normal = (O + root1*D) / radius;
			if( rtReportIntersection( 0 ) )
			check_second = false;
		}
		if( check_second ) {
			float root2 = (-b + sdisc);
			if( rtPotentialIntersection( root2 ) ) {
				normal = (O + root2*D) / radius;
				rtReportIntersection( 0 );
			}
		}
	}
}
{% endhighlight %}

注意到我们在这段相交程式里面忽略了prim_index参数，它代表该几何体只有一个图元；并且将材质索引0传给了rtReportIntersection函数（译注：它代表几何实例的0号材质）。包围盒程式非常简单：

{% highlight c++%}
RT_PROGRAM void bound_sphere( int, float result[6] ) {
	float3 cen = make_float3( sphere.x, sphere.y, sphere.z );
	float3 rad = make_float3( sphere.w, sphere.w, sphere.w );
	// compute the minimal and maximal corners of
	// the axis-aligned bounding box
	float3 min = cen - rad;
	float3 max = cen + rad;
	// store results in order
	result[0] = min.x;
	result[1] = min.y;
	result[2] = min.z;
	result[3] = max.x;
	result[4] = max.y;
	result[5] = max.z;
}
{% endhighlight %}

## 4.9 选择程式











{% highlight c++%}
{% endhighlight %}


