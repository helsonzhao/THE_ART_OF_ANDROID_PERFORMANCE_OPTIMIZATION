The value of optimizing speed and fluency is obvious. In terms of optimizing this area, there are many things we need to do, such as optimizing startup speed, page opening speed, business loading speed, UI rendering fluency, List component scrolling fluency, and so on. Precisely because there are so many optimization points involved, including various pages, various business operations, and various components, our optimization efforts are likely to be fragmented and difficult to form a systematic approach, so the results will not be very good.&#x20;

Actually, we can look at speed and fluency optimization from a different perspective. For startup speed optimization, it actually means how to execute all the code instructions from the attachBaseContext lifecycle start to the display of the main page as quickly as possible; for page opening speed optimization, it actually means how to execute all the code instructions from startActivity to the display of the Activity interface as quickly as possible; for page rendering or component sliding fluency, it actually means how to execute all the instructions from the current frame to the next frame within 16 milliseconds (60 frames per second refresh rate) as quickly as possible. From these examples, we can see that all relevant optimizations related to speed and fluency can be converted into the same optimization: that is, how to execute the code instructions from point A to point B as quickly as possible. Based on this optimization, it will be easier and more systematic to build optimization solutions.

If you want to execute the instruction from point A to point B as quickly as possible, then the CPU, cache, and task scheduling are the most fundamental factors. Next, we will delve into the principles to understand how these three factors affect the speed and smoothness of the program.&#x20;

# 4.1 CPU

The impact of CPU on the speed and fluency of the program is huge. It is thanks to the increasingly powerful CPU that the performance of mobile phones is getting better and better, and the programs that can be run are becoming more and more complex. In this section, we will understand how the CPU works so that we can find optimization points to improve the speed and fluency of the program.

## 4.1.1 Structure of CPU

The CPU is mainly composed of an arithmetic unit, a Control Unit, and a storage unit. The storage unit contains registers and cache, and its structure is shown in Figure 4-1.&#x20;

![Figure 4-1 CPU Structure ](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter4_img_1.png)

The corresponding functions of each module are shown in the following table.&#x20;

| Module                | Function Description                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Arithmetic Logic Unit | Used to perform arithmetic and logical operations, such as addition, subtraction, AND, OR, etc.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| Control Unit          | The Control Unit is responsible for coordinating and controlling the internal operations of the CPU. It is mainly composed of modules such as the Instruction Register, Instruction Counter, Instruction Decoder, and Operation Controller. The Control Unit fetches instructions from the cache and places them in the Instruction Register, and the Instruction Decoder converts the instructions into internal operands or control signals. Finally, the Operation Controller sends out control signals to drive modules such as the Arithmetic Logic Unit, Registers, and Cache to complete the execution of instructions.  |
| Cache                 | The existence of cache is to address the speed difference between the CPU and main memory. The access speed of main memory is relatively slow, while the CPU needs to frequently read and write data when executing instructions. By adding cache inside the CPU, frequently accessed data can be temporarily stored closer to the CPU to speed up data access. This way, the CPU can obtain the required data more quickly without having to read from main memory every time.                                                                                                                                                 |
| Registers             | Registers are used to temporarily store instructions, data, intermediate results, etc. Their access speed is much faster than that of cache, but their capacity is very small.                                                                                                                                                                                                                                                                                                                                                                                                                                                  |

## 4.1.2 CPU Workflow

The CPU's job is to execute program instructions, and this process can be roughly divided into the following steps:&#x20;

1. Read Instruction: The Control Unit accesses the memory address of the next instruction to be executed and loads the instruction read from that address into the Instruction Register.&#x20;

2. Decoding Instruction: The Control Unit parses the instruction in the Instruction Register to determine the type and parameters of the instruction&#x20;

3. Execution Instruction: The Control Unit executes corresponding operations based on the type and parameters of the instruction, such as arithmetic operations, logical operations, data movement, control transfer, etc. For operations like arithmetic and logical operations, they are executed through the Arithmetic Logic Unit.&#x20;

