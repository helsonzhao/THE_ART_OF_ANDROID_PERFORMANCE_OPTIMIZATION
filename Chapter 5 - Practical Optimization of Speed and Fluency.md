In the previous chapter, we learned the theoretical knowledge that affects speed and fluency, and summarized three optimization methodologies, namely improving CPU execution efficiency, enhancing cache efficiency, and improving task scheduling efficiency. Therefore, in this chapter, I will introduce multiple practical cases of speed and fluency optimization based on these three methodologies to help everyone gain a deeper understanding of speed and fluency optimization.&#x20;

When we optimize speed and fluency, among the optimization solutions that can be implemented and put into practice, most of them are mainly focused on improving CPU execution efficiency. Therefore, this chapter will also introduce more optimization cases in this direction, including multiple cases such as "making full use of CPU idle time", "reducing CPU waiting", "binding CPU big cores", and "GC suppression". The introduced solutions include "cache strategy optimization" and "Dex file reordering". Finally, in the direction of improving task scheduling efficiency, the introduced solutions include "increasing the priority of core threads" and "thread pool optimization".&#x20;

# 5.1 Make full use of CPU idle moments

Except for a few categories of applications such as games, most applications do not continuously consume CPU resources at a high level. Therefore, during the program execution, the CPU will be idle at many times, such as when the user is inactive or the application is running in the background. If we can make full use of the idle moments of the CPU to pre-execute or load tasks and data that may be needed later, it is a very effective optimization solution to improve CPU efficiency. To make full use of the idle moments of the CPU, we first need to determine that the CPU is idle. I will introduce two solutions here:&#x20;

1. Judgment is made based on the CPU data under the proc node file

2. Determine through the `times` function

## 5.1.1 proc File Scheme

On Linux systems, most information about devices and applications is recorded in a file under the proc directory, such as the maps file that records process memory mapping information, the stat file that records CPU information, etc. We can obtain the Total Cost time of the device CPU by reading the data from the /proc/stat file, and obtain the CPU consumption time of a process corresponding to a certain process ID by reading the /proc/{pid}/stat file.

### 1. Total CPU Cost

View the /proc/stat file in the terminal using the command  `adb shell cat /proc/stat`, and its data is as follows:

```shell
cpu  125008 117667 128037 3196237 16160 18733 11734 0 0 0
cpu0 25839 24942 30963 355685 4113 6000 2280 0 0 0
cpu1 23695 27365 27443 363280 2860 3407 2416 0 0 0
cpu2 14162 10115 20652 398174 1798 2782 2451 0 0 0
cpu3 12507 9615 21847 397652 2491 3437 2433 0 0 0
cpu4 10043 11091 5759 424106 867 783 324 0 0 0
cpu5 11832 9604 5749 423895 911 690 311 0 0 0
cpu6 14558 12965 7616 415413 1614 799 410 0 0 0
cpu7 12367 11967 8005 418027 1502 832 1104 0 0 0
intr 14033565 0 0 0 0 2212446 0 138913 137167 3850 0 197 7170 0 0 0 0 56433 39773 ……
ctxt 20963274
btime 1666009901
processes 25537
procs_running 1
procs_blocked 0
softirq 5378992 18820 1620970 6861 558158 660031 0 673436 869906 0 970810
```

Rows 1 to 9 in the data represent the cumulative CPU time consumed under different dimensions from system startup to the current moment, where Row 1 represents the overall cumulative data of all CPU cores, and the remaining rows correspond to data for each individual CPU core. I will use the overall CPU usage data in Row 1 for explanation, and the data from left to right in this row is explained as follows:

* cpu: Represents the overall CPU usage.

* 125008 (user): User Mode, which is the CPU time consumed by application processes

* 117667 (nice): CPU time consumed by high-priority processes in User Mode

* 128037 (ystem): System Mode, which is the CPU time consumed by kernel processes

* 3196237 (idle): Idle state, indicating the time when the CPU is in an idle state

* 16160 (io\_wait): IO wait time, the time the CPU waits for I/O operations to complete.

* 18733 (irp): Time to handle hard interrupts, the time taken by the CPU to handle hardware interrupts.

* 11734 (soft\_irp): Time to handle soft interrupts, which is the time for the CPU to handle software interrupts.

* 0 0 0: Invalid field

Starting from the intr line, the meaning of the data represented by each line is as follows:

* intr: Cumulative number of interrupts since system startup

* ctxt: Cumulative number of context switches since system startup

* btime: System startup duration

* processes: The number of processes created after the system starts

* procs\_running: The number of processes currently in the running state

* procs\_blocked: The number of processes currently waiting for IO

* softirq: Time spent by the CPU handling soft interrupts since system startup

The data recorded in the /proc/stat file is very helpful for our performance analysis. Based on the data inside, we can know the total running time of the CPU, which can be obtained by simply adding up the data from columns 2 to 8 in the first row of CPU data. The implementation code is as follows:&#x20;

```java
RandomAccessFile mProcStatFile;

long getTotalCPUCostTime(){
    // open /proc/stat file
    if(mProcStatFile == null && mAppStatFile == null){
            mProcStatFile = RandomAccessFile("/proc/stat", "r");
    }else{
        //move the pointer to the head
        mProcStatFile.seek(0);
    }

    // read the first line of file
    String procStat = mProcStatFile.readLine();
    // split data
    String[] procStats = procStat.split(" ");
    // The sum of the 2,3,4,5,6,7,8th data items is the total CPU time consumed.
    return Long.parseLong(procStats[2]) + Long.parseLong(procStats[3]) +
        Long.parseLong(procStats[4]) + Long.parseLong(procStats[5]) +
        Long.parseLong(procStats[6]) + Long.parseLong(procStats[7]) +
        Long.parseLong(rocStats[8]);
}
```

I implemented this using Java code here. On some high-version models, Java layer code may not have read permission for stat files, but Native layer code does. Therefore, you can implement the above logic through C++ code in the Native layer.&#x20;

### 2. Process CPU Consumption

Next, let's look at the CPU consumption of the sample program. Its process ID is 19700. Therefore, by using the command `adb shell cat /proc/19700/stat`, the data of the corresponding node file is as follows:

```shell
19700 (example.android_perference) S 1271 1271 0 0 -1 1077952832 179904 0 28356 0 651 310 0 0 10 -10 42 0 529919 15416832000 25731 18446744073709551615 1 1 0 0 0 0 4612 1 1073775864 0 0 0 17 4 0 0 0 0 0 0 0 0 0 0 0 0 0
```

This file contains only one line of data, but the number of data items in that line is very large. It not only includes the CPU data consumed by the process but also includes a lot of performance-related information about the process. Here, some commonly used data are introduced from left to right, and the detailed explanations are as shown in the following table:&#x20;

| Data Item Index | Data Content                      | Description                                                                                                         |
| --------------- | --------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| 1               | 19700                             | Process ID                                                                                                          |
| 2               | (com.example.android\_perference) | Process Name                                                                                                        |
| 3               | S                                 | The state of a process, represented by a single character, such as R (Running), S (Sleeping), T (Terminated), etc.  |
| 4               | 1271                              | Parent Process ID                                                                                                   |
| 5               | 1271                              | Process Group ID                                                                                                    |
| 14              | 651                               | CPU time of the process in User Mode                                                                                |
| 15              | 310                               | CPU time of the process in Kernel Mode                                                                              |
| 16              | 0                                 | The cumulative CPU time that the current process waits for its child processes in User Mode                         |
| 17              | 0                                 | The cumulative CPU time of the current process waiting for the child process in the system state                    |
| 18              | 10                                | Process Priority                                                                                                    |
| 19              | -10                               | Process nice value                                                                                                  |
| 20              | 42                                | Number of Threads                                                                                                   |
| 22              | 529919                            | Total Process Startup Duration                                                                                      |
| 23              | 15416832000                       | The virtual memory size of the process, in bytes                                                                    |
| 24              | 25731                             | Process exclusive memory + shared libraries, in pages (4KB)                                                         |

As can be seen from the explanation of the above fields, to obtain the CPU consumption time of a process, simply add the User Mode CPU time of item 14 and the Kernel Mode CPU time of item 15. The code implementation is as follows:&#x20;

```java
RandomAccessFile mAppStatFile;
long getAppProcessCPUTime(){
    if(mAppStatFile == null){
        mAppStatFile = 
                RandomAccessFile("/proc/" + android.os.Process.myPid() + "/stat", "r");
    }else{
        mAppStatFile.seek(0);
    }
    
    String appStat = mAppStatFile.readLine();
    String[] appStats = appStat.split(" ");
    // The sum of the 14th ,15th is the process's CPU Time
    return Long.parseLong(appStats[14]) + Long.parseLong(appStats[15]);  
}
```

### 3. CPU Idle Notification

CPU utilization refers to the proportion of CPU time consumed by the application process relative to the total CPU runtime within a certain time range. If the proportion is low, it indicates that the application process consumes less CPU, which means the process is in an idle state. With the above data, we can then determine whether the CPU is in an idle state, but there are also two values that need to be confirmed here:

1. Time range for data acquisition: This value should neither be too short nor too long; any value between 10 seconds and 60 seconds is acceptable. If it is too short, it will waste more resources on data acquisition and computation, while if it is too long, it will result in a too low frequency of triggering idle states. We can continuously adjust it based on experience and the business type of the program to achieve an optimal value.

2. Usage rate: For an 8-core CPU device, under extreme full load conditions where all cores are serving this process, the CPU utilization can approach 800%. For a 4-core CPU device, the CPU utilization can only approach 400% under extreme conditions. Therefore, for high-performance phones, the idle threshold of the usage rate can be set higher, while for low-performance phones, this threshold needs to be lower. Thus, there is no exact value for this parameter, but rather it needs to be adjusted in combination with the program's situation.

Here, I use 10 seconds as the frequency for data collection and 30% as the threshold for the CPU idle state to implement the solution. The code is as follows:&#x20;

```java
float CPU_USAGE_IDLE_VALUE = 0.3;
//A periodic task that runs every 10 seconds is implemented using a scheduled thread pool.
mScheduledThreadPool.schedule(new Runnable() {
    @Override
    public void run() {
        //gain CPU usage rate
        float cpuUsage = getCpuUsage();
        if(CPUUsage < CPU_USAGE_IDLE_VALUE){
            //CPU is idle，we can perform some tasks now
            ……
        }
    }
}, 10, TimeUnit.SECONDS);

long beforeTotalCpuTime;
long beforeAppCpuTime;
float getCpuUsage(){
    long curTotalCpuTime = getTotalCpuCostTime();
    long curAppCpuTime = getAppProcessCpuTime();
   
    // first gain 
    if(beforeAppCpuTime == 0){
        return 0;
    }
    //calculate CPU usage rate
    float CpuUsage = (curTotalCpuTime - beforeTotalCpuTime)/
        (float)(curAppCpuTime - beforeAppCpuTime);
   
    beforeTotalCpuTime= curTotalCpuTime ；
    beforeAppCpuTime= curAppCpuTime ；
    
    return CpuUsage; 
}
```

When the CPU usage is less than the threshold, we can perform logical operations such as pre-creating page components, pre-fetching data, and pre-creating key objects of secondary pages. However, performing all restricted tasks within this if condition will lead to coupling of business code. Therefore, we can use the observer pattern to notify each business of the CPU idle state, and then perform logical operations such as preloading within the business module.

## 5.1.2 Times Function Solution

Reading and parsing the stat file incurs a certain amount of performance overhead, especially in scenarios with high-frequency calls, where the performance overhead introduced by this method may be unacceptable. Therefore, when faced with scenarios that require high-frequency CPU idle judgment, we can use the times function to determine whether the process CPU is idle. The times function is a system function, which can be called by including the sys/times.h file at the Native layer. The function is as follows.&#x20;

```c++
clock_t times(struct tms *buf);
struct tms {
    clock_t tms_utime; // user CPU time 
    clock_t tms_stime; // system CPU time
    clock_t tms_cutime; // sub process user CPU time 
    clock_t tms_cstime; // sub process system CPU time 
}
```

The times function places the returned data in the tms structure. Through the tms\_utime and tms\_stime parameters in the tms structure, we can know the CPU consumption time of the current process. The implementation code is as follows:&#x20;

```c++
#include <sys/times.h>

float getCPUTimes(JNIEnv *env) {
    struct tms currentTms;
    times(&currentTms);
    return currentTms.tms_utime + currentTms.tms_stime;
}
```

When calculating the CPU usage of a process, the CPU time consumed by the process within a unit time is the numerator, and the Total Cost of CPU time is the denominator. Here we obtain the CPU time consumed by the application through the times function, but we cannot obtain the Total Cost of CPU time. Since the system does not have a corresponding interface to directly obtain the Total Cost of CPU time, we still have to read and parse the stat file. Therefore, at this point, we can switch to another metric, using the CPU usage rate of the process to replace the CPU usage, and then make a judgment on whether it is idle.&#x20;

CPU rate refers to the load level of CPU utilization within a certain time period, which can be calculated by the formula "CPU time consumed by the process within the time interval / (time interval × CPU clock tick frequency per unit time)". The denominator "time interval × CPU clock tick frequency per unit time" in the formula represents the theoretically full load time of the CPU during this time interval. We can obtain the CPU clock tick frequency per second through the sysconf(\_SC\_CLK\_TCK) function, and the code is as follows.

```c++
#include <bits/sysconf.h>

int getCpuTick(JNIEnv *env) {
    return sysconf(_SC_CLK_TCK);
}
```

The CPU time consumed by the process obtained in the Times function is in seconds, and the value returned by the sysconf(\_SC\_CLK\_TCK) function is also the CPU clock frequency per second. Therefore, when calculating the CPU usage rate of the process, it is also necessary to perform the calculation with seconds as the frequency. The implementation of the solution is as follows.&#x20;

```java
float beforeAppTime  =  0;
long beforeSysTime = 0;
float CPU_SPEED_IDLE_VALUE = 0.1;
//Calculate the CPU usage rate over a 10-second period.
mScheduledThreadPool.schedule(new Runnable() {
    @Override
    public void run() {
        float curAppTime = getCPUTimes();
        long curSysTime = System.CurrentTime()/1000;
        float CpuSpeed =  (beforeAppTime - curAppTime) / 
                ((curSysTime - beforeSysTime) * getCpuTick())
        if(CpuSpeed < CPU_SPEED_IDLE_VALUE){
            //CPU is idle，we can perform some tasks now
            ……
        }
        beforeAppTime  = curAppTime; 
        beforeSysTime = curSysTime ;
    }
}, 10, TimeUnit.SECONDS);
```

When the application is in an idle state, the CPU rate is basically below 0.1. In actual scenarios, we can set a threshold for idle determination based on the characteristics of the application and through empirical values. The solution of calculating the CPU rate using the Times function to determine whether the CPU is idle is simpler to implement and has better performance, so it is more suitable for scenarios where high-frequency CPU state judgment is required.&#x20;

