# 1.1 Why This Book Was Written

Android Performance optimization is a very important area, and its importance is reflected in two aspects: helping applications bring greater value and helping Android developers enhance their professional competitiveness.

In terms of enhancing the value of a program, performance optimization can reduce program instability, improve user experience such as running speed and smoothness, thereby increasing customer satisfaction score, boosting user retention rate, and promoting business growth. For medium and large companies, each program has a dedicated performance quality team responsible for optimizing program performance, which fully demonstrates the importance of performance optimization in enhancing program value.&#x20;

For Android developers, proficiency in performance optimization can enhance our competitiveness and bring us better performance in the workplace. In daily work, business requirements rarely reflect the gap in individual technical capabilities. However, if we can bring more value to the business through our performance optimization skills beyond the development of business requirements, we will naturally gain more recognition. In interviews, performance optimization is also a topic that is bound to be tested. It is a manifestation of a developer's technical strength. For developers who are good at performance optimization, they can also stand out in interviews and increase the success rate of the interview.&#x20;

The importance of Performance optimization is self-evident. Therefore, there are many articles on Performance optimization on the Internet, and there are also quite a few books on Performance optimization on the market. However, most of these articles and books merely explain individual specific Performance optimization cases. After reading these cases, we only know what to do in the same scenario. But in actual development, the scenarios we face are diverse and complex, including diverse business types and various performance hardware. Therefore, in many cases, we may not know where to start because we cannot find an optimization solution for the same scenario online, or we may find that the optimization effect is poor after referring to others' solutions, or we may enthusiastically propose an optimization solution but be questioned and refuted by others and thus unable to implement it.&#x20;

To do a good job in Performance optimization, it is completely insufficient to simply learn some fragmented optimization solutions from others through blogs or other means. We need to have a solid and systematic grasp of knowledge points from multiple levels such as the hardware layer, system layer, and application layer. Therefore, I wrote this book not only to focus on explaining some specific practical cases of Performance optimization, but also to delve into the knowledge system required for Performance optimization, and strive to build a methodology for Performance optimization based on this knowledge system and experience cases, helping readers achieve thorough understanding, draw inferences from one instance, and truly master Performance optimization.

# 1.2 How to do a good job in Performance optimization

Before explaining the main content of this book, I would like to first explain how to do a good job in performance optimization. The subsequent content of this book is all centered around how to do a good job in performance optimization. Therefore, we need to have an overall understanding of how to do a good job in performance optimization so that we can learn more directionally and purposefully later.&#x20;

## 1.2.1 The essence of Performance optimization

When doing anything, if we don't understand its essence, it is difficult for us to formulate truly effective solutions. Therefore, understanding the essence of performance optimization is a very important thing. So, what is the essence of performance optimization? I believes that the essence of performance optimization is "to improve the program experience and achieve benefits by making full and reasonable use of the device's hardware resources".&#x20;

**1) Fully and reasonably utilize hardware resources**

We all know that the purpose of performance optimization is to achieve a better program experience, but how can we achieve a better experience? We can come up with many solutions without hesitation, such as using preloading, multi-threading, caching, and so on. But can these solutions really improve the program experience? The answer is uncertain. They may have a positive effect, but they may also have no effect, or even have negative effects. Why? We need to consider this issue from the perspective of hardware resources.&#x20;

If the current CPU usage of the program is already high, using preloading tasks and multi-threaded concurrent execution of tasks will undoubtedly be a counterproductive operation, bringing no optimization but rather degradation. Only when CPU usage is insufficient can we achieve better optimization results by using these optimization schemes. If the current memory usage is already high, using more caches will only cause the system to frequently trigger GC and may even lead to OOM crashes. In many cases, the reason for poor results in Performance optimization is that when formulating optimization schemes, we do not think based on the essence of Performance optimization. Only when we formulate optimization schemes based on the essence of making full and reasonable use of hardware resources can we confidently say that optimization is effective.&#x20;

Hardware resources include CPU, memory, disk, power, etc. Since the hardware resources of different devices vary, when we conduct Performance optimization based on the essence, the proposed solutions will be different from the previous ones.

* CPU: When conducting performance optimization based on how to use the CPU reasonably and fully, we need to adopt different optimization strategies for CPUs with different performance levels. For mid- and low-end devices, due to the poor CPU performance, CPU usage can easily become overloaded. At this time, we need to consider reducing CPU consumption to use CPU resources reasonably. Common strategies include reducing the number of threads, reducing and shutting down preloading tasks, etc., and allocating more CPU resources to the main thread or core scenarios. For high-end devices, with good CPU performance, our optimization strategy is often how to make full use of CPU resources. Therefore, the optimization strategy at this time is just the opposite of that for mid- and low-end devices, and we can use more threads to execute tasks concurrently and use more preloading tasks to enhance the user experience.