4. Result Write-Back: The Control Unit writes the result of instruction execution into the register or cache, and the program counter is updated to the address of the next instruction to be executed

Through the above four steps, an instruction completes its execution. The CPU will continuously repeat this process at a fixed frequency until the program ends or an exception occurs. From Figure 4-2, we can more intuitively understand the CPU's workflow.

![Figure 4-2 CPU Execution Process](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter4_img_2.png)

The faster the CPU executes instructions, the faster the program runs. The speed of this process is mainly affected by the following aspects:&#x20;

* CPU operating frequency: This refers to the number of clock cycles that a CPU can execute per second, usually expressed in Hz. The higher the operating frequency, the faster the CPU can execute instructions. As shown in Figure 4-2, which depicts the operating frequencies of Qualcomm Snapdragon series CPUs, it can be seen that the more advanced the CPU, the higher its operating frequency.

  ![Figure 4-2 Operating Frequency of Snapdragon Series CPUs](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter4_img_3.png)

* CPU Cache: The cache is used to temporarily store data and instructions frequently accessed by the CPU, thereby reducing the data exchange time between the CPU and memory. The larger the cache capacity, the better the performance of the CPU in accessing instructions and data.

* Number of CPU cores: The number of cores represents the quantity of independent computing units within the CPU. The more cores there are, the stronger the CPU's parallel processing capability for executing instructions. Currently, the CPUs of mainstream mobile phones generally have 8 cores.

* CPU Instruction Set: Different instruction sets have different complexities and functions. Under the RISC architecture, instructions are fewer and simpler, while under the CISC architecture, instructions are more numerous and complex. Therefore, RISC instructions execute faster and consume less power, but CISC offers more powerful functionality and better compatibility.

## 4.1.3 Assembly Instructions

Having understood the structure of the CPU and the execution flow of instructions, we will now proceed to understand the assembly instructions under the ARM architecture. Understanding assembly instructions not only allows us to gain a deeper understanding of the underlying principles of computers, but also helps us better understand the implementation principles of technologies such as Native Hook.&#x20;

In the ARM platform, the format of an assembly instruction is as follows, where the opcode is the mnemonic for the instruction operation, the destination register represents the register that stores the result after the instruction operation, and the operand is used to perform the operation. Here, we mainly look at the opcode and operand.&#x20;

```c++
<opcode> <destination register>, <operand1>, <operand2>
```

* Opcode: The number of opcodes is very large, but we don't need to memorize them by rote. We only need to be able to master some of the most commonly used operations based on the types and functions of these opcodes. The common opcodes are shown in the following table.&#x20;

  | **Type**          | **Function**                                              | **Opcode**                                                                                                                                                                                                                                                                                                                |
  | ----------------- | --------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
  | Jump              | Used to control program jumps                             | B: Unconditionally jump the program to a specified address<br />BL: Similar to the B instruction, but saves the return address to the Link Register (LR) before jumping. This way, after the jump is completed, the program can continue executing the next instruction by returning to the address in the Link Register. |
  | Data Processing   | Used to perform various arithmetic and logical operations | ADD: Adds the values of two operands and stores the result in the destination register<br />SUB: Subtract the value of another operand from the operand and store the result in the destination register                                                                                                                  |
  | Data Transmission | is used to transfer data between registers and memory     | LDR: Load data from memory into a register<br />STR: Store the data in the register into memory<br />MOV: Copies the value of one register to another register                                                                                                                                                            |

* operand

  Operands can be immediate values (i.e., values that appear directly in the instruction and do not need to be loaded from other locations), registers, memory addresses, etc. For example, in the plt table of the malloc function in the previous chapter, the instruction "add ip, ip, #12288" appears, where "add" represents the addition opcode, the following "ip" is the register where the result of the addition is written, the second "ip" is the first operand, which is a register operand, and "#12288" is the second operand, which is an immediate value. Therefore, this instruction means adding the value of the ip register to the decimal number 12288, and then writing the result into the ip register.

# 4.2 Cache