# 5.2 Reduce CPU waiting

In the Android system, the rendering of the interface is performed on the main thread. Therefore, when the CPU is executing code instructions on the main thread, it is necessary to minimize the time wasted by CPU waiting as much as possible, so that the interface can be displayed faster and more smoothly. There are mainly two situations that cause CPU waiting: waiting for locks and waiting for I/O. So let's take a look at how to optimize in these two directions.&#x20;

## 5.2.1 Lock Wait Optimization

During Java development, we often use \`synchronize\` to lock methods or data to avoid concurrency issues in multi-threaded scenarios. When a lock is held by a thread, other threads need to wait for the lock to be released and then acquire it before they can access the method or data. When a thread tries to acquire a lock, it first checks whether the lock is held by another thread. If it is, the thread will repeatedly check whether the lock has been released through multiple loops, which can cause the CPU to spin idly. If the lock cannot be acquired after multiple spins, the thread requesting the lock will go to sleep and be added to the waiting queue, and will be woken up after the lock is released. From this process, we can see that when requesting a lock, both spinning and sleeping can cause the current thread to be unable to obtain CPU resources. If there are too many locks in the main thread and the rendering thread, it will lead to a poor user experience in the application.&#x20;

Here, I still simulates a scenario in the sample program where the main thread needs to acquire a lock. As shown in Figure 5-1, the child thread in the code holds the lock StabilityExampleActivity.this and will not release it, while the main thread will also attempt to acquire this lock. Since the lock cannot be acquired here, it will cause the main thread to be unresponsive for a long time, resulting in an ANR. This is an extreme example of lock waiting, but through this extreme example, we can gain a deeper understanding of the optimization process for lock waiting.&#x20;

![Figure 5-1 Code for lock waiting](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter5_img_1.png)

### 1. Trace File Capture

To optimize the waiting time of locks, the first step is to capture a Trace file. A Trace file is a file used for performance analysis and debugging, which records the running status of an application or system, including detailed information such as CPU usage, memory allocation, thread activity, function calls, and system events. There are also many ways to capture a Trace, such as capturing a systrace via the systrace.py script (this method has been deprecated in Android 10 and above), capturing via Android Studio's Profile, capturing via Perfetto, and other methods. I mainly introduce the method of capturing via Perfetto here, which is a Trace capture tool provided by Android 10 and above versions and is very powerful.

There are also many ways to capture traces with Perfetto, such as capturing through the visualization website provided by Perfetto ([https://ui.perfetto.dev/](https://ui.perfetto.dev/)), as shown in Figure 5-2. After connecting the device via USB, you can click to capture on the Record page.

![Figure 5-2 Pefetto fetch trace interface](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter5_img_2.png)

You can also open system tracing in the device's developer mode to capture, as shown in Figure 5-3. Configure the content to be captured through categories, and then click "Record Trace" to capture the Trace.&#x20;

![Figure 5-3 Trace entry in developer mode](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter5_img_3.png)

It can also be captured via the command line, which is a method I recommended because it is more flexible. Here, let's take a look at how to capture Perfetto's Trace logs via the command line. Perfetto is implemented based on Android's system tracing service, which is enabled by default after Android 11. However, for devices running Android 11 or lower, the system tracing service needs to be manually enabled using the following command.&#x20;

```c++
adb shell setprop persist.traced.enable 1
```

After enabling the system tracing service, you can start capturing the corresponding trace. Enter the device's command interface via adb shell, and execute the command "perfetto \[options] \[categories]" to capture the trace file, where options are configuration selections and categories are the types of data to be captured. You can use perfetto --help to view available options and categories. Common options configurations are as follows.&#x20;

| Parameter    | Explanation                                                                                                  |
| ------------ | ------------------------------------------------------------------------------------------------------------ |
| -o or --out  | Specify an output file to save the captured trace data. The output file must be in the.perfetto-trace format |
| -t or --time | Specify the duration of the capture, with the unit being seconds                                             |
| -b or --buf  | Specify the size of the buffer, with the unit being MB                                                       |

Common categories configurations include the following.

| Type           | Explanation                                                                                                                                           |
| -------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| sched          | Capture data related to CPU scheduling, including events such as the running, switching, and waking up of processes and threads.                      |
| freq           | Capture CPU frequency-related data, including information such as the current frequency, maximum frequency, and minimum frequency of each CPU.        |
| gfx            | Capture graphics-related data, including events such as rendering, composition, and swapping of components like SurfaceFlinger, Vulkan, OpenGL, etc.  |
| input          | Capture input-related data, including the handling and distribution of input events such as touch, keystrokes, and mouse events.                      |
| view           | Capture view-related data, including events such as creation, destruction, layout, and drawing of components like View, Window, Activity, etc.        |
| wm             | Capture window management-related data, including events such as window addition, removal, adjustment, and focus.                                     |
| am             | Capture activity management-related data, including events such as application startup, stop, switch, pause, and resume.                              |
| sm             | Capture system management-related data, including events such as system startup, shutdown, restart, hibernation, wakeup, etc.                         |
| mem            | Capture memory-related data, including events such as system and application memory usage, allocation, recycling, and compression.                    |
| power          | Capture power-related data, including information such as battery charge level, voltage, temperature, and charging status                             |
| aidl           | Capture AIDL-related data, including events such as calls, returns, and exceptions in inter-process communication                                     |
| binder\_driver | Capture data related to the Binder driver, including events such as the sending, receiving, processing, and completion of Binder transactions         |
| binder\_lock   | Capture data related to Binder locks, including events such as Binder lock contention, acquisition, and release                                       |
| app            | Capture application-related data, including application-defined trace events, recorded by the Trace API.                                              |

After getting familiar with the use of the command line, enter the following command to start Trace capture, then operate the scenario in the sample program that will generate lock waits, and you can successfully capture the Trace during this 20-second operation period.&#x20;

```shell
perfetto -o /data/misc/perfetto-traces/trace_file.perfetto-trace -t 20s sched freq am wm gfx input view mem binder_driver binder_lock 
```

### 2. Trace File Analysis

Once the Trace file has been captured, analysis can begin. Pefetto provides a visualization website specifically designed to parse the Trace files we capture. After opening the Pefetto website, simply import the captured "trace\_file.perfetto-trace" file.&#x20;

In the parsed data, directly locate the main thread of the sample program. As shown in Figure 5-4, it can be intuitively seen that the main thread is in a state of waiting for a lock, and through Details, it can be seen that the waiting time has already reached 12 seconds, and the lock on line 29980 of StabilityExampleActivity.java:27 is held by the thread with thread ID 29980.&#x20;

![Figure 5-4 Main thread information](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter5_img_4.png)

Next, let's take a look at the thread information with id 29980. As shown in Figure 5-5, it has been in the Running state.

![Figure 5-5 The running state of the thread holding the lock](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter5_img_5.png)

Here, I use a relatively extreme case to show readers how to analyze lock occupancy through Perfetto's Trace. The Trace captured by Perfetto contains very detailed data, which we can use to analyze many things, such as the time taken by each method in the main thread, the duration and reason of sleep, CPU consumption, etc. Developers need to make full use of the Trace file to optimize speed and fluency.&#x20;

### 3. Lock Optimization Solution

After locating the CPU wait caused by the lock of the main thread through Trace analysis, the next step is to optimize the lock. There are several lock optimization solutions as follows:&#x20;

1. No Locking: Determine whether it is necessary to lock the business logic. If concurrency is impossible, then locking is not required. In addition, there are also solutions such as thread-local storage and biased locking, all of which belong to lock-free optimization.&#x20;

2. Reasonably refine the granularity of locks: Optimize lock performance by reducing the number of synchronized code blocks, for example, refining the use of Synchronize to lock the entire method into only locking the code blocks within the method that may cause thread safety issues.

3. Reasonably coarsen the granularity of locks: Performance can also be optimized by appropriately coarsening locks. For example, when the code logic calls the StringBuffer.append method provided by Java multiple times simultaneously, the virtual machine will coarsen the locks inside each append method, so that multiple consecutive append methods share only one lock.

4. Reasonably increase the number of locks: Similar to refining the granularity of locks, but by increasing the number of locks to refine the granularity.

We need to conduct a specific analysis based on the business scenario before we can determine which optimization solution to use. For example, for the code in the example, we can adopt a lock-free solution for optimization. To enable readers to gain a further understanding of lock optimization, here I introduce a scenario that everyone is familiar with: the lock optimization solution of ConcurrentHashMap in Java 7.

In Java 7, ConcurrentHashMap is mainly composed of two parts: Segments and HashEntry. The Segments array is used to store arrays of HashEntry objects, and HashEntry objects store Key and Value. Their relationship and implementation are shown in the following code:&#x20;

```java
public class ConcurrentHashMap<K, V> extends AbstractMap<K, V>  
        implements ConcurrentMap<K, V>, Serializable {      
    final int segmentMask;    
    final int segmentShift;   
    final Segment<K,V>[] segments;  
    ……
 }

static final class Segment<K,V> extends ReentrantLock implements Serializable {  
    private static final long serialVersionUID = 2249069246763182397L;  
    transient volatile int count;  
    transient int modCount;   
    transient int threshold;  
    transient volatile HashEntry<K,V>[] table;  
    final float loadFactor;
    ……
}

static final class HashEntry<K,V> {
        final int hash;
        final K key;
        volatile V value;
        volatile HashEntry<K,V> next;
        ……
}

```

Next, let's look at how the key function put of ConcurrentHashMap locks when storing data. The code implementation of this function is as follows:

```java
public V put(K key, V value) {
    Segment<K,V> s;
    if (value == null)
        throw new NullPointerException();
    //get the hash value of the key
    int hash = hash(key);
    //locate the segment
    int j = (hash >>> segmentShift) & segmentMask;
    if ((s = (Segment<K,V>)UNSAFE.getObject          
         (segments, (j << SSHIFT) + SBASE)) == null) 
        s = ensureSegment(j);
    //ivoke Segment's put method
    return s.put(key, hash, value, false);
}

```

The put method of ConcurrentHashMap mainly obtains the hash function of the Key, then gets the Segment based on the hash value, and then calls the put method of the Segment. Its implementation is as follows. From the code, we can see that ConcurrentHashMap does not lock the entire put method in the put function, but only locks the Segment after finding it. Let's take a look at the put method of the Segment object. The code is as follows.

```java
 V put(K key, int hash, V value, boolean onlyIfAbsent) {  
     //lock segment
     lock();    
     try {  
         int c = count;  
         if (c++ > threshold) // If the rehash threshold is exceeded
             // Perform rehashing, the length of the table array will be doubled 
             rehash(); 
         HashEntry<K,V>[] tab = table;  

      
         int index = hash & (tab.length - 1);  

         //Locate the specific bucket corresponding to the hash code
         HashEntry<K,V> first = tab[index];  
         HashEntry<K,V> e = first;  
         while (e != null && (e.hash != hash || !key.equals(e.key)))  
             e = e.next;  

         V oldValue;  
         if (e != null) { // If the key-value pair already exists
             oldValue = e.value;  
             if (!onlyIfAbsent)  
                 e.value = value; // Set the value 
         }  
         else {  // Key-value pair does not exist 
             oldValue = null;  
             ++modCount; 

             // Create a new node and add it to the head of the linked list
             tab[index] = new HashEntry<K,V>(key, hash, first, value);  
             count = c; 
         }  
         return oldValue;  
     } finally {  
         unlock(); // Release the lock
     }  
 } 

```

Having understood the put function, let's continue to look at the implementation of another function, the get function, whose code is as follows. From the code, we can see that since get does not cause thread safety issues, we can simply retrieve the HashEntry from the Segment without any locking logic.&#x20;

```java
public V get(Object key) {
    Segment<K,V> s; 
    HashEntry<K,V>[] tab;
    int h = hash(key);
    long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
    //first locate the Segment，the locate the HashEntry
    if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null &&
                                                     (tab = s.table) != null) {
        for (HashEntry<K,V> e = (HashEntry<K,V>) UNSAFE.getObjectVolatile
          (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);
          e != null; e = e.next) {
             K k;
             if ((k = e.key) == key || (e.hash == h && key.equals(k)))
                 return e.value;
        }
    }
    return null;
}
```

Here, I summarize the lock optimization schemes in ConcurrentHashMap, mainly including the following:&#x20;

* Lock Refinement: ConcurrentHashMap uses a segmented lock scheme, as shown in Figure 5.6. This scheme divides the data into several Segment segments, where each segment is equivalent to an independent HashMap and has its own independent lock. Thus, when operating on a particular segment, only that segment needs to be locked without affecting the concurrent access to other segments, thereby improving concurrent performance.

![Figure 5-6 ConcurrentHashMap locking method](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter5_img_6.png)

* Lock Elimination: The get method of ConcurrentHashMap is not locked because it only reads data and does not modify it, so there will be no concurrency issues.

In Java 8, the segmented lock approach has been abandoned, and the Segment array segments are no longer present. All HashEntry objects are stored in the Node array, and the locking mechanism of CAS + Synchronize is adopted. In the put method, it first checks whether the position of the Node to be stored has a value, i.e., whether a HASH collision will occur. If there is no value, it directly uses CAS for locking and stores the HashEntry; if there is, it uses Synchronize for locking before performing the storage logic. Interested readers can refer to the implementation of ConcurrentHashMap in Java 8 and further analyze the advantages of the new solution in concurrency.&#x20;

## 5.2.2 IO Wait Optimization

Reducing IO time consumption is the most commonly used ways to improve CPU execution efficiency. We can analyze functions with long IO time consumption through various methods such as function time consumption or Trace analysis. The method of capturing and analyzing Trace has already been introduced in the previous section, so I won't elaborate on it here. Therefore, let's directly look at how to optimize functions once we have located those with long IO time consumption. The commonly used IO optimization solutions are as follows.

1. Asynchronous IO

When an IO task is encountered in the main thread, the IO task can be placed in a child thread for processing, allowing the main thread to continue execution. When the logic of the main thread needs to use this data, it can first check if the data already exists. If it does, it can be used directly; if not, it continues to wait. For Java code, we can use CountDownLatch to implement this mechanism. As shown in the code, when initializing CountDownLatch, the value of the counter needs to be set. In the code, the value is set to 1. Then, the child thread executes the IO task to obtain the data, while the main thread continues to execute its logic. When it needs the data obtained by the IO task, it calls the latch.await method. If CountDownLatch is 0 at this time, it means the data has been obtained, and the code logic continues without blocking. If it is not 0, it waits until the data is obtained in the child thread and the value of the CountDownLatch counter is decremented by one.&#x20;

```java
CountDownLatch latch = new CountDownLatch(1);

new Thread(new Runnable(){
    @Override
    public void run() {
        try {
            // Simulate a sub-thread executing a task, which takes 1 second.
            getData();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            // sub-thread execute countDown
            latch.countDown();
        }
    }
}).start();

//execute other code that don't need data
……