* Memory: When performing performance optimization based on how to use memory resources reasonably and fully, we need to set different cache sizes according to the size of memory resources. For devices with large memory, more data can be cached, while for devices with small memory, the amount of cached data should be reduced. The size of cached data should always be controlled within a reasonable range, so as to ensure that the performance of the program can be improved by fully utilizing memory resources, but without causing stability issues such as OOM (Out Of Memory) due to excessive memory usage.&#x20;

Through the two examples above, have we noticed that optimization plans formulated based on the essence are more effective?

**2) Obtain benefits**

Developing and optimizing solutions is only part of the work in Performance optimization. There is another equally important thing we need to do, which is how to achieve benefits? This requires us to do two things: one is how to define metrics, and the other is how to collect metrics.

* Develop metrics

Since we aim to achieve benefits, the first step is to establish metrics. We usually select common performance metrics to measure the effectiveness and benefits of our optimization. For example, metrics for measuring speed include startup speed, page load speed, etc.; metrics for measuring memory include PSS (the actual physical memory occupied by the program in the system), Java memory usage, Native memory usage, etc.; metrics for measuring stability include Crash rate, OOM rate, etc. However, when selecting these metrics, we need to further consider whether they can truly reflect the benefits.

For example, when performing memory optimization, we are likely to choose the PSS metric to measure the benefits of memory optimization. After a series of optimizations, we successfully reduced PSS by 100 MB. At this point, we are likely to think that our optimization has brought good benefits. However, in reality, it is uncertain whether the program's performance is better or worse after PSS is reduced by 100 MB. It is possible that because we reduced cached data, PSS decreased by 100 MB, and at this time the opening speed of some pages of the program slowed down, resulting in a degraded performance experience; it is also possible that due to the reduction of 100 MB of PSS, the OOM rate of the program decreased significantly, resulting in an improved performance experience.&#x20;

However, if we change the metrics for memory optimization from PSS value to metrics such as OOM rate and number of GC (Garbage Collector) cycles, we can clearly measure the quality of the program experience. Optimizing these metrics is the benefit of our performance optimization efforts. Therefore, when formulating performance optimization metrics, we should truly select those that can genuinely reflect the user experience and benefits of the program.&#x20;

* Collection metrics

Once we have established the metrics, the second step is to collect them. When collecting metrics, we need to ensure accuracy, consistency, and minimize the performance impact of data collection.

Accuracy is something we all understand, which means that the collected performance metrics must be accurate. So what is consistency? There are many metrics that are relatively subjective, such as page load speed. This metric involves the selection of the end point of page loading. Under what circumstances is page loading considered complete? There is no standard answer. We can consider the completion of rendering of certain key components as the end point, or the display of most of the UI as the end point, or simply the completion of the first frame rendering as the end point. We need to combine the characteristics of our own program, select a point that everyone can agree on as the end point, and always use this point as the end point in the future, without changing it after just two versions. In Performance optimization, metrics are a very important matter, so their benchmarks need to always remain consistent. If the standards are constantly changing, then the benchmarks become meaningless, and we will not be able to clearly compare whether different versions have been optimized or degraded.

During the indicator collection process, we also need to pay attention to the impact of the collection itself on performance. Many indicators require IO operations, such as memory-related data and CPU-related data. Read and write IO operations themselves are resource-intensive, so we need to control the frequency well and minimize the performance loss as much as possible.

## 1.2.2 Dimensions of Performance Optimization

Having understood the essence of performance optimization, we are already able to design effective optimization solutions, but this does not mean that we can design more systematic optimization solutions. After all, performance optimization is a vast topic. At this point, we may have some fragmented ideas, but our solutions cannot be systematic and comprehensive. Only a systematic and comprehensive optimization solution can achieve the best optimization results and bring the greatest improvement to the program experience. As shown in Figure 1-1, to create a systematic optimization solution, we still need to proceed based on the three dimensions of application layer, system layer, and hardware layer.&#x20;

1. **Application layer**

The application layer mainly refers to the programs we develop, and optimization targeting the application layer is the most common type of optimization carried out by developers. When performing optimization, we usually understand the business logic and then optimize through means such as multi-threading, preloading, and caching. However, just doing this will result in very limited optimization effects. We need to think based on the essence of performance optimization, that is, how to fully and reasonably utilize hardware resources. Therefore, optimization targeting the application layer usually has two directions: one is how to enable the business to make more full use of hardware resources such as CPU and cache to enhance the experience; the other is how to manage and control the business side to use these resources reasonably.&#x20;

