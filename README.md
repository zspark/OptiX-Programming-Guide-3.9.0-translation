对OptiX编程指南3.9版本的中文翻译

**翻译全部转移到[博客][1]，这里不再做更新**

将下列术语暂歇统一翻译：
semantics：语义
program：程式（专门指需要上载到G端的程序）

* 没有翻译3.6节 Rendering on the VCA
* 没有翻译4.10节 Callable programs

4.1节中的The nvcc compiler that is distributed with the CUDA SDK is used to
create PTX in conjunction with the OptiX header files.应该被翻译为：发布于CUDA SDK的nvcc编译器结合OptiX的头文件用来创建PTX文件。还是应该为：发布于CUDA SDK的nvcc编译器用来创建PTX文件且处理OptiX头文件？

翻译中的疑问：
>> 3.1.3: Within an OptiX program, the rtPrintf function works similarly to C's
printf. Each invocation of rtPrintf will be atomically deposited into the
print output buffer, but separate invocations by the same thread or by different
threads will be interleaved arbitrarily.

多线程的输出交错能理解，单线程的多次调用怎么还会任意交错输出？

[1]:http://www.zspark.net/post/cg/OptiX-Programming-Guide-3.9.0-translation.html