Having learned about CPU knowledge, let's take a look at how cache affects the performance of programs.&#x20;

## 4.2.1 Structure of the Cache

The storage devices of mobile phones or computers are organized into a memory hierarchy. In this hierarchy, from top to bottom, the access speed of the devices becomes slower, but the capacity becomes larger and the price becomes cheaper. As shown in Figure 4-3, it presents the general data cache structure and its access time.&#x20;

![Figure 4-3 Access Time of Different Cache Types](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter4_img_4.png)

As can be seen in the structure diagram, the register has the fastest access speed, typically one clock cycle. In the diagram, one clock cycle is represented as 0.5ns, and this value varies under different CPUs. For example, the clock cycle of the large core of the Snapdragon 8 Gen 2 is 0.3ns. The speed of L1 is generally several clock cycles, which is also related to the design of L1. The access speed of each level of cache is several to dozens of times faster than that of the previous level. Therefore, we should try to place data in the top-level cache container to ensure better performance of the program.&#x20;

## 4.2.2 Register

Here, I mainly introduce the registers under the ARM32-bit CPU architecture. There are a total of 16 registers in this architecture, numbered from R0 to R15, and their functions are shown in the following table.&#x20;

| Register Number | Name                 | Function                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| --------------- | -------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| R0 to R12       | General Register     | They can be used for any purpose, such as arithmetic operations, logical operations, data movement, string manipulation, etc.                                                                                                                                                                                                                                                                                                                             |
| R13             | Stack Pointer (SP)   | It is used to point to the current stack top address and is used to store local variables, intermediate results, function parameters, return addresses, etc.                                                                                                                                                                                                                                                                                              |
| R14             | Link Register (LR)   | is used to save the return address of a function or an exception, and is used to implement function calls or exception returns.                                                                                                                                                                                                                                                                                                                           |
| R15             | Program Counter (PC) | After an instruction is executed, the PC register will automatically update to the address of the next instruction, ensuring the continuous execution of instructions. Due to the design of the pipelined processor architecture, the value of PC points to the address of the instruction two steps ahead of the current instruction. In ARM32, one address is 4 bytes, so the value of PC is the address value of the current instruction plus 8 bytes. |

Under the ARM32 architecture, the number of registers is only 16, while under the ARM64 architecture, there are 33 registers. When the number of registers increases, the execution speed of the program will naturally become faster.

## 4.2.3 Cache

Cache is typically designed within the CPU and divided into multiple levels, where L1 cache is closest to the CPU, with the fastest speed but the smallest capacity, while L2 and L3 caches have larger capacities but slower speeds. In the Snapdragon 8 Gen 2 chip, the L1 cache of the large core is 128KB, the L2 cache is 1MB, and the L3 cache is 8MB in size.

When the CPU needs to access a certain address in memory, it first checks whether the corresponding data exists in the cache. If it does, it reads directly from the cache; if not, it reads the data from memory and copies it to the cache. When reading data from memory into the cache, in addition to reading the target data, the data surrounding the target data is also read into the cache. The total size of the data read is typically 32 bytes under a 32-bit CPU and 64 bytes under a 64-bit CPU. This size of 32 bytes or 64 bytes is the basic unit of the cache, also known as the Cache Line.&#x20;

The cache is divided into several cache lines according to its own size. Taking the Snapdragon 8 Gen 2 chip as an example, since it is a 64-bit architecture chip, its L1 cache has 2000 (128000 ÷ 64) cache lines, as shown in Figure 4-4.&#x20;

![Figure 4-4 Number of L1 Cache Lines](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter4_img_5.png)

## 4.2.4 Main Memory

Main memory, also known as physical memory, although its speed is not as fast as cache, is still much faster than disk. Therefore, loading data from disk into main memory as much as possible is one of the most effective ways to improve performance.

