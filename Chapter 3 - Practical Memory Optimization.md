In the previous chapter, I summarized three methodologies for memory optimization, namely timely data cleaning, reducing data loading, and increasing memory size. All memory optimization solutions are derived around these three points. Therefore, in this chapter, I will introduce multiple optimization cases based on these three methodologies to strengthen everyone's understanding of memory optimization.&#x20;

In the direction of timely data cleaning, this chapter will introduce two solutions: "Java Memory Leak Detection" and "Native Memory Leak Detection"; in the direction of reducing data loading, this chapter will introduce the solution of "Bitmap Governance"; in the direction of increasing memory size, this chapter will introduce two solutions: "Thread Stack Optimization" and "Default WebView Memory Release". When introducing these practical solutions, I will also delve into the technologies used in the implementation of these solutions, such as Native Hook technology, byte code instrumentation technology, etc., which are all essential technologies to master in Android advancement. Through the combination of the principle section and the practical section, it is hoped that readers can completely eliminate the obstacles in memory optimization caused by insufficient basic knowledge and technical reserves.&#x20;

# 3.1 Java Memory Leak Detection

At the end of the business or when the memory is approaching the threshold, we only need to promptly clean up the data that is no longer in use to achieve good memory optimization results. Although this solution is relatively simple, we still often encounter memory issues caused by incomplete data cleanup. The main reason is often that these data are used in some code that we cannot effectively detect, such as an Activity held by a long-lived object, memory allocated by a certain so library but not released, etc., which leads to the data not being effectively cleaned up and thus causing memory leaks. Therefore, the detection and management of memory leaks is one of the guarantees to ensure that we can promptly clean up useless data and thereby increase available memory.

I simulated a memory leak scenario in the example program, as shown in Figure 3-1. On line 27, the main thread holds JavaLeakActivity, and even when JavaLeakActivity exits, the main thread will not release this Activity within 50000 ms.&#x20;

![Figure 3-1 Simulated Memory Leak Scenario](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter3_img_1.png)

Although the example is a very simple simulated scenario, and we can clearly know from the code that there will be a memory leak here, memory leaks in actual projects are often very hidden, and it is difficult for us to directly detect them from the code. Therefore, taking this simulated scenario as an example, let's take a look at how to analyze memory leaks in actual projects.&#x20;

Memory leaks can be analyzed in two ways: manual analysis and automatic analysis. Let's first look at the first method, manual analysis.&#x20;

## 3.1.1 Manual Analysis

The method of manually analyzing memory leaks requires us to capture the Hprof (Heap Profile, memory snapshot) file through commands. This file is a type of heap dump file of the virtual machine, which records the heap memory allocation and object usage during the program's runtime, and is used to analyze the memory usage of Java applications. Then, by analyzing the reference chain of objects through the Hprof file, we can discover and resolve memory leak issues.&#x20;

The Hprof file of the program can be captured using the command “adb shell am dumpheap \<process\_id> \<output\_file>”. The code for capturing the Hprof file of the sample program in this book is as follows.&#x20;

```shell
 adb shell am dumpheap com.example.performance_optimize /data/local/tmp/demo.hprof
```

After capturing the Hprof file, you can analyze it after pulling the file from the phone directory to the local computer directory using the pull command. We have two ways to perform the analysis: one is to directly analyze it using the Profile tool built into Android Studio, and the other is to analyze it using  the MAT tool .&#x20;

### 1. Android Studio Analysis

Open the Profile tool in Android Studio, directly import the Hprof file just captured into it for analysis. As shown in Figure 3-2, click Leaks, and you can see that JavaLeakActivity has leaked.

![Figure 3-2 Android Studio Analyzing Hprof Interface ](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter3_img_2.png)

By clicking References, we can see the reference chain of JavaLeakActivity, thereby discovering that this Activity is held by the inner class JavaLeakActivity$1, resulting in a memory leak. Right-clicking on Jump to Source allows us to directly jump to the source code path. In the analysis interface, we can see that the leaked Activity has Shallow Size and Retained Size, and the explanations for these two "Size" are as follows:&#x20;

* Shallow size: The size of the memory occupied by the object itself, excluding the memory of the objects it references. It can be seen that the memory size of the JavaLeakActivity object itself is only 328 bytes.

* Retained size : The sum of the object's own memory size and the memory sizes of objects that can be directly or indirectly accessed from this object. If a memory leak occurs for this object, the retained heap size is the size of the leak.

By retained size, we can know that JavaLeakActivity has caused a memory leak of 109460 bytes.&#x20;

### 2. MAT Analysis

In addition to using Android Studio, we can also use the MAT tool in Eclipse to analyze Hprof files, which can be downloaded from the official website of [Eclipse](https://eclipse.dev/mat/downloads.php). However, the Hprof file captured by the previous "dumpheap" command cannot be directly used in MAT and needs to be converted using the hprof-conv tool in the platform-tools directory of the Android SDK. The conversion command line is as follows:&#x20;

```shell
hprof-conv demo.hprof after_demo.hprof
```

Drag the converted after\_demo.hprof into the MAT tool, and you will see the interface shown in Figure 3-3.

![Figure 3-3 MAT Analysis Hprof Interface](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter3_img_3.png)

&#x20;The main functions in the interface are explained as follows:&#x20;

* Histogram: Displays the number of instances, shallow heap, and retained heap size of each object, allowing you to view them in different sorting and grouping methods, or use filters to screen objects of interest.

* Dominator Tree: Displays the shallow heap, retained heap size, and reference relationships of each object.

* Top consumers: Displays the objects in the heap that occupy the most memory

* Leak Suspects: Displays objects that may have memory leaks. MAT traverses objects in the heap memory, identifies all objects held by GC Roots, and then analyzes whether an object has a leak based on information such as its retained heap size, reference path, and class. It then sorts the objects in descending order of retained heap size and selects the top few as suspected memory leak objects. It should be noted that the results here only indicate possible leaks, not actual ones, and we still need to further judge and analyze the results.

Click Leak Suspects, as shown in Figure 3-4 interface , you can see and did not find JavaLeakActivity appeared in the list of leaks, which is mainly due to the difference between MAT and Android Studio in memory leak detection mechanism, MAT is a general memory analysis tool for Java, so it will not be a special leak judgment on Activity, and Android Studio will be a special leak logic judgment for Activity, Fragment and other objects, that is, after onDestory is executed, Activity, Fragment and other objects are not recycled, which means a memory leak has occurred.

![Figure 3-4 MAT Detection of Leakage Interface](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter3_img_4.png)

Since no useful information was found in Leak Suspects, how can MAT  analyze memory leaks? In fact, the main use of MAT is to analyze more detailed information such as the reference chain of an object when we already know that a certain object has leaked, thereby helping us solve the problem. By clicking on the Dominator Tree and entering JavaLeakActivity at the top, the reference chain of the object can be displayed, as shown in Figure 3-5.

![Figure 3-5 Reference Chain of JavaLeakActivity ](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter3_img_5.png)

The reference chain appearing in the interface at this time includes all  Incoming references and Outgoing references , so there are many entries, making it difficult to analyze. Therefore, we need to right-click on List objects and select with Incoming references, that is, select incoming references, as shown in Figure 3-6.&#x20;

![Figure 3-6 Incoming Reference Selection](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter3_img_6.png)

Here, I explain what Incoming references and Outgoing references are. In Figure 3-7, the incoming references of object C are object A, object B, and C's class object, while the outgoing references of object C are object D, object E, and C's class object. Therefore, when other objects hold references to this object, these objects are called incoming reference objects; when an object holds references to other objects, these objects are considered outgoing reference objects. Through incoming references, we can know which objects hold JavaLeakActivity, and then cut off the reference chain to fix memory leaks.&#x20;

After performing the "Incoming references" filter, you can see all objects that reference JavaLeakActivity. Here, we need to continue filtering. Click "Path To GC Roots" and exclude all reference chains such as soft references and weak references that do not cause memory leaks by selecting "exclude all phantom/weak/soft etc. references", as shown in Figure 3-8.&#x20;

![Figure 3-8 Excluding Reference Chains ](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter3_img_7.png)

At this point, we can see that JavaLeakActivity is held by the inner class JavaLeakActivity$1, resulting in a memory leak. As shown in Figure 3-9, the analysis result is consistent with the result in Android Studio.

![Figure 3-9 Reference Chain Leading to Leakage](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter3_img_8.png)

MAT has more powerful features and better performance compared to AndroidStudio, making it more suitable for complex scenarios, but it is also more complex to use. AndroidStudio, on the other hand, is easier to use, and for less complex scenarios such as those with short reference chain paths and simple reference relationships, AndroidStudio can be used for analysis.

## 3.1.2 Automatic Analysis

After discussing the method of detecting memory leaks through manual analysis of Hprof, let's now look at the method of automatic analysis and detection. This method can help us discover memory leak issues more quickly and promptly. The commonly used automatic analysis tool is mainly LeakCanary.&#x20;

### 1. LeakCanary Usage

Using LeakCanary is relatively simple. We only need tointroduce the LeakCanary library in the dependencies configurationto start using it. In the latest version of LeakCanary, there's no need for initialization code either; simply introducing it enables usage.

```groovy
dependencies {
  debugImplementation 'com.squareup.leakcanary:leakcanary-android:3.0-alpha-1'
}
```

After introducing LeakCanary and running the memory leak case in the sample program again, you can see that LeakCanary takes effect and automatically detects that a leak has occurred. After detecting the leak, it will capture the stack and send a reminder via the notification bar, as shown in Figure 3-10. Clicking on the notification bar will jump to the details page, as shown in Figure 3-11. Through the details page, we can see which objects are leaking and the reference chains that hold these objects. Since LeakCanary captures and analyzes the Hprof file when a memory leak occurs, this process can cause the program to become laggy, so LeakCanary is only recommended for use in test packages.&#x20;
|![Figure 3-10 LeakCanary Notification Alert](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter3_img_9.png)| ![Figure 3-11 Memory Leak Details](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter3_img_10.png) |
| :---: | :---: |

### 2. Principle of LeakCanary

LeakCanary is very practical and widely used in actual projects. However, simply knowing how to use LeakCanary is not enough to handle complex scenarios. For example, we may need to use LeakCanary for secondary development, add some customized detection capabilities, or configure some customized attributes. Therefore, we also need to understand the principle of LeakCanary, and the detection principle of LeakCanary is also one of the common test points in interviews.

LeakCanary essentially leverages the characteristics of the WeakReference, a weak reference object in Java. If an object is only referenced by a weak reference, the weak reference object will be cleared when the GC executes. LeakCanary encapsulates an Activity into a WeakReference object. After the Activity executes onDestroy, it actively performs a GC operation. If the weak reference of the Activity is not reclaimed at this time, it is determined that a memory leak has occurred in the Activity. I deepens readers' understanding of the detection process through simplified process code, and the main process is as follows.

1\) LeakCanary, upon app startup, registers to monitor the lifecycle of each Activity via the ActivityLifecycleCallbacks provided by the system. The code implementation is as follows: it can be seen that LeakCanary creates an ActivityRefWatcher object that inherits from ActivityLifecycleCallbacks, and calls the RefWatcher.watch method when the Activity executes the destruction callback of onDestroy.

```java

public class ActivityRefWatcher implements Application.ActivityLifecycleCallbacks {
  private final RefWatcher refWatcher;

  public ActivityRefWatcher(RefWatcher refWatcher) {
    this.refWatcher = refWatcher;
  }

  @Override
  public void onActivityDestroyed(Activity activity) {
    // When Activity destoryed，call the RefWatcher.watch
    refWatcher.watch(activity);
  }
}
```

2\) In the RefWatcher.watch method, the reference to this Activity will be wrapped into a KeyedWeakReference object and added to a custom ReferenceQueue object. The code is as follows.