// main thread call await method，wait for sub-thread finished
latch.await();
//execute the code that need data
……
```

In real-world environments, there are many scenarios where we can use asynchronous IO optimization. For example, when we want to render a certain interface, we can execute IO tasks through a child thread at the beginning of the code to read the interface data, and then the main thread can continue with tasks that do not require this data for the time being, such as parsing and creating views, layout measurement, etc., until the data is needed, at which point we call the latch.await method.&#x20;

* Coroutine

Java threads actually go to sleep when performing IO operations until the IO is completed. Since the state transition is controlled by the operating system, we cannot make the threads avoid going to sleep or do other things. However, with coroutine tasks, we can execute other tasks while waiting for IO, thus ensuring that the process does not go to sleep but remains in a running state. In fact, the flexible use of coroutines is very helpful for IO-intensive applications.

However, I needs to remind readers that the commonly used Kotlin coroutines are not true coroutines. They do not implement their own scheduler and context switching, but still rely on the operating system's thread scheduling. Therefore, using Kotlin coroutines does not optimize IO-intensive tasks. If we want to optimize IO tasks through coroutines, we can use the Rust language to implement true coroutines.&#x20;

Rust is a new language. We first need to download the Rust Development Environment before we can start development. Then, we use Android's NDK tools to compile Rust code into an so library. Next, Android project code can execute the methods in this so library through JNI calls. Rust development is beyond the scope of this book, so I will not elaborate further here. Readers interested can conduct their own research.&#x20;

* Batch Processing

Batch processing refers to combining multiple IO operations into one, thereby reducing the overhead of network transmission and disk seek. For example, we can combine multiple database queries into one query, or multiple network requests into one request, which can improve the throughput and efficiency of IO.

I uses transactions in database operations to explain an optimization case of batch processing. When it is necessary to perform batch insertions, updates, or deletions on the database, we can split the batch tasks into multiple single tasks for execution. Undoubtedly, this approach will incur performance losses due to a large number of I/O operations. Therefore, we can use the transaction mechanism provided by the database to complete batch database operations in a single operation. The code for using transactions is as follows: first, call the beginTransaction method provided by the SQLite database, then multiple database operations can be executed, and finally call the endTransaction method to end the transaction. Although multiple database operations are called in the middle, these operations will be combined into an atomic operation, so these operations will either all succeed or all fail. By using transactions, not only can the performance of batch operations be significantly improved, but also data consistency can be ensured.&#x20;

```java

SQLiteDatabase db = dbHelper.getWritableDatabase();

db.beginTransaction();
try {
    // Perform multiple database operations.
    ContentValues values1 = new ContentValues();
    values1.put("column1", "value1");
    db.insert("table_name", null, values1);

    ContentValues values2 = new ContentValues();
    values2.put("column1", "value2");
    db.insert("table_name", null, values2);

    // if all operations are successful, set transaction successful
    db.setTransactionSuccessful();
} catch (Exception e) {
    e.printStackTrace();
} finally {
    db.endTransaction();
}
```

# 5.3 Bind to CPU Big Core

Currently, the CPUs of mobile phones are all multi-core. For example, the Snapdragon 8 Gen3 CPU has 8 cores, among which the large core Cortex-X4 has the best performance, with a clock cycle frequency of 3.3 GHz, while the performance of other cores is much worse. Among them, the clock frequency of the two small cores Cortex-A520 is only 2.27 GHz. If the large core is used to execute the main thread, it will naturally enable the main thread to have a faster speed when executing logic such as UI rendering.

## 5.3.1 Thread Binding Core Function

The Linux system provides  pthread\_setaffinity\_np  and  sched\_setaffinity  these two functions to bind a specified thread to a specified core. However, in the Android system, the use of the pthread\_setaffinity\_np function is blocked, so we can only perform core binding operations through the sched\_setaffinity function. The function is as follows:&#x20;

```c++
#include <sched.h>
int sched_setaffinity(pid_t pid, size_t cpusetsize,cpu_set_t *mask);
```

* The first input parameter pid refers to the thread ID. If the value of pid is 0, it indicates the main thread&#x20;

* The second input parameter cpusetsize is the length of the third input parameter mask&#x20;

* The third input parameter mask is the mask of the CPU sequence to be bound&#x20;

The code to implement thread core binding through this function is as follows:&#x20;

```c
void bindCore(int coreNum){
    cpu_set_t mask;    //the set of CPU core
    CPU_ZERO(&mask);    //set zero of mask
    // Set the mask with the CPU core number to bind to
    CPU_SET(coreNum,&mask);  
    // Bind the main thread to the specified CPU core
    if (sched_setaffinity(0, sizeof(mask), &mask) == -1){    
         printf("bind core fail");
    }
}
```

Through the above code, the main thread is bound to the CPU core with the core sequence number coreNum. Next, we still need to further determine which CPU core is the big core.

## 5.3.2 Obtain the large core sequence

By checking the files under the `/sys/devices/system/cpu/` directory, you can see how many CPU cores the current device has. I used a Pixel3 for testing and can see that there are a total of 8 CPU cores from cpu0 to cpu7.&#x20;

```c++
/sys/devices/system/cpu $ ls 
core_ctl_isolated  cpu1  cpu3  cpu5  cpu7     cpuidle                hang_detect_gold    hotplug   kernel_max  offline  possible  present
cpu0               cpu2  cpu4  cpu6  cpufreq  gladiator_hang_detect  hang_detect_silver  isolated  modalias    online   power     uevent
```

Then, enter the cpufreq file corresponding to a specific CPU core to view the detailed parameters of that particular CPU core. Below is the detailed data for the CPU core with sequence number 0:&#x20;

```c++
/sys/devices/system/cpu/cpu0/cpufreq $ ls
affected_cpus     cpuinfo_max_freq  cpuinfo_transition_latency  scaling_available_frequencies  scaling_boost_frequencies  scaling_driver    scaling_max_freq  scaling_setspeed  stats
cpuinfo_cur_freq  cpuinfo_min_freq  related_cpus                scaling_available_governors    scaling_cur_freq           scaling_governor  scaling_min_freq  schedutil
```

The cpuinfo\_max\_freq under this file is the clock cycle frequency of the current CPU core. Below are the clock cycle frequencies of each core of my test device, Piexl3 with the Snapdragon 845 chip. It can be seen that cores 4, 5, 6, and 7 are all big cores with a clock frequency of 2.8 GHz, while the other small cores only have a frequency of 1.7 GHz.&#x20;

```c++
/sys/devices/system/cpu $ cat cpu0/cpufreq/cpuinfo_max_freq
1766400
/sys/devices/system/cpu $ cat cpu1/cpufreq/cpuinfo_max_freq
1766400
/sys/devices/system/cpu $ cat cpu2/cpufreq/cpuinfo_max_freq
1766400
/sys/devices/system/cpu $ cat cpu3/cpufreq/cpuinfo_max_freq
1766400
/sys/devices/system/cpu $ cat cpu4/cpufreq/cpuinfo_max_freq
2803200
/sys/devices/system/cpu $ cat cpu5/cpufreq/cpuinfo_max_freq
2803200
/sys/devices/system/cpu $ cat cpu6/cpufreq/cpuinfo_max_freq
2803200
/sys/devices/system/cpu $ cat cpu7/cpufreq/cpuinfo_max_freq
2803200
```

Therefore, in the code implementation, you only need to traverse the CPU nodes under the `/sys/devices/system/cpu/directory`, and then read the value of the cpuinfo\_max\_freq file to find the big core. The following is the detailed code implementation:&#x20;

1. Count how many cores the CPU of this device has.&#x20;

```java
int getNumberOfCPUCores() {
    int cores = 0;
    DIR *dir;
    struct dirent *ent;
    if ((dir = opendir("/sys/devices/system/cpu/")) != NULL) {
        while ((ent = readdir(dir)) != NULL) {
            std::string path = ent->d_name;
            if (path.find("cpu") == 0) {
                bool isCore = true;
                for (int i = 3; i < path.length(); i++) {
                    if (path[i] < '0' || path[i] > '9') {
                        isCore = false;
                        break;
                    }
                }
                if (isCore) {
                    cores++;
                }
            }
        }
        closedir(dir);
    }
    return cores;
}
```

* Read rows to iterate through each core and find the core with the highest clock frequency.&#x20;

```java
int getMaxFreqCPU() {
    int maxFreq = -1;
    for (int i = 0; i < getNumberOfCPUCores(); i++) {
        std::string filename = "/sys/devices/system/cpu/cpu" + 
                                std::to_string(i) + "/cpufreq/cpuinfo_max_freq";
        std::ifstream cpuInfoMaxFreqFile(filename);
        if (cpuInfoMaxFreqFile.is_open()) {
            std::string line;
            if (std::getline(cpuInfoMaxFreqFile, line)) {
                try {
                    int freqBound = std::stoi(line);
                    if (freqBound > maxFreq) maxFreq = freqBound;
                } catch (const std::invalid_argument& e) {
                    
                }
            }
            cpuInfoMaxFreqFile.close();
        }
    }
    return maxFreq;
}
```

At this point, the sequence of the big core has been identified, and then sched\_setaffinity can be called to bind the core. In addition to the main thread, other core threads, such as the rendering thread, can also be bound to the big core according to business requirements. After we bind the main thread to the big core through the above logic, we can verify which core the target thread is running on by capturing Trace to Pefetto, thereby confirming whether the binding is successful.

# 5.4 GC Inhibition

When optimizing CPU efficiency, we often use the Profile tool built into Android Studio to analyze CPU usage. At this time, it is very likely that we will find that the HeapTaskDaemon thread occupies a relatively high amount of CPU time. As shown in Figure 5-7, it can be seen that the HeapTaskDaemon thread has large chunks of time in the Running state.&#x20;

![Figure 5-7 HeapTaskDaemon Thread in Profiler](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter5_img_7.png)

This thread is actually used by the virtual machine to perform GC operations. When the ART virtual machine is performing GC, it will preempt a lot of CPU resources. For devices with poor performance, it is very easy to become laggy or slow because they cannot obtain enough CPU time slices. Since GC operations are the most core operations of the virtual machine, it is naturally impossible to turn off this operation, otherwise memory cannot be reclaimed, leading to serious problems. However, if we briefly suppress the execution of GC when performing core scenarios, such as starting, opening a page, or scrolling a list, we can allow core tasks to obtain more CPU time, thereby improving the performance experience.&#x20;

## 5.4.1 Process of GC Execution

Since the GC process is the system's logic, if we want to inhibit GC execution, we first need to familiarize ourselves with the GC execution process, then find a breakthrough point from it, and achieve the goal by modifying its logic through Native Hook technology. Since the HeapTaskDaemon thread preempts a relatively large amount of CPU, we will directly analyze this thread to see what it actually does.

### 1. HeapTaskDaemon Thread Analysis

By globally searching for the keyword "HeapTaskDaemon" in the Android system source code, it was found that it is a thread created at the Java layer, located in Daemons.java object. Analyzing the source code shows that HeapTaskDaemon inherits from the Daemon object, and the code is as follows.

```c++
private static class HeapTaskDaemon extends Daemon {
    private static final HeapTaskDaemon INSTANCE = new HeapTaskDaemon();

    HeapTaskDaemon() {
        super("HeapTaskDaemon");
    }

   public void runInternal() {
        ……
        VMRuntime.getRuntime().runHeapTasks();
    }
}
```

The Daemon object is actually a Runnable object. After calling the start method of this object, a thread named "HeapTaskDaemon" will be created to execute the current Daemon Runnable. The simplified code is as follows. By now, we know the origin of this thread.&#x20;

```c++
private static abstract class Daemon implements Runnable {
    @UnsupportedAppUsage
    private Thread thread;
    private String name;
    private boolean postZygoteFork;

    protected Daemon(String name) {
        this.name = name;
    }

    @UnsupportedAppUsage
    public synchronized void start() {
        startInternal();
    }

    public synchronized void startPostZygoteFork() {
        postZygoteFork = true;
        startInternal();
    }

    // The current thread starts as soon as the Zygote process is launched.
    public void startInternal() {
        if (thread != null) {
            throw new IllegalStateException("already running");
        }
        thread = new Thread(ThreadGroup.systemThreadGroup, this, name);
        thread.setDaemon(true);
        thread.setSystemDaemon(true);
        thread.start();
    }

    public final void run() {
        ……
        try {
            runInternal();
        } catch (Throwable ex) {
           ……
            throw ex;
        }
    }

    public abstract void runInternal();

    ……
    
}
```

HeapTaskDaemon is a daemon thread that starts along with the Zygote process. The run method of this thread is relatively simple, which is to execute the abstract function runInternal. In the implementation of this abstract function, the method `VMRuntime.getRuntime().runHeapTasks` will be executed, and this method will execute the Native function RunAllTasks.

```c++
static void VMRuntime_runHeapTasks(JNIEnv* env, jobject) {
  Runtime::Current()->GetHeap()->GetTaskProcessor()->RunAllTasks(ThreadForEnv(env));
}
```

The RunAllTasks function is located in task\_processor.cc, and this method actually just continuously loops to call the GetTask function to obtain HeapTask and execute it.

```c++
void TaskProcessor::RunAllTasks(Thread* self) {
  while (true) {
    HeapTask* task = GetTask(self);
    if (task != nullptr) {
      task->Run(self);
      task->Finalize();
    } else if (!IsRunning()) {
      break;
    }
  }
}
```

The GetTask function continuously retrieves HeapTasks from the tasks collection for execution, and for HeapTasks that require a delay, it blocks until the target time arrives.&#x20;

```c++
std::multiset<HeapTask*, CompareByTargetRunTime> tasks_ ;

HeapTask* TaskProcessor::GetTask(Thread* self) {
  ……
  while (true) {
    if (tasks_.empty()) {
      //If tasks is empty，then wait
      cond_.Wait(self);  
    } else {
      // If task isn't empty，then get the first HeapTask
      const uint64_t current_time = NanoTime();
      HeapTask* task = *tasks_.begin();
      
      uint64_t target_time = task->GetTargetRunTime();
      if (!is_running_ || target_time <= current_time) {
        tasks_.erase(tasks_.begin());
        return task;
      }
      // For the delayed HeapTask，wait here until the target time is reached
      const uint64_t delta_time = target_time - current_time;
      const uint64_t ms_delta = NsToMs(delta_time);
      const uint64_t ns_delta = delta_time - MsToNs(ms_delta);
      cond_.TimedWait(self, static_cast<int64_t>(ms_delta), static_cast<int32_t>(ns_delta));
    }
  }
  UNREACHABLE();
}
```

After analyzing the entire process, the idea of suppressing GC emerged, and we have two approaches:

1. Add a custom HeapTask to the tasks collection, and perform a sleep operation within the custom HeapTask. This will block the HeapTaskDaemon thread, achieving the goal of suppressing the execution of this thread.

2. Obtaining the system's HeapTask and putting this HeapTask to sleep can also achieve the purpose of suppressing the execution of the HeapTaskDaemon thread&#x20;

For the system, the first solution is very simple because the system can directly obtain the TaskProcessor object and simply add a custom task to it. Starting from Android 8, the system has adopted this solution when the program starts, adding a task to delay the execution of GC by 2 seconds. However, for user programs, this solution is relatively complex because it is difficult to obtain and operate the TaskProcessor object. Therefore, in this chapter, I will introduce the second solution.&#x20;

### 2. HeapTask Analysis

To ensure the smooth implementation of the plan, it is necessary to continue analyzing what HeapTask does. This object is located in the task\_processor.h file, and the source code is as follows

```c++
class HeapTask : public SelfDeletingTask {
 public:
  explicit HeapTask(uint64_t target_run_time) : target_run_time_(target_run_time) {
  }
  uint64_t GetTargetRunTime() const {
    return target_run_time_;
  }