Although the main memory of mainstream mobile devices is relatively large nowadays, this does not mean that we can load all data into the main memory without restraint in order to improve the speed of the program. Because if the physical memory usage is high, in addition to the program crashing easily due to insufficient memory, the operating system will also frequently perform page replacement operations due to insufficient physical memory. During page replacement operations, the kswapd daemon will select the least used pages and move them to the disk to provide Memory Space for new data, and the process of frequent page replacement will consume more CPU resources, thus affecting the speed and fluency of the program. Therefore, how to optimize the cache design of the main memory and how to use the main memory more reasonably are important directions for improving program performance.&#x20;

# 4.3 Task Scheduling

There are many tasks running on the operating system, but CPU resources are limited. Which tasks can obtain CPU resources to execute instructions and which tasks have to wait completely depend on the operating system's task scheduling mechanism. Therefore, we need to have a deeper understanding of the operating system's task scheduling mechanism so that tasks in our programs can have more opportunities to be scheduled and executed, thereby achieving faster speed and better fluency.

## 4.3.1 Process and Thread States

A process is an instance of a running program in the system, with independent resources and execution environment. A process consists of one or more threads, which can execute concurrently, sharing the process's resources. Threads are also the smallest units for task execution in the system. In the Linux system, a thread is actually a process, but a lightweight one. The scheduling algorithms for lightweight processes and ordinary processes are the same, with the only difference being that lightweight processes can share the logical address space and system resources with other processes. As we already know from the previous chapter, when creating a thread via new Thread, it actually calls the Thread::CreateNativeThread method at the Native layer, which internally calls the pthread\_create function provided by the Linux system to create a lightweight process.&#x20;

Having understood processes and threads, let's take a look at their states during the running process. The following are some of the most commonly encountered states:&#x20;

* Running State (Running, R): A process or thread is executing or ready to execute on the CPU.

* Ready state (Runnable, R): The process or thread is ready to run but is waiting for a CPU time slice.&#x20;

* Interruptible Sleep State (Interruptible Sleep, S): A process or thread is waiting for a condition, such as the completion of an I/O operation, and can be awakened by a signal.

* Uninterruptible Sleep State (Uninterruptible Sleep, D): Similar to the interruptible sleep state, but the process or thread cannot be awakened by signals and can only wait for the occurrence of specific events&#x20;

* Stopped State (T): The process or thread has stopped executing&#x20;

Through Figure 4-5, we can more clearly understand the state relationships of processes or threads and their switching conditions.

![Figure 4-5 Thread States and Transitions](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter4_img_6.png)

The state of a thread is very helpful when we analyze trace files. By analyzing the thread state in trace information, we can achieve the following optimizations.&#x20;

* Identify performance bottlenecks: threads are frequently in the ready state, indicating possible performance issues such as CPU resource constraints

* Concurrent issue detected: The thread frequently switches between running and sleeping states, potentially leading to lock contention or other synchronization issues.

* Troubleshoot IO issues: If a thread remains in an uninterruptible sleep state for an extended period, there may be an issue with IO latency&#x20;

* Optimize task scheduling: Analyzing thread states can help optimize thread priorities and reduce the overhead of context switching.&#x20;

## 4.3.2 Process Scheduling

When the system's scheduling manager schedules processes, there will be a certain amount of performance loss, which is mainly reflected in the following aspects:&#x20;

* Context Switching: When a thread needs to switch to another thread, the CPU needs to save the context information of the current thread, including the state of registers, stack pointer, etc. Then, the CPU needs to load the context information of the next thread so that it can continue execution. This process consumes a certain amount of resources, especially when the number of threads increases, the overhead of context switching will be greater.

* Cache invalidation: Each thread has its own working set, which contains the data and instructions it needs. When a thread is switched out, the CPU needs to load the working set of the next thread into the cache, which may cause the previous cache contents to become invalid. Cache invalidation leads to additional memory accesses, increases access latency, and thus affects the execution efficiency of the thread.

* Scheduling Overhead: Thread switching is managed by the operating system's scheduler. The scheduler needs to maintain the state of threads and scheduling queues to determine the next thread to execute. These scheduling overheads include the overhead of scheduling algorithms, the overhead of maintaining thread queues, etc.&#x20;