```java
public class RefWatcher {
  private final Set<String> retainedKeys = new HashSet<>();
  private final ReferenceQueue<Object> queue = new ReferenceQueue<>();
  private final WatchExecutor watchExecutor;

  public RefWatcher(WatchExecutor watchExecutor) {
    this.watchExecutor = watchExecutor;
  }

  public void watch(Object watchedReference) {
    // generate a randon key
    String key = UUID.randomUUID().toString();
    // wrap the object into an KeyedWeakReference，and add to the ReferenceQueue
    KeyedWeakReference reference = new KeyedWeakReference(watchedReference, key, queue);
    // add the key into the retainedKeys
    retainedKeys.add(key);
    // Asynchronously check for memory leaks
    ensureGoneAsync(reference);
  }
}

public class KeyedWeakReference extends WeakReference<Object> {
  public final String key;

  public KeyedWeakReference(Object referent, String key, ReferenceQueue<Object> queue) {
    super(referent, queue);
    this.key = key;
  }
}

```

3\) Next, RefWatcher will call the ensureGoneAsync method to detect memory leaks. It should be noted that after this method is called, memory leak detection will not be performed immediately. Instead, it will be detected by a WatchExecutor when the main thread is idle. The detection method is to actively trigger a GC and check whether the references in the ReferenceQueue have been reclaimed. If an unreclaimed reference is found, it indicates that a memory leak has occurred in the object. RefWatcher will dump the heap memory into an Hprof file through the AndroidHeapDumper object and pass it to the HeapAnalyzer for analysis. The code is as follows.

```java
private void ensureGoneAsync(final KeyedWeakReference reference) {
    // execute task when main thread is idle
    watchExecutor.execute(new Runnable() {
      @Override
      public void run() {
        // Triger GC
        GcTrigger.DEFAULT.runGc();
        // Remove reclaimed references
        removeWeaklyReachableReferences();
        // Check for any remaining unreclaimed references
        if (retainedKeys.contains(reference.key)) {
          // Have a memeory leak, dump memeory heap
          File heapDumpFile = AndroidHeapDumper.dumpHeap();
          // Analyze memory heap, generate leak info
          LeakTrace leakTrace = HeapAnalyzer.analyze(heapDumpFile, reference.key);
          // Display leak info
          DisplayLeakService.showLeakNotification(leakTrace);
        }
      }
    });
}

private void removeWeaklyReachableReferences() {
    KeyedWeakReference ref;
    while ((ref = (KeyedWeakReference) queue.poll()) != null) {
      retainedKeys.remove(ref.key);
    }
}
```

4\) HeapAnalyzer will then parse the Hprof file, find the reference chain of the leaked object, and generate a LeakTrace object containing information such as the class name, field name, and size of the leak. It will also send the LeakTrace object to DisplayLeakService, a service running in another process, which will display the leak information in the notification bar and provide a LeakActivity to view detailed information about the leak.

LeakActivity performs Hprof capture and reference chain analysis through the third-party library HAHA, so i will not elaborate further. If readers interested in LeakActivity can delve deeper into analyzing the code details.

# 3.2 Native Memory Leak Detection

The main cause of native memory leaks is that the code in the .so library calls the malloc function to allocate memory, but don't call the free function to release the memory after the business operation ends. As the program runs, more and more memory leaks occur, eventually leading to excessive memory consumption and causing the program to malfunction.&#x20;

To detect memory leaks in Native, we usually need to intercept the malloc function and free function in the so library, and insert our own logic to count the memory size of malloc and free. If the memory allocated by a so library minus the memory freed exceeds a threshold we set, we consider that a memory leak has occurred in this so library.

I applied for nearly 95MB of memory in the Native layer of the example program, as shown in Figure 3-12, and packaged this Native code into the example.so library.&#x20;

![Figure 3-12 Abnormal Application for Native Memory](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter3_img_11.png)

Next, we load this .so library in the Activity and call this method, as shown in Figure 3-13, which allows us to simulate a scenario of abnormal memory allocation in Native. I then use this example program to gradually troubleshoot and detect this abnormal Native memory allocation, thereby helping readers master the techniques and steps for detecting Native memory leaks.

![Figure 3-13 Call Exception Native Function ](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter3_img_12.png)

## 3.2.1 Intercept the malloc and free functions

To intercept and modify the malloc and free functions in the .so library, Native Hook technology needs to be used. Here, I introduce one such Native Hook technology: PLT Hook technology.

### 1. PLT Hook Technology

The running process of a program is a continuous process of calling and executing functions, and function calls must know the address of the function in memory. When native code is packaged into a so library, each function is assigned an offset address. Therefore, if it is a mutual call between functions within the so library, the function call can be completed directly through the offset address assigned to the function during compilation.&#x20;

However, when we call a function from an external so library, we can only call it through the absolute address of the function in memory, which is the starting address of the external so library in memory plus the offset address of the function. So, what is the call process of an external function? Here, I take the example.so library in the sample program as an example. Since the malloc function is a function located in the external library libc.so, when the code in the example.so library calls the malloc function, it first searches the internal `.plt` procedure linkage table, which is a code segment containing jump instructions. Through the jump instructions, it then jumps to the `.got` table corresponding to the malloc function, which is a data segment containing the address of the external function and is located in the `.dynamic` segment. The `.got` table records the real address of the malloc function, and the process is shown in Figure 3-14. However, during compilation, the address of the malloc function cannot be determined in the `.got` table, so the initial address is 0. During the program execution, the Linker (dynamic linker), a system program, will write the real address of the malloc function into the corresponding `.got` global offset table of the example.so library when the malloc function is called.&#x20;

What are the PLT table and the GOT table? We know that an so library is actually a file in ELF format, which contains various data segments such as the code segment (.text), data segment (.data), BSS segment (.bss), dynamic table (`.dynamic`), etc. When a program runs a certain so library, all these data segments of the so library will be loaded into memory. The PLT table, that is, the Procedure Linkage Table, is actually a table located in the code segment, which records the code segment of the GOT table corresponding to the function that jumps to an external function. The GOT table is a table located below the data segment, which records the addresses of external library functions. The addresses of the functions in the external library are written back to the GOT table by the dynamic linker (Linker), a system program, during program runtime.

By using the objdump tool that comes with the Android NDK, executing the command "objdump -D libexample.so" allows you to view the assembly code corresponding to example.so. We find the assembly code corresponding to the mallocLeak function in the example scenario, as shown in Figure 3-15.&#x20;

![Figure 3-15 Assembly code of the mallocLeak function](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter3_img_13.png)

From the above assembly code, we can see that the assembly instruction "blx 1caec \<malloc@plt>" corresponds to address 1ede6, where blx is a function call instruction, 1caec is the address corresponding to the function, that is, the address of the `.plt` table corresponding to the malloc function, namely the \<malloc@plt> function. It can be seen that through this instruction, the call to the malloc function will jump to the corresponding `.plt` table.

Next, let's take a look at the assembly code of malloc in the `.plt` table. As shown in Figure 3-16, it is a code segment containing three instructions, with explanations as follows:

* The first instruction "add ip, pc, #0, 12" means shifting 0 left by 12 bits, adding it to the value of the PC (Program Counter) register, and writing the result to the ip register. Since in the ARM architecture, the value of PC is the address of the current instruction plus 8 bytes, the value of the ip register at this time is: 1caec + 8 = 1caf4.&#x20;

* The second instruction “add ip, ip, #217088 ; 0x35000” means adding the decimal value 217088, whose hexadecimal equivalent is 0x35000, to the value of the ip register, and storing the result back into the ip register. Therefore, the value of the ip register at this time is 1caf4 + 35000 = 51af4.&#x20;

* The third instruction "ldr pc, \[ip, #2432]! ; 0x980" means adding the decimal value #2200 (whose hexadecimal value is 0x980) to the value of the ip register, and storing the result in the PC register. Therefore, the value of the PC register at this time is 51af4 + 978 = 52474.&#x20;

![Figure 3-16 plt Table of the malloc Function](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter3_img_14.png)

Therefore, the value 52474 in the PC register is the address to which the instruction will jump next. Readers who do not fully understand these three instructions at this point need not worry, you can revisit this after becoming familiar with common assembly instructions and registers in Chapter 4, "Optimizing Speed and Smoothness." We only need to know that the next jump will be to the address 52474.

Continuing to find the code corresponding to 52474, as shown in Figure 3-17, we can see that it is located in the `.got` table, and the value 0001cac0 corresponding to this address is the real address of the malloc function. Why is all the data in the `.got` table the address 0001cac0? In fact, the address corresponding to 0001cac0 will jump to a dynamic linking code segment. When the program runs and calls the malloc function, this code segment will call the dynamic linker (Linker) to write in the real address of malloc.&#x20;

![Figure 3-17 Value of the malloc function in the got table](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter3_img_15.png)

After understanding the principle of the above external function call, we can start to learn about the PLT Hook technique. If we replace the address data corresponding to the offset address 52474 in the libexample.so library with the absolute address of our custom function during the program's execution, then all function calls to malloc in this library will jump to our custom function. When the logic of the custom function is executed and jumps to the address originally recorded at address 52474 at the end of the custom function, the original malloc function logic can continue to execute. Through this process, the interception operation of the malloc function is completed. This process is shown in Figure 3-18.&#x20;

Having understood the process and thinking, we can now implement it through code. The code for the main process is as follows:&#x20;