Therefore, to achieve better results, we not only need to understand and be familiar with each business logic, but also clearly know the resource consumption of each business, such as how much memory resources each business consumes, how much CPU resources each business consumes, how many threads each business uses, etc. Only after we have thoroughly understood all these aspects can we start optimization. For large-scale applications, with numerous businesses, resource consumption is often overloaded, and our optimization plan mainly focuses on how to allocate and manage the resource usage of businesses. Therefore, we can use frameworks such as the startup framework, preloading framework, and degradation framework to constrain and manage the resource usage of business parties. For small and medium-sized applications, with fewer businesses, resource consumption is often insufficient, so we can use more preloading tasks, multi-threading, and other solutions to improve resource utilization.&#x20;

* **System Layer**

The system layer refers to the Android system and Linux system, and optimizing the system layer is much more difficult than optimizing the application layer. This is because optimizing the system layer requires familiarity with system knowledge, and understanding the principles and characteristics of the Android and Linux systems is far more complex than understanding the business logic of our own programs. Secondly, since we cannot directly control the logic of the system layer, we often need some complex techniques, such as Native Hook technology, to modify some system logic in order to achieve the goal of optimization.&#x20;

Optimization at the system level usually focuses on reducing resource consumption caused by the system itself. For example, when the system is performing GC, frequent thread switching, frequent page faults, and page replacement all consume a large amount of CPU resources. Therefore, our optimization strategy is to reduce the resource consumption of these system logics or mitigate their impact on applications during execution.

Although optimization at the system level is much more complex than that at the application level, fortunately, optimizations at the system level are generally universal, so we can fully draw on and reuse some solutions, such as performing GC suppression during startup and designing a reasonable thread pool.&#x20;

* **Hardware layer**

Optimization for the hardware layer mainly involves understanding the characteristics of the hardware and then identifying optimization points. Most optimization solutions for the hardware layer are targeted at the two hardware features of CPU and cache.

For example, if the CPU consists of big and small cores, with the big core having a high operating frequency and the small core having a low operating frequency, we can run the main thread on the big core; if the manufacturer provides a corresponding overclocking API interface, we can also increase the CPU frequency in core scenarios to improve performance.&#x20;

The architecture of the cache consists of multi-level caches, memory, and disks. When optimizing this hardware, we can consider how to improve the hit rate of the cache and the hit rate of the memory cache.&#x20;

## 1.2.3 Difficulties in Performance Optimization

By now, we already know the essence and dimensions of doing well in performance optimization. However, at this point, we only know where the road ahead lies. To reach the destination, we still need to keep moving forward and overcome the numerous obstacles encountered along the way. The obstacles on this road, which are also the difficulties of performance optimization, are mainly reflected in these four aspects: knowledge reserve, perspective of thinking, way of thinking, and a complete optimization closed-loop.&#x20;

1. **Knowledge reserve**

To do a good job in performance optimization, the first difficulty is the need to master a complete and systematic knowledge. As we learned earlier, performance optimization should be carried out from three dimensions: application layer, system layer, and hardware layer, which means we also need to solidly master the knowledge points of these three levels.

* Application layer

The more we want to achieve good results in performance optimization for the application layer, the more we need to have a better understanding of the applications we develop. We need to know which threads our responsible APP has, what they do, which business uses them, and the CPU consumption of these threads; how much memory is occupied, which business occupies it, and what the cache hit rate is; what tasks are performed during the startup process and the opening process of core pages, how long the IO blockage takes, how long the logic takes, and what CPU usage is.&#x20;

* System Layer

Compared to the knowledge points at the application layer, those at the system layer are even more extensive and complex. For example, Linux knowledge includes process management and scheduling, memory management, virtual memory, locks, IPC communication, etc.; Android system knowledge includes virtual machines, core services such as AMS (ActivityManagerService), WMS (WindowManagerService), etc., rendering, and some core processes such as startup, opening an Activity, installation, and so on.

Performance optimization at the system level must be based on our understanding of the system's mechanisms and processes. If we do not understand the process scheduling mechanism of the Linux system, we cannot fully utilize process priority to help us improve performance; if we are not familiar with the Android virtual machine, then some related optimizations around this virtual machine, such as OOM optimization or GC optimization, cannot be well carried out and implemented.

* Hardware layer