 private:
  void SetTargetRunTime(uint64_t new_target_run_time) { 
    target_run_time_ = new_target_run_time;
  }

  uint64_t target_run_time_;
  friend class TaskProcessor;
};

class SelfDeletingTask : public Task {
 public:
  virtual ~SelfDeletingTask() { }
  virtual void Finalize() {
    delete this;
  }
};

class Task : public Closure {
 public:
  virtual void Finalize() { }
};

class Closure {
 public:
  virtual ~Closure() { }
  virtual void Run(Thread* self) = 0;
};

```

Through source code analysis, it can be found that HeapTask actually inherits from the three classes SelfDeletingTask, Task, and Closure in sequence. The Task class defines the virtual function Finalize, and the Closure class defines the virtual function Run. What is a virtual function? We can first understand it as Java's abstract function. The virtual keyword is similar to Java's abstract keyword. SelfDeletingTask implements the virtual function Finalize, which is used for object destruction. The implementation of the Run function will be left to the subclasses of HeapTask. Still through global source code search, it is found that the subclasses that inherit from HeapTask in the Android system are as follows:

* ConcurrentGCTask: When Java memory reaches the threshold, this task will be executed to perform concurrent GC.

* CollectorTransitionTask: This task is executed when switching between foreground and background, used to switch the type of GC. For example, when switching to the background, it will switch to a GC mechanism such as copy collection.

* HeapTrimTask: After GC is completed, if it is necessary to return the idle memory in the heap to the kernel, this task will be executed to handle it.

* TriggerPostForkCCGcTask: Starting from Android 8, to avoid GC operations during startup, the system will execute this task, blocking the HeapTaskDaemon thread for 2 seconds.

* ReduceTargetFootprintTask: Used in conjunction with TriggerPostForkCCGcTask.

* ClearedReferenceTask: When an object is reclaimed, this Task will be executed. The Task calls the ReferenceQueue.add method in the Java layer to add the reference of the reclaimed object to the ReferenceQueue.

* NotifyStartupCompletedTask: A task executed after startup completion, used for verification purposes.

Among these HeapTasks, ConcurrentGCTask is the most frequently executed and has the greatest impact on performance. Therefore, we will mainly analyze ConcurrentGCTask.

### 3. ConcurrentGCTask Analysis

As mentioned in Chapter 2  "Principles of Memory Optimization" , when creating an object, the virtual machine calls the  AllocObjectWithAllocator  method to allocate memory space for this object. Since it involves memory allocation, naturally, memory deallocation follows, and the deallocation process also takes place in this function. The simplified source code is as follows:&#x20;

```c++
inline mirror::Object* Heap::AllocObjectWithAllocator(Thread* self,
                                                      ObjPtr<mirror::Class> klass,
                                                      size_t byte_count,
                                                      AllocatorType allocator,
                                                      const PreFenceVisitor& pre_fence_visitor) {
  ……
  bool need_gc = false;
  uint32_t starting_gc_num;  // GC number at which we observed need for GC.
  {
    ……
    if (bytes_tl_bulk_allocated > 0) {
      ……
      // need_gc is set to true if it is Concurrent GC or the threshold is reached.
      if (IsGcConcurrent() && UNLIKELY(ShouldConcurrentGCForJava(new_num_bytes_allocated))) {
        need_gc = true;
      }
      ……
    }
  }
  ……
  if (need_gc) {
    //Triger GC
    RequestConcurrentGCAndSaveObject(self, /*force_full=*/ false, starting_gc_num, &obj);
  }
  ……
  return obj.Ptr();
}

inline bool Heap::ShouldConcurrentGCForJava(size_t new_num_bytes_allocated) {
  return new_num_bytes_allocated >= concurrent_start_bytes_;
}
```

From the source code, it can be seen that if it is determined to be a concurrent GC, or when the heap memory reaches the concurrent\_start\_bytes\_ threshold, the RequestConcurrentGCAndSaveObject method will be called. The source code is as follows:

```c++
void Heap::RequestConcurrentGCAndSaveObject(Thread* self,
                                            bool force_full,
                                            uint32_t observed_gc_num,
                                            ObjPtr<mirror::Object>* obj) {
  RequestConcurrentGC(self, kGcCauseBackground, force_full, observed_gc_num);
}

bool Heap::RequestConcurrentGC(Thread* self,
                               GcCause cause,
                               bool force_full,
                               uint32_t observed_gc_num) {
  uint32_t max_gc_requested = max_gc_requested_.load(std::memory_order_relaxed);
  if (!GCNumberLt(observed_gc_num, max_gc_requested)) {
    if (CanAddHeapTask(self)) {
      if (max_gc_requested_.CompareAndSetStrongRelaxed(max_gc_requested, 
                                                      observed_gc_num + 1)) {
        //Create ConcurrentGCTask and add to task_processor_.
        task_processor_->AddTask(self, new ConcurrentGCTask(NanoTime(),  
                                                            cause,
                                                            force_full,
                                                            observed_gc_num + 1));
      }
      ……
      return true;
    }
    return false;
  }
  return true;  
}