* Synchronization Mechanism: During the process of thread switching, the synchronization issues between threads also need to be considered. If one thread is accessing a shared resource while another thread needs to wait for the release of that resource, synchronization operations between threads are required. This may involve the use of synchronization mechanisms such as locks, semaphores, and condition variables, and these mechanisms themselves also introduce a certain amount of overhead.&#x20;

Therefore, in order to reduce the losses caused by scheduling, we need to have a certain understanding of the system's scheduling mechanism. The Linux system divides processes into two categories: real-time processes and ordinary processes. Real-time processes can respond quickly and timely to events, such as audio playback processes and camera video capture processes, which need to ensure real-time and continuity. In contrast, ordinary processes do not guarantee immediate response to events. The functions of the two types of processes are different, and their scheduling rules are also different.&#x20;

**1) Real-time process**

First, there are the scheduling rules for real-time processes. The Linux system has two scheduling policies for real-time processes: First-In-First-Out (SCHED\_FIFO) and Round Robin (SCHED\_RR). However, Android only uses the SCHED\_FIFO policy. Under this scheduling policy, if a real-time process occupies the CPU, it will continuously monopolize the CPU resources until the process voluntarily releases them. For example, when an audio playback thread is playing audio, it will always monopolize a CPU core and will not be switched to other threads by the scheduling manager at this time, thus ensuring that the memory of the played audio is continuous. Since the monopolization of the CPU by real-time processes may cause other processes to be unable to obtain CPU resources, the number of real-time processes in the Android system is limited. Only some core system processes with high real-time requirements are real-time processes, and for application processes, they cannot be set as real-time processes.

**2) Non-real-time process**

Non-real-time processes, also known as ordinary processes, are scheduled by the Linux system using a scheduling algorithm called Completely Fair Scheduler (CFS) to implement process switching and scheduling. In the CFS algorithm, the priority of a process is represented by the nice value, where a lower nice value indicates a higher priority. However, the scheduler does not directly use the nice value as the priority for task scheduling; instead, it selects the process with the least running time among all executable processes to execute. It should be noted that the running time here is not the actual physical running time, but the virtual running time calculated with weighting based on the process priority, i.e., the nice value. Under this mechanism, the virtual running time of a high-priority process running for 10ms may be the same as that of a low-priority process running for 5ms.

## 4.3.3 Coroutines and Threads

Since a thread is essentially a lightweight process, it is managed by the system's task scheduling mechanism. As it is the system's task scheduling mechanism, the performance overhead caused by thread switching is inevitable. Can we manage task scheduling and switching ourselves at the application layer to reduce performance overhead? Of course, we can implement a user-level task scheduling manager to manage tasks, and the units running on the scheduling management in user space are called coroutines. As shown in Figure 4-6, for threads, one thread represents a process in the kernel, and each thread is managed by the system's process scheduler. For coroutines, multiple coroutines correspond to one process in the kernel, and each coroutine is managed by the coroutine scheduler in user space.

![Figure 4-6 Correspondence between threads, coroutines, and processes](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter4_img_7.png)

Compared to threads, coroutines mainly have the following advantages:

* Low performance loss: Thread scheduling is performed within the kernel, so operations such as the switching between User Mode and Kernel Mode, and context switching, are relatively resource-intensive for system performance. Since the scheduling of coroutine tasks is carried out in user space, the performance overhead of task switching is far less than that of threads.

* No IO overhead: When the logic in a thread encounters IO, the process corresponding to that thread needs to wait for the IO to complete before it can continue execution. Therefore, IO tasks are a significant factor affecting speed and process flow. However, through coroutines, the current process can continue to execute other coroutine tasks, and then resume execution of the coroutine after its IO is completed. In this way, we can ensure that the process is always in a running state without waiting, thus eliminating the overhead caused by waiting for IO.&#x20;

Since coroutines have advantages, they naturally also have disadvantages, mainly in the following aspects:&#x20;

* Since coroutines run within a single process, they cannot fully leverage the parallel capabilities of multi-core CPUs.