For the hardware layer, we need to be familiar with the characteristics of hardware such as CPU,  cache, etc.  If we know how many cores a CPU consists of, which are big cores, and which are small cores, we will naturally think of whether we can improve performance by binding core threads to big cores; if we understand the design of registers, caches, and main memory in the storage structure, we will naturally consider whether we can improve performance based on this characteristic, such as placing core data in the cache as much as possible to improve performance.&#x20;

* Other

In addition to the knowledge in the three aspects mentioned above, if we want to make further progress in performance optimization, we also need to master more knowledge, such as assembly, compilers, programming languages, reverse engineering, etc. For example, writing code in C++ runs faster than in Java, and we can improve performance by replacing some business logic with C++; for example, by optimizing compiler inlining, dead code elimination, and other solutions to reduce package size; for example, by using reverse engineering techniques to optimize system logic and make the program perform better.&#x20;

It can be seen that mastering performance optimization requires a vast knowledge reserve, so performance optimization is highly indicative of a developer's technical depth and breadth. Whether in interviews or at work, if we are proficient in performance optimization, it will surely add significant value to us.&#x20;

* **Angle of thinking**

When dealing with a complex and large-scale matter, we should first perform layering and classification. Different layers and categories represent different perspectives. For example, when designing the architecture of a client program, a layered architecture is usually adopted, where the logic and responsibilities of different layers vary; when conducting performance optimization of a system, we can optimize it from the application layer, system layer, and hardware layer respectively, and the optimization schemes for each layer also differ.&#x20;