1. By reading the maps file line by line, we can find and parse out the address of libexample.so. Of course, we can also use the [dl\_iterate\_phdr](https://man7.org/linux/man-pages/man3/dl_iterate_phdr.3.html) function provided by the Linux system to more conveniently find the base address of libexample.so.&#x20;

   ```c++
   FILE *fp = fopen("/proc/self/maps", "r");
   char line[1024];
   uintptr_t base_addr = 0;
   /// Read the maps file line by line
   while (fgets(line, sizeof(line), fp)) {
       __android_log_print(ANDROID_LOG_DEBUG, "hookMallocByPLTHook", "line:%s", line);
       if (NULL != strstr(line, "libexample.so")) {
           std::string targetLine = line;
           std::size_t pos = targetLine.find('-');
           if (pos != std::string::npos) {
               std::string addressStr = targetLine.substr(0, pos);
               // The stoull function converts a string to an unsigned long long integer
               base_addr = std::stoull(addressStr, nullptr, 16);
               break;
           }
       }
   }
   fclose(fp);
   ```

2. Then, based on the device platform, convert the obtained base address of the so library into the Elf32\_Ehdr or Elf64\_Ehd data structure, which is the corresponding data structure after the ELF file is loaded into memory. After including the \<linux/elf.h> Header File, this data structure can be used. My platform environment is 32-bit, so in the following text, the 32-bit data structure will be used uniformly for code demonstration. However, in actual use, it is necessary to check the platform version and then select the corresponding ELF structure.&#x20;

   ```c++
   // Data structure of 32-bit ELF file header
   typedef struct {
       unsigned char e_ident[EI_NIDENT]; /* Magic number and other information */
       Elf32_Half    e_type;             /* File type */
       Elf32_Half    e_machine;          /* Target architecture */
       Elf32_Word    e_version;          /* File version */
       Elf32_Addr    e_entry;            /* Program entry address */
       Elf32_Off     e_phoff;            /* File offset of the program header table */
       Elf32_Off     e_shoff;            /* File offset of the section header table */
       Elf32_Word    e_flags;            /* Processor-specific flags */
       Elf32_Half    e_ehsize;           /* Size of the ELF header */
       Elf32_Half    e_phentsize;        /* Size of each entry in the program header table */
       Elf32_Half    e_phnum;            /* Number of entries in the program header table */
       Elf32_Half    e_shentsize;        /* Size of each entry in the section header table */
       Elf32_Half    e_shnum;            /* Number of entries in the section header table */
       Elf32_Half    e_shstrndx;         /* Index of the section name string table in section headers */
   } Elf32_Ehdr;
   ------------------------------------------------------------------------------
   //Cast base_addr to the Elf_Ehdr format
   Elf32_Ehdr *header = (Elf32_Ehdr *) (base_addr); 
   ```

3. Obtain the entry address of the program header table through the data structure of Elf\_Ehdr, which is e\_phoff (the offset address of the program header table) plus the base address of the so library. After obtaining the address of the program header table, still convert it to the corresponding data structure Elf32\_Phdr of the program header table, which is also defined in the elf.h file. Then we can traverse the data structure of the program header table, find the segment with p\_type being PT\_DYNAMIC, that is, the `.dynamic` segment, and obtain the address and size of this segment. The code implementation is as follows:

```c++
typedef struct {
    Elf32_Word p_type;   /* Segment type */
    Elf32_Off  p_offset; /* File offset of the segment */
    Elf32_Addr p_vaddr;  /* Virtual address of the segment in memory */
    Elf32_Addr p_paddr;  /* Physical address of the segment in memory */
    Elf32_Word p_filesz; /* Size of the segment in the file */
    Elf32_Word p_memsz;  /* Size of the segment in memory */
    Elf32_Word p_flags;  /* Segment flags */
    Elf32_Word p_align;  /* Segment alignment */} Elf32_Phdr;
----------------------------------------------------------------------------
// Number of program header entries
size_t phr_count = header->e_phnum;  
// Address of the program header table
Elf32_Phdr *phdr_table = (Elf32_Phdr *) (base_addr + header->e_phoff);  
unsigned long dynamicAddr = 0;
unsigned int dynamicSize = 0;
for (int i = 0; i < phr_count; i++) {
    if (phdr_table[i].p_type == PT_DYNAMIC) {
        // The base address of the so plus the offset address of the dynamic segment equals the actual address of the dynamic segment
        dynamicAddr = phdr_table[i].p_vaddr + base_addr;
        dynamicSize = phdr_table[i].p_memsz;
        break;
    }
}
```

* Traverse the found `.dynamic` section. When d\_tag is DT\_PLTREL, it is the section pointing to the plt table, and we can obtain the address of the plt table through d\_val. The code implementation is as follows.

  ```c++
  typedef struct {
      Elf32_Sword d_tag;   /* Dynamic entry type */
      union {
          Elf32_Word d_val;  /* Integer value */
          Elf32_Addr d_ptr;  /* Address value */
      } d_un;
  } Elf32_Dyn;
  _________________________________________________________
  uintptr_t symbolTableAddr;
  Elf32_Dyn *dynamic_table = (Elf32_Dyn *) dynamicAddr;
  for (int i = 0; i < dynamicSize; i++) {
      if (dynamic_table[i].d_tag == DT_PLTGOT) {
          symbolTableAddr = dynamic_table[i].d_un.d_ptr + base_addr;
          break;
      }
  }
  ```

1. Modify the memory attributes to writable, traverse the `.plt` table, find the value recorded in the corresponding got table based on the address of the malloc function in the got table (52474 + the base address of the so library), and replace this value with the address of our custom function. Meanwhile, we need to save this value, and after executing the custom function, continue to execute this value, which is the real address corresponding to the malloc function.

   ```c++
   //Modify the memory attributes to writable
   mprotect((void *)symbolTableAddr, PAGE_SIZE,PROT_READ|PROT_WRITE);
   //The offset address of target function
   originFunc = 0x52474 + base_addr;
   //replace the address of our custom function
   uintptr_t newFunc = (uintptr_t) &malloc_hook_by_plt;
   int *symbolTable = (int *) symbolTableAddr;
   for (int i = 0;; i++) {
       if ((uintptr_t) &symbolTable[i] == originFunc) {
           //save the value in plt table,which is address of malloc function
           originFunc = symbolTable[i];
           //Replace the value of originFunc address with the new function
           symbolTable[i] = newFunc;
           break;
       }
   }
   ```

2. Implement the desired logic in the custom interception function, such as printing the stack for the logic of excessive memory allocation, recording the total memory allocated by the so library, etc., and simply execute the original replaced function address at the end of the function. The code is as follows.

   ```c++
   void *malloc_hook_by_plt(size_t len) { 
       __android_log_print(ANDROID_LOG_DEBUG, "hookMallocByPLTHook", 
                                               "origin malloc size:%d", len);
       if(len > 20*1024*1024){
           __android_log_print(ANDROID_LOG_DEBUG, "hookMallocByPLTHook", 
                                                   "do somethings");
           printNativeStack();
       }
       // call original function
       return reinterpret_cast<void *(*)(size_t)>((void*)originFunc)(len);
   }
   ```

Run the Demo program, and through the log, as shown in Figure 3-19, we can see that we have successfully hooked the malloc function

![Figure 3-19 hook success log](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter3_img_16.png)

As can be seen, implementing this process is not very complicated, but there is still an issue we haven't resolved. In the above process, the entry address of the malloc function in the got table is 52474, but this address is not fixed. It's possible that after new code is added to the so library and it's repackaged, this address will change. So we need to dynamically obtain the address of the malloc function in the got table, which in fact can be found by simply looking in the `.rel.plt` table. Earlier, we calculated through three instructions that the next jump address of the plt is 52474, and the reason these three instructions know to jump to this address is also because the jump address is recorded in the `.rel.plt` table. The `.rel.plt` table contains the information required for relocating the entries in the PLT, as well as the symbol information required for relocation. By looking at the .`rel.plt` table in the assembly code, as shown in Figure 3-20, we can see the entry 52474.&#x20;

![Figure 3-20 .rel.plt table data](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter3_img_17.png)

The `.rel.plt` table is also located in the `.dynamic` section. We can use the d\_tag DT\_JMPREL to determine whether it is this table. The `.rel.plt` table contains the address of the malloc function in the `.got` table and the corresponding symbol of the malloc function. When traversing the `.dynamic` section, we can also obtain the symbol table DT\_SYMTAB and the size of the `.rel.plt` table DT\_PLTRELSZ. These two pieces of data will be used later. The implementation code is as follows:&#x20;

```c++
Elf32_Rel *rela;
Elf32_Sym *sym;
size_t pltrel_size = 0;
// Traverse .dynamic section
for (int i = 0; i < dynamicSize; i++) {
    if (dynamic_table[i].d_tag == DT_JMPREL) {
        //Obtain.rel.plt table
        rela = (Elf32_Rel *)(dynamic_table[i].d_un.d_ptr + base_addr);
        __android_log_print(ANDROID_LOG_DEBUG, "hookMallocByPLTHook", 
                            "DT_PLTRELSZ2 size:%d", dynamic_table[i].d_un.d_val);
    } else if(dynamic_table[i].d_tag == DT_PLTRELSZ){
        //Obtain.rel.plt table size
        pltrel_size = dynamic_table[i].d_un.d_val;
    } else if(dynamic_table[i].d_tag == DT_SYMTAB){
        //Obtain Symbol table
        sym = (Elf32_Sym *)(dynamic_table[i].d_un.d_val + base_addr);
    }
}
```

After obtaining the `.rel.plt` table, we then traverse the table, and use the symbol recorded in the table entry to obtain the name information of the symbol from the symbol table. If the name information string contains "malloc", it indicates that this entry is the target item we need to find. The code implementation is as follows:&#x20;

```c++
//Calculate the number of entries in the .rel.plt table
size_t entries = pltrel_size / sizeof(Elf32_Rel);
for (size_t i = 0; i < entries; ++i) {
    //Get the entry at index i
    Elf32_Rel *reloc = &rela[i];
    size_t symbol_index = ELF32_R_SYM(reloc->r_info);
    // Get the symbol from the symbol table based on symbol_index
    Elf32_Sym *sym = sym[symbol_index];
    // Get the name of the symbol based on the symbol
    std::string name = getSymbolNameByValue(base_addr,sym);
    if(name.find("malloc")!= std::string::npos){
        //Found the entry address of the malloc function in the GOT
        originFunc = reloc->r_offset + base_addr;
        break;
    }
}
```

In the above code, we obtained the symbol of the entry through the index of the entry, but the symbol only contains index data and no data for the symbol name. Therefore, we still need to call the getSymbolNameByValue method and obtain the corresponding symbol name based on this symbol. The table that records the symbol names is located in the symtab section. Therefore, we need to traverse the sections of the so and find the symtab section (SHT\_STRTAB). The code implementation is as follows. The process involves a lot of knowledge about symbols. If readers are not familiar with this part of the knowledge, they can read the more in-depth explanation of symbols in Chapter 5 and then come back to look at the process here.&#x20;

```c++
std::string getSymbolNameByValue(uintptr_t base_addr , Elf32_Sym *sym) {
    Elf32_Ehdr *header = (Elf32_Ehdr *) (base_addr);
    // Get the address of the section header table
    Elf32_Shdr *seg_table = (Elf32_Shdr *) (base_addr + header->e_shoff);
    // Get the number of sections
    size_t seg_count = header->e_shnum;
    Elf32_Shdr* stringTableHeader = nullptr;
    // Traverse the sections to find the symbol string table header
    for (int i = 0; i < seg_count ; i++) {
        if (seg_table[i].sh_type == SHT_STRTAB) {
            stringTableHeader = &seg_table[i];
            break;
        }
    }
    // Get the string table for symbol names
    char* stringTable = (char*)(base_addr + stringTableHeader->sh_offset);
    // Get the string of the symbol based on the symbol index
    std::string symbolName = std::string(stringTable + sym->st_name);
    return symbolName;
}
```

By now, we have truly and completely implemented the process of  PLT Hook . Readers can follow the code above to practice on their own, thereby deepening their understanding of the process.&#x20;

### 2. Use an open source framework&#x20;

Previously, we completed the implementation of PLT Hook step by step through code. Its principle is actually not very complicated, but the entire process involves a large number of operations to search for and modify target addresses in ELF files. Therefore, to become familiar with the entire process, we needs to have a relatively in-depth understanding of the so file format. Implementing it is still quite cumbersome, and errors can easily occur if one is not careful. When dealing with online environments, it is even more important to handle comprehensive compatibility and exceptions properly.

Native Hook is a well-established technology, and there are many relevant open-source libraries on GitHub. Therefore, after we understand the principles and processes, we don't need to reinvent the wheel and implement a complete PLT Hook library ourselves. I recommends several mainstream open-source PLT Hook libraries here:

* bhook: <https://github.com/bytedance/bhook>

* profilo: <https://github.com/facebookincubator/profilo/tree/main/deps/plthooks>

Here, I take the open-source plt hook library bhook as an example to hook the malloc function in the testmalloc.so library of the sample program. By using the bytehook\_hook\_single interface provided by bhook, one can easily implement the hooking of the malloc function in the libexample.so library. The code implementation is as follows.&#x20;

```c++
Java_com_example_performance_1optimize_memory_NativeLeakActivity_hookMallocByBHook(
                                                                        JNIEnv *env,
                                                                        jobject thiz) {
    bytehook_stub_t stub = bytehook_hook_single(
            "libexample.so",
            nullptr,
            "malloc",
            reinterpret_cast<void *>(malloc_hook),
            nullptr,
            nullptr);
}
```

After the Activity in the upper layer calls this method, as shown in Figure 3-21 through the Log, we can see that we have successfully detected a memory allocation of 100000000 ByteDance. By using a third-party open-source library, we can complete PLT Hook with just a few lines of code and ensure better stability and compatibility.&#x20;

![Figure 3-21 bhook Execution Success Log](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter3_img_18.png)

## 3.2.2 Obtain Native stack

Once an abnormal memory allocation is detected in a custom function, we need to obtain the stack to help us locate the problem. In Java code, we only need to call the Debug.DumpHeap method to obtain the stack of the current Java function, while in Native code, there is no direct method to obtain the stack, so we need to implement it ourselves. Here, I introduce a solution for obtaining the Native stack through CFI (Call Frame Information), which is currently the most commonly used solution in Android. For example, the stack output during a Native Crash and some official Android Native debugging tools all adopt this solution.&#x20;

During program execution, when a Native function enters the stack instruction, it writes information such as the address of the corresponding instruction into the `.eh_frame` and `.eh_frame_hdr sections`. These two sections are also part of the segment composition of the so ELF file. Therefore, to obtain the Native stack, one only needs to read the data from these two sections.&#x20;

In actual projects, we don't need to implement this solution ourselves. In the Android system, we can directly use the libunwind library to obtain Native stack information. However, the underlying principle of the libunwind library is actually implemented by reading CFI. The usage of the libunwind library is shown in the following code, where the \_Unwind\_Backtrace function is the function provided by the libunwind library for obtaining stack information. The input parameters of this function require passing in a callback function and a pointer data, and we can obtain this data in the callback function. The callback function can obtain the data returned by the \_Unwind\_Backtrace function during the stack traceback process, and can also control whether to continue the stack traceback. The pointer data input parameter can pass in a custom BacktraceState structure, which is used to store the data returned by the callback and to limit the maximum stack traceback depth.&#x20;

```c++
#include <unwind.h> 

struct BacktraceState {
    void **current;
    void **end;
};
void printNativeStack() {
    const size_t max = 30; // Max call stack layer
    void *buffer[max];    //used for saving stack frame address
    BacktraceState state = {buffer, buffer + max}; 
    _Unwind_Backtrace(unwindCallback, &state);   //cpature stack
    //get the depth of stack
    size_t depth =  state.current - buffer;
    //print the stack
    dumpBacktrace(buffer, depth);
}
```

After passing the custom callback function unwindCallback to the \_Unwind\_Backtrace method, we can receive the callback data in the callback function. At this point, we can extract the address of the stack frame and store it in the buffer container of the BacktraceState structure passed earlier. When the depth of the cached stack exceeds the previously configured threshold of 30, return \_URC\_END\_OF\_STACK to exit stack backtracking; in other cases, return \_URC\_NO\_REASON to indicate continuing backtracking. The code flow is as follows.&#x20;

```c++
// Callback function
static _Unwind_Reason_Code unwindCallback(struct _Unwind_Context *context, void *arg) {
    BacktraceState *state = static_cast<BacktraceState *>(arg);
    uintptr_t pc = _Unwind_GetIP(context);
    if (pc) {
         // Exit if the stack depth reaches the threshold
        if (state->current == state->end) {
            return _URC_END_OF_STACK;
        } else {
            // Store the address data into the buffer
            *state->current++ = (void *)pc;
        }
    }
    // Continue unwinding
    return _URC_NO_REASON;
}
```

Once the stack information has been cached in the buffer container, we can print out the stack information. The code is as follows.

```java
void dumpBacktrace(void **buffer, size_t depth) {
    for (size_t idx = 0; idx < depth; ++idx) {
        void *addr = buffer[idx];
        __android_log_print(ANDROID_LOG_DEBUG, "MallocHook",
                            "# %d : %p",
                            idx,
                            addr);
    }
}
```

## 3.2.3 Native Stack Information Restoration

When obtaining stack information through the \_Unwind\_Backtrace function, we can actually only obtain the hexadecimal address information of the stack. The log is shown in Figure 3-22. Based on these addresses, it is impossible to view valid information. Therefore, we also need to restore the addresses to the corresponding detailed function information.&#x20;

![Figure 3-22 Stack information obtained by Unwind](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter3_img_19.png)

Restoring the hexadecimal address stack to a stack with valid information can be done in two ways: online and offline:

1\)**Online stack information restoration**