* In CPU-intensive scenarios, since the CPU is fully loaded and there is not much loss caused by IO, coroutines do not achieve better results in this scenario.&#x20;

* Coroutines cannot have blocking operations; otherwise, it will cause the entire process to be blocked, thereby affecting the operation of other coroutines

&#x20;From the perspective of the advantages and disadvantages of coroutines, in IO-intensive scenarios, coroutines are a better choice than threads. However, in CPU-intensive scenarios, threads are the better option. In actual project scenarios, IO-intensive and CPU-intensive scenarios often coexist and intertwine, so we need both threads and coroutines to work together to achieve better performance in the program. However, readers should note that Kotlin's coroutines are not real coroutines, and each Kotlin's coroutine still corresponds to a thread(process), so Kotlin's coroutines cannot fully exert their true coroutine capabilities.

# 4.4 Methodology for Speed and Fluency Optimization

Once we understand the relevant knowledge points of the three factors that determine speed and fluency, namely CPU, cache, and task scheduling, we can then focus on the three directions of improving CPU execution efficiency, enhancing cache efficiency, and boosting task scheduling efficiency to develop a methodology for optimizing the speed and fluency of programs.&#x20;

## 4.4.1 Improve CPU Execution Efficiency

The time consumed by the CPU to execute instructions from point A to point B can be referred to as CPU execution time. Improving the CPU's execution efficiency can reduce CPU execution time. To effectively improve CPU execution efficiency, we can start with the formula "CPU execution time = number of instructions in the program x average time per instruction executed by the CPU". By reducing the number of instructions or decreasing the average time per instruction executed by the CPU, we can effectively improve CPU execution efficiency and reduce CPU execution time.&#x20;

1. **Reduce the number of program instructions**

Improving speed by reducing the number of instructions in a program is one of the most commonly used optimization methods. Here, I introduces some common solutions for improving speed by reducing the number of instructions.

* Utilize the multi-core capabilities of the CPU

  If instructions with the same logic can be assigned to multiple CPUs for simultaneous execution, then for a single CPU core, the number of instructions to be executed decreases, and the execution time of the CPU naturally decreases. How can we leverage the multi-core capabilities of the CPU? Using multi-threading is the answer. If our mobile phone CPU has 8 cores, then theoretically, it can concurrently run at least 8 threads. In real-world scenarios, the difficulty of concurrency lies in task splitting. We need to split and orchestrate tasks based on code logic and characteristics to reduce dependencies between concurrent tasks and thus achieve effective concurrency.&#x20;

* More concise code logic

  For the same functionality or logic, if implemented with more concise or optimized code, the number of instructions will also decrease, and the speed will naturally increase. We can analyze the time consumption by capturing traces or measuring the time before and after the function, and implement these time-consuming methods in a more optimized way.&#x20;

* Reduce CPU consumption unrelated to the scene

  During the program execution, there are many business operations or logics unrelated to the current scenario that consume CPU resources. These operations or logics may originate from the system layer or other parts of the program. By eliminating as much as possible the consumption of CPU resources by unrelated business operations, we can effectively reduce the number of instructions the CPU needs to execute, resulting in a better performance experience.&#x20;

* Reduce CPU idle time

  By executing preloading logic such as pre-creating interfaces and pre-preparing data when the CPU is idle, it is also an optimization solution to reduce the number of instructions, because the number of instructions for the scenarios that need to be run later is reduced due to the execution of preloading, and the running speed of the scenarios naturally becomes faster.&#x20;

* Reduce the number of instructions in the current device's program through other devices

  By breaking out of the current machine limitations, many optimization solutions can also be derived. For example, Google Play will upload the machine code of programs in certain devices, so that when other users download this program, their own devices do not need to perform compilation operations, thus improving the installation or startup speed. For example, when opening some webview pages, the server will pre-render and process all IO data, directly presenting a static page to the user, which can greatly improve the page opening speed.&#x20;

The optimization schemes mentioned above are all relatively commonly used, and the essence behind these optimization schemes is to reduce the number of instructions. Based on the methodology of reducing the number of instructions, more optimization schemes can be derived, which will not be listed one by one here. Readers can also use their imagination to think about what other schemes there are.&#x20;