In addition to thinking from different perspectives inherent in the matter itself, we can also obtain more inspiration by stepping outside the matter itself. For example, by thinking outside the current device, we can consider whether other devices can help us accelerate startup. Google Play (Google's app store) has similar optimizations. Google Play uploads some machine code that has already been compiled on other devices, and when the same device downloads this app, it also downloads this compiled machine code. Another commonly used technique is server-side rendering, which allows the server to pre-render the interface and then directly load static modules to improve page loading speed. Or, from the user's perspective, we can think about what optimizations are beneficial to the user's perception. For example, sometimes when optimizing startup and page loading speed, we give the user a fake static page to make them feel that the page has already opened, and then bind the real data.&#x20;

When performing performance optimization, the more comprehensive the perspectives considered, the more effective and numerous our optimization solutions will be.&#x20;

* **Way of thinking**

When dealing with complex matters, we need to divide them into different perspectives. With more perspectives, there will be more solutions. But how should we divide these perspectives? What if we just can't come up with them? In fact, all these are related to our way of thinking. Through different ways of thinking, we can obtain different perspectives. The most common ones are top-down and bottom-up thinking.&#x20;

* Top-down

The top-down thinking approach is a way of thinking that gradually breaks down from the whole to the parts and tackles each part individually. In the performance optimization of large-scale applications, top-down is a very common approach. Large-scale applications have a large number of business operations, and the teams for different business operations vary. Therefore, when we conduct performance optimization, we need to consider from the overall perspective how to manage and allocate the consumption and use of resources by business operations. We can design some global frameworks to manage the use of resources by business operations, such as preloading frameworks and degradation frameworks; design some global monitoring mechanisms to measure the use of resources by business operations. After we have overall management and monitoring, we can then move on to the local perspective and optimize each business operation that consumes a large amount of resources one by one.&#x20;

* Bottom-up

The bottom-up thinking approach is just the opposite, starting from details or underlying principles and gradually building up the overall solution step by step. For example, when we perform performance optimization, we directly consider factors such as the CPU and cache that affect speed, think about how to improve CPU utilization and cache hit rate, and conduct in-depth analysis from the hardware layer, system layer, and application layer bottom-up to construct a comprehensive optimization plan.&#x20;

Different ways of thinking will ultimately lead us to design different optimization solutions, but it doesn't mean that one way of thinking is superior to the other. When facing complex problems, we need to try using different ways of thinking to think about and solve the problem.&#x20;

* **Optimized process**

In the actual process of Performance optimization, how to optimize is only one part of it. We also need to do more, as shown in Figure 1-2, including monitoring, optimization, data benefit acquisition, and anti-deterioration. These parts form a closed loop to constitute a complete optimization process. During Performance optimization, we need to consider and do well in all aspects.&#x20;

* Monitoring: That is, monitoring various performance indicators during the application's operation. To do a good job in monitoring, in addition to minimizing the performance overhead caused by monitoring logic, it is also necessary to monitor the root causes as much as possible. For example, in memory monitoring, besides monitoring the memory indicator data of the application, it should also be able to monitor the memory usage proportion of each business, as well as root cause items such as large collections, large images, and large objects. In this way, we can directly pinpoint the problem through monitoring. A complete and excellent monitoring solution enables us to more efficiently detect and resolve anomalies.

* Optimization: Many developers may think that performance optimization is simply optimization, but in fact, optimization is only one aspect of performance optimization, not the entirety of it. The reason we have such a misunderstanding is often because we lack a systematic understanding and awareness of performance optimization.&#x20;

* Data Benefit Acquisition: This stage is not simply about observing changes in indicators; we need to learn how to conduct A/B tests and focus on indicators of core value. For example, when performing memory optimization, we cannot blindly pursue reducing the PSS occupancy of application memory. The amount of memory occupancy does not necessarily represent the real User Experience. Therefore, when optimizing memory, we should preferably combine indicators that reflect the core value of the experience, such as memory peak hit rate, crash rate, user retention, etc., to obtain the benefits and value of memory optimization.

* Anti-deterioration: There are also many things that can be done for anti-deterioration, such as establishing a comprehensive offline performance test and online monitoring alarms. Taking memory optimization as an example again, we can run memory leak tests daily offline through Monkey and address them in advance, which is anti-deterioration work.

# 1.3 Features and Main Content of This Book

In the previous section, I explained the essence, dimensions, and difficulties of performance optimization. The characteristic of this book is to help readers overcome the difficulties in the process of performance optimization, and based on the essence of performance optimization, carry out practical optimization from multiple dimensions. Therefore, this book will systematically explain the knowledge system required for each performance optimization direction, which covers from the hardware layer, operating system layer to the application layer. Based on this knowledge reserve, I will also construct the methodology for optimizing this performance direction, which is a profound manifestation of our ability to have diverse thinking perspectives, complete thinking methods, and a complete optimization process. Finally, I will strengthen our consolidation of the knowledge system and mastery of the methodology through specific optimization cases from multiple dimensions such as the application layer, system layer, and hardware layer in the monitoring, optimization, and anti-deterioration processes.&#x20;

The content of this book includes comprehensive Performance optimization topics such as memory optimization, speed and fluency optimization, stability optimization, package size optimization, as well as power consumption optimization, disk usage, and traffic optimization. Most of these topics are divided into a theory section and a practical section. The theory section mainly explains the basic knowledge, while the practical section, based on the basic knowledge, further explains optimization cases and the technologies and principles used in these cases.&#x20;

The knowledge points related to the Android system in this book are mainly explained based on Android 14 However, when performing performance optimization, considering compatibility, we often need to base it on each system version. Therefore, this book will also cover the explanation of source code for system versions other than Android 14. For the cases explained through sample programs in this book, source code will mostly be provided, and details of the source code can be found in the following link.&#x20;

> <https://github.com/helsonzhao/android\_performance>

The content of this book is the my personal thoughts and summaries based on past work experience and excellent industry practices. However, technology is boundless, and individual capabilities are relatively limited. Therefore, errors are inevitable in the text, and we hope readers will kindly forgive and offer their advice.&#x20;

# 1.4 Target Audience of This Book

The content of this book includes both basic knowledge systems and in-depth practical cases, covering from the Java layer to the Native layer. Therefore, the content of this book is suitable for readers at all stages, including the following readers.&#x20;

* Developers with extensive Android development experience and a desire to further break through Android technology&#x20;

* Android developers with some work experience, having a certain understanding of performance optimization but wishing to study it more deeply&#x20;

* New Android developers who have just started working and want to systematically learn the basic knowledge of Android

# 1.5 How to Read This Book

Different readers can choose to read different chapters according to their own fields and levels.&#x20;

For new employees who have just started working, it is recommended to read through the knowledge principle section of this book. These knowledge points can help Android newcomers build a systematic knowledge. With this knowledge, they can integrate into daily work and development more quickly. As for the practical chapters, they can be gradually read and practiced during subsequent development work.&#x20;

For readers with some work experience or those who are already engaged in performance optimization work, it is recommended to read the entire book thoroughly to comprehensively understand the performance optimization of Android in various directions and the knowledge points covered by these optimizations. There are quite a few optimization cases in this book that use relatively complex technologies, such as Native Hook technology, Bytecode instrumentation, etc. These technologies are all knowledge points that need to be mastered for Android advancement and can help readers take their skills to the next level in the Android field.

For experienced Android developers, they can selectively read the chapters on practical projects in this book. Many optimization cases in this book are quite novel. Through these cases, readers can also gain some new ideas and inspire more inspiration in their subsequent work.&#x20;
