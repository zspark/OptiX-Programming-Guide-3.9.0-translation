对OptiX编程指南3.9版本的中文翻译

翻译中的疑问：
>> 3.1.3: Within an OptiX program, the rtPrintf function works similarly to C's
printf. Each invocation of rtPrintf will be atomically deposited into the
print output buffer, but separate invocations by the same thread or by different
threads will be interleaved arbitrarily.

多线程的输出交错能理解，单线程的多次调用怎么还会任意交错输出？