First, we need to know which so library the address corresponds to. There are multiple ways to confirm the so library name, such as by parsing the maps file and then comparing the address range to confirm which so file it is. However, in actual business, we will call the dladdr function provided by the Linux system, which is specifically used to obtain information about the shared library where the specified address is located. The prototype of the function is as follows. In the function, the input parameter addr is the address to be queried, and info is a pointer to a Dl\_info structure used to store the query results.

```c++
#include <dlfcn.h>
int dladdr(const void *addr, Dl_info *info);

typedef struct {
    // SO name corresponding to the address
    const char *dli_fname;   
    // Base address of the corresponding SO library
    void       *dli_fbase;   
    /* If the SO library has a symbol table, 
    this displays the function symbol closest to the address*/
    const char *dli_sname;   
    // Address of the function closest to the address in the symbol table
    void       *dli_saddr;   
} Dl_info;
```

By calling this function in the sample program and printing the information, we can see that dladdr can not only obtain the so name corresponding to the address, but also obtain the symbol name corresponding to the address. Therefore, we can print out the address, so library name, and symbol name together to make the stack information more complete.&#x20;

```c++
void dumpBacktrace(void **buffer, size_t count) {
    for (size_t idx = 0; idx < count; ++idx) {
        void *addr = buffer[idx];
        Dl_info info;
        if (dladdr(addr, &info)) {
            //Obtain offset address
            const uintptr_t addr_relative =
                    ((uintptr_t) addr - (uintptr_t) info.dli_fbase);
            //print stack info
            __android_log_print(ANDROID_LOG_DEBUG, "MallocHook",
                                "# %d : %p : %s(%p)(%s)(%p)",
                                idx,
                                addr, info.dli_fname, addr_relative, info.dli_sname,
                                info.dli_saddr);
        }
    }
}
```

After running the program, as shown in Figure 3-23, more complete stack information can be seen. By the symbolic names of the functions in the stack, we can basically locate which functions have exceptions. The information in lines #0 and #1 of the stack log is actually our Hook function and stack capture function in the liboptimize.so library, so the information in line #2 is the location of the memory allocation exception, with the corresponding so named libexample.so and the exception function being "Java\_com\_example\_performance\_1optimize\_memory\_NativeLeakActivity\_mallocLeak".&#x20;

![Figure 3-23 Stack after Confirming so Library and Symbol Name](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter3_img_20.png)

Through the logs, we can also find that for so libraries with symbol tables, such as liboptimize.so and libexample.so, dli\_sname can display the correct symbol name of the function, but for so libraries whose symbol tables have been removed, such as libart.so, it will display as null. For official online packages, considering security and package size, we generally also remove the symbol tables of so libraries. After removal, the libexample.so library can no longer obtain symbols normally, and we will not be able to locate which function has a problem. At this time, we can perform offline stack restoration based on the stack address.&#x20;

2\)**Offline stack information restoration**

In the offline environment, we can use the addr2line tool to restore stack information. This tool will obtain information such as the function name and line number corresponding to the offset address based on the function's offset address. What is the offset address of a function? The hexadecimal value in the stack is the absolute address of the function, which is the address within the entire virtual Memory Space, while the offset address is the internal address within the so library. Therefore, the offset address can be obtained by simply subtracting the relative address of the so library from the absolute address. In the previous code for online stack restoration, we have actually already calculated the offset address of each address, as shown in the following code.&#x20;

```c++
const uintptr_t addr_relative = ((uintptr_t) addr - (uintptr_t) info.dli_fbase);
```

The offset address is supplemented. Through the Log, we can know that the offset address of the exception function is 0x1edea **.&#x20;**&#x20;Next, we can use the addr2line tool to restore the function name and corresponding line. This tool is already provided in Android's NDK. Execute the following command:&#x20;

```c++
addr2line -C -f -e libexample.so  0x1edea
```

In the command, -C indicates decoding low-level symbol names into user-level names, -f indicates displaying the function name while showing the file name and line number information, and -e is used to specify the name of the executable file whose addresses need to be converted. The execution result is shown in Figure 3-24. As can be seen, the result shows that the offset address is located on line 10 of the mock\_native\_leak.cpp file. Based on this information, we can accurately locate the problematic area.&#x20;

![Figure 3-24 addr2line Stack Restoration](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter3_img_21.png)

It should be noted that the so library parsed by the addr2line tool needs to have a symbol table; otherwise, stack restoration cannot be performed correctly. The so library with a symbol table can be found in the merged\_native\_libs (Figure 3-26) of the compilation output (Figure 3-25). There is also a stripped\_native\_libs file in the compilation output files, where all the so libraries have had their symbol tables removed, and these so libraries are used in the official online packages.

![3-25 Project Compilation Artifacts](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter3_img_22.png)

![3-26 so library with symbol table](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter3_img_23.png)

By now, we have completed the detection and location of the function with abnormal memory allocation in the sample program. Although the example demonstrates an obvious memory allocation exception, in actual projects, more often than not, it is relatively hidden memory leaks that occur. However, the principles and procedures for troubleshooting and location are the same as those explained here. We only need to additionally intercept the free memory release function, and then, at a fixed frequency, such as once every 10 minutes, subtract the total memory released by the free function from the total memory allocated by the malloc function to calculate the difference. If the difference continuously increases and exceeds the threshold we set, such as 512MB, then it is considered that the so library has a memory leak. The entire process of analyzing and addressing memory anomalies in the so library is relatively long, and there are also quite a few knowledge points. Here, it is recommended that readers operate it themselves once, which can help everyone better understand and recognize the entire process and the knowledge points involved.&#x20;

For third-party SDKs, the symbol tables have already been removed, so it is also impossible to view the corresponding function and line number through add2line. Even if the symbol table of the third-party SDK's so library has not been removed, we cannot modify it without the source code. Therefore, for third-party so libraries, we only need to use Native Hook to check whether there are memory leaks or abnormal issues in the third-party so library. If so, replace the so library with a stable and normal version.&#x20;

## 3.2.4 Introduction to Open Source Tools

Actually, developing a highly stable Native memory detection tool that can be used online requires a great deal of effort. If readers do not have the energy to develop a complete set of so library abnormal memory detection tools, we can also use existing open-source tools. Here I introduce the following two:

* malloc\_debug

malloc\_debug is a Native analysis tool officially provided by Googl&#x65;**,&#x20;**&#x61;nd its technical principle is consistent with the process described above. However, it intercepts functions related to memory allocation in the entire Zygote process and can only be used on rooted phones. It is not very flexible to use, has poor performance, and can only be used for offline work.

* memory-leak-detector

MemoryLeakDetector is an open-source Native memory leak monitoring tool developed by ByteDance, featuring easy integration, wide monitoring scope, excellent performance, and good stability, and it has been verified in numerous ByteDance apps in production.

The official usage documentation for mature and stable third-party open-source tools is always very detailed, so I will not repeat the explanation of how to use them here. It is recommended that readers try out these two tools and run through the process. Those who are interested can also read the source code of these two libraries, as the basic principles in the source code are actually similar to what was explained earlier.&#x20;

# 3.3 Bitmap Governance

In the optimization direction of reducing data loading, we usually have two steps: the first step is to discover low-frequency, redundant, or excessively large data in the program through manual analysis of business code and stack, or automated analysis and monitoring mechanisms; the second step is to optimize the data discovered through the previous analysis, and at this time, the optimization solutions are generally relatively simple, which is nothing more than not loading or reducing the amount of data loaded. Manual analysis of business code has low universality, so it will not be introduced in detail here. I mainly focus on  how to discover and optimize large images through automated mechanisms.&#x20;

For most applications, Bitmaps usually account for a large portion of memory usage because as long as an application uses images, it will use Bitmaps. Starting from Android 8, the memory usage of Bitmaps is counted in Native memory, while in systems prior to Android 8, Bitmaps were counted in Java memory. Although the proportion of devices running Android versions below 8 on the market is no longer significant, regardless of whether it consumes Native or Java memory, optimizing Bitmaps is one of the directions with relatively high returns in memory optimization.&#x20;

The key to managing and optimizing Bitmaps lies in how to identify Bitmaps that are used unreasonably in the application, such as Bitmaps with high memory usage or leaked Bitmaps. Once we identify these abnormal Bitmaps, we can then adopt some general solutions, such as reducing the image resolution or quality, and promptly clearing Bitmap references, to complete the optimization and management of these abnormal Bitmaps. To identify abnormal Bitmaps through an automated mechanism, we still need to use Hook technology. Earlier, we learned how to perform Hook technology in Native layer code, and here the I will continue to introduce Hook technology in Java layer code: bytecode manipulation technology.&#x20;

## 3.3.1 Bytecode Operations

During the packaging process, the project will go through two steps: compiling Java files into .class bytecode files, and then generating dex files from the.class bytecode files, as shown in Figure 3-27.&#x20;

![Figure 3-27 Java File Compilation Process](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter3_img_24.png)