class Heap::ConcurrentGCTask : public HeapTask {
 public:
  ConcurrentGCTask(uint64_t target_time, GcCause cause, bool force_full, uint32_t gc_num)
      : HeapTask(target_time), cause_(cause), force_full_(force_full), my_gc_num_(gc_num) {}
  void Run(Thread* self) override { 
    Runtime* runtime = Runtime::Current();
    gc::Heap* heap = runtime->GetHeap();
    ……
    //Triger GC
    heap->ConcurrentGC(self, cause_, force_full_, my_gc_num_);
    ……
  }
};
```

In this method, a ConcurrentGCTask will be created, and the AddTask method of the task\_processor\_ object will be called to add it to the tasks collection. Immediately afterwards, the Run function of ConcurrentGCTask will be executed, triggering concurrent GC. The mechanism and principle of GC will not be elaborated on here. Let's continue to look at the implementation scheme of GC suppression.&#x20;

## 5.4.2 Scheme for Suppressing GC Execution

If you want to suppress the execution of GC, you need to intercept the Run function of ConcurrentGCTask and then execute our custom sleep logic. In Chapter 3 "Practical Memory Optimization", we have already learned about PLT Hook, a Native Hook technique. However, this Hook technique is not applicable in the current scenario because the Run function of ConcurrentGCTask is not a function of an external library but an internal function, so there is no jump logic in the plt table. Therefore, I introduce another Native Hook technique here: Inline Hook, to implement the Hook operation of internal functions in the so library.&#x20;

### 1. Inline Hook Technology

Inline Hook is a type of hook method that changes the execution flow of a program by dynamically modifying the assembly instructions in memory during program runtime. The basic idea is to insert a jump instruction into an existing code segment to redirect the execution flow of the code to our own function. The main implementation process of this solution consists of three steps:

1\) Replace the assembly instructions at the head of the original function with jump instructions that can jump to our custom function

2\) After the logical execution of the custom function is completed, restore the instructions of the original function that has been overwritten by the jump instruction to ensure the integrity of the original function.

3\) Finally, call the original function again in the custom function

The steps of this process are shown in Figure 5-8, and it can be seen that the idea of this process is not very complex.&#x20;

![Figure 5-8 Flow Logic of Inline Hook](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter5_img_8.png)

Once you understand the process and concept of this technology, you can then see how to modify the Run method of the ConcurrentGCTask object through Inline Hook. This method is located in libart.so. In systems below Android 10, libart.so is located in the /system/lib directory, while in Android 10 and above, libart.so is located in the /apex/com.android.art/lib64/ directory. My test device is an Android device running Android 10 or above, so by using the following add pull command, libart.so can be pulled to the local machine for further analysis later.

```c++
adb pull /apex/com.android.art/lib64/libart.so libart.so
```

In the previous chapter, we have already learned the usefulness of the objdump tool. Therefore, by using the command "objdump -d libart.so", we can view the assembly instructions of libart.so. Some of the assembly instructions for the Run method of the ConcurrentGCTask object are shown in Figure 5-9, where we can see that the offset address of the Run function is 0x46f518.&#x20;

![Figure 5-9 Assembly code for ConcurrentGCTask](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter5_img_9.png)

If you want to replace the instruction at the head with an instruction that jumps to our own function, you need to first understand the jump instructions on the ARM platform.&#x20;

We can perform jumps in two ways. One is relative jump, which means that during the jump, it will be based on the current instruction address plus the position to jump, and the relative jump is implemented through the 'B' instruction. Since our Hook function and the Run function to be intercepted are not in the same so library, the relative jump method cannot be achieved. The other is absolute jump, which means directly jumping to a given address. The implementation method is through the 'LDR' assignment instruction, which directly assigns the target address to jump to the PC register. We know that the role of the PC register is to record the address of the next instruction to be executed, so after assigning an address to the PC, the next instruction will start executing from this address. Therefore, the purpose of jumping to our custom function can be achieved through the absolute jump method. The instruction implementation of this method is as follows:

```c++
LDR r15, [PC, #-4]
xxx //address
```

In the above code, the first line of instruction is "LDR r15, \[PC, #-4]", and the corresponding hexadecimal machine code for this instruction is "0xe51ff004". Here, "LDR" means loading data into a register, "r15" represents the PC register on the ARM32-bit platform, and "\[PC, #-4]" means loading data from the address that is 4 bytes subtracted from the address pointed to by the PC register, which is the location of the second line of instruction. The instruction on the second line is an address value, which is the address value of the function to be jumped to. Next, let's take a look at the code implementation of the solution.&#x20;

```c++
int originCode1;
int originCode2;
uintptr_t targetFunc;

void hook(void *target, void *new_func) { 
    // Get the base address of libart.so in memory
    uintptr_t artBaseAddress = findArtBaseAddress();
    // 0x46f518 is the offset address of the Run function of the ConcurrentGCTask object
    targetFunc = 0x46f518 + artBaseAddress; 
    mprotect(page, PAGE_SIZE, PROT_READ | PROT_WRITE)
    // Save the first two instructions of the function
    originCode1 = ((uint32_t *) targetFunc)[0];
    originCode2 = ((uint32_t *) targetFunc)[1];
    // Modify the memory attributes to readable and writable
    mprotect(targetFunc, 8, PROT_READ | PROT_WRITE | PROT_EXEC);
    // Replace the first instruction with "LDR r15, [PC, #-4]"
    ((uint32_t *) targetFunc)[0] = 0xe51ff004;
    // Replace the second instruction with the function to jump to
     ((uint32_t *) targetFunc)[1] = (uint32_t)new_func;
}
```

Through the above code, we can achieve the goal of jumping to our custom function new\_func when the Run method of ConcurrentGCTask is executed. At this point, we can sleep for 2 seconds in the custom function new\_func, then restore the instructions of the original function and continue to execute the logic of the original function. Since the Run function of ConcurrentGCTask has a Thread parameter, we also need to pass the Thread\* parameter when executing the original method. The code implementation is as follows:&#x20;

```c++
void new_func(){
    // Sleep for 2 seconds
    sleep(2000);
    // Restore the original instructions
    unHook(); 
    // Get the current thread
    pthread_t pthread = pthread_self();
    // Execute the original Run method
    ((void (*)(Thread*))targetFunc)((Thread*)pthread); 
}

// Restore the original function's bytes
void unHook(void *target) {
    // Modify memory attributes to readable and writable
    mprotect(targetFunc, 8, PROT_READ | PROT_WRITE | PROT_EXEC);
    // Restore the original first two instructions of the function
    ((uint32_t *) targetFunc)[0] = originCode1;  
    ((uint32_t *) targetFunc)[1] = originCode2;
}
```

Through the above process, we have completed the Hook operation on the Run function in the ConcurrentGCTask object using Inline Hook and achieved the ability to suppress GC. However, there is still an unsolved problem in the process, which is the address of the Run function. In the above code, the address of the Run function was directly known to be 0xe51ff004 through the assembly code of the libart.so library. But in the online environment, for compatibility and stability reasons, it is necessary to dynamically obtain this address.

### 2. Get the address of the target function

Previously, I mentioned the Run method of the ConcurrentGCTask object, whose full name is `art::gc::Heap::ConcurrentGCTask::Run(art::Thread*)`. The name of this method in the assembly code is `_ZN3art2gc4Heap16ConcurrentGCTask3RunEPNS_6ThreadE`, and this name is the symbol name of the Run function. In the previous chapter, we have learned the generation rules of symbols, so at this point, we can understand this symbol. Its generation rules are as follows:

* \_ZN is a prefix indicating that this is a C++ function symbol.

* 3art indicates that the name of the first namespace is art, and 3 is the length of the name.

* 2gc indicates that the name of the second namespace is gc, and 2 is the length of the name.

* 4Heap indicates that the name of the third namespace is heap, and 4 is the length of the name.

* 14ConcurrentGCTask indicates that the name of the class is ConcurrentGCTask, and 14 is the length of the name.

* 3Run indicates that the name of the function is Run, and 3 is the length of the name.

* EPNS\_6ThreadE indicates that the parameter of the function is a pointer to the art::Thread class, where E is the end marker of the parameter list, P is the pointer marker, NS\_6Thread represents the nested name of the art::Thread class, N is the start marker of the nested name, S\_ indicates repeating the previously appeared name, 6Thread indicates that the class name is Thread, and 6 is the length of the name.

According to this rule, we can know the symbol corresponding to the `art::gc::Heap::ConcurrentGCTask::Run(art::Thread*)` function, but we cannot ensure that this symbol is retained in the libart.so library. In the libart.so library file, many methods have symbols. The reason for retaining these symbols is mainly for debugging or exception location. Through the symbols, we can find the corresponding function address. However, there are also many methods that do not have symbols. The reason why we do not adopt the solution of adding our custom task to the TaskProcessor to suppress GC, as mentioned earlier, is mainly because the `add` method of the TaskProcessor object does not have a symbol, so we cannot obtain and execute this function address.

By using the readelf tool provided in the Android NDK, execute the command "readelf -s libart.so" to read all symbols in libart. It can be seen that there are a large number of symbols in this library, more than 20,000. By searching, the symbol corresponding to the Run function can be found, which is located at line 16846 in the symbol table.&#x20;

```c
Symbol table '.symtab' contains 26281 entries:
   Num:    Value  Size Type    Bind   Vis       Ndx Name
     0: 00000000     0 NOTYPE  LOCAL  DEFAULT   UND 
     1: 00000000     0 FILE    LOCAL  DEFAULT   ABS crtbegin_so.c
     2: 000acf34     0 NOTYPE  LOCAL  DEFAULT    12 $a
     3: 000acf50     0 NOTYPE  LOCAL  DEFAULT    12 $d
     4: 0045a410     0 NOTYPE  LOCAL  DEFAULT    18 $d
     5: 00462000     0 NOTYPE  LOCAL  DEFAULT    23 $d
    ……
 16846: 001b0ff1    36 FUNC    LOCAL  HIDDEN     12 _ZN3art2gc4Heap16ConcurrentGCTask3RunEPNS_6ThreadE
    ……
```

The Linux system provides the dlsym function, which can directly obtain the address of a function based on its symbol name. Simply include the dlfcn.h Header File, and the usage is as follows:&#x20;

```c
#include <dlfcn.h>
void* findRunAddressSymbol() {
    void* libraryHandle = dlopen("libart.so", RTLD_LAZY);
    if (libraryHandle == nullptr) {
        return nullptr;
    }
    // find symbol
    void* symbolAddress = dlsym(libraryHandle, 
            "_ZN3art2gc4Heap16ConcurrentGCTask3RunEPNS_6ThreadE");
    if (symbolAddress == nullptr) {
        // fail
        dlclose(libraryHandle);  
        return nullptr;
    }
    // return the address of the symbol
    return symbolAddress;
}
```

In the above code, the file handle of libart.so is obtained through the dlopen function, and then the dlsym function is called, passing in the handle and the symbol name of the function, and the address can be directly obtained. The code implementation is relatively simple, but when we actually run it, we will find that on Android 7 and above models, the dlopen function will fail to open the so library. This is because in Android 7 and above versions, the Android system, for security reasons, has prohibited the ability of C++ code to directly call the dlopen function to open system libraries.&#x20;

However, we can use some unconventional technical means to break through this limitation. Since these technologies are relatively complex, I will introduce the principle of the technology here. Readers only need to have a general understanding of the process and principle, and are not required to fully master it.

The dlopen function is located in the libdl.cpp file of the bionic library (libc.so), and the function code is shown in Figure 5-10. Among them, the \_\_builtin\_return\_address(0) method returns the address of the calling function, which is the address of caller\_addr. In the subsequent process, it will be determined whether caller\_addr is from a system library or a non-system library address. If it is a non-system library address, that is, the address of a user program, an exception will be thrown.&#x20;

![Figure 5-10 source code fo dlopen function](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter5_img_10.png)

The value returned by the `__builtin_ereturn_address(0)` function is actually the value of the LR register (Link Register), which is used to store the return address when a function is called, so that the function can return to the calling point after execution. If we can modify the value of the LR register to the address of the system library, when the dlopen function is called, the system will think that the caller is a system caller, and thus the dlopen function can be used normally.&#x20;

However, bypassing the system's restrictions on dlopen by modifying the value of the LR register is complex and can easily lead to system crashes due to exceptions. Therefore, we can use online open-source libraries to bypass the restrictions. Here I introduce [ndk\_dlopen](https://github.com/Rprop/ndk_dlopen), an open-source library, to implement the capabilities of dlopen and dlsym. The implementation principle of this open-source library is similar. By reading the source code of this library, as shown in Figure 5-11, it can be seen that it takes (\*env)->FatalError to replace the address of the LR register, thereby achieving the purpose of bypassing system checks.&#x20;

![Figure 5-11 ndk\_dlopen open source library part of the code](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter5_img_11.png)

&#x20;ndk\_dlopen open source tool is also very easy to use, and the usage code is as follows:

```c++
// Initialize ndk_dlopen
ndk_init(env);

// Open the dynamic library libart.so in RTLD_NOW mode to obtain a handle. 
// RTLD_NOW mode immediately resolves all symbols.
void *handle = ndk_dlopen("libart.so", RTLD_NOW);

// Get the address through the symbol
void *runAddress = ndk_dlsym(handle, "_ZN3art2gc4Heap16ConcurrentGCTask3RunEPNS_6ThreadE");
```

With just three simple lines of code, we successfully obtained the address of the Run function of ConcurrentGCTask. At this point, all we need to do is insert our own code, modify this function to make it sleep, and we can successfully block the HeapTaskDaemon thread.

By this point, we have successfully suppressed the logic of the HeapTaskDaemon thread executing GC. However, some readers may worry that suppressing GC could lead to issues such as an increased OOM rate. In fact, this is not the case. We do not need to suppress GC for an extended period; we only need to suppress it for a few seconds during startup, when the List component is scrolling, and when a page is opened. Moreover, starting from Android 8, the system has also added the logic to suppress GC for 2 seconds during app startup.&#x20;

Besides the solution introduced here , there are many other solutions to suppress GC, such as using the characteristics of virtual functions to hook the Run function. For example, we can also analyze one by one the logic executed by the Run function in HeapTask to find out if there are any callback methods in this logic, and if so, we can directly perform the sleep operation in the callback method. Taking the previously mentioned ClearedReferenceTask as an example, it will execute the Java layer method ReferenceQueue.add in the Run function, so we can perform the sleep operation in this add method to suppress GC. Readers who are interested can also explore more feasible solutions on their own.&#x20;

### 3. Use an open source framework&#x20;

In the above explanation, the implementation of Inline Hook was presented using the ARM 32 platform as an example. In actual online usage, we need to be compatible with multiple platforms such as ARM32, ARM64, X64, and x32, and also need to handle exceptions and fallback properly, all of which involve a significant amount of work. Therefore, I suggest that when using it in a real environment, try to choose a stable open-source library. There are many open-source frameworks for Inline Hook, and the one I uses most frequently is [ShadowHook](https://github.com/bytedance/android-inline-hook), an Inline Hook framework open-sourced by ByteDance, which has been tested in numerous online projects and is therefore relatively stable.&#x20;

The detailed usage will not be introduced here, as it has been explained in great detail in the project's documentation. Readers can refer to it themselves. After importing the ShadowHook library into the project, only one shadowhook\_hook\_sym\_name method is needed to complete the Hook operation on the ConcurrentGCTask::Run function. The code is as follows.&#x20;

```c
shadowhook_hook_sym_name("libart.so",
                        "_ZN3art2gc4Heap16ConcurrentGCTask3RunEPNS_6ThreadE",
                        (void *)newFunc,
                        nullptr);
```

After sleeping in our custom Hook function, we can complete the call to the original function by executing the SHADOWHOOK\_CALL\_PREV method provided by ShadowHook. The code implementation is as follows.

```c
void newFunc(Thread* self){
    SHADOWHOOK_STACK_SCOPE();
    sleep(2000);
    //execute original Run method
    SHADOWHOOK_CALL_PREV(self);
}
```

# 5.5 Cache Strategy Optimization

Caching is crucial for improving speed, but the cache capacity is limited. Therefore, when using caching to enhance performance, it is always necessary to consider how to maximize the cache hit rate within the limited capacity. To improve the cache hit rate, it is necessary to design more appropriate cache eviction strategies based on the business scenario.&#x20;

I will use WhatsApp as an example to explain the selection of cache eviction strategies. Communication apps generally have a conversation page, and external content pages shared in chats can also be opened on the conversation page. Whether it's the images on the conversation page or those on the external content pages, they are often cached to improve the loading speed of images the next time. If this scenario runs on a low-end device with limited memory, due to the small amount of device memory, the capacity of cached images is limited. To enable readers to better understand the cache eviction strategy, here we assume that only 5 images can be cached. When we open a conversation page and cache the images on it, if we then continue to open an externally shared content item containing a large number of images on the conversation page, we will quickly reach the capacity limit of the image cache container. At this point, we will face the choice of what strategy to use to evict images from the cache.&#x20;

## 5.5.1 Common Eviction Strategies

Before choosing the optimal eviction strategy, I will first introduce some commonly used caching strategies here.&#x20;

* LRU: Least Recently Used eviction strategy. When the cache space is insufficient, it prioritizes evicting data that has not been accessed for the longest time. As shown in Figure 5-12, in this case, it will first evict the images in the session page at the end of the queue and move the images in the external content page to the head of the queue.

![Figure 5-12 The process of eliminating images in the cache through LRU strategy](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter5_img_12.png)

* LFU: Least Frequently Used Eviction Strategy, which evicts the data with the lowest access frequency based on the frequency of data access. As shown in Figure 5-13, at this time, since the images in the session page have a higher usage frequency, they will not be evicted first; instead, the images on the official account page with a lower usage frequency will be evicted first.

* &#x20;LFU: Least Frequently Used Eviction Strategy, which evicts the data with the lowest access frequency based on the frequency of data access. As shown in Figure 5-13, at this time, since the images in the session page have a higher usage frequency, they will not be evicted first; instead, the images in the external content pages with a lower usage frequency will be evicted first.&#x20;

![Figure 5-13 The process of eliminating images in the cache through the LFR elimination strategy](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter5_img_13.png)

* FIFO: First-In-First-Out eviction strategy, which evicts the data that entered the cache earliest based on the order of data entry into the cache. As shown in Figure 5-14, it will evict the image most recently placed in the cache.

![Figure 5-14 The process of eliminating images in the cache through the FIFO elimination strategy](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter5_img_14.png)

* Random: Random replacement eviction strategy, randomly selects a piece of data for eviction, and needs to ensure that all data have an equal probability of being replaced. This eviction strategy has the lowest performance overhead, but the hit rate is not high. It is mainly used in scenarios where system resources are limited, performance overhead has strict requirements, and the cache hit rate requirement is not high or the access pattern is difficult to predict.

Based on the commonly used cache eviction strategies introduced above, if we choose to use the LruCache provided by Android by default, which is the Least Recently Used (LRU) eviction strategy cache, the images on the conversation page in the cache will easily be replaced by the images on external content pages. This naturally slows down the page display speed of the frequently used conversation page significantly, while we rarely open these external content pages again after viewing them once, so even if the images are cached, it doesn't have much value. Therefore, this strategy is clearly not suitable for the current scenario. On the other hand, the Least Frequently Used (LFU) eviction strategy is clearly the most suitable for the current scenario. This strategy can retain hot data, where the images on the conversation page are hot data, while the images on external content pages are non-hot data and will be evicted first.

## 5.5.2 LFUCache Cache

Once the optimal cache eviction strategy is determined based on the characteristics of the scenario, implementing it is not too complicated. The steps for designing the cache of LFUCache are as follows:&#x20;

1. First, determine the data structure that the cache container needs to maintain. For LFUCache, there are mainly three mapping tables&#x20;

   * is used to store a hash table that maps keys to values, facilitating quick data retrieval.&#x20;

   * is used to store a hash table that maps keys to freq (frequency), facilitating the quick update of the usage frequency of data&#x20;

   * is a hash table used to store the mapping from freq (frequency) to a list of keys, which maps frequencies to the set of all keys with that frequency. This set is usually sorted in insertion order so that the earliest inserted key can be quickly found. When a key needs to be removed, this table can be used to quickly find the key with the lowest frequency and earliest insertion time.

   In addition to these three mapping tables, a field is needed to record the capacity of the cache, and a field is needed to record the minimum frequency for quickly locating and evicting data. The structure code is as follows.&#x20;

   ```java
   class LFUCache {
       // Mapping table from key to value
       HashMap<Integer, Integer> keyToVal;
       // Mapping table from key to frequency
       HashMap<Integer, Integer> keyToFreq;
       // Mapping table from frequency to key list
       HashMap<Integer, LinkedHashSet<Integer>> freqToKeys;
       // Record the minimum frequency
       int minFreq;
       // Record the maximum capacity of the LFU cache
       int cap;

       public LFUCache(int capacity) {
           keyToVal = new HashMap<>();
           keyToFreq = new HashMap<>();
           freqToKeys = new HashMap<>();
           this.cap = capacity;
           this.minFreq = 0;
       }

       public int get(int key) {}

       public void put(int key, int val) {}
   }
   ```

2. The second step is to design the get and put methods in the cache. For the get method, it is necessary to first look up the data in the hash table from key to value. If it does not exist, return null; if it exists, return the value, and call increaseFreq to update the usage frequency of the data and sort it according to the frequency.&#x20;

   ```java
   public int get(int key) {
       if (!keyToVal.containsKey(key)) {
           return -1;
       }
       // Increase the frequency of the key
       increaseFreq(key);
       return keyToVal.get(key);
   }

   private void increaseFreq(int key) {
       int freq = keyToFreq.get(key);
       // Update KF table
       keyToFreq.put(key, freq + 1);
       // Remove key from the list corresponding to freq
       freqToKeys.get(freq).remove(key);
       // Add key to the list corresponding to freq + 1
       freqToKeys.putIfAbsent(freq + 1, new LinkedHashSet<>());
       freqToKeys.get(freq + 1).add(key);
       // If the list for freq becomes empty, remove this freq
       if (freqToKeys.get(freq).isEmpty()) {
           freqToKeys.remove(freq);
           // If this freq happens to be minFreq, update minFreq
           if (freq == this.minFreq) {
               this.minFreq++;
           }
       }
   }
   ```

3. For the put method, it is necessary to first look up the data in the hash table from key to value based on the key. If the data does not exist, create a new data node, add it to the hash tables from key to value and from key to freq, and set its usage frequency to 1. If the cache is full at this time, then delete the data corresponding to the minimum frequency. If the data exists, call the increaseFreq method to increment the frequency corresponding to the data by one and reorder it according to the frequency.&#x20;

   ```java
   public void put(int key, int val) {
       if (this.cap <= 0) return;

       // If the key already exists, simply update the corresponding val
       if (keyToVal.containsKey(key)) {
           keyToVal.put(key, val);
           // Increment the freq of the key
           increaseFreq(key);
           return;
       }

       // Key doesn't exist, need to insert. If capacity is full, evict a key with the minimum freq
       if (this.cap <= keyToVal.size()) {
           removeMinFreqKey();
       }

       // Insert key and val, corresponding freq is 1
       keyToVal.put(key, val);
       // Insert into KF table
       keyToFreq.put(key, 1);
       // Insert into FK table
       freqToKeys.putIfAbsent(1, new LinkedHashSet<>());
       freqToKeys.get(1).add(key);
       // After inserting a new key, the minimum freq must be 1
       this.minFreq = 1;
   }

   private void removeMinFreqKey() {
       // List of keys with the minimum freq
       LinkedHashSet<Integer> keyList = freqToKeys.get(this.minFreq);
       // The key that was inserted first among them is the one to be evicted
       int deletedKey = keyList.iterator().next();
       // Update FK table
       keyList.remove(deletedKey);
       if (keyList.isEmpty()) {
           freqToKeys.remove(this.minFreq);
       }
       // Update KV table
       keyToVal.remove(deletedKey);
       // Update KF table
       keyToFreq.remove(deletedKey);
   }
   ```

By now, a simple LFU (Least Frequently Used) cache container with a minimum usage frequency eviction policy has been designed. If you are a careful reader, you may have noticed a drawback of LFUCache from the above code, that is, the longer the data has been in the cache, the higher its usage frequency will be, and it may never be evicted in the end. Therefore, we also need to add some auxiliary eviction strategies, such as incorporating the mechanism mentioned in memory optimization, which is to evict all caches when the memory reaches a threshold. When facing different business scenarios, we need to fully evaluate and then select appropriate cache strategies. In some complex scenarios, multiple cache strategies can also be combined to design a multi-level, multi-strategy cache container.&#x20;

# 5.6 Dex File Reordering

In the previous chapter, we learned what a cache is. Its read and write speeds are much higher than those of memory. If we can fully improve the utilization of the cache, we can greatly enhance the performance of the program. Therefore, I introduce a classic optimization solution here to improve cache efficiency: Dex file reordering.&#x20;

## 5.6.1 Principle of Locality

When the cache reads data, it will read data up to the size of the cache line at once. If the size of the cache line is 64 bytes, that means even if the data the CPU needs may be just 1 byte, the cache will still read 64 bytes of data. To ensure that this extra read data is likely to be used by the CPU in the future, the computer uses the principle of spatial locality. Under this principle, during the startup process of a program, when an object's data is first used, since the cache does not have this data, it needs to read this object's data from the main memory, and it will also read some objects immediately following this object until the amount of read data reaches 64 bytes. If the objects immediately adjacent to this object in memory are the ones that will be used next, the cache does not need to read data multiple times, and the CPU also reduces the time waiting for data to be read, enabling it to execute program instructions faster and allowing the program to run more quickly.&#x20;

When our project is compiled into an APK package, all class files will be integrated and placed in individual dex files. At this time, the order of class files in the dex files is not arranged according to the program execution order, because the compilation process does not know the execution order of class files either. If we can run the program once in advance, collect the usage order of class objects, and then adjust the order of class files in the dex files according to this order, we can fully leverage the characteristics of the locality principle and accelerate the program execution speed. We usually use this optimization to improve the startup speed, because the execution order of class files during the startup phase is often fixed, so it will have better optimization results.&#x20;

To achieve this optimization, after instrumenting each object, the program needs to be run once to collect the execution order of the objects. Then, according to the execution order of the objects and the data structure of the dex file, the class files inside are reordered. Implementing this set of processes is relatively complex and cumbersome. In actual development, we hardly ever implement this solution ourselves. Instead, we use Facebook's open-source solution:[Redex](https://github.com/facebook/redex) to achieve this optimization. Therefore, I mainly introduce the usage of Redex here.

## 5.6.2 Redex Usage Process

First, download the Redex project source code and related environment. The instructions are as follows.

```shell
// clone redex
git clone https://github.com/facebook/redex.git
// download env
xcode-select --install
brew install autoconf automake libtool python3
brew install boost jsoncpp
```

Then open the config file under the config directory of Redex, add the InterDexPass field in the configuration file to enable dex class rearrangement optimization, and add the coldstart\_classes field to specify the directory for the order of class files.&#x20;

```json
//open redex config
cd redex/config/
vim default.config

//Add the InterDexPass enable setting in the configuration file, and include the new `coldstart_classes` to specify the class loading order.
{
  "redex" : {
    "passes" : [
      "ReBindRefsPass",
      "BridgePass",
      "SynthPass",
      "FinalInlinePass",
      "DelSuperPass",
      "SingleImplPass",
      "SimpleInlinePass",
      "StaticReloPass",
      "RemoveEmptyClassesPass",
      "ShortenSrcStringsPass",
      "InterDexPass"
    ],
   "coldstart_classes":"app_list_of_classes.txt" //class invocation order list
  }
}
```

Then enable configuration and compile using the command "./configure && make"&#x20;

```shell
// start compile
cd redex
autoreconf -ivf && ./configure && make
sudo make install
```

Next, we need to output the classes used during the startup process and their order to the "app\_list\_of\_classes.txt" file. We can obtain the list of startup class loading order by following these steps:

1\) Start the application and obtain the application's pid via the ps command

```json
adb shell ps | grep app_packageName
```

2\) Capture the memory snapshot of the application

```json
adb shell am dumpheap pid /data/local/tmp/SOMEDUMP.hprof
```

3\) Analyze the captured memory snapshot using redex scripts and output it to the "app\_list\_of\_classes.txt" file specified by coldstart\_classes

```shell
# Pull the memory snapshot file to the local computer
adb pull /data/local/tmp/SOMEDUMP.hprof local_path

# Use the Python script provided by redex to parse the heap memory and generate a class loading order list
python redex/tools/hprof/dump_classes_from_hprof.py --hprof SOMEDUMP.hprof > app_list_of_classes.txt
```

4\) Generate a new package using the tools provided by Redex, and finally re-sign the new package.

```json
//execute redex，generate a new apk
redex input.apk -o output.apk
```

Redex has many optimization items, and dex class file rearrangement is just one of them. It is relatively simple to use and is an optimization that is easy to implement. According to the statistical experimental data, after dex class file rearrangement, the cold start speed can be increased by about 10%.

# 5.7 Increase Core Thread Priority

To increase the priority of a thread, we first need to understand the rules of thread priority. Processes in Linux are divided into two types: real-time processes and normal processes. Real-time processes generally use the RTPRI (Real Time Priority) value to describe their priority, with a range from 0 to 99. Normal processes generally use the Nice value to describe their priority, with a range from -20 to 19. Since a thread is essentially a process, the priority rules also apply to threads. We can enter the shell interface of the mobile phone and execute "ps -A -l" to view the Nice and Prio values of all processes. Some data is shown as below, where it can be seen that the Nice value of the main thread of the example program defaults to 0, and the Nice value of the rendering thread defaults to -4.

```plain&#x20;text
ps -A -l

USER       PID   PPID    VSIZE  RSS     PRIO  NICE  RTPRI SCHED  WCHAN    PC          NAME
root       393   1      1554500 5256     20     0    0     0    ffffffff 0000 S      zygote
system     762   328    338336  9844     12    -8    0     0    ffffffff 00000000 S surfaceflinger
……
// The main thread and rendering thread of the test program
u0_a45    16632 393     2401604 60140    20     0    0     0    ffffffff 00000000 S com.example.android_performance
u0_a45    16725 16632   2401604 60140    16    -4    0     0    ffffffff 00000000 S RenderThread
……
```

In Android, only some underlying core processes are real-time processes, such as audio and video service processes, while most processes are ordinary processes. We cannot adjust ordinary processes to real-time processes, nor can we adjust real-time processes to ordinary processes; only the operating system has this privilege. However, there is an exception: on a rooted phone, by modifying the sys.use\_fifo\_ui field in the build.prop file under the /system directory to 1, the main thread and rendering thread of an application can be adjusted to real-time processes. However, this requires a rooted device to operate, and on normal devices, this value is always 0, so this method is not universal.&#x20;

## 5.7.1 Methods of Adjusting Thread Priority

All threads in the application belong to the level of normal processes, so regarding thread priority, the only thing that can be manipulated is to modify the Nice value of the thread. Generally, there are two ways to adjust the Nice value of a thread, which are as follows.

1. Change the thread priority through the Process.setThreadPriority function.

2. Change the priority of a thread through the Thread.setPriority function

Let's take a look at the differences between these two methods separately.

**1) Process.setThreadPriority function**

The first way is through the following interfaces provided in the Android system.&#x20;

```java
public static native void setThreadPriority(int tid, int priority);
```

Among them, the input parameter tid is the thread ID, which can be omitted and will default to the current thread. The input parameter priority is the Nice value, which can be any value between -20 and 19, but it is recommended to directly use the priority definition constants provided by Android, as shown in the following table. Such code has higher readability, and directly passing in a numerical value is not conducive to code understanding.

| **Priority Definition**                   | **Nice value&#x20;** | **Usage Scenarios**                         |
| ----------------------------------------- | -------------------- | ------------------------------------------- |
| Process.THREAD\_PRIORITY\_DEFAULT         | 0                    | Default Priority                            |
| Process.THREAD\_PRIORITY\_LOWEST          | 19                   | Lowest Priority                             |
| Process.THREAD\_PRIORITY\_BACKGROUND      | 10                   | Recommended priority for background threads |
| Process.THREAD\_PRIORITY\_LESS\_FAVORABLE | 1                    | Slightly lower than default                 |
| Process.THREAD\_PRIORITY\_MORE\_FAVORABLE | -1                   | Slightly higher than default                |
| Process.THREAD\_PRIORITY\_FOREGROUND      | -2                   | Foreground Thread Priority                  |
| Process.THREAD\_PRIORITY\_DISPLAY         | -4                   | Display thread recommended priority         |
| Process.THREAD\_PRIORITY\_URGENT\_DISPLAY | -8                   | Displays the highest level of the thread    |
| Process.THREAD\_PRIORITY\_AUDIO           | -16                  | Audio thread recommended priority           |
| Process.THREAD\_PRIORITY\_URGENT\_AUDIO   | -19                  | Audio thread highest priority               |

Before any adjustment, the Nice value of the main thread defaults to 0, and the default Nice value of the rendering thread is -4. Therefore, we can further increase the priority of the main thread and the rendering thread to improve the responsiveness of these two threads.

**2) Thread.setPriority function**

The second way is through the following interface provided by Java.&#x20;

```java
public final void setPriority(int priority)
```

The input parameter priority of the above interface is not the Nice value, but rather the definition and rules of thread priority provided by Java itself. However, ultimately, these rules will be converted into the corresponding Nice value sizes. The rules for the priority provided by Java threads and their corresponding Nice values are shown in the following table. This method can set fewer priorities, is not very flexible, and is not conducive to understanding. Moreover, due to system timing issues, when setting the priority of a child thread, it may be set to the main thread's priority because the child thread was not created successfully, which can lead to abnormal priority of the main thread. Therefore, I recommend using the first method to set thread priority and avoiding the second method.&#x20;

| Priority Definition      | **Nice value&#x20;** |
| ------------------------ | -------------------- |
| Thread.MAX\_PRIORITY（10） | -8                   |
| Thread.MIN\_PRIORITY（0）  | 19                   |
| Thread.NORM\_PRIORITY（5） | 0                    |

## 5.7.2 Threads that need priority adjustment

Having understood the way to adjust thread priority, let's take a look at which threads need adjustment. In the most common cases, there are mainly two threads that need priority elevation: the main thread and the rendering thread (RenderThread).&#x20;

Why do we need to adjust these two threads? Because these two threads are very important for any application. Starting from Android 5, the main thread is only responsible for the measurement (Measure) and layout (Layout) work of layout files, while the rendering work is assigned to the rendering thread. Only when these two threads work in coordination can the application's interface be displayed normally. Therefore, by increasing the priority of these two threads, they can obtain more CPU time, and the page display speed will naturally be faster.

Adjusting the priority of the main thread is relatively simple. Directly call Process.setThreadPriority(-19) in the attach lifecycle of the Application, without needing to pass in the id of the main thread, and the main thread will be defaulted to the highest level of priority. However, if you want to adjust the priority of the rendering thread, you first need to know the thread id of the rendering thread. Let's take a look at how to find the thread id of the rendering thread below.&#x20;

Information about threads in the application is recorded in the file "/proc/pid/task". Taking the data of process 11548 as an example, as shown below, it can be seen that the task file records all threads in the current process.&#x20;

```c++
/proc/11548/task $ ls
11548  11554  11556  11558  11560  11564  11566  12879  12883  12890  12917  14501  14617  15596  15598  15600  15602  15614
11553  11555  11557  11559  11562  11565  12878  12881  12884  12894  12920  14555  15585  15597  15599  15601  15613  15617
```

By checking the stat node of the thread in this directory, we can specifically view detailed information about the thread, such as Name, pid, etc. The main thread ID of process 11548 is 11548, and its stat data is as follows:&#x20;

```c++
blueline:/proc/11548/task $ cat 11548/stat                                      
11548 (example.android_performance) S 1271 1271 0 0 -1 1077952832 12835 0 1617 0 52 19 0 0 10 -10 36 0 59569858 15359959040 23690 18446744073709551615 1 1 0 0 0 0 4612 1 1073775864 0 0 0 17 4 0 0 0 0 0 0 0 0 0 0 0 0 0
```

In the previous Section 5.1 "Process CPU Consumption", the meaning of each parameter in the stat data has been introduced in detail, where the first parameter represents the thread ID and the second parameter represents the thread name. Therefore, we only need to iterate through this file and search for the thread named "render" to find the ID of the rendering thread. The following is the specific code implementation.&#x20;

```java
public static int getRenderThreadTid() {
    File taskParent = new File("/proc/" + Process.myPid() + "/task/");
    if (taskParent.isDirectory()) {
        File[] taskFiles = taskParent.listFiles();
        if (taskFiles != null) {
            for (File taskFile : taskFiles) {        
                BufferedReader br = null;
                String line= "";
                try {
                    br = new BufferedReader(
                            new FileReader(taskFile.getPath() + "/stat"), 100);
                    line = br.readLine();
                    if (!line.isEmpty()) {
                        String param[] = line.split(" ");
                        if (param.length < 2) {
                            continue;
                        }
                        //read thread name
                        String threadName = param[1];
                        //the first data is tid
                        if (threadName.equals("(RenderThread)")) {
                            return Integer.parseInt(param[0]);
                        }
                    }
                } catch (Throwable throwable) {
                    
                } finally {
                    if (br != null) {
                        br.close();
                    }
                }
            }
        }
    }
    return -1;
}
```

After obtaining the ID of the rendering thread, simply call Process.setThreadPriority(pid, -19) to set the rendering thread to the highest priority.&#x20;

Of course, these two are not the only threads whose priority needs to be increased. We can increase the priority of core threads and decrease the priority of other non-core threads according to business requirements. This operation can be uniformly adjusted through the thread factory in the thread pool. Only by increasing the priority of core threads and decreasing the priority of non-core threads in combination can we fully enhance the effectiveness of this optimization of adjusting thread priority and more effectively improve the speed of the application.

# 5.8 Thread Pool Optimization

Threads are the basic units for executing tasks, and their importance is self-evident. By using threads reasonably, we can give full play to the CPU's performance and greatly enhance the program experience. How can we use threads more reasonably? This requires us to do a lot of things.

For example, it is necessary to control the number of threads in the program within an appropriate range, neither too many nor too few. Too many threads will waste excessive CPU resources on thread scheduling and may cause performance issues such as laggy in the main thread of the program due to insufficient CPU resources. On the other hand, too few threads cannot fully utilize the CPU's performance, which also leads to poor performance experience of the program. For example, it is necessary to minimize the performance loss caused by the threads themselves. Frequent creation and destruction of threads, as well as frequent state switching, will result in excessive CPU consumption.

Using threads reasonably is not an easy task, but we can achieve more reasonable use of threads through thread pools. Therefore, the importance of thread pools cannot be ignored, and it is knowledge that every developer needs to master. In this chapter, let's take a look at how to correctly use and optimize thread pools to fully improve the scheduling efficiency of thread pools and make the program faster and more fluent.

## 5.8.1 Default Thread Pool Creation Method

Let's first take a look at how to create a thread pool. As a fundamental capability, the Java library provides the default Executors utility class to create thread pools. This utility class contains more than a dozen static methods named "newThreadPool" for creating thread pools, as shown in Figure 5-15. If a new developer has limited knowledge of thread pools, they will surely be troubled by which creation method to choose. So let's take a look at how these methods provided in the Executors object of the Java library create thread pools.

![Figure 5-15 Thread pool creation method provided by Executors](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter5_img_15.png)

These thread pool creation methods can be divided into three categories. The first category includes the newSingleThreadExecutor, newFixedThreadPool, and newCacheThreadPool methods. The code implementation is as follows. As can be seen, these methods actually create ThreadPoolExecutor objects with different input parameters.

```java
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}

