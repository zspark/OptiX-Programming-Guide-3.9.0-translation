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







{% highlight c++%}
{% endhighlight %}