In these two processes, we can modify the source code through the following techniques:&#x20;

* APT: Annotation Processor, which can operate in three stages: pre-compilation, compilation, and runtime. The commonly used @Override annotation belongs to pre-compilation annotation, which operates during the pre-compilation stage of the process. The well-known ButterKnife framework belongs to compilation annotation, which automatically generates repetitive code such as findViewById during the compilation process.&#x20;

* AspectJ: AspectJ can modify files during the stage when Java files are compiled into class files. AspectJ inserts the newly added bytecode into the bytecode of the original file through a dedicated compiler to enhance the functionality or logic of the original code, but it does not directly modify the bytecode of the original file to change the original logic.

* ASM and Javaassist: Both tools can modify .class files when they are compiled into dex files. They can both modify the bytecode of the original file to change the functionality or logic of the source code. The difference between the two is that ASM has higher flexibility and can precisely control the generation and modification process of bytecode, but it requires a certain understanding of bytecode structure and operations, while Javaassist hides the complexity of the underlying bytecode, allowing developers to directly manipulate Java classes, methods, fields, etc., without directly operating on bytecode.

In Android, the most widely used method is still to modify code through ASM. The process of using ASM to modify, transform, or enhance Java bytecode is bytecode manipulation. Bytecode manipulation is very widely used in actual projects and is commonly used for function expansion, performance optimization, dynamic debugging, and so on.&#x20;

To enable readers to gain a deeper understanding of bytecode operations, here we use bytecode operations to implement a simple case: adding the ability to print "hello world" logs to the original function. Such operations that do not modify the logic of the original function but only enhance the function's capabilities are also known as instrumentation operations.&#x20;

Android uses Gradle scripts to package and compile projects. So, how can we use ASM in Gradle to implement instrumentation? In fact, when Android compiles a project through Gradle, at some specific stages, the bytecode of the compiled Java files in the project is passed back to the script in Gradle for further processing. This stage is called the Transform stage. Therefore, we can write a custom Gradle script and register it in the Transform stage. In our custom script, we can obtain the bytecode in the project and use ASM to instrument the methods in the bytecode. There are two main steps in the entire instrumentation process:&#x20;

* First, register the custom script to the Transform stage&#x20;

* Second, perform instrumentation through ASM in custom scripts&#x20;

### 1. **Transform Script Registration**

Let's first look at the first thing, the registration of the Transform custom script. The implementation process is as follows.

1\) Create a new buildSrc module under the root directory, and introduce the ASM library in the gradle file of this module. Since the gradle script in the my example program is written in Groovy, it is also necessary to introduce the Groovy library through implementation localGroovy() in the library dependency configuration, and enable the Groovy plugin through the code apply plugin: 'groovy'. For Android projects, buildSrc is a special module, and we do not need to configure module introduction; the project can automatically recognize this module.

```c++
apply plugin: 'groovy'

dependencies {
    implementation 'com.android.tools.build:gradle:4.1.1'
    implementation localGroovy()
    implementation 'org.ow2.asm:asm:7.1'
    implementation 'org.ow2.asm:asm-util:7.1'
    implementation 'org.ow2.asm:asm-commons:7.1'
}
```

2\) Then we can start writing the Gradle script. The code implementation is as follows: create a new script MyAsmPlugin that inherits from the Plugin base class provided by Gradle. In the apply callback method, obtain the AppExtension, and register our custom AsmTransform script into the Transform phase through the registerTransform function. AppExtension is an extended object for the configuration and properties of an Android program. Through AppExtension, we can customize various settings such as build types, dependencies, signing configurations, and resource processing.

```c++
class MyAsmPlugin implements Plugin<Project> {
    @Override
    void apply(Project project) {
        // Obtain AppExtension
        AppExtension appExtension = project.getExtensions().getByType(AppExtension.class);
        // Register custom Transform Script
        appExtension.registerTransform(new MyAsmTransform());
    }
}
```

3\) Create a new "plugin\_name.properties" file in the resouces/META-INF/gradle-plugins/ directory of buildSrc, as shown in Figure 3-28, and configure the entry script in this file. Then, enable the script we configured by applying the plugin: 'MyAsmPlugin' in the gradle script of the app module.

![Figure 3-28 Gradle Script Configuration](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter3_img_25.png)

At this point, the registration of the custom Transform script is completed. When the project is compiled and reaches the Transform phase during execution, our custom script will execute normally.&#x20;

### 2. **Perform bytecode instrumentation&#x20;**

Having already registered the custom Transform script previously, the next step is to modify the method in the source code within the custom MyAsmTransform script to insert the logic for printing "hello world".&#x20;

MyAsmTransform inherits from the system's Transform base class. In the transform callback method, we can iterate through and obtain all class bytecode files in the project. After we obtain these class bytecode files, we can perform bytecode operations through the capabilities provided by ASM to insert our own logic.

ASM provides three classes, ClassReader, ClassVisitor, and ClassWriter, which work together to perform bytecode operations. Among them, ClassReader is used to read and parse the bytecode of a class and trigger corresponding callback methods to ClassVisitor. ClassVisitor then operates on these bytecode files in the callback function, and finally ClassWriter writes back the modified bytecode. At this point, we can implement the code logic. As shown below, in the code, after obtaining the class bytecode file through file traversal, it is converted into a stream and passed to ClassReader, and bytecode operations are performed in the custom TestClassVisitor.

```c++
class MyAsmTransform extends Transform {
    @Override
    void transform(TransformInvocation transformInvocation)
        // Get files from the project
        Collection<TransformInput> inputs = transformInvocation.getInputs();
        TransformOutputProvider outputProvider = 
                transformInvocation.getOutputProvider();
        for (TransformInput input : inputs) {
            Collection<DirectoryInput> directoryInputs = input.getDirectoryInputs()
            // Traverse through class files
            for (DirectoryInput directoryInput : directoryInputs) {
                File dstFile = outputProvider.getContentLocation(
                        directoryInput.getName(),
                        directoryInput.getContentTypes(),
                        directoryInput.getScopes(),
                        Format.DIRECTORY);
    
                // Create BufferedInputStream and BufferedOutputStream based on bytecode files
                BufferedInputStream bis = 
                        new BufferedInputStream(directoryInput.getFile())
                BufferedOutputStream bos = new BufferedOutputStream(dstFile)
                // Read input stream through ClassReader
                ClassReader reader = new ClassReader(bis)
                ClassWriter writer = new ClassWriter(reader)
                // Use ASM to manipulate methods in class files within the custom TestClassVisitor
                ClassVisitor cv = new TestClassVisitor(writer)
                reader.accept(cv);
                bos.write(writer.toByteArray());
            }
        }
    }
}
```

TestClassVisitor inherits from ASM's ClassVisitor class. ClassVisitor provides a callback method visitMethod, which represents method access. In this callback method, we can further use AdviceAdapter to divide the logic of method access into two timings: onMethodEnter when entering the method and onMethodExit when exiting the method. Therefore, at the timing of entering the method, we can use the MethodVisitor object provided by the ASM library for accessing and modifying instructions in the method bytecode to add the logic of printing "Hello world" to the method. The code implementation is as follows.

```c++
class TestClassVisitor extends ClassVisitor {

    TestClassVisitor(int api) {
        super(api)
    }
    
    @Override
    MethodVisitor visitMethod(int access, String name, String desc, String signature, String[] exceptions) {
        MethodVisitor mv = super.visitMethod(access, name, desc, signature, exceptions);
        // Take out all methods that need to be modified and create visitors one by one to process them
        mv = new TestMethodVisitor(changer, mClassName, mv, access, name, desc);
        return mv;
    }

    private static class TestMethodVisitor extends AdviceAdapter {
        protected TestMethodVisitor(int api, MethodVisitor mv, int access, String name, String desc) {
            super(api, mv, access, name, desc)
        }
    
        @Override
        protected void onMethodEnter() {
            /* Get the Out object. The visitFieldInsn method is used to access bytecode instructions related to fields, 
            GETSTATIC is the instruction for getting static fields */
            mv.visitFieldInsn(Opcodes.GETSTATIC,
                    "java/lang/System", "out", "Ljava/io/PrintStream;")
                    
            /* Load constant. The visitLdcInsn method is used to load constants (such as strings, integers, floating-point numbers, etc.) onto the operand stack */
            mv.visitLdcInsn("hello world")
            
            /* Call the print method. The visitMethodInsn method is used to access bytecode related to method invocation,
            INVOKEVIRTUAL is the command for virtual method invocation */
            mv.visitMethodInsn(Opcodes.INVOKEVIRTUAL,
                    "java/io/PrintStream", "print", "(Ljava/lang/String;)V", false)
            super.onMethodEnter()
        }
        
        @Override
        protected void onMethodExit(int opcode) {

        }
    }
}
```

By now, the instrumentation to add "hello world" log output to all methods in the project has been completed. Regarding the detailed rules of bytecode, we don't need to memorize them either. We can use the javap command to convert Java code into readable bytecode, or directly view the bytecode of Java code through an AS plugin. By now, we have completed a simple method instrumentation case through ASM bytecode manipulation. It is recommended that readers also try it themselves to deepen their understanding and usage of ASM.&#x20;