public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}

public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
```

The second category includes newSingleThreadScheduledExecutor and ScheduledThreadPoolExecutor. These two types of methods are used to create scheduled thread pools, which can be used to execute delayed tasks or periodic tasks. From the source code, it can be seen that they both create ScheduledThreadPoolExecutor objects. And ScheduledThreadPoolExecutor actually also inherits from the ThreadPoolExecutor object.&#x20;

```java
public static ScheduledExecutorService newSingleThreadScheduledExecutor() {
    return new DelegatedScheduledExecutorService
        (new ScheduledThreadPoolExecutor(1));
}

public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
    return new ScheduledThreadPoolExecutor(corePoolSize);
}
```

The remaining methods such as newWorkStealingPool use the ForkJoinPool thread pool, the code is as follows. It is actually a type of thread pool that only appeared in Java 8, specifically designed to handle concurrent algorithms. Due to its limited use cases, it is rarely used.&#x20;

```java
public static ExecutorService newWorkStealingPool(int parallelism) {
    return new ForkJoinPool
        (parallelism,
         ForkJoinPool.defaultForkJoinWorkerThreadFactory,
         null, true);
}
```

## 5.8.2 Thread Pool Configuration Parsing

Analyzing the implementation of the Executors object to create a thread pool, we can find that these thread pools are all implementations of different imported parameters of ThreadPoolExecutor . If we can be familiar with all the imported parameters in the constructor function of ThreadPoolExecutor, then we have mastered the usage of the thread pool. Therefore, let's take a look at what imported parameters are in the constructor function of the object. The constructor function of ThreadPoolExecutor object is as follows.

```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler rejectedExecutionHandler)
```

The detailed explanation of each input parameter in this function is as follows:&#x20;

| Input Parameter          | Explanation                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| corePoolSize<br />       | When a thread pool is created, a certain number of core threads are pre-created and kept active to execute tasks immediately. Unless the allowCoreThreadTimeOut method is manually called to indicate that core threads need to exit, core threads remain alive and do not exit after starting, even if there are no tasks to execute currently, and they will not be destroyed. By setting an appropriate number of core threads, the performance and resource consumption of the thread pool can be balanced. If the number of core threads is set too small, it may not be able to handle incoming tasks in a timely manner, leading to performance degradation. On the other hand, setting it too large may waste system resources.  |
| maximumPoolSize          | When new tasks arrive and all core threads are busy executing tasks and unable to respond to these new tasks, these new tasks will be placed in the cache queue. If the cache queue is also full, the thread pool will start new threads to execute these tasks. These threads are called non-core threads, and the sum of the number of non-core threads and the number of core threads is the maximum number of threads in the thread pool.                                                                                                                                                                                                                                                                                            |
| keepAliveTime            | keepAliveTime defines the survival time of non-core threads in an idle state. If the idle time of a non-core thread reaches the value set by keepAliveTime, it will be reclaimed and destroyed by the thread pool to reduce resource consumption.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| unit                     | Time unit of keepAliveTime, such as seconds, minutes, etc.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| workQueue                | The cache queue in the thread pool is used to store tasks waiting to be executed. Common cache queues include LinkedBlockingDeque and SynchronousQueue: LinkedBlockingDeque is a bidirectional concurrent queue, mainly used in CPU thread pools; although SynchronousQueue is also a queue, it cannot store tasks, so this queue will directly hand over the added tasks to new threads for processing without storing these tasks, mainly used in IO thread pools. The task queue plays an important role in the thread pool, as it can help control the number of concurrent tasks, balance the production and consumption speed of tasks, and provide a task queuing mechanism.                                                      |
| threadFactory            | A factory object used to create threads. It can be used to customize the creation method and properties of threads, including thread name, priority, thread group, etc. When optimizing virtual memory, it was also mentioned that a custom thread factory can be used to create threads with a stack space of only 512 KB.                                                                                                                                                                                                                                                                                                                                                                                                              |
| rejectedExecutionHandler | When the thread pool is saturated and unable to accept new tasks, the rejection policy defines how to handle these new tasks. The default policy throws a RejectedExecutionException and prevents task submission.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |

It should be noted here that only when the capacity of the workQueue cache queue is full will the creation of non-core threads for the execution of new tasks begin. We can verify this through the implementation of the execute method of ThreadPoolExecutor, and the implementation code is as follows.

```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();

    int c = ctl.get();
    // If the current worker count is less than the core pool size, create a new thread to execute the task
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    // If the core threads are all busy and the pool is running, add the task to the workQueue buffer
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        if (! isRunning(recheck) && remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)
            // If the worker count is 0, call addWorker to create a new thread
            addWorker(null, false);
    }
    // Only when adding the task to the queue fails, call addWorker to start a new thread
    else if (!addWorker(command, false))
        reject(command);
}
```

Through understanding the input parameters of a thread pool, we know that the configuration items of a thread pool are diverse and need to be reasonably configured in combination with the performance of the device and the characteristics of the business to improve the scheduling efficiency of the thread pool. However, the thread pool creation methods provided by the Executors object do not allow flexible configuration of these parameters, so this will result in the thread pool created by the default method being unable to fully improve the scheduling efficiency. Therefore, let's continue to see how to customize the creation of a thread pool.&#x20;

## 5.8.3 Thread Pool Types and Creation

To create a more reasonable thread pool, we still need to further understand what types of thread pools there are and what characteristics each of them has. Only in this way can we customize a more suitable thread pool according to the business scenario. The most frequently used thread pools in business mainly include the scheduled thread pool, CPU thread pool, IO thread pool, and these three types of thread pools. Different types of thread pools have different responsibilities and are specifically designed to handle corresponding types of tasks. The scheduled thread pool is used to handle periodic or delayed tasks, such as the collection of performance metrics; the CPU thread pool is used to handle CPU-intensive tasks, such as computation, logical operations, UI rendering, etc.; the IO thread pool is used to handle IO-intensive tasks, such as fetching network data, reading and writing data to disk, etc.

Let's first take a look at the scheduled thread pool, which inherits from ThreadPoolExecutor, is a wrapper and extension of ThreadPoolExecutor, and the constructor of the scheduled thread pool is as follows.

```c++
public ScheduledThreadPoolExecutor(int corePoolSize,
                                   ThreadFactory threadFactory,
                                   RejectedExecutionHandler handler) {
    super(corePoolSize, Integer.MAX_VALUE,
          DEFAULT_KEEPALIVE_MILLIS, MILLISECONDS,
          new DelayedWorkQueue(), threadFactory, handler);
}
```

It can be seen that the constructor has already defined most of the input parameters, and the only input parameters we can set are the number of core threads, the thread factory, and the rejection policy. Therefore, for the scheduling thread pool, there is no need to consider how to define the input parameters; it can simply be created using the default provided method. So, what we need to focus on are the CPU thread pool and the IO thread pool. Let's take a detailed look at these two types of thread pools below.&#x20;

### 1. CPU Thread Pool

The main function of the CPU thread pool is to effectively manage and execute CPU-intensive tasks, aiming to fully utilize CPU resources and improve application performance. With this purpose in mind, let's take a look at how to set the input parameters of the CPU thread pool.&#x20;

* corePoolSize Core

  First is the corePoolSize, the number of core threads. The CPU thread pool is used to execute CPU-type tasks, so the number of its core threads is generally equal to the number of CPU cores. Ideally, each CPU core runs one thread, which can not only fully utilize the CPU's performance but also reduce CPU consumption caused by frequent scheduling. Although the program cannot achieve the ideal situation during actual operation, setting the number of core threads to the number of CPU cores remains the most reliable configuration.&#x20;

* maximumPoolSize

  For CPU thread pools, each CPU core corresponds to one thread, which can fully utilize the CPU. If the number of threads exceeds the number of CPU cores, it will only lead to performance degradation caused by unnecessary CPU context switching and scheduling. Therefore, the maximum number of threads in a CPU thread pool is equal to the number of core threads. When the threads in the CPU thread pool are already busy and unable to handle new tasks, the newly arrived tasks are placed in the task cache container.&#x20;

* keepAliveTime

  Since the CPU thread pool has no non-core threads, the value of keepAliveTime, which represents the survival time of non-core threads, can be set to 0.

* workQueue

  LinkedBlockingDeque is generally used in CPU thread pools, which is a queue that can set capacity and support concurrency. Since the number of threads in the CPU thread pool is relatively small, if more tasks arrive and there are no idle core threads to execute them, these tasks need to be placed in the cache queue. By default, the capacity of the cache queue is infinite, but such a capacity setting is not a good configuration. If there is some abnormal infinite loop logic in the program that continuously adds tasks to the queue, and the queue can always cache tasks, it will be difficult to detect the anomaly. However, when we set the queue to a limited size, such as 512, the abnormal infinite loop will fill the queue, and subsequent tasks will enter the logic of the rejection policy. In this way, we can add monitoring to the rejection policy and detect this anomaly in a timely manner.&#x20;

* Rejection Strategy

  When the tasks stored in the cache queue reach the upper limit, and there are no available non-core threads to handle these tasks that cannot be placed in the cache queue, then these tasks will enter an exceptional fallback function rejectedExecution. The thread pool created by the Executors object uses the default fallback strategy, and its code implementation is as follows, where it can be seen that an exception will be directly thrown at this time.&#x20;

  ```c++
  public static class AbortPolicy implements RejectedExecutionHandler {
     
      public AbortPolicy() { }

      public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
          throw new RejectedExecutionException("Task " + r.toString() +
                                               " rejected from " +
                                               e.toString());
      }
  }
  ```

  However, directly throwing an exception will cause the program to crash, which will affect the user experience. To provide a better experience, we need to customize a rejection policy, report the exceptional tasks and thread pool for subsequent troubleshooting and fixing of issues. At the same time, we can also add these tasks to a Handler capable of executing concurrent tasks, allowing the Handler to perform fallback execution of these tasks, minimizing the impact on the program. The code implementation is as follows.

  ```c++
  class CoreRejectedExecutionHandler implements RejectedExecutionHandler {

      @Override
      public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
          String taskName = r.getClass()
          // Exception reporting
          report(taskName, executor);
          if (rejectHandlerThread == null) {
              HandlerThread rejectHandlerThread = new HandlerThread("core-reject");
              rejectHandlerThread.start();
              sRejectThreadHandler = new LarkHandler(rejectHandlerThread.getLooper());
          }
          // Use the handler as a fallback to execute the task
          sRejectThreadHandler.post(task);
      }
  }
  ```

* ThreadFactory&#x20;

  Thread factories can be used to set thread properties, so we can use thread factories to uniformly name threads. Uniformly named threads are very helpful when analyzing and troubleshooting exceptions or performance issues. We can also uniformly increase the priority of threads in the CPU thread pool to improve the efficiency of task execution in the CPU thread pool. The implementation code for the custom thread factory solution is as follows. In the implementation of the solution, the prefix name of the thread and the thread priority are passed in through the constructor. To ensure the success rate of setting the thread priority, the Runnable is wrapped, and then when the thread actually runs, that is, in the run method of the Runnable, the priority is set through the Process.setThreadPriority method.

  ```java
  public class CoreThreadFactory implements ThreadFactory {
      private static final String TAG = "CoreThreadFactory";

      private final AtomicInteger mThreadNum = new AtomicInteger(1);
      private final String mPrefix;
      private final int priority;

      public CoreThreadFactory(String prefix, int priority) {
          this.mPrefix = prefix;
          this.priority = priority;
      }

      @Override
      public Thread newThread(Runnable runnable) {
          String name = mPrefix + "-" + mThreadNum.getAndIncrement();
          Thread ret = new Thread(new AdjustThreadPriority(priority, runnable), name);
          return ret;
      }

      public static class AdjustThreadPriority implements Runnable {
          private final int priority;
          private final Runnable task;

          public AdjustThreadPriority(int priority, Runnable runnable) {
              this.priority = priority;
              task = runnable;
          }

          @Override
          public void run() {
              try {
                  Process.setThreadPriority(priority);
              } catch (Exception e) {
                  Log.e(TAG, "AdjustThreadPriority run: ", e);
              }
              task.run();
          }
      }
  }
  ```

After understanding how to configure these parameters, let's take a look at how to create a useful CPU thread pool. Considering the scalability of the architecture, we can create a CoreThreadPoolExecutor class that inherits from ThreadPoolExecutor. Subsequently, both the CPU thread pool and the IO thread pool can inherit from this CoreThreadPoolExecutor object, rather than directly inheriting from ThreadPoolExecutor. Its UML diagram is shown in Figure 5-16.&#x20;

![Figure 5-16 UML diagram of thread pool architecture](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter5_img_16.png)

Set some basic configurations in this CoreThreadPoolExecutor, such as rejection policies and other configurations. The implementation code of the solution is as follows.

```c++
public class CoreThreadPoolExecutor extends ThreadPoolExecutor {
    public CoreThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              BlockingQueue<Runnable> queue,
                              CoreThreadFactory threadFactory) {
        super(corePoolSize, maximumPoolSize, keepAliveTime, TimeUnit.SECONDS, 
                queue, threadFactory);
        setRejectedExecutionHandler(sRejectedExecutionHandler);
    }
    
    public void execute(Runnable command) {
        super.execute(command);
    }
    
    public Future<?> submit(@NotNull Runnable task) {
        super.submit(task);
    }
    
    ……
}
```

Next, we will implement the CPU thread pool, which can be named CpuThreadPoolExecutor or FixedThreadPoolExecutor. Both class names can reflect the characteristics of the thread pool. Here, I use CpuThreadPoolExecutor as the name. The object needs to provide a static method getThreadPool to create or obtain the CPU thread pool. For the CPU thread pool, we can set its priority higher, such as the Process.THREAD\_PRIORITY\_DISPLAY level. The implementation code for the solution is as follows.

```java
class CPUThreadPoolExecutor extends CoreThreadPoolExecutor {
    private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();
    protected static final int CORE_POOL_SIZE = CPU_COUNT;
    protected static final int MAX_POOL_SIZE = CPU_COUNT;
    private static final int BLOCK_QUEUE_CAPACITY = 512;
    private static ThreadPoolExecutor coreCPUThreadPoolExecutor;
    