* **Reduce instruction execution time**

Next, let's look at the second point: how to reduce the time taken for CPU instruction execution, which is largely affected by the operating frequency. Therefore, the most effective way to reduce the average time taken for each instruction is to increase the CPU's operating frequency. At this point, some readers may think that the optimization we can do in this area is limited, because we cannot ask users to purchase devices with better CPU models. However, based on the basic knowledge about CPU we learned earlier, we can still identify quite a few implementable optimization solutions.

We know that the architecture of a CPU is divided into big cores and small cores. Therefore, ensuring that core threads such as the main thread can always run on big cores is an optimization that increases the frequency of CPU instruction execution. In addition, modern CPUs generally can increase their operating frequency by raising the operating voltage, a mechanism known as overclocking. Currently, some mainstream mobile phone manufacturers provide API interfaces to implement overclocking capabilities, which are used in many gaming applications. In applications, we can also use the API interfaces provided by manufacturers to implement overclocking, thereby enhancing the user experience. However, overclocking results in higher power consumption and heat generation, so it should be used with caution.&#x20;

We can also think from a different perspective, such as converting the improvement of CPU operating frequency into how to prevent CPU frequency reduction. For example, when a mobile phone overheats, the device often reduces CPU frequency to slow down the heating. At this time, our optimization becomes the management of the device overheating scenario. For example, by reducing the time taken for CPU read and write IO operations to reduce the execution time of instructions, our optimization then becomes IO optimization, and in this optimization direction, many implementable solutions have emerged.&#x20;

## 4.4.2 Improve cache efficiency

There are only two ways to improve the efficiency of cache: one is to increase the speed of the cache, and the other is to increase the cache hit rate.&#x20;

1. **Improve cache speed**

In the hierarchical structure of caches, the closer a cache is to the top level, the faster it is. Therefore, we naturally think about how to store core data as much as possible in the top-level cache, from which a large number of optimization schemes can be derived. For example, data stored on the server should be stored on the local disk as much as possible, and data stored on the local disk should be loaded into memory as much as possible. Commonly used image loading frameworks like Fresco and network request frameworks like OkHttp all adopt multi-level cache design to store data in faster caches as much as possible, thereby improving speed.&#x20;

* **Improve cache hit rate**

Although we can improve cache efficiency by placing data in faster cache levels, the capacity of caches is always limited, and the higher the cache level, the smaller its capacity. Therefore, we can only store a limited amount of data in the cache. Ensuring that only the data actually used during program execution is stored within the limited cache capacity, that is, optimizing the cache hit rate, is one of the most challenging yet valuable tasks in cache efficiency optimization.&#x20;

The operating system uses a large amount of the principle of locality to improve the hit rate. Locality has two forms: temporal locality and spatial locality. Temporal locality means that data that has been used once is likely to be used multiple times later. Spatial locality means that if a piece of data is used once, then the data near it is also likely to be used next. From the knowledge above, we also know that cache reads data according to the spatial locality strategy. Under this strategy, the system will load the target data and the data around it into the cache together.&#x20;

In project development, it is necessary to select the most suitable data loading and data eviction strategies based on the characteristics of the scenario in order to effectively improve the cache hit rate.&#x20;

## 4.4.3 Improve task scheduling efficiency

Based on the knowledge of task scheduling learned earlier, it can be known that there are mainly two ways to improve task scheduling efficiency. The first is to increase thread priority. Threads with higher priority can obtain more CPU time slices, thus achieving better performance in terms of speed and fluency. The second is to reduce the overhead of thread and thread state switching. In actual projects, the overhead caused by thread switching is usually relatively large, especially in large-scale projects with a large number of threads. To make better optimizations in this direction, not only the number of threads needs to be reduced, but also the capabilities of thread pools need to be fully utilized to reduce overhead. Therefore, in the subsequent practical chapters, more in-depth explanations will also be provided on the optimization of thread pools.