Readers need to note that starting from AGP (Android Gradle Plugin) 7.0, the way to register custom Gradle tasks has changed significantly, and it needs to be registered through [AndroidComponentsExtension](https://developer.android.com/studio/build/extend-agp?hl=zh-cn), and Transform has also been replaced by BytecodeTransformationPlugin. I will not elaborate on the usage of AGP 7.0 and above here. Interested readers can conduct your own research on the changes or complete the adaptation of bytecode injection for AGP 7.0 and above.

### 3. Use an open source framework&#x20;

The way of performing bytecode instrumentation by manually writing bytecode is not very easy to understand, so it is error-prone and has a high learning cost. Fortunately, there are many open-source bytecode instrumentation tools that encapsulate bytecode operations and provide simpler instrumentation methods to complete bytecode instrumentation. Here, I introduce a simple, easy-to-use, and mature open-source framework for bytecode operations: [Lancet](https://github.com/eleme/lancet), and uses it to quickly implement bytecode instrumentation.&#x20;

Lancet also modifies bytecode through ASM, but we no longer need to modify the bytecode ourselves. We only need to specify the points to be modified in the form of annotations, and the framework will automatically help us complete the bytecode modification. Let's take a look at the example provided in the official documentation. With just a few simple lines of code below, we can replace all Log.i(tag, msg) method code with Log.i(tag, msg + "lancet").&#x20;

```c++
@Proxy("i")
@TargetClass("android.util.Log")
public static int anyName(String tag, String msg){
    msg = msg + "lancet";
    return (int) Origin.call();
}
```

In this code, the annotation @TargetClass specifies the target class android.util.Log into which the code will be woven, and the annotation @Proxy specifies the target method i into which the code will be woven. The weaving method is Proxy, indicating that the original method will be replaced. Finally, Origin.call() is called to execute the original method Log.i(). I will not provide too much introduction to the usage of Lancet here, as it is explained in detail in the official documentation, and readers can check it themselves.&#x20;

Although it is very easy to perform bytecode operations with Lancet, and Lancet is also widely used within ByteDance for bytecode operations, the Lancet library is not compatible with the newer AGP because it has not been updated for a long time. If we need to be compatible with the newer AGP, we can download the Lancet source code and then modify and adapt it or use other open source framework, l. The Lancet used within ByteDance is also a modified and adapted version. Of course, if you don't want to adapt it ourselves, there are also many libraries adapted to the latest AGP on Github, like FlyJingFish:AndroidAOP, and readers can find them by searching on Github.

## 3.3.2 Optimization of Super Large Bitmaps

Once we have mastered the bytecode instrumentation technology, we can optimize the oversized Bitmaps in the program. This optimization is divided into two steps: first, intercept the creation of Bitmaps and identify those with high memory usage; second, optimize the abnormal Bitmaps in the interception logic. I then implemented this optimization solution through Lancet.

### 1. **Intercept Bitmap Creation&#x20;**

To detect abnormal Bitmaps, the best time is when the Bitmap is created. Therefore, we need to intercept it before the Bitmap is created. To find the entry point for interception, we need to first analyze the source code of [Bitmap](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/graphics/java/android/graphics/Bitmap.java?q=bitmap\&ss=android%2Fplatform%2Fsuperproject) to understand its creation process.

Bitmap is created through the Bitmap.createBitmap static function, and the createBitmap static function in turn calls the Bitmap.cpp object in the Native layer to create the real Bitmap. The final Bitmap is actually just a memory area created by calling the calloc function, used to store the metadata of our image. The creation process is shown in Figure 3-29.

![Figure 3-29 Bitmap Creation Process ](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter3_img_26.png)

Combining with the above flow chart, it can be found that there are two entry points for interception. One is to intercept when creating a Bitmap at the Java layer, and the other is to intercept the Bitmap creation function at the Native layer through Native Hook technology. However, intercepting the Bitmap creation process at the Native layer is relatively complex, with poorer stability. Moreover, even when we intercept Bitmaps at the Native layer and detect an exception, we still need to obtain the Java layer stack through JNI calls to effectively locate the position of the abnormally requested Bitmap. Therefore, it is recommended here to intercept Bitmap creation at the Java layer. The main static methods for creating Bitmaps at the Java layer are as follows.

```java
public static Bitmap createBitmap(int width, int height, Bitmap.Config config) 
public static Bitmap createBitmap(Bitmap src)
public static Bitmap createBitmap(Bitmap source, int x, int y, int width, int height)
public static Bitmap createBitmap(Bitmap source, int x, int y, int width, int height, Matrix m, boolean filter)
public static Bitmap createBitmap(int width, int height, Bitmap.Config config, boolean hasAlpha)
public static Bitmap createBitmap(DisplayMetrics display, int width, int height, Bitmap.Config config, boolean hasAlpha)
public static Bitmap createBitmap(DisplayMetrics display, int width, int height, Bitmap.Config config, boolean hasAlpha, ColorSpace colorSpace)
```

Therefore, we only need to intercept these few methods to detect the creation of Bitmaps in the program. Here, I take one of the creation methods as an example and intercepts it through Lancet. The code implementation is as follows. As can be seen, with Lancet, we don't need to write Gradle scripts or perform any bytecode operations; we can directly implement bytecode operations through Java code and annotations. In the intercepted code logic, the memory footprint of the Bitmap being created will be detected and logged before the Bitmap is created. Different Bitmap formats have different memory consumption. The common ARGB\_8888 format is 4 bytes in size, so when we use this format to display an image, the memory footprint is the width × height × 4 bytes of the image size. Other formats such as ARGB\_4444 and RGB\_565 are 2 bytes.

```sql
@TargetClass(value = "android.graphics.Bitmap")
@Proxy(value = "createBitmap")
public static Bitmap createBitmap(int width, int height, Bitmap.Config config) {
    monitorAndOptimizeBitmap(width, height, config);   
    return (Bitmap) Origin.call();
}

private static Object[] monitorAndOptimizeBitmap(int width, int height, 
                                                    Bitmap.Config config) {
    float factor = 1;
    if (config.name().equals(Bitmap.Config.ARGB_8888.name())) {
        factor = 4;
    } else if (config.name().equals(Bitmap.Config.ARGB_4444.name()) 
            || config.name().equals(Bitmap.Config.RGB_565.name())) {
        factor = 2;
    } else if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O 
            && config.toString().equals(Bitmap.Config.RGBA_F16.name())) {
        factor = 8;
    }
    //Calcuate Bitmap Memory
    float size = width * height * factor / (1024f * 1024f);
    Log.i(TAG, "Crete Bitmap Size:" + size + "M" + 
                            " Width:" + width + 
                            " Height:" + height );
}
```

After running, through the log, as shown in Figure 3-30, it can be seen that the creation of Bitmap was successfully detected and the size of the created Bitmap was output.&#x20;

![Figure 3-30 Successful Interception Log](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter3_img_27.png)

### 2. **Improve interception logic**

Simply printing the size of a Bitmap is actually not very helpful for optimizing extremely large Bitmaps, so it is necessary to continue improving the instrumentation logic. We can set a Bitmap size threshold, which can be determined based on the actual scenario. My strategy here is to set it according to the device model and screen resolution. For example, for a low-end mobile phone with a resolution of 1920\*1080, its maximum Bitmap threshold is 15M, which is the memory size occupied by an image that just fills the entire phone screen and has a format of ARGB8888. For Bitmaps that exceed this threshold, the stack needs to be printed and the exception reported. With this information, we can locate the specific position of the image and further investigate whether the image is abnormal.&#x20;

In addition to detecting abnormal images and collecting log data, we can also perform fallback optimization. For example, when the available memory on low-end devices is limited, we can scale down images that exceed the threshold proportionally in the interception logic. The scaling rule can be to reduce the width of the image to the width of the screen while scaling the height by the same proportion. We can also reduce the format from ARG8888 to ARGB565, which directly cuts the memory usage of the image in half.&#x20;

After combining the two strategies mentioned above, the implementation code of the further optimized Bitmap interception function is as follows. In the code, simply modifying the attribute configurations such as width, height, or Config in the input parameters can complete the resetting of the image to achieve fallback optimization. To prevent some "collateral damage" during fallback optimization, we can further add an allowlist mechanism, and fallback optimization is only performed in Activity scenarios that are not on the allowlist.&#x20;

```java
@TargetClass(value = "android.graphics.Bitmap")
@Proxy(value = "createBitmap")
public static Bitmap createBitmap(int width, int height, Bitmap.Config config) {
    // Intercept bitmap creation, perform detection and fallback processing
    Object[] objects = monitorAndOptimizeBitmap(width, height, config);
    if (objects.length == 3) {
        // Reset image width, height, and config
        width = (int) objects[0];
        height = (int) objects[1];
        config = (Bitmap.Config) objects[2];
    }
    return (Bitmap) Origin.call();
}

private static Object[] monitorAndOptimizeBitmap(int width, int height, 
                                                    Bitmap.Config config) {
    float factor = 1;
    if (config.name().equals(Bitmap.Config.ARGB_8888.name())) {
        factor = 4;
    } else if (config.name().equals(Bitmap.Config.ARGB_4444.name()) 
            || config.name().equals(Bitmap.Config.RGB_565.name())) {
        factor = 2;
    } else if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O 
            && config.toString().equals(Bitmap.Config.RGBA_F16.name())) {
        factor = 8;
    }
    float size = width * height * factor / (1024f * 1024f);

    if(isLowDevice() && bitmapLimit == 0) {
        // On low-end devices, set the abnormal image threshold to the size of the entire screen
        bitmapLimit = UIUtils.getScreenWidth(AppContext.getApplication()) *
             UIUtils.getScreenHeight(AppContext.getApplication()) * 4 / (1024f * 1024f);
    }
    if (bitmapLimit != 0 && size > bitmapLimit) {
        // For images exceeding the threshold, print business scenario (Activity dimension), stack trace and other detailed information
        Log.e(TAG, "Bitmap over "+bitmapLimit + "M limit:" + size + "M" + 
                " Width:" + width + " Height:" + height + 
                " Scene:" + scene, new Throwable());

        if (isLowDevice() && !isSceneInWhiteList(scene)) {
            // On low-end models, perform fallback processing on images by reducing image quality
            if (config.name().equals(Bitmap.Config.ARGB_8888.name())) {
                config = Bitmap.Config.RGB_565;
                Log.i(TAG, "bitmap optimized");
            }
            return new Object[]{width, height, config};
        }
    } 
    return new Object[]{width, height, config};
}
```

## 3.3.3 Bitmap Leak Optimization

In addition to automatically detecting and optimizing extremely large Bitmaps, another commonly implemented Bitmap optimization strategy is the management of Bitmap memory leaks. Although Bitmaps are ultimately created in the Native layer, the management of Bitmap memory leaks can actually be transformed into the management of Java object memory leaks. Since Bitmaps utilize the auxiliary automatic Native memory reclamation technology provided by Android, we only need to clear the Java objects of Bitmaps at the Java layer, and the memory allocated in the Native layer will also be automatically released.&#x20;

Below is the constructor function of Bitmap, where the NativeAllocationRegistry object binds the Java layer and Native layer of this Bitmap. After the Java object of the Bitmap is reclaimed due to GC, NativeAllocationRegistry can assist in reclaiming the Native memory allocated by this Bitmap object.

```c++
Bitmap(long nativeBitmap, int width, int height, int density,
        boolean isMutable, boolean requestPremultiplied,
        byte[] ninePatchChunk, NinePatch.InsetStruct ninePatchInsets) {
    ...
    mNativePtr = nativeBitmap;
    long nativeSize = NATIVE_ALLOCATION_SIZE + getAllocationByteCount();
    //Auxiliary reclaim native memory
    NativeAllocationRegistry registry = new NativeAllocationRegistry(
        Bitmap.class.getClassLoader(), nativeGetNativeFinalizer(), nativeSize);
    registry.registerNativeAllocation(this, nativeBitmap);
    if (ResourcesImpl.TRACE_FOR_DETAILED_PRELOAD) {
        sPreloadTracingNumInstantiatedBitmaps++;
        sPreloadTracingTotalBitmapsSize += nativeSize;
    }
}
```

NativeAllocationRegistry can help us better avoid memory leak issues in the Native layer. When developing Android Native, I also recommends that everyone try to use this technology to reduce memory problems. Generally speaking, when the Bitmap in the Java layer is released, the Bitmap in the Native layer is also released. Knowing this, we only need to find the leaked Bitmap objects in the Java layer and then recycle them by setting them to null.

At this point, it is easy to find the leaked Bitmap, just like finding a leaked Java object. After the business ends, we manually perform a GC, capture the Hprof file, then use MAT or the tool built into Android Studio to find the Bitmap objects that have not been released, and then analyze whether there is a leak. If it is determined that a leak has occurred, the way to fix it is also the same as for Java objects: by analyzing the reference chain, find the GC Root that holds the Bitmap object, and promptly set it to null.

# 3.4 Thread Stack Optimization

There is still a relatively large proportion of 32-bit models in the market, so we often encounter program crashes caused by insufficient virtual memory space. Therefore, this section will mainly introduce the optimization of virtual memory. For optimization in this direction, the most commonly used solution is to release unused memory in the virtual memory space to increase the size of virtual memory.&#x20;

Since we need to release the unused memory in the virtual memory space, we inevitably have to analyze the maps file and find the memory that can be released from it. By examining the maps file, as shown in Figure 3-31, we can find that the virtual memory size occupied by each thread stack (anno: stack\_and\_tls) is approximately 1MB. For a slightly larger application, it is quite normal to use hundreds of threads during its operation, and the total virtual memory consumed by these threads can reach up to hundreds MB. Therefore, optimizing the thread stack size is also one of the effective virtual memory optimization solutions.

![Figure 3-31 maps File Data ](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter3_img_28.png)

## 3.4.1 Thread Creation Process

In the Android system, each thread will at least request 1MB of virtual space as the stack space size. I will first lead you through the thread creation process to verify this point.&#x20;

When we use threads to execute tasks, we usually first call new Thread(Runnable runnable) to create an instance of the [Thread.java](https://cs.android.com/android/platform/superproject/+/master:libcore/ojluni/src/main/java/java/lang/Thread.java;bpv=0;bpt=1) object. In the constructor of Thread, the stackSize variable is set to 0. This stackSize variable determines the size of the thread stack. Then we execute the start method provided by the Thread instance to run this thread. The start method calls the nativeCreate Native function to create and run a thread at the system level. The code flow is as follows.&#x20;

```java
Thread(ThreadGroup group, String name, int priority, boolean daemon) {
    ……
    this.stackSize = 0;
}

public synchronized void start() {
    if (started)
        throw new IllegalThreadStateException();
    group.add(this);
    started = false;
    try {
        nativeCreate(this, stackSize, daemon);
        started = true;
    } finally {
        try {
            if (!started) {
                group.threadStartFailed(this);
            }
        } catch (Throwable ignore) {

        }
    }
}
```

From the source code of the start function above, we can see that nativeCreate passes in stackSize, but its default value is 0. So why does a thread still have a default stack space of 1MB? We need to continue looking at the source code implementation of the nativeCreate function, whose implementation class is [java\_lang\_Thread.cc](https://cs.android.com/android/platform/superproject/+/master:art/runtime/native/java_lang_Thread.cc?q=Thread_nativeCreate), and the source code is as follows.

```c++
static void Thread_nativeCreate(JNIEnv* env, jclass, jobject java_thread, jlong stack_size, jboolean daemon) {
  Runtime* runtime = Runtime::Current();
  if (runtime->IsZygote() && runtime->IsZygoteNoThreadSection()) {
    jclass internal_error = env->FindClass("java/lang/InternalError");
    CHECK(internal_error != nullptr);
    env->ThrowNew(internal_error, "Cannot create threads in zygote");
    return;
  }
  //Chreate Thread
  Thread::CreateNativeThread(env, java_thread, stack_size, daemon == JNI_TRUE);
}
```

nativeCreate will execute the Thread::CreateNativeThread function, which is the ultimate place where the thread is created. Its implementation is in [Thread.cc](https://cs.android.com/android/platform/superproject/+/master:art/runtime/thread.cc?q=thread.cc), and within this function, the FixStackSize method will be called to adjust stack\_size to 1MB. So the previous question is resolved here. Even if we set stack\_size to 0, it will still be adjusted here. The simplified code logic is as follows.

```c++
void Thread::CreateNativeThread(JNIEnv* env, jobject java_peer, size_t stack_size, bool is_daemon) {
  ……
  // Adjust stack_size，Default 1 MB
  stack_size = FixStackSize(stack_size);
  ……
  
  if (child_jni_env_ext.get() != nullptr) {
    pthread_t new_pthread;
    pthread_attr_t attr;
    child_thread->tlsPtr_.tmp_jni_env = child_jni_env_ext.get();
    CHECK_PTHREAD_CALL(pthread_attr_init, (&attr), "new thread");
    CHECK_PTHREAD_CALL(pthread_attr_setdetachstate, (&attr, PTHREAD_CREATE_DETACHED),
                       "PTHREAD_CREATE_DETACHED");
    CHECK_PTHREAD_CALL(pthread_attr_setstacksize, (&attr, stack_size), stack_size);
    // Create Thread
    pthread_create_result = pthread_create(&new_pthread,
                                           &attr,
                                           Thread::CreateCallback,
                                           child_thread);
    CHECK_PTHREAD_CALL(pthread_attr_destroy, (&attr), "new thread");

    if (pthread_create_result == 0) {
      child_jni_env_ext.release();  
      return;
    }
  }

  ……
}
```

In the simplified code above, we can see that the source code implementation of CreateNativeThread ultimately calls the pthread\_create function, which is a Linux function, and the pthread\_create function will ultimately call [clone](https://man7.org/linux/man-pages/man2/clone.2.html) this kernel function. The main role of this function is to create a new process that shares specified resources with the calling process. The clone function will, based on the size of the stack passed in, allocate a block of virtual memory of the corresponding size through the mmap function and create a process. Therefore, for Linux systems, threads are actually lightweight processes that can share resources.&#x20;

```c++
int clone(int (*fn)(void * arg), void *stack, int flags, void *arg);
```

Having understood that a thread occupies 1MB of virtual memory, we can naturally think of two optimization strategies: reducing the number of threads and reducing the amount of virtual memory occupied by each thread.&#x20;

## 3.4.2 Reduce the number of threads

Let's first look at how to reduce the number of threads, mainly in two ways:

1. Use a unified thread pool in the program to manage the number of threads&#x20;

2. Converge wild threads and wild thread pools in the program

ThreadPool is a very important piece of knowledge that Android developers need to be familiar with its principles and proficient in using. ThreadPool greatly helps improve the performance of applications, as it can assist us in using threads more efficiently and reasonably, thereby enhancing application performance. Chapter 5, "Practical Optimization of Speed and Smoothness," will introduce in detail and depth the use of thread pools, so I will not elaborate on it here. Instead, this section mainly focuses on the direction of how to reduce the number of threads and introduces the optimal setting of the number of threads in a thread pool.

For thread pools, we need to manually set the number of core threads and the maximum number of threads. Core threads are threads that will not exit and will always exist after being created by the thread pool. The maximum number of threads is the maximum number of threads that the thread pool can reach. When the maximum number of threads is reached, subsequent tasks accepted by the thread pool are treated as exceptions and handled in the fallback logic. So, what is the appropriate number to set for the number of core threads and the maximum number of threads? These two values should be configured differently based on the type of thread pool, namely CPU thread and IO thread pool.&#x20;

* CPU Thread Pool

  The CPU thread pool is used to handle CPU-type tasks, such as operations like computation and logic, which require quick response but should not take too long. Therefore, for the CPU thread pool, we set the number of core threads to the number of CPU cores of the phone. Ideally, each core can run one thread, which can reduce the scheduling overhead of the CPU thread pool and fully leverage the CPU performance.&#x20;

  As for the maximum number of threads in the CPU thread pool, it is sufficient to keep it consistent with the number of core threads. Because when the maximum number of threads exceeds the number of core threads, it will actually reduce CPU utilization, as more CPU resources will be used for thread scheduling at this time. If the number of threads corresponding to the number of CPU cores cannot meet our business needs, it is very likely that there is a problem with our use of the CPU thread pool, such as executing IO-blocking tasks in CPU threads.&#x20;

* IO Thread Pool

  Tasks that take a long time, such as file read/write, network requests, and other IO operations, are handled by the IO thread pool, which is specifically designed to handle tasks that are time-consuming and do not require a very rapid response. For the IO thread pool, we usually set the number of core threads to 0, and since the IO thread pool does not require timely response, setting the number of resident threads to 0 can reduce the number of threads in the application. However, this does not mean that it must be set to 0; if our business has a relatively large number of IO tasks, we can also set a small number of core IO threads.

  Regarding the maximum number of threads in the IO thread pool, it can be set according to the complexity of the application. For small and medium-sized applications with relatively simple business, setting it to 64 is sufficient; for large applications with multiple and complex business, it can be set to 128 or even more.

According to such settings, even under extreme conditions where threads are fully utilized, there are only a little over a hundred, and the virtual memory consumed is only a little over 100 MB, which doesn't seem to take up much memory. However, in reality, there are always many places in the program that do not follow the specifications and create threads or thread pools independently, which we call wild threads or wild thread pools. So how can we rein in wild threads and wild thread pools?

For simple applications, we can conduct a one-by-one inspection. By performing a global search for keywords such as "new Thread" or "newFixedThreadPool", we can identify the code for creating threads and thread pools in the project. Then, we can modify the non-compliant code and converge it into the common thread pool.&#x20;

However, if it is a medium to large application that also extensively introduces second-party libraries, third-party libraries, and AAR packages, then global search will no longer work. At this point, we need to use bytecode operation technology. I still demonstrates the code through Lancet, and the implementation is as follows: by intercepting the function that creates a thread pool using newFixedThreadPool and replacing the creation of the thread pool with our common thread pool within the function, we can complete the convergence of the thread pool.

```java
public class ThreadPoolLancet {

    @TargetClass("java.util.concurrent.Executors")
    @Proxy(value = "newFixedThreadPool")
    public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
        // Replace with and return our public thread pool
        ……
    }

    @TargetClass("java.util.concurrent.Executors")
    @Proxy(value = "newFixedThreadPool")
    public static ExecutorService newFixedThreadPool(int nThreads) {
       // Replace with and return our public thread pool
        ……
    }
}
```

After converging the wild thread pool, how should we converge the wild threads created directly using "new Thread"? For wild threads in third-party libraries, we don't have very good means of convergence, because even if the constructor of Thread is intercepted, we cannot converge them into the common thread pool. Fortunately, most of the third-party libraries we use are already very mature and have been verified by a large number of users, so there will be few places where wild threads are directly used. We can intercept the constructor of Thread and print the stack to determine whether this thread was created through a thread pool or a wild thread. If there are indeed a large number of wild threads in a third-party library, then we can download the source code and manually modify it.&#x20;

## 3.4.3 Reduce the default stack memory of threads

As I mentioned earlier when explaining the source code of CreateNativeThread, this function will execute the FixStackSize method to adjust stack\_size to 1MB. Considering the previous cases of various hooks, it is easy for us to think that by intercepting the FixStackSize function through Native Hook, can we reduce stack\_size from 1MB to 512KB? Of course we can, because CreateNativeThread is a function located in libart.so, but CreateNativeThread actually calls pthread\_create to create threads, and pthread\_create is a function located in the libc.so library. If pthread\_create is called in CreateNativeThread, function calls need to be made through the PLT table and GOT table. Therefore, we can use PLT Hook technology to intercept the pthread\_create function in the libc.so library and directly set stack\_size in the input parameter \&attr to 512KB.&#x20;

However, this approach lacks flexibility because in actual projects, we usually do not adjust the stack size of all threads. For some threads with heavy tasks, we will retain their original stack size. Therefore, we need to use an allowlist and dynamic configuration scheme to exclude those threads that do not need adjustment. So the best way for us is to be able to configure the stack space size of a thread when creating it at the Java layer.

In the process of thread creation in the previous thread, we learned that when creating a thread at the Java layer, the stack\_size is passed to the Native layer, and the default value of stack\_size in the Java layer is always 0. In the FixStackSize function at the Native layer, the stack\_size will then be adjusted, and the code implementation is as follows. It can be seen that the final size of stack\_size is stack\_size += 1 \* MB. If the stack\_size we pass in is 0, the default size is 1 MB; if the stack\_size we pass in is -512KB, the stack\_size will become 512KB (1M - 512KB).&#x20;

```c++
static size_t FixStackSize(size_t stack_size) {

  if (stack_size == 0) {
    stack_size = Runtime::Current()->GetDefaultStackSize();
  }

  stack_size += 1 * MB;

  ……

  return stack_size;
}
```

Therefore, we only need to use the constructor with the stackSize parameter (as shown in the following code) to create a thread, and set stackSize to -512, then we can reduce the consumption of thread stack space by half.

```java
public Thread(ThreadGroup group, Runnable target, String name,
                  long stackSize) {
    this(group, target, name, stackSize, null, true);
}
```

However, since there are too many places in the application where threads are created, it seems difficult for us to modify all of them one by one. In fact, we don't need to manually modify them one by one. In the previous optimization, we have already converged most of the threads in the application to the common thread pool for creation. Therefore, at this time, we only need to modify the way threads are created in the common thread pool, and the thread pool just happens to support us creating threads ourselves. So we only need to pass in a custom ThreadFactory to meet the requirement. In our custom ThreadFactory, we create threads with a stack\_size of -512 KB, which can reduce the virtual memory occupied by the threads.&#x20;

When we change the size of all thread stacks in the application to 512 KB, it may cause stack overflow in some threads with heavy tasks. At this time, we can collect the threads that will experience stack overflow through event tracking and control not to modify the size of these threads via the allowlist.&#x20;

# 3.5 Default webview memory release

As previously mentioned, all allocated virtual memory in the application process is recorded in the maps file. Therefore, if we want to optimize virtual memory space, we need to analyze the application's maps file, find the virtual memory space that has been allocated but is not being used in the application, and then figure out a way to release this space, which can optimize a significant amount of virtual memory.&#x20;

By analyzing the maps file of the sample program in this book, it can be found that there is an allocation of virtual Memory Space for anno:libwebview reservation, as shown in Figure 3-32. The size of this block of memory is 1GB (720e862000 - 71ce862000 = 1GB).&#x20;

![Figure 3-32 Webview memory data recorded in the maps file](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter3_img_29.png)

This block of virtual memory is actually the space reserved for webview, with a size of 1GB on 64-bit machines, 130MB on 32-bit machines, and 190MB on other non-ARM machines. This space is actually allocated in the Zygote process. During the Zygote startup process, the so library webviewchromiun\_loader is loaded and this block of virtual space is allocated, and a block of virtual memory is allocated through the reserveAddressSpaceInZygote method in the [WebViewLibraryLoader.java](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/webkit/WebViewLibraryLoader.java) object, as shown in Figure 3-33. Subsequently, all application processes are forked from the Zygote process, so they will also retain this area.&#x20;

![Figure 3-33 Memory Allocation Function for Default Webview](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter3_img_30.png)

If our application does not need to use the system webview, or if we have already moved the usage scenarios of webview to a child process, we can completely free up this space in the main process and save this portion of virtual memory. For the example program, there are no webview pages, so this portion of virtual memory is actually not needed. Even in real projects, we often run webview in a child process, so for the main process, this webview memory is also not required.&#x20;

## 3.5.1 Find the address through the maps file

From our previous learning, we know that virtual memory is always allocated through the mmap function, and to release virtual memory, simply call munmap.

```c++
int munmap(void *start, size_t length)
```

So if we want to release this part of memory, we only need to call the following code at the Native layer.&#x20;

```c++
// 720e862000 is the starting address, 1073741824 is the size of 1GB converted to bytes
munmap(0x720e862000, 1073741824)
```

However, since the address of the space reserved by libwebview reservation is not fixed, we cannot hard code the address to 720e862000. Moreover, 1GB is the size for 64-bit machines, and on 64-bit machines we do not need to worry about insufficient virtual memory, so we only need to check if it is a 32-bit machine. If it is a 32-bit machine, then read the maps file, parse libwebview reservation, extract the start and end addresses, calculate the space size, and then call the munmap function. The process implementation is as follows.

1\) Parse maps and find the address of the libwebview reservation area. The implementation logic is the same as the previous method for finding the base address of so.

```c++
FILE *fp = fopen("/proc/self/maps", "r");
char line[1024];
uintptr_t webview_addr = 0;
size_t reservedSpaceSize = 0;
while (fgets(line, sizeof(line), fp)) {
    if (NULL != strstr(line, "[anon:libwebview reservation]")) {
        std::string targetLine = line;
        std::size_t pos = targetLine.find('-');
        if (pos != std::string::npos) {
            // Extract the starting address
            std::string addressStr = targetLine.substr(0, pos);
            // The stoull function converts a string to an unsigned long long integer
            webview_addr = std::stoull(addressStr, nullptr, 16);
            
            // Extract the ending address
            addressStr = targetLine.substr(pos+1, 2*pos);
            webview_addr_end = std::stoull(addressStr, nullptr, 16);
            // Calculate the size of the reserved space
            reservedSpaceSize = static_cast<size_t>(webview_addr_end - webview_addr);
            break;
        }
    }
}
fclose(fp);
```

2\) By releasing the virtual memory in the libwebview reservation area.

```c++
//Release virtual memory
munmap(webview_addr, reservedSpaceSize)
```

Upon seeing this, readers may think this solution is quite simple, but at this point, only half of it has been implemented. When we run this solution on a device with Android 9, we will find that the virtual memory space named libwebview reservation cannot be found in the maps file, thus rendering the solution ineffective. By examining the Android 9 source code, as shown in Figure 3-34, we can see that this virtual memory is allocated in the form of anonymous (MAP\_ANONYMOUS)

![Figure 3-34 WebView Memory Allocation Function in Android 9](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter3_img_31.png)

Actually, it was only starting from the Android 10 system that when applying for this virtual memory space through the mmap function, the prctl function would then be called to name this area "libwebview reservation", as shown in Figure 3-35 of the source code. Therefore, to implement this optimization on devices running Android versions below 10, our solution still needs further improvement.

![Figure 3-35 WebView Memory Allocation Function in Android 10](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter3_img_32.png)

## 3.5.2 Find the address through system variables

For devices running Android versions below 10, this area is anonymous in the maps file, and we cannot find this area based on the "libwebview reservation" field. Therefore, the solution of parsing maps becomes ineffective. How to find the address of this area is the difficulty of the entire solution.&#x20;

By examining the source code for memory allocation, such as the DoReserveAddressSpace method shown in Figure 3-36, we can find that after the source code allocates this memory via mmap, it then assigns the address `addr` and size `vsize` to the two static variables gReservedAddress and gReservedSize respectively. Therefore, we only need to figure out how to obtain the values of these two variables to solve this difficult problem. There are multiple solutions for obtaining the values of these two variables, and I will introduce one of them here.

First, by globally searching for gReservedAddress in the Android source code, as shown in Figure 3-36, it is found that it is only used by [loader.cpp](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/native/webview/loader/loader.cpp) in the three methods DoReserveAddressSpace, DoCreateRelroFile, and DoLoadWithRelroFile of this object.

![Figure 3-36 Code using gReservedAddress](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter3_img_33.png)

In the DoCreateRelroFile (source code shown in Figure 3-37) and DoLoadWithRelroFile (source code shown in Figure 3-38) methods, we can find that gReservedAddress and gReservedSize are encapsulated in the extinfo structure and passed as input parameters to the android\_dlopen\_ext function, which is a function in the libdl.so library.&#x20;

![Figure 3-37 Source code of DoCreateRelroFile ](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter3_img_34.png)

![Figure 3-38 Source Code of DoLoadWithRelroFile](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter3_img_35.png)

Previously, we have learned about PLT Hook technology, which is specifically designed to intercept the call functions of external libraries. For the webviewchromiun\_loader library, android\_dlopen\_ext happens to be an external function. Therefore, we only need to intercept the `android_dlopen_ext` function in the `webviewchromiun_loader` so through PLT Hook technology to obtain the extinfo data, and then obtain the values of gReservedAddress and gReservedSize. Here, I still uses bhook as a tool to demonstrate the specific code implementation:

```c++
// Use bhook to hook the android_dlopen_ext function in the webviewchromium_loader so library 
bytehook_stub_t bytehook_hook_single(
    "libwebviewchromium_loader.so",
    null,
    reinterpret_cast<void*>(android_dlopen_ext),
    reinterpret_cast<void*>(android_dlopen_ext_hook),
    bytehook_hooked_t hooked,
    void *hooked_arg);
```

By using the bytehook\_hook\_single method, we can complete the interception of the android\_dlopen\_ext function call in the libwebviewchromium\_loader.so library. Next, we retrieve sReservedSpaceStart and sReservedSpaceSize in the custom hook function and release them. The code implementation is as follows. At this point, we will encounter a problem commonly encountered in Native Hook, that is, we do not have the structure of the input parameter function in the intercepted function. In this case, we only need to redefine one according to the original data structure.&#x20;

```c++
/* extinfo is actually an android_dlextinfo structure,
   but since we cannot directly use this structure in our hook function, 
   we need to define one according to the original structure's data layout */
typedef struct {
    uint64_t flags;
    void* reserved_addr;
    size_t reserved_size;
    int relro_fd;
    int library_fd;
    off64_t library_fd_offset;
    struct android_namespace_t* library_namespace;
} android_dlextinfo;

// Get the values of gReservedAddress and gReservedSize in the hook function
static void* android_dlopen_ext_hook(const char* filepath, int flags, void* extinfo) {
    // Cast extinfo to the android_dlextinfo structure
    auto android_extinfo = reinterpret_cast<android_dlextinfo*>(extinfo);
    // Now we can directly obtain the values of reserved_addr and reserved_size
    sReservedSpaceStart = android_extinfo->reserved_addr;
    sReservedSpaceSize = android_extinfo->reserved_size;
    // Release the corresponding virtual memory space
    munmap(sReservedSpaceStart, sReservedSpaceSize);
    // Call the original function
    BYTEHOOK_CALL_PREV();
}
```

After completing the above series of operations, we will find that the solution still does not take effect. This is because if the system's webview is not used in the process, neither the DoCreateRelroFile nor the DoLoadWithRelroFile function will be executed. If neither of these two functions is executed, android\_dlopen\_ext will not be called at all, and naturally the desired data cannot be obtained. Therefore, we need to actively call either of these two functions through code in the application. However, we cannot call these two functions by normally starting the webview, as this goes against the original intention of this optimization, which is to perform this optimization only when the process does not need to use the webview.

How can these two functions be executed? After analyzing the source code, it will be found that it is actually very simple to actively call these two functions, because these two functions can actually be called through the two JNI methods nativeCreateRelroFile and LoadWithRelroFile, as shown in Figure 3-39.

![Figure 3-39 JNI call function for DoCreateRelroFile and DoLoadWithRelroFile](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter3_img_36.png)

At this point, readers may wonder if it is possible to achieve the goal simply by directly calling System.loadLibrary("webviewchromiun\_loader") in the Java layer and then invoking one of the JNI methods. The answer is no, because starting from Android 7.0, applications are no longer allowed to load system so libraries, and webviewchromiun\_loader is a system so library, so it cannot be loaded properly. Since the so library cannot be loaded properly, it is naturally impossible to directly call the two native functions LoadWithRelroFile or CreateRelroFile in the Java layer.

Although these two methods cannot be called at the Java layer, they can be called at the Native layer. By using the CallStaticIntMethod provided by JNI, the call to the static method at the Java layer can be completed. Here, I take the call to the nativeLoadWithRelroFile function as an example. Its input parameters include "lib, relro, clazzLoader", where lib is the name of the so library webviewchromiun\_loader, and relro is the path of this so library. We only need to obtain the corresponding Java object and methodId of this function through env->FindClass, and then we can execute this method. The code implementation process is as follows:

```c++
// Call the nativeLoadWithRelroFile function at the JNI layer
static bool LocateReservedSpaceByProbing(JNIEnv* env,
        jint sdk_ver, jobject class_loader) { 
    jclass loaderClazz = env->FindClass("android/webkit/WebViewLibraryLoader");
    const char* methodName = "nativeLoadWithRelroFile";
    jmethodID methodID = nullptr;
    jint probeMethodRet = 0;
    // Get the methodId of nativeLoadWithRelroFile via GetStaticMethodID
    methodID = (*env)->GetStaticMethodID(env, loaderClazz, methodName, 
            "(Ljava/lang/String;Ljava/lang/String;Ljava/lang/ClassLoader;)I");
    if (methodID != nullptr) {
        /* Execute the nativeLoadWithRelroFile method. Since we don't actually need to load webviewchromium_loader,
           we can pass fake paths for lib and relro */
        probeMethodRet = env->CallStaticIntMethod(loaderClazz, methodID,
                "/dev/test/","/dev/test/", class_loader);
        env->ExceptionClear();
    }
}
```

Through the above process, the application on 32-bit devices has an additional 130M of available virtual memory. In addition to the above-mentioned solution, we have other solutions that can achieve the same goal. For example, as an uninitialized global variable, gReservedAddress will be stored in the bss section. Therefore, after parsing the maps file, finding the address of the webviewchromiun\_loader so library and converting it to ELF format, the value of gReservedAddress can be obtained by searching and traversing the BSS section. In fact, the webviewchromiun\_loader so library has only 7 global variables, so there are only 7 entries in the BSS section, and we can easily find the value of gReservedAddress. Readers interested in this solution can also try it out themselves.&#x20;

| Source code appearing in this chapter:<br />MAT Official Website: <https://eclipse.dev/mat/downloads.php><br />malloc\_debug: <https://android.googlesource.com/platform/bionic/+/master/libc/malloc\_debug/><br />memory-leak-detector: <https://github.com/bytedance/memory-leak-detector><br />Thread.java: <br /><https://cs.android.com/android/platform/superproject/+/android-14.0.0\_r9:libcore/ojluni/src/main/java/java/lang/Thread.java><br />java\_lang\_Thread.cc: <br /><https://cs.android.com/android/platform/superproject/+/android-14.0.0\_r9:art/runtime/native/java\_lang\_Thread.cc><br />thread.cc: <https://cs.android.com/android/platform/superproject/+/android-14.0.0\_r9:art/runtime/thread.cc> |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