    private CoreCPUThreadPoolExecutor(BlockingQueue<Runnable> blockingQueue, 
                                        CoreThreadFactory threadFactory) {
        super(CORE_POOL_SIZE, 
            MAX_POOL_SIZE, 
            0, 
            blockingQueue, 
            threadFactory);
    } 

    public static ThreadPoolExecutor getThreadPool() {
        if(coreCPUThreadPoolExecutor == null){
            synchronized (CPUThreadPoolExecutor.class) {
                if (coreCPUThreadPoolExecutor == null) {                  
                    coreCPUThreadPoolExecutor = new CPUThreadPoolExecutor(
                            new LinkedBlockingDeque<Runnable>(BLOCK_QUEUE_CAPACITY),
                            new CoreThreadFactory("CPU", 
                                     Process.THREAD_PRIORITY_DISPLAY));
                }
            }
            
        }
        return coreCPUThreadPoolExecutor; 
    }     
}
```

### 2. IO Thread Pool

When the system is performing IO operations, it will be handed over to DMA (Direct Memory Access) hardware for processing, so data transfer can be carried out without going through the CPU. Therefore, IO tasks consume very little CPU resources. For the IO thread pool, since IO tasks do not consume much CPU resources, each incoming IO task can be directly assigned to an independent thread for execution without being placed in a cache queue. This ensures that each IO task can be responded to in a timely manner. If multiple IO tasks reuse the same thread, when one IO task blocks the thread, it will cause other IO tasks to be unable to execute. Having understood this characteristic, let's take a look at how to set the input parameters of the IO thread pool.&#x20;

* corePoolSize

  There is no fixed rule for the corePoolSize (number of core threads) of the IO thread pool; it is related to the business scenario of our application. If there are many IO tasks, it should be set to a larger value, because setting it too low will result in performance degradation due to frequent creation and destruction of IO threads. If there are few IO tasks in the business scenario, setting it directly to 0 is also acceptable; the number of core threads in the IO thread pool created by Executors is 0.&#x20;

* maximumPoolSize

  In fact, the CPU resources actually consumed by IO tasks are very small. When data needs to be read or written, the system will hand it over to the DMA chip for operation. At this time, the scheduler will put the current thread to sleep and switch the CPU resources to other threads for use. Therefore, for the maximum number of threads (maximumPoolSize) in the IO thread pool, it can be set to a larger value to ensure that each IO task has a corresponding thread to execute, which can ensure that IO tasks can be executed as soon as possible. Generally speaking, setting dozens of threads is sufficient for small and medium-sized applications. Even for large applications, it is not recommended to set the number to be particularly large. For example, the maximum number of threads in the IO thread pool created by Executors is infinite, which will cause the tasks in the IO thread pool to fail to enter the rejection policy when an exception such as an infinite loop occurs.&#x20;

* Cache Queue

  For the IO thread pool, there is no need to cache tasks because every time a task arrives, the thread pool will start an independent thread to execute this task. Therefore, for the IO thread pool, it is generally passed the SynchronousQueue, a queue with a capacity of 0.&#x20;

* keepAliveTime&#x20;

  The survival time of non-core threads also needs to be determined based on the business scenario. If the business frequently encounters scenarios with a large amount of IO, the survival time can be set longer; if it is a low-frequency scenario with a large amount of IO, the survival time can be set shorter, which can reduce the consumption of memory resources by useless threads.&#x20;

* Exception fallback strategy

  The fallback strategy for the IO thread pool can be the same as that for the CPU thread pool: report the exception, and then execute the fallback task using a fallback thread.&#x20;

Based on the above input parameter configuration, the code for creating the IO thread pool is as follows. The priority of IO threads is slightly lower than that of CPU threads, so it can be set to the THREAD\_PRIORITY\_DISPLAY level.

```java
class IOThreadPoolExecutor extends CoreThreadPoolExecutor {
    private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();
    private static final int CORE_POOL_SIZE = 1;
    private static final int MAX_POOL_SIZE = 64;
    private static final int KEEP_ALIVE_TIME = 30;
    private static ThreadPoolExecutor coreIOThreadPoolExecutor;


    private IOThreadPoolExecutor(BlockingQueue<Runnable> blockingQueue, 
                                        CoreThreadFactory threadFactory) {
        super(CORE_POOL_SIZE, 
            MAX_POOL_SIZE, 
            KEEP_ALIVE_TIME, 
            blockingQueue, 
            threadFactory);
    }


    public static CoreThreadPoolExecutor getThreadPool() {
        if(coreIOThreadPoolExecutor == null){
            synchronized (IOThreadPoolExecutor.class) {
                if (coreIOThreadPoolExecutor == null) {
                    coreIOThreadPoolExecutor = new CoreIOThreadPoolExecutor(
                            new SynchronousQueue<Runnable>(),
                            new CoreThreadFactory("IO",
                                 Process.THREAD_PRIORITY_LESS_FAVORABLE));
                }
            } 
        }
        return coreIOThreadPoolExecutor;
    }

}
```

## 5.8.4 Thread Pool Monitoring

After creating a suitable thread pool, we can further improve its functionality. For example, we can monitor the time taken by tasks running in the thread pool. If the time taken exceeds the set threshold, we can output it through logs or report it as an exception. Such capabilities are of great help for the reasonable use of thread pools.&#x20;

### 1. Task Duration Monitoring

To monitor the time taken by tasks, you only need to encapsulate the Runnable. By using the public base class of CoreThreadPoolExecutor we defined earlier, you can encapsulate the Runnable in methods such as execute and submit. The code is as follows.

```java
public class CoreThreadPoolExecutor extends ThreadPoolExecutor{
    
    ……

    public void execute(Runnable command) {
        Runnable newTask = new CoreTask(command, this);
        super.execute(newTask);
    }

}
```

In the encapsulated object CoreTask, the time-consuming threshold can be further set according to the type of the thread pool. For task processes that exceed the time-consuming threshold, log printing and reporting are performed. The implementation code of the solution is as follows. In the displayed code, I set the timeout threshold of the CPU thread pool to 1 second and the timeout threshold of the IO thread pool to 8 seconds. In actual development, we need to set the timeout threshold of thread pool tasks according to the characteristics of the business scenario.&#x20;

```java
public class CoreTask implements Runnable{
    private final CoreThreadPoolExecutor mExecutor;
    protected final Runnable mCommand;
    public CoreTask(@NonNull Runnable r, @Nullable ICoreThreadPool executor) {
        mExecutor = executor;
        mCommand = r;
    }
    
    @Override
    public void run() {
        long mTaskBeginExecTime = SystemClock.uptimeMillis();
        try {
            mCommand.run();
        } finally {
            runTime = SystemClock.uptimeMillis() - mTaskBeginExecTime;    
            boolean bTaskOverLimit = false;               
            if (mExecutor != null) {
                //根据不同的线程池类型，设置超时阈值。
                if (mExecutor instanceof CPUThreadPoolExecutor) {
                    if (1000 < runtime) {
                        bTaskOverLimit = true;
                    }
                } else if (mExecutor instanceof IOThreadPoolExecutor) {      
                    if (8000 < runtime) {
                        bTaskOverLimit = true;
                    }
                }
            } 

            if (bTaskOverLimit) {
                Log.w(TAG, poolName + ", taskname: " + orgTaskName + 
                        ", dispatchtime & runtime is(ms) " + 
                        dispatchTime + ", " + runtime +
                        "maxQueueWaitTime & MaxRunTime is " + 
                        maxQueueWaitTimeMS + ", " + maxRunTimeMS);
                //report data
                ifNeedReport(……) 
            }
        }
    }
}
```

### 2. Task Deadlock Monitoring

We have monitored high-time-consuming tasks in the thread pool. In addition to high-time-consuming tasks, deadlock tasks are also a factor that has a significant impact on the performance of the thread pool. When a deadlock occurs in a task, it will cause the task to be unable to exit for a long time, which will make the thread unavailable. Moreover, when a deadlock occurs, it is often not just one thread trying to acquire a lock, but multiple threads will be in an unavailable state because they cannot acquire the lock. For a thread pool with a limited number of threads, especially a CPU thread pool with only a few threads, not having enough threads to execute tasks will inevitably have a significant impact on performance.&#x20;

When a task encounters a deadlock, it does not exit, so it is impossible to determine whether the task has deadlocked based on its elapsed time. In this case, we can change our approach to determine deadlocks. We can put the task name into a container before the task starts, and then remove it from the container when the task ends. For task names that have not been removed from the container for a long time, we can determine that the task has deadlocked. Based on this idea, I will introduce the implementation of the solution in detail below:&#x20;

1\) The key step in deadlock monitoring is to put the task name into a container before the task starts execution. Here, a Map can be used as the container, with the key being the task name and the value being the timestamp. Based on design specifications, this Map container can be placed in the CoreThreadPoolExecutor base class of the thread pool, and CoreThreadPoolExecutor inherits an abstract interface that defines the addTaskRecord and removeTaskRecord methods. In the implementation of these two interfaces, the task name and timestamp are added and removed. The code implementation is as follows.

```c++
public interface ICoreThreadPool {
    String getThreadPoolName();

    void addTaskRecord(String taskName,int taskHash);

    void removeTaskRecord(String taskName,int taskHash);
    
    HashMap<String, Long> getRunningTaskMap();

}

public class CoreThreadPoolExecutor extends ThreadPoolExecutor 
        implements ICoreThreadPool {
    ……
    private HashMap<String, Long> mTaskMap = new HashMap<>();
    
    @Override
    public void addTaskRecord(String taskName, int taskHash) {
        if (CoreThreadPool.getCoreThreadPoolServiceSwitch()) {
            synchronized (mTaskMap) {
                mTaskMap.put(taskName + "#" + taskHash, System.currentTimeMillis());
            }
        }
    }
    
    @Override
    public void removeTaskRecord(String taskName, int taskHash) {
        long taskBeginTime = 0;
        synchronized (mTaskMap) {
            if(mTaskMap.get(taskName + "#" + taskHash) == null){
                return;
            }
            taskBeginTime = mTaskMap.remove(taskName + "#" + taskHash);
        }
    }
    
    @Override
    HashMap<String, Long> getRunningTaskMap(){
        return mTaskMap;
    }
}
```

2\) In our custom CoreTask, the thread pool object has already been passed in. Since the thread pool implements the ICoreThreadPool interface, addTaskRecord and removeTaskRecord can be directly used before and after task execution for task name recording and exception handling. The code implementation is as follows.

```c++
public class CoreTask implements Runnable{
    private final CoreThreadPoolExecutor mExecutor;
    protected final Runnable mCommand;
    public CoreTask(@NonNull Runnable r, @Nullable ICoreThreadPool executor) {
        mExecutor = executor;
        mCommand = r;
    }
    
    @Override
    public void run() {
        long mTaskBeginExecTime = SystemClock.uptimeMillis();
        try {
            ……
            //add task into executor
            mExecutor.addTaskRecord(mCommand.getClass().toString(), hashCode());
            //执行真正的Task
            mCommand.run();
        } finally {
            ……
            //remove task from executor
            mExecutor.removeTaskRecord(mCommand.getClass().toString(), hashCode());
        }
    }
}
```

3\) Finally, we also need a dedicated thread to monitor whether there are tasks in the Map container that have not been removed for a long time. Therefore, we can use a periodic task thread pool for monitoring, and the frequency can be adjusted according to the business. Here, it is set to once every 10 seconds. When detecting tasks in the container, if a task that has not completed execution for more than 30 seconds is found, it is considered that the task has deadlocked. At this time, log printing and data reporting can be performed. The code implementation is as follows.

```c++
// Start periodic task
Executors.newSingleThreadScheduledExecutor()
        .scheduleWithFixedDelay.
        scheduleWithFixedDelay(new TestTask(), 10, 10, TimeUnit.SECONDS);

public static class CheckLockedTask implements Runnable {
    @Override
    public void run() {
        ICoreThreadPool cpuThreadPool = CoreCPUThreadPoolExecutor.getThreadPool();
        // Check if there are any task deadlocks in the CPU thread pool
        synchronized (cpuThreadPool.getRunningTaskMap()) {
            checkLongRunTask(cpuThreadPool.getRunningTaskMap(), 
                    cpuThreadPool.getThreadPoolName(), 30*1000);
        }
        // Check if there are any task deadlocks in the IO thread pool
        ICoreThreadPool ioThreadPool = CoreIOThreadPoolExecutor.getThreadPool()
        synchronized (ioThreadPool.getRunningTaskMap()) {
            checkLongRunTask(ioThreadPool.getRunningTaskMap(), 
                    ioThreadPool.getThreadPoolName(), 30*1000);
        }
    }
}

private static void checkLongRunTask(HashMap<String, Long> taskMap, 
        String threadPoolName, int maxTime) {
    for (Map.Entry<String, Long> entry : taskMap.entrySet()) {
        // Determine if task execution time exceeds the threshold
        if (System.currentTimeMillis() - entry.getValue() >= maxTime) {
            // Key is concatenated by taskname#hashcode
            String taskName = entry.getKey().split("#")[0];
            long runningTime = System.currentTimeMillis() - entry.getValue();
            // Print error log or report                
            Log.w(TAG, threadPoolName + ", taskname: " + 
                    taskName + "run time over " + runningTime);
            
        }
    }
}
```

| Source code addresses appearing in this chapter: <br />Daemons: <br /><https://cs.android.com/android/platform/superproject/+/android-14.0.0\_r9:libcore/libart/src/main/java/java/lang/Daemons.java><br />task\_processor.h:<br /><https://cs.android.com/android/platform/superproject/+/android-14.0.0\_r9:art/runtime/gc/task\_processor.h><br />task\_processor.cc:<br /><https://cs.android.com/android/platform/superproject/+/android-14.0.0\_r9:art/runtime/gc/task\_processor.cc><br />ndk\_dlopen: <https://github.com/Rprop/ndk\_dlopen><br />redex: <https://github.com/facebook/redex><br />libdl.cpp：<br /><https://cs.android.com/android/platform/superproject/+/android-14.0.0\_r9:bionic/libdl/libdl.cpp><br />ShadowHook：<https://github.com/bytedance/android-inline-hook><br />ThreadPoolExecutor.java: <br /><https://cs.android.com/android/platform/superproject/+/android-14.0.0\_r9:libcore/ojluni/src/main/java/java/util/concurrent/ThreadPoolExecutor.java> |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
