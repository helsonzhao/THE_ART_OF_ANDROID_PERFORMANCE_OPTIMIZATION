As we have learned from the previous chapters, effective stability optimization cannot be achieved without the three key elements of monitoring, analysis, and anti-deterioration. Therefore, in this chapter, I will also focus on these three points to explain the practical aspects of stability optimization.&#x20;

Comprehensive stability monitoring is the cornerstone of effective stability optimization, and stability monitoring is often more difficult than analysis and governance. Therefore, this chapter will introduce monitoring solutions including "Native Crash Monitoring Solution", "ANR Monitoring Solution", "OOM Monitoring Solution", etc. With monitoring, when an anomaly occurs, we can conduct anomaly analysis based on the information reported by the monitoring. So this chapter will introduce conventional stability analysis ideas such as "Native Crash Analysis Approach" and "ANR Analysis Approach". Finally, we also need a mechanism to prevent degradation to ensure better results in stability optimization. Most degradation prevention mechanisms are mainly based on offline Monkey testing, which is a very conventional and effective solution, usually implemented by a dedicated testing team. Therefore, it will not be introduced in detail here. As a supplement, I will introduce the online degradation prevention solution of "Slow Function Monitoring".

It is also hoped that readers can form a complete stability optimization system through learning practical cases in the three directions of monitoring, analysis, and anti-deterioration, and with the support of this system, not only can we do a good job in stability optimization, but also write code with increasingly higher stability in daily project development.&#x20;

# 7.1 Native Crash Monitoring Solution

Most stability monitoring solutions cannot do without two steps: anomaly capture and acquisition of key anomaly logs, and the monitoring of Native Crash is no exception. So let's take a look at how these two steps are implemented in the monitoring solution for Native Crash.

## 7.1.1 Abnormal Signal Capture

In the previous chapter, we learned about the signal capture function sigaction. Through this function, we can implement monitoring of Native Crash. There are numerous signals at the Native layer, but it is not necessary to capture all of them. Just capturing the signals SIGSEGV, SIGABRT, SIGBUS, SIGILL, and SIGFPE can basically cover all exceptions at the Native layer. The code implementation is as follows:&#x20;

```c++
// Save the old signal handler
static struct sigaction old_sa[NSIG];

static void setup_signal_handler() {
    struct sigaction sa;
    // Pass in the exception handling callback function
    sa.sa_sigaction = signal_handler;
    sigemptyset(&sa.sa_mask);
    // Set signal handling options
    sa.sa_flags = SA_RESTART | SA_SIGINFO;
    // Set the signals to catch
    sigaction(SIGSEGV, &sa, &old_sa[SIGSEGV]);
    sigaction(SIGABRT, &sa, &old_sa[SIGABRT]);
    sigaction(SIGBUS, &sa, &old_sa[SIGBUS]);
    sigaction(SIGILL, &sa, &old_sa[SIGILL]);
    sigaction(SIGFPE, &sa, &old_sa[SIGFPE]);
}
```

The signal\_handler function in the above code is our custom callback function after exception capture, and the prototype of this callback function is the function pointer sa\_sigaction.&#x20;

```c++
void (*sa_sigaction)(int, siginfo_t *, void *);
```

It can be seen that this callback function has three callback data. The first callback data is of type int, representing the received semaphore. The second callback data is a pointer to a siginfo\_t structure, which contains additional information about the signal, such as the sender's process ID. The third callback data is a pointer to the structure of the current context environment. In the Android Native layer, the context structure is represented by ucontext\_t, through which data such as register status and stack information can be obtained.&#x20;

In the custom signal\_handler function, it is mainly used to capture key logs. However, considering the success rate of reporting, generally, data reporting is not immediately performed after capturing the logs here. Instead, the logs are recorded locally and reported after the program is restarted. After the logs are successfully captured, the original processing function will be executed to ensure the integrity of the call chain. The code implementation is as follows:&#x20;

```c++
static void signal_handler(int sig, siginfo_t *info, void *context) {
    // get and save crash log
    saveStack(context);
    // call old signal handler
    if (old_sa[sig].sa_handler) {
        old_sa[sig].sa_handler(sig);
    }
}
```

In the exception handling function, two main things are done: one is to call saveStack to obtain and store crash logs, and the other is to execute the original signal handling functions stored in the old\_sa array to avoid overwriting or ignoring the signal handling logic of other modules.&#x20;

As seen from the code implementation, the process of capturing the exception signal of a Native Crash is not complicated, but to make the solution more robust, a new stack space is usually created for the signal handling function to use. If a new stack space is not created, the handling function will execute on the original default stack. If the default stack space is insufficient at this time, such as when an OOM exception due to stack overflow occurs, the handling function will not be able to execute properly.&#x20;

A new stack space can be set using the sigaltstack function, thereby resolving this issue. The code implementation for creating a new stack space is as follows: in the code, a space of size SIGSTKSZ is allocated via mmap, where SIGSTKSZ is a macro definition with a size of 8KB, and then passed to the sigaltstack function. Before capturing signals using the sigaction function, after creating a new stack space with the following logic, the signal\_handler signal handling function will automatically execute within this new 8KB stack space.&#x20;

```c++
static void setup_alternate_stack() {
    stack_t stack;
    //apply for space of SIGSTKSZ(8KB)
    stack.ss_sp = mmap(
            NULL, SIGSTKSZ, PROT_READ | PROT_WRITE, MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
    stack.ss_size = SIGSTKSZ;
    stack.ss_flags = 0;
    if (stack.ss_sp != MAP_FAILED) {
        sigaltstack(&stack, NULL);
    }
}
```

## 7.1.2 Obtain Native Stack

In the above signal\_handler exception handling function, the saveStack method is called to obtain and store the Native stack. In Chapter 3, "Practical Memory Optimization", the solution of obtaining the Native stack through the unwind library has already been explained. Although this solution has better compatibility, the retrieval speed is relatively slow. When an exception occurs, we need to capture the stack as quickly as possible before the process is terminated. Therefore, I introduce a faster way to obtain the Native stack here: obtaining the stack via the FP (Frame Pointer) register.&#x20;

### 1. FP Register

In the previous chapter, we have learned the function of the SP (Stack Pointer) register, which is used to point to the stack top address of the current function and is used to store local variables, intermediate results, function parameters, return addresses, etc. The FP (Frame Pointer) register, on the contrary to the SP register, points to the starting position of the stack frame of the current function. The value of the FP register remains unchanged during the execution of this function, while the SP register changes as the stack expands and contracts during the function execution. The relationship between these two registers is shown in Figure 7-1. In Chapter 3, we learned that the stack space is allocated from high addresses to low addresses, so the FP register points to the high address of the stack, and the SP points to the low address of the stack.&#x20;

<img src="https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter7_img_1.png" alt="Figure 7-1 Schematic diagram of FP and SP" width="350" style="margin:auto;"/>



Once we understand SP and FP, we can also understand what a stack frame is. It is a contiguous Memory Space created at runtime during a function call to maintain the function's local variables, parameters, and other information related to the function's execution. Whenever a function is called, a new stack frame is created in the program's stack space to store the function's local variables, parameters, and other necessary information. When the function finishes executing, its stack frame is destroyed, thereby releasing the corresponding Memory Space.&#x20;

### 2. Stack Frame Backtrace

When a function call is made, the first few instructions of the current function push the values of the FP register, LR register, and SP register of the previous function onto the stack of the current function, and then move the SP pointer to allocate stack space for the current function. I will use the Native method that appears in the example program to explain here. By using the dumpobj tool to view the assembly code of the so library, it can be seen that the headers of most functions have three identical instructions, as shown in Figure 7.2.&#x20;

![Figure 7-2 Assembly code for the function](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter7_img_2.png)

The explanations for these three instructions are as follows:&#x20;

* Among them, the instruction "push {r7, lr}" performs a stack operation, pushing the values of the r7 register and the lr register onto the stack. In the ARM32 architecture, r7 is usually the frame pointer register, and lr is the link register. It should be noted that at this time, the r7 register and the lr register still hold the data from the previous function, which means this instruction will push the values of the FP pointer and the LR pointer from the previous function onto the stack of the current function.

* "mov r7, sp", this instruction assigns the address of the current stack top, which is the value of the SP register, to the r7 register. The purpose of doing this is to save the address of the stack top in r7 for use in subsequent code.&#x20;

* "sub sp, #16", this instruction subtracts 16 from the value of the SP register. Since the stack space expands from high addresses to low addresses, subtracting 16 bytes from the SP pointer is equivalent to allocating 16 bytes of stack space.&#x20;

Therefore, in a scenario where multiple function calls occur, such as function A calling function B, and function B calling function C, its stack frame model is shown in Figure 7-3.

<img src="https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter7_img_3.png" alt="Figure 7-3 FP and SP model diagrams" width="350" style="margin:auto;"/>



From the model diagram, it can be found that by using the value of the next address of the FP register, the data of the LR register can be obtained, which is the return address of the previous function. By using the value of the address two positions after the FP register, the FP address of the previous function can be obtained. Next, the LR and FP data stored in the stack of the previous function can be found in the same way, and thus the address of the function before the previous one and its stack bottom address are also known. This loop constitutes a stack traceback process.

Following this line of thought, the code implementation of stack traceback is as follows. In the code, we obtain the FP and PC registers through the ucontext\_t context object. FP points to the bottom of the current stack frame, and the stack grows from high addresses to low addresses. Therefore, the addresses of FP - 1 and FP - 2 represent the return address of the current function and the FP address of the previous function respectively. We use a while loop to continuously obtain these two values until all stack frames have been traversed.

```java
static void saveStack(void *secret) {
    // Get context information
    ucontext_t *uc = (ucontext_t *)secret;
    int i = 0;
    Dl_info  dl_info;
    const void **frame_pointer = (const void **)uc->uc_mcontext.arm_fp;
    const void *return_address = (const void *)uc->uc_mcontext.arm_pc;
    printf("\nStack trace:");
    while (return_address) {
        memset(&dl_info, 0, sizeof(Dl_info));
        if (!dladdr((void *)return_address, &dl_info))        break;
        const char *sname = dl_info.dli_sname;
        //Print function call information, including counter, return address, function name, offset, and file name
        printf("%02d: %p <%s + %u> (%s)", ++i, return_address, sname,
               ((uintptr_t)return_address - (uintptr_t)dl_info.dli_saddr),
               dl_info.dli_fname);
        //If the frame pointer is null, it means the end of the call chain has been reached, so break out of the loop
        if (!frame_pointer)        break;
        //Get the value of the previous function's LR (link register), which is the return address
        return_address = frame_pointer[-1];
        //Get the value of the previous function's FP (frame pointer)
        frame_pointer = (const void **)frame_pointer[-2];

    }
    printf("Stack trace end.");
}
```

The solution of obtaining the stack through stack traceback is relatively simple to implement, and the speed of obtaining the stack is also faster. However, to implement this solution, a dedicated general-purpose register needs to be used as the FP register. For devices on 32-bit platforms with only 13 general-purpose registers, this will undoubtedly have a certain impact on performance. Therefore, in Android, the FP register is turned off by default, which means there is no such register by default. But for devices on 64-bit platforms with 31 general-purpose registers, the impact on performance is limited. So we can turn on the FP register only on 64-bit platforms. The way to turn it on is to add the -fno-omit-frame-pointer flag when compiling the so library to tell the compiler not to turn off the FP register. The methods of adding it when using Android.mk or CMakeLists.txt as the build configuration are as follows respectively.&#x20;

```c++
# Android.mk
LOCAL_CFLAGS += -fno-omit-frame-pointer

# CMakeLists.txt
add_compile_options(-fno-omit-frame-pointer)
```

### 3. Register Data

A complete Native stack log should also include register data, so we can also print out the register data. The uc\_mcontext structure of ucontext\_t contains information about all registers, as shown in Figure 7-4. Through "uc->uc\_mcontext.arm\_r0" to "uc->uc\_mcontext.arm\_r10", we can obtain information about all general-purpose registers. Through uc->uc\_mcontext.arm\_ip, uc->uc\_mcontext.arm\_sp, uc->uc\_mcontext.arm\_lr, and uc->uc\_mcontext.arm\_pc, we can obtain information about all special-purpose registers. After obtaining the addresses of these registers, we add them to the previously captured Native stack information, and this is a complete Native stack log.&#x20;

![Figure 7-4 Data in uc\_mcontext structure](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter7_img_4.png)

## 7.1.3 Use Open Source Libraries

After understanding the principles and processes of the Native Crash capture solution, we can develop our own monitoring solution. When implementing it in practice, we need to be compatible with different platforms, such as ARM32, AMR64, X86, etc. Under different command platforms, the details of the solution vary. Therefore, designing a complete and stable monitoring solution on our own is actually a very cumbersome and complex task. To prevent readers from expending a great deal of effort on reinventing the wheel, I introduces a mature and stable open-source library for Native Crash capture : [ Breakpad ](https://github.com/google/breakpad), which is a cross-platform crash monitoring and analysis framework launched by Google. There are mainly three steps to using this framework:&#x20;

1\) Source code compilation: Since Breakpad is cross-platform, when using it on Android, it must first be compiled into a so library on the Android platform before it can be used.

2\) Crash Capture: After being compiled into a so library in the previous step, it can be directly used in the Native layer of Android. By calling the methods provided by Breakpad, crash capture can be completed.

3\) Log Parsing: Considering factors such as security, the Native Crash logs captured by Breakpad are binary files. Therefore, we also need to use the parser provided by Breakpad to convert the binary files back into a human-readable text format.

Let's take a detailed look at each step below.

### 1. Source code compilation

First, compile on the Android platform, and clone the Breakpad source code to the local file according to the source code address on GitHub

```shell
git clone https://github.com/google/breakpad.git
```

Since breakpad depends on the third-party lss library, after the clone of breakpad's source code is completed, enter the root directory and continue to pull the third-party lss library into the src/third\_party directory of breakpad. The git commands are as follows:

```shell
git clone https://chromium.googlesource.com/linux-syscall-support src/third_party/lss
```

After the source code download is completed, copy the entire src directory files into the cpp directory of the Android project, as shown in Figure 7-5. You can create a separate breakpad directory under the cpp directory to place the source code.&#x20;

![Figure 7-4 included breakpad file](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter7_img_5.png)

The android/google\_breakpad/Android.mk file in the Breakpad project already tells us which source files to include. Therefore, in the Android.mk or CMakeLists.txt configuration build file of our own project, we can include the source code in the way provided by the project. I use CMakeLists to build Native code in the example program, so the configuration is as follows:&#x20;

```c++
#breakpad
add_library(breakpad SHARED
        breakpad/src/client/linux/crash_generation/crash_generation_client.cc
        breakpad/src/client/linux/handler/exception_handler.cc
        breakpad/src/client/linux/handler/minidump_descriptor.cc
        breakpad/src/client/linux/log/log.cc
        breakpad/src/client/linux/dump_writer_common/thread_info.cc
        breakpad/src/client/linux/dump_writer_common/seccomp_unwinder.cc
        breakpad/src/client/linux/dump_writer_common/ucontext_reader.cc
        breakpad/src/client/linux/microdump_writer/microdump_writer.cc
        breakpad/src/client/linux/minidump_writer/linux_dumper.cc
        breakpad/src/client/linux/minidump_writer/linux_ptrace_dumper.cc
        breakpad/src/client/linux/minidump_writer/minidump_writer.cc
        breakpad/src/client/minidump_file_writer.cc
        breakpad/src/common/android/breakpad_getcontext.S
        breakpad/src/common/convert_UTF.c
        breakpad/src/common/md5.cc
        breakpad/src/common/string_conversion.cc
        breakpad/src/common/linux/elfutils.cc
        breakpad/src/common/linux/file_id.cc
        breakpad/src/common/linux/guid_creator.cc
        breakpad/src/common/linux/linux_libc_support.cc
        breakpad/src/common/linux/memory_mapped_file.cc
        breakpad/src/common/linux/safe_readlink.cc)
set_target_properties(breakpad PROPERTIES ANDROID_ARM_MODE arm)
set_property(SOURCE breakpad/src/common/android/breakpad_getcontext.S PROPERTY LANGUAGE C)
target_include_directories(breakpad PUBLIC
            breakpad/src/common/android/include
            breakpad/src)    
```

In the CMakeLists configuration file, you need to use the add\_library function to import the source code that needs to be packaged in Breakpad, generate the Breakpad library, and then link it to our own so library. In the example program, it will be linked to the optimize.so library. The configuration is as follows:&#x20;

```c
target_link_libraries(
        optimize
        ${log-lib}
        breakpad)
```

### 2. Crash Capture

After configuration, Breakpad can be directly used in the project's Native code to capture Native crashes. The usage is as shown in the following code. The path for saving crash logs is set via the descriptor, which is generally saved under our own program path. Then, simply set the descriptor and callback function in the ExceptionHandler. The code is as follows. After capturing a Native crash via Breakpad in the code, a crash is manually generated to verify the capture effect.&#x20;

```c++
#include <jni.h>
#include <android/log.h>
#include "breakpad/src/client/linux/handler/exception_handler.h"
#include "breakpad/src/client/linux/handler/minidump_descriptor.h"

bool dumpCallback(const google_breakpad::MinidumpDescriptor &descriptor,
                  void *context,
                  bool succeeded) {

    __android_log_print(ANDROID_LOG_DEBUG, "breakpad",
                        "Wrote breakpad minidump at %s succeeded=%d\n",
                         descriptor.path(),
                        succeeded);
    return false;
}

extern "C"
JNIEXPORT void JNICALL
Java_com_example_performance_1optimize_stability_StabilityExampleActivity_captureNativeCrash(JNIEnv *env, jobject thiz) {
    //set log save path
    google_breakpad::MinidumpDescriptor 
            descriptor("/data/data/com.example.performance_optimize/");
    google_breakpad::ExceptionHandler 
            eh(descriptor, NULL, dumpCallback, NULL, true, -1);
    //mock Crash 
    int* ptr = nullptr;
    *ptr = 10;
}
```

### 3. Log Parsing

After the above Native method is called, it can be seen that a crash log is generated in the specified directory, as shown in Figure 7-6. However, at this time, the log is a binary file, and valid information cannot be directly viewed. Therefore, the log needs to be parsed and restored using the tools provided by Breakpad.&#x20;

![Figure 7-6 Native stack log generated by breakpad](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter7_img_6.png)

The parsing method is as follows:&#x20;

1\) Navigate to the root directory of Breakpad downloaded earlier, and execute "make clean" in the command window to initialize data configuration, as shown in Figure 7-7.

![Figure 7-7 Executing the clean command](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter7_img_7.png)

2\) Execute the commands./configure & make to run the configuration information and compile, as shown in Figure 7-8.

![Figure 7-8 Executing the compile command](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter7_img_8.png)

3\) Navigate to the src/processor directory of Breakpad and execute the command "./minidump\_stackwalk source.dmp > output.txt", which can parse the binary crash log into text format, as shown in Figure 7-9.

![Figure 7-9 Parsing crash files](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter7_img_9.png)

The parsed Breakpad crash log is shown in Figure 7-10. As can be seen, the stacks in the log when converted to text format are all hexadecimal addresses. We can use the addr2line tool mentioned in the previous chapter to restore the binary addresses to detailed stack information, or we can use the dump\_syms tool in src/tools under the Breakpad directory to export the symbols of the so library with symbols, and then use the minidump\_stackwalk tool for parsing, so that the addresses in the parsed log can be restored.&#x20;

![Figure 7-10 Parsed crash log](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter7_img_10.png)

# 7.2 ANR Monitoring Solution

When an ANR occurs in the program, the system will pop up an ANR dialog box and write the ANR log information to a file in the /data/anr/ directory. However, we do not have a direct interface to detect the occurrence of an ANR, nor do we have the permission to read the files in the /data/anr/ directory. But in order to improve the stability of the program, it is essential to effectively monitor ANRs in the production environment. Therefore, it is necessary to implement a set of ANR monitoring solutions in the program.

## 7.2.1 Signal Acquisition Detection Scheme

When an ANR occurs, the system sends a SIGQUIT signal to the corresponding process. Since it is a signal, just like monitoring Native Crash, we can capture the ANR by capturing this SIGQUIT signal. Based on the knowledge from before, we can quickly write the code. Similarly, we use the sigaction function, add the capture of the SIGQUIT signal, and in the custom signal handling function, determine whether it is SIGQUIT and perform corresponding processing. The code is as follows:

```c++
sigaction(SIGQUIT, &sa, &old_sa[SIGILL]);
```

However, after actually running this code, we will find that our custom exception handling function does not catch the SIGQUIT signal, because the system blocks the SIGQUIT signal and does not allow sigaction to receive the SIGQUIT signal.

When the program starts, it will start a SignalCatcher thread, which will block and listen for the SIGQUIT signal through the sigwait function. The source code implementation is shown in Figure 7-11, and it is located in the signal\_catcher.cc file. The sigwait function is also a function for listening to signals. Compared with the sigaction function, it uses a synchronous receiving method, which means only one place is allowed to listen for the specified signal, while the sigaction function is asynchronous and can listen for and process the execution signal in multiple places.&#x20;

![Figure 7-11 SignalCatcher thread listens to SIGQUIT signal source code](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter7_img_11.png)

Although the system blocks the asynchronous way of receiving the SIGQUIT signal via sigaction, it cannot block the synchronous monitoring of the SIGQUIT signal via sigwait. This also ensures that after the SignalCatcher thread receives the SIGQUIT signal, it can normally obtain information about each thread in the process and output it to the /data/anr/traces.txt file.&#x20;

Although the SIGQUIT signal is blocked by the system, we can use  pthread\_sigmask  function  to remove the SIGQUIT signal from the signal mask set of the current thread , and the implementation is as follows&#x20;

```c++
// Define a signal set
sigset_t new_set, old_set;
// Initialize and clear the signal set
sigemptyset(&new_set);
// Add the SIGQUIT signal to the signal set
sigaddset(&new_set, SIGQUIT);
// Set the current thread's signal mask to the complement of the signal set, i.e., unblock the SIGQUIT signal
pthread_sigmask(SIG_UNBLOCK, &new_set, &old_set);
```

When the masking of  the SIGQUIT signal  is lifted, the above capture of the SIGQUIT signal will take effect. We can implement the handling of ANR in the signal handling function. In addition to capturing the ANR Trace data, we also need to ensure that the original SignalCatcher thread can respond to the SIGQUIT signal. Since the SignalCatcher thread does not respond to the SIGQUIT signal through sigaction, directly executing old\_sa will not take effect. At this time, we can use the tgkill signal sending function to send a SIGQUIT signal to the SignalCatcher thread. The tgkill function needs to know the thread ID when sending a signal to a specified thread, so we also need to traverse all the thread data recorded in the /proc/{pid}/task directory under this process to obtain the thread ID corresponding to the thread named "SignalCatcher". The code implementation is as follows.&#x20;

```c++
// Signal handler function
static void signal_handler(int sig, siginfo_t *info, void *secret) {
    if (sig == SIGQUIT) {  
        // Capture ANR information      
        dealAnr();
        // Send SIGQUIT signal to the SignalCatcher thread
        tgkill(getpid(), getSignalCatcherThreadId(), SIGQUIT);
    }
}

// Traverse the /proc/[pid]/task directory to find the tid of the SignalCatcher thread
int getSignalCatcherThreadId() {
    // Construct the path to the /proc/[pid] directory
    string proc_path = "/proc/" + to_string(getpid());
    DIR* dir = opendir(proc_path.c_str());
    struct dirent* entry;
    while ((entry = readdir(dir)) != NULL) {
        std::string name = entry->d_name;
        // Check if the name is a number
        if (std::all_of(name.begin(), name.end(), ::isdigit)) {
            // Construct the path to the /proc/[pid]/task/[tid]/status file
            std::string status_path = proc_path + "/task/" + name + "/status";
            std::ifstream status_file(status_path);
            // Read the file content
            std::string line;
            while (std::getline(status_file, line)) {
                // Look for the line: Name: SignalCatcher
                if (line == "Name:\tSignalCatcher") {
                    // Found it, return the tid
                    int tid = std::stoi(name);
                    closedir(dir);
                    return tid;
                }
            }
        }
    }
    closedir(dir);
    return -1;
}
```

In the above process, we have implemented monitoring whether an ANR has occurred by listening for the SIGQUIT signal. However, in actual situations, when a process receives the SIGQUIT signal, it only indicates that the current process may have experienced an ANR, and it cannot be 100% certain that an ANR has occurred. For example, when an ANR occurs in another application, processes with relatively high CPU usage will also receive the SIGQUIT signal, and other processes or threads can also manually send the SIGQUIT signal to the current process through functions such as kill or tgkill. Therefore, receiving the SIGQUIT signal is only a necessary but not sufficient condition for a process to experience an ANR. So in actual scenarios, additional supplementary measures are also used for secondary judgment to increase the success rate of ANR determination. Therefore, I will continue to introduce supplementary measures.&#x20;

## 7.2.2 AMS Interface Detection Solution

After the SIGQUIT signal is captured in the logic of front signal capture, the Java layer method can be called through JNI to notify the Java layer to perform a secondary confirmation of ANR. The code implementation is as follows, where the onANRDumpTrace function of the Java layer is executed through the CallStaticVoidMethod function.&#x20;

```c++
void anrDumpTraceCallback(JNIEnv *env) {
    jclass myUtilsClass = env->FindClass("com/example/app/MyUtils");
    jmethodID onANRDumpTraceMethod = 
            env->GetStaticMethodID(myUtilsClass, "onANRDumpTrace", "()V");
    // call method
    env->CallStaticVoidMethod(myUtilsClass, onANRDumpTraceMethod);
}
```

Before the ActivityManagerService notifies the process to launch the ANR pop-up window, it sets a flag of "NOT\_RESPONDING" for the process that has experienced an ANR, indicating that the process has encountered an exception, and this flag can be obtained through the getProcessesInErrorState method of ActivityManager. Therefore, in the onANRDumpTrace function, we can call this method to obtain the error state of the process. If the process is in the NOT\_RESPONDING state, it indicates that the process has experienced an ANR, and at this time, the secondary confirmation of the ANR is completed. The code implementation is as follows.

```java
void onANRDumpTrace() {
    ActivityManager am = (ActivityManager) getSystemService(Context.ACTIVITY_SERVICE);
    if (am != null) {
        List<ActivityManager.ProcessErrorStateInfo> errorList 
            = am.getProcessesInErrorState();
        
        if (errorList != null && !errorList.isEmpty()) {
            for (ActivityManager.ProcessErrorStateInfo info : errorList) {
                if (info.condition 
                        == ActivityManager.ProcessErrorStateInfo.NOT_RESPONDING) {
                    Log.e(TAG, "ANR detected in process: " + info.processName);
                    //confirmation of ANR, used for ANR recording or reporting.
                    ...
                }
            }
        } 
    }
}
```

After the secondary ANR confirmation is completed, we can further confirm whether to delete the locally mis-captured ANR logs or retain them for subsequent upload to the server. In addition to the solution introduced above, there are quite a few ANR detection solutions, such as detecting ANR pop-ups or detecting function delays in the main thread through various methods. However, the solution introduced in this chapter, which captures signals and then performs secondary confirmation, is often the optimal solution in terms of success rate and performance. If the signal capture solution is not used, ANR can only be detected through polling, and these methods naturally incur relatively high performance overhead.&#x20;

## 7.2.3 Capture Trace File

When we capture the occurrence of an ANR through the previous solution, the most important thing at this time is to grab the ANR trace log. We know that the /data/anr/traces.txt file is a powerful tool for ANR analysis. The content of this file is very comprehensive, including various states, locks, and stack information of all threads, which is very helpful for troubleshooting ANR issues. However, applications do not have access rights to this file, so it is impossible for online programs to obtain ANR log information by directly accessing this file. Although we cannot directly obtain this file, we can indirectly obtain the data content of this file.

After receiving the  SIGQUIT signal , the SignalCatcher thread will obtain the Trace information of each thread and write the Trace data to the /data/anr/traces.txt file through the system's write function. If we can hook this write method, we can obtain the ANR Trace data written to the traces.txt file. Here, we use the PLT Hook technique learned earlier, that is, after catching the SIGQUIT signal, we intercept the write function in the libc.so library through the PLT Hook technique, so that we can obtain the content of the Trace data written by the write function. I implement the interception of the write function through bytehook here, which is relatively simple to implement, and the code is as follows.&#x20;

```c++
void dealAnr(){
    bytehook_hook_all(
            "libc.so",
            "write",
            (void *)my_write,
            nullptr,
            nullptr);
}
```

In the custom write interception function above, the my\_write function, we can write ANR data into the program's own directory. We can use C++'s fstream to perform the file data writing operation, and the code implementation is as follows:&#x20;

```c++
#include <fstream>
ssize_t my_write(int fd, const void* const buf, size_t count) {
    BYTEHOOK_STACK_SCOPE();
    if (buf != nullptr) {
        char *content = (char *) buf;
        std::ofstream file("/data/data/com.example.performance_optimize/example_anr.txt"
                , std::ios::app);
        if (file.is_open()) {
            file << content;
            file.close();
        }
    }
    return BYTEHOOK_CALL_PREV(my_write,fd, buf,count);
}
```

Through the above solution, as shown in Figure 7-12, it can be seen that the ANR log has been successfully captured, and the data is exactly the same as that in the /data/anr/traces.txt file. When the program starts next time, this ANR log can be uploaded to the server for subsequent ANR analysis and repair.

![Figure 7-12 ANR logs captured by Hook technology](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter7_img_12.png)

## 7.2.4 Use Open Source Frameworks

ANR monitoring is one of the most important tasks in stability management, so there are many open-source libraries that implement this task. For example, Tencent's Matrix, iQIYI's xCrash, and ANR-WatchDog, etc. The implementation principles of these libraries are similar to what is described in this book, but they have been verified by a large number of users, so they have sufficient stability and performance guarantees. In addition to ANR monitoring, many open-source libraries also integrate monitoring of Java Crash, Native Crash, etc., to form a complete set of monitoring tools. The main differences among these three open-source frameworks are shown in the table. Readers can delve into the advantages and disadvantages of these open-source libraries, such as stability, user base, update frequency, etc., and choose a suitable open-source library to use based on the business scenario.

| Features                 | Tencent Matrix                                                                  | iQiyi xCrash                                         | ANR-WatchDog                                                 |
| ------------------------ | ------------------------------------------------------------------------------- | ---------------------------------------------------- | ------------------------------------------------------------ |
| Function Scope           | laggy, memory, resource abuse, ANR, etc.                                        | Focus on Crash and ANR capture                       | Only ANR monitoring                                          |
| Configuration Complexity | High                                                                            | Medium                                               | Low                                                          |
| Performance Impact       | has a significant impact                                                        | Mild Impact                                          | Less Impact                                                  |
| Usage Scenarios          | Medium and large-scale projects requiring comprehensive performance monitoring  | Projects that need to monitor Crash and ANR captures | Small projects that need to quickly integrate ANR monitoring |

# 7.3 OOM Monitoring Solution

In the previous chapter, we learned that as a special type of Java Crash, OOM shares the same exception capture approach as Java Crash, both of which are handled through the global ExceptionHandler. The difference lies in that OOM requires obtaining memory snapshot data before effectively troubleshooting exceptions and resolving issues. Both the monitoring of Java Crash and the capture of memory snapshots have direct interfaces available for use, and these processes are not difficult points.&#x20;

However, I continues to explain the OOM monitoring solution here because the original Hprof memory snapshot file is usually quite large. When OOM occurs, the memory size basically exceeds 512MB, and the captured Hprof memory snapshot file at this time will also basically have the same size. For such a large file, the success rate of uploading it to the server is relatively low. Therefore, we usually perform a certain amount of pruning on Hprof. The pruned Hprof file only retains the data necessary for analyzing OOM, which not only reduces the size but also improves data security.&#x20;

## 7.3.1 Hprof File Structure

To trim an Hprof file, one must first have a certain understanding of the data composition of the Hprof file. The structure of an Hprof file consists of a Header and multiple Record data items, and its simplified structure is shown in Figure 7-13.

![Figure 7-13 Simplified structure of Hprof](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter7_img_13.png)



Next, let's take a detailed look at the data composition of the two data segments, Header and Record.

1\) Header: Records the meta information of the Hprof file, such as version number, identifier size, timestamp, etc. The data format of the Header is shown in Figure 7-14.

![Figure 7-14 Data structure of Header](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter7_img_14.png)



The data structure defined according to the Header format is as follows.&#x20;

```java
class HprofHeader {
    private byte[] format; // Occupies 18 bytes, used to identify the file format
    private int version; // Indicates the version information of the hprof file
    private int highTime; // The creation timestamp of the hprof file
    private int lowTime; // The creation timestamp of the hprof file; the high and low fields together represent a 64-bit timestamp
}
```

2\) Record: The specific content of the Hprof file, which consists of multiple Records. Each Record is composed of four parts: type, timestamp, data length, and data content, as shown in Figure 7-15.

![Figure 7-15 Structure of Record Data Segment](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter7_img_15.png)



The data structure defined according to the Record data segment structure is as follows.&#x20;

```java
class HprofRecord {
    private byte tag;   // Identifies the type of record, 1 byte
    private int time;  // Indicates the timestamp of the record
    private int length; // Indicates the length of the data section of the record
    private byte[] data; // Indicates the data section of the record, variable length
}

```

The data of the first ByteDance of each Record entry represents the type (TAG) of the entry. Through the [hprof.cc](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r9:art/runtime/hprof/hprof.cc) file in the Android source code, the value definition of each TAG can be seen, as shown in Figure 7-16&#x20;

![Figure 7-16 Definition of the value of Tag in the hprof.cc file](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter7_img_16.png)

Some of the main types are explained as shown in the table below:&#x20;

| TAG Value | Type          | Explanation                                                                                                                                                                                             |
| --------- | ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 0x01      | STRING        | records information about string objects                                                                                                                                                                |
| 0x02      | LOAD\_CLASS   | records information about the loaded classes, including the object identifier of the class, the string ID of the class name, etc.                                                                       |
| 0x04      | STACK\_FRAME  | records information about the stack frame, including ID, method name, class name, source file name, line number, etc.                                                                                   |
| 0x05      | STACK\_TRACE  | records stack trace information, including sequence number, thread ID, number of stack frames, stack frames, etc.                                                                                       |
| 0x07      | HEAP\_SUMMARY | records the overall situation of heap memory, including used memory, total memory, number of objects, etc.                                                                                              |
| 0x0C      | HEAP\_DUMP    | A file containing a complete memory snapshot of the application at runtime, recording the state and content of all objects in the application.                                                          |
|  0x1C     | DUMP\_SEGMENT | records the state, reference relationships, and other relevant information of all objects. By analyzing heap memory snapshots, we can identify memory leaks, optimize memory usage, and locate issues.  |
| 0x0D      | CPU\_SAMPLES  | Record CPU usage, including thread ID, number of samples, stack frames, etc.                                                                                                                            |

When analyzing the cause of OOM through Hprof, we only need to know the object's dependencies and data size to complete the analysis. Therefore, we do not need to know the specific content of the data, and all data that does not affect OOM analysis can be trimmed.&#x20;

For Hprof files, most of the data content is in the two data entries HEAP\_DUMP and DUMP\_SEGMENT. Therefore, we need to further understand the detailed data content under this data entry. Through the first ByteDance of the data content in this data entry, we can further identify the subtype of the data content. The hprof.cc file has the value definitions of the sub-data types of the HEAP\_DUMP and DUMP\_SEGMENT entries, as shown in Figure 7-17.&#x20;

![Figure 7-17 Value definition of sub-data entry Tag](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter7_img_17.png)

Here are some major sub-data types, as shown in the table.

| TAG Value | Data Type              | Description                                                                       |
| --------- | ---------------------- | --------------------------------------------------------------------------------- |
| 0x01      | ROOT\_JNI\_GLOBAL      | This record type represents a global reference created through JNI code           |
| 0x02      | ROOT\_JNI\_LOCAL       | This record type represents a local reference created through JNI code.           |
| 0x03      | ROOT\_JAVA\_FRAME      | This record type represents the frame information in the Java method call stack   |
| 0x04      | ROOT\_NATIVE\_STACK    | This record type represents the frame information in the local method call stack  |
| 0x20      | CLASS\_DUMP            | The record contains information about Java classes                                |
| 0x21      | INSTANCE\_DUMP         | records the detailed information of the object instance                           |
| 0x22      | OBJECT\_ARRAY\_DUMP    | Record the data of the object array                                               |
| 0x23      | PRIMITIVE\_ARRAY\_DUMP | records the data of the basic type array                                          |

When analyzing OOM, we only need to focus on the size of objects and their reference relationships. Therefore, data containing key information about reference chains, such as ROOT\_JNI\_GLOBAL, ROOT\_JNI\_LOCAL, and ROOT\_JAVA\_FRAME, should all be retained. However, for data segments that record metadata, such as INSTANCE\_DUMP and PRIMITIVE\_ARRAY\_DUMP, we can delete them all.&#x20;

## 7.3.2 Hprof Tailoring Solution

Once you understand the data structure of the Hprof file, you can start trimming it. The technical principle of trimming is not complicated, mainly involving routine operations on file streams. We need to read the ByteFlow of the file, then restore the ByteFlow of the file back to the corresponding data based on the file's data structure, then modify the data, and finally write it back to a new file through the ByteFlow. Here, I only take trimming the data content of PRIMITIVE\_ARRAY\_DUMP as an example to explain the code.

First, use the Input Stream object DataInputStream provided by Java to read the ByteFlow of the Hprof file. According to the data structure of HprofHeader defined earlier, read out the corresponding data in sequence. The read operation of the DataInputStream object will move the index of the ByteFlow to the position after the read data, so there is no need for us to manually set the index position of the file stream reading. While reading the data stream of the original file, we can simultaneously write the modified data stream into a new file through the Output Stream object DataOutputStream provided by Java. For data that does not need to be cropped or modified, simply write it as the original data. The code implementation is as follows.&#x20;

```java
DataInputStream dataStream = new DataInputStream(new FileInputStream("input.hprof"));
DataOutputStream dataOutStream = new DataOutputStream(new FileOutputStream("out.hprof")

//Read Hprof header data
HprofHeader hprofHeader = new HprofHeader();
//Read data of specified byte length using readFully
hprofHeader.format= dataStream.readFully(new byte[18]);
hprofHeader.version = dataStream.readInt();
hprofHeader.highTime = dataStream.readInt();
hprofHeader.lowTime = dataStream.readInt();
//Write the Hprof header data back to the new file
dataOutStream.write(hprofHeader.format);
dataOutStream.writeInt(hprofHeader.version);
dataOutStream.writeInt(hprofHeader.highTime);
dataOutStream.writeInt(hprofHeader.lowTime);
```

After reading the data in the Header, the byte stream index of the file then moves to the position of the Record data entry. Next, we find the data segment of HEAP\_DUMP through the value of the first-level Tag, and then find the data segment of PRIMITIVE\_ARRAY\_DUMP under the HEAP\_DUMP segment through the value of the second-level Tag. For the data in this segment, when writing the data back, we skip writing this data, thus completing the pruning of PRIMITIVE\_ARRAY\_DUMP. The code implementation is as follows:&#x20;

```java
int tag;
while ((tag = dataStream.read()) != -1) {
    HprofRecord hprofRecord = new HprofRecord();
    hprofRecord.tag = tag;
    hprofRecord.time = dataStream.readInt();
    hprofRecord.length = dataStream.readInt();
    // Read data of the corresponding length according to the previously recorded data length
    hprofRecord.data = dataStream.readFully(new byte[hprofRecord.length]);
    ByteBuffer buffer = ByteBuffer.wrap(hprofRecord.data);
    // Read the first 4 bytes of the data segment to determine the type of the sub-data segment.
    int subTag = buffer.getInt();
    // Determine whether the target data segment is found based on the primary tag and secondary tag
    if (hprofRecord.tag == RECORD_TAG_HEAP_DUMP && subTag == PRIMITIVE_ARRAY_DUMP) {
       // If it is PRIMITIVE_ARRAY_DUMP data, do not write this data segment to the file
       continue;
    }; 
    // For data that is not trimmed, write the original data back to the file
    dataOutStream.writeByte(hprofRecord.tag);
    dataOutStream.writeInt(hprofRecord.time);
    dataOutStream.writeInt(hprofRecord.length);
    dataOutStream.write(hprofRecord.data);
}

```

After running the above code, we have completed the Hprof trimming process. As long as one is familiar with the Hprof file format and understands the basic operations of file streams, they can easily understand and implement this optimization solution. I have only trimmed the data content of PRIMITIVE\_ARRAY\_DUMP here, and readers can follow the above process to further trim the unused data in the Hprof file.&#x20;

There are two mainstream directions for cropping Hprof files. The first is to crop and upload the original Hprof file after capturing a complete Hprof file via Debug.dumpHprofData, which is the direction introduced above. The second is to intercept the system's write function through Native Hook, and then crop the data while the system is writing memory snapshot data. Each of these two methods has its own advantages and disadvantages. The former solution is simple and does not affect the original Hprof file, while the latter solution is more efficient, but because it uses Native Hook technology, compatibility and stability issues are inevitable.

## 7.3.3 Use open source frameworks

In the above process, only one data segment, PRIMITIVE\_ARRAY\_DUMP, was trimmed. In real-world scenarios, we will trim more data to achieve better results. To ensure better stability and trimming effectiveness when used online, I still recommend using open-source third-party frameworks. There are also many open-source frameworks in this area, but their principles are similar to those described earlier. Here, the [Tailor](https://github.com/bytedance/tailor) framework open-sourced by ByteDance is recommended. In addition to trimming the data within Hprof, this tool further compresses the data after trimming. Since the data has been compressed, the Hprof file needs to be decompressed using the decompression script provided by Tailor before it can be used. Thanks to the dual effects of trimming and compression, this framework can reduce a Hprof file of several hundred megabytes to within a few dozen megabytes or even a dozen megabytes, greatly improving the success rate of capturing and uploading memory snapshots. The detailed usage instructions are provided in the official Github documentation, so I will not elaborate further.&#x20;

# 7.4 Native Crash Analysis Approach

Android developers come into contact with Native development much less frequently than Java layer development. Therefore, when faced with Native crashes, they often find it more difficult to locate and fix them. However, the management of Native crashes is an unavoidable aspect of stability optimization. We need to systematically master the relevant basic knowledge and methodology of Native crash management to handle Native crash management more effectively.

Here, I use a simple Native Crash to guide readers from shallow to deep in understanding the basic knowledge and analysis ideas when dealing with Native Crashes. As shown in Figure 7-18, the sample program assigns a value to a null pointer in the mockCrash function at the Native layer. When we call this Native function at the Java layer, a Native Crash will occur.&#x20;

![Figure 7-18 Null pointer exception](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter7_img_18.png)

## 7.4.1 Preliminary Analysis

After a Native Crash occurs, we can capture detailed logs through the Native Crash monitoring solution learned earlier, or find the Native Crash log captured by the system in the /data/tomstoms/ directory on the phone, with some data of this log shown in Figure 7-19.&#x20;

![Figure 7-19 Native Crash Log](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter7_img_19.png)

This crash log is very long. First, we need to conduct preliminary analysis and location through some key information. The main points of the preliminary analysis are as follows:&#x20;

* Signal: Based on the signal shown in line 10 of the log, we can make a preliminary attribution of this crash. The signal in this log is 11 (SIGSEGV), with an error code of 1, from which we can know that this error is caused by the code attempting to access a memory region not mapped to its address space, i.e., a null pointer.&#x20;

* Thread: As shown in line 8 of the log, we can know that the tid of the thread where the Crash occurred is 16760. In many cases, the Crash does not occur in the main thread. Therefore, we need to know the id of the thread where the Crash occurred, and then go to the stack log of the corresponding thread for further analysis. If this id is consistent with the main process pid, it indicates that the Crash occurred in the main thread.

* Crash stack: As shown starting from line 17 in the log, this is the stack information of the main thread when the crash occurred. Since the Crash shown in the case happened in the main thread, we only need to analyze the stack here. If the Crash was caused by a non-main thread, we need to locate the stack information of the corresponding thread before further analysis. The crash log of tomstoms will record the stack information of all threads. Based on the crash stack, we can initially analyze the so library where the exception occurred. If the so library has not had its symbol table removed, we can also directly know the function symbol name where the crash occurred.

## 7.4.2 Stack Analysis

After a preliminary analysis of Native Crash, detailed analysis and location can then be carried out based on the stack information.&#x20;

In the stack format of Native Crash, each line starts with #, followed by a number indicating the stack depth, starting from 0. Then comes the value of the PC register, representing the address of the current instruction. Next is the path and name of the module, indicating the library or executable file where the current instruction is located. Finally, there is the name and offset of the function, indicating the function where the current instruction is located and the offset relative to the starting address of the function. If there is a BuildId, it represents the Unique Device Identifier of the module, used for symbolization and debugging.&#x20;

Since the so library of the example program retains symbols, it is possible to directly see from the stack that the function where the crash occurred is the mockCrash function, and the crash occurred at the instruction at the 14th ByteDance of this function, as shown in Figure 7-20.

![Figure 7-20 The function of collapse](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter7_img_20.png)

For online programs, considering security and package size, we usually remove the symbol table. In this case, we can use the addr2line tool in conjunction with the so library with the symbol table to perform stack restoration. We also learned this part of knowledge in Chapter 3 "Practical Memory Optimization". According to the stack information, we can see that the offset address where the exception occurred is 1edfe. By executing the instruction "addr2line -C -f -e libexample.so 0x0001edfe", the result is shown in Figure 7-21. From the third line, we can see that the crash is accurately located at line 12 of mock\_native\_crash.cpp.&#x20;

![Figure 7-21 Crash Localization](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter7_img_21.png)

## 7.4.3 Instruction Analysis

For so libraries without a symbol table, we cannot perform stack unwinding. In this case, we generally use tools such as objdump to parse the so library into assembly code and then perform instruction analysis. For the crash that occurred in the text, we already know that its crash address is 0x1edfe, so we look into the assembly code of libexample.so, and the corresponding code is shown in Figure 7-22.&#x20;

![Figure 7-22 assembly code for the crash](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter7_img_22.png)

In the instruction "str r1, \[r0, #0]" in the code, "str" is a store instruction, which means storing the value in register r1 to the memory address with an offset of 0 from the address in register r0. Looking back at the register information in the previous log, as shown in Figure 7-23, we can see that the value of r0 is 0, and thus we can know that the cause of the exception is writing data to a null address.&#x20;

![Figure 7-23 Crash register information](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter7_img_23.png)

Here, a simple case is used to explain the troubleshooting approach for Native Crash. In real-world scenarios, troubleshooting Native Crash may be much more complex than this case, requiring a deep understanding of knowledge such as the ARM instruction set and Native. This requires us to go through a longer period of learning and more practical troubleshooting cases. However, mastering the troubleshooting approach for this simple Native Crash in this chapter is the first step we take.&#x20;

# 7.5 ANR Analysis Approach

Here, I still simulate an ANR in the sample program to guide readers to master the thinking of ANR analysis. As shown in Figure 7-24, the main thread will generate an ANR due to waiting for a lock.&#x20;

![Figure 7-24 Simulated ANR](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter7_img_24.png)

Analysis and troubleshooting of ANR are often much more complex than those of Crash. Therefore, after an ANR occurs, the most important thing is to obtain sufficient log information for analysis. The most critical logs include the log information under the data/anr directory and Log log information. We can use the previously introduced methods to capture these logs, or obtain this log information through offline means such as BugReport. Once we have this log information, we can start analyzing the ANR.

## 7.5.1 Preliminary Analysis

When analyzing ANR, we first need to conduct a preliminary analysis based on the logs to determine the time, type, and approximate cause of the ANR. After running the example logic above, an ANR will occur and an ANR log will be generated in the /data/anr directory, with some ANR log information shown in Figure 7-25.&#x20;

![Figure 7-25 Partial ANR Log Information](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter7_img_25.png)

As shown in the log, on line 1, it can be determined that the ANR type is InputDispatching TimedOut, which means input event dispatch timeout. Then, from the following lines, information such as the time when the ANR occurred, the thread ID, etc., can be known. Starting from line 10, further information about the main thread, such as its priority, locks, running state, stack, etc., can be known, among which the most critical is the running state of the main thread, which commonly has the following states&#x20;

* Runnable: The thread is runnable or is running

* Sleeping: The code logic calls functions such as wait, sleep, or join to put the thread into a dormant state&#x20;

* Blocked: The thread is blocked, usually waiting to acquire a lock object at this time

* Waiting: The execution of a wait function without a timeout parameter set in the code causes the thread to enter a waiting state

It can be seen that the state of the main thread in the log is Blocked, and at this time we can basically determine that this is an ANR caused by waiting for a lock.&#x20;

## 7.5.2 Performance Analysis

Since the ANR in the example is a relatively simple one, through basic analysis, we can know that the cause of the ANR is due to the main thread waiting for a lock. However, in reality, many ANRs are not that simple, especially when the state of the main thread is in the Runnable state. At this time, the main thread is in the running state but an ANR occurs. In this case, the cause generally cannot be located through simple preliminary analysis. At this time, we then need to conduct performance analysis to confirm whether the ANR anomaly is caused by performance issues.

Detailed performance-related data is not in the Trace log but in the Log log. According to the log information in Logcat, as shown in Figure 7-26, the performance status at the time of ANR occurrence can be seen.&#x20;

![Figure 7-26 Performance information in the Log log when ANR occurs](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter7_img_26.png)

When analyzing performance, the main directions we focus on are as follows:

1. View CPU LOAD Average (Load Average) data

In the performance log, you can see the line "Load: 0.74 / 0.56 / 0.55", which represents the average number of processes that are using or waiting to use the CPU within the past 1, 5, and 15 minutes. This value, also known as the load average, is an indicator used to measure system load. Generally, the load average should not exceed the total number of processor cores; if it does, it indicates that the system load is high and CPU resources are insufficient.&#x20;

* Check CPU usage of the process&#x20;

From the line "CPU usage from 254840ms to 0ms ago" in the log, we can know the CPU usage within the past time period (from 254840 ms to 0 ms ago). The log will print out the process information in descending order of CPU usage, including detailed data such as CPU usage, process ID, process name, CPU usage percentage of user processes and kernel processes, etc.&#x20;

* View other key processes or indicators&#x20;

Check keywords such as kswapd0 and iowait to determine whether ANR is caused by system memory or IO issues. kswapd0 is a process that manages memory. When Memory Space is insufficient, the kswapd0 process is responsible for swapping out infrequently used pages to the swap space. If the occupancy rate of this process is high, it indicates that the device is in a low-memory state and frequent page swapping operations are occurring.&#x20;

iowait is an indicator of system state, representing the proportion of time the CPU spends waiting for IO operations to complete. A high iowait value usually indicates an anomaly in IO operations within the system, which may lead to CPU resource idling and thus cause ANR.

* Memory and other performance data

Finally, you can check the memory usage, including analyzing memory information such as total allocated bytes, freed bytes, available memory, etc., to determine whether there are issues such as insufficient memory or frequent memory swapping. Additionally, you can combine comprehensive factors such as the number of threads, disk usage, and device model to further determine whether there are any performance anomalies.

Through the analysis of these performance data, we can basically confirm whether the ANR is caused by performance issues. If it is confirmed that the ANR is caused by performance problems, we need to shift to addressing performance issues such as memory and CPU. If it is confirmed that the ANR is not caused by performance, we need to further analyze the reasons for the ANR.

## 7.5.3 Direct and Indirect Analysis

If the state of the thread is in states such as Block or Waiting, this usually means that the main thread is blocked or waiting for certain operations to complete. Due to the existence of blocking tasks, the main thread cannot respond to the tasks of the four major components within the specified time, resulting in the occurrence of ANR, as shown in Figure 7-27. At this time, we can directly analyze the stack of the main thread to see if there are situations such as waiting for a lock or deadlock in the current task of the main thread, or if there are time-consuming operations such as database read and write or large file IO, and then we can analyze the cause of ANR. From the 10th line of the ANR log, we can see that the main thread is in the Block state. Further looking at the log stack, we can see the log line "waiting to lock held by thread 4", indicating that the main thread is waiting for the lock held by thread number 4. Then, through the stack, we can clearly locate the place where the lock is being waited for, which is on line 34 of StabilityExampleActivity, and this is the root cause of the ANR.&#x20;

![Figure 7-27 A task blocking causes ANR](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter7_img_27.png)



If the state of the main thread is in the Running state, it indicates that the ANR is not caused by the current stack, but by certain functions before the current stack. In this case, it may be due to too many tasks in the main thread, or some tasks in the main thread taking a long time. These factors accumulate and cause the main thread to fail to respond to the tasks of the four major components in a timely manner, as shown in Figure 7-28.&#x20;

![Figure 7-28 Task stacking leads to ANR](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter7_img_28.png)



This type of ANR cannot be troubleshooted through direct stack analysis, so it is generally quite complex to troubleshoot. In this case, it is usually indirectly converted into an analysis of time-consuming functions. Common time-consuming functions are mainly caused by factors such as complex logic, slow I/O, and waiting for locks. We need to reduce the time consumption of functions through offline Trace logs or online collection of time-consuming methods, thereby indirectly reducing the probability of this type of ANR occurring. The slow function monitoring to be discussed below is a solution for indirectly analyzing and managing ANR.&#x20;

# 7.6 Slow Function Monitoring

Slow functions are functions that take a long time to execute. Slow functions in the main thread not only affect the page opening and rendering speed but also easily degrade into ANR. For example, a slow function that takes 3 to 4 seconds to execute theoretically does not cause ANR, but once in the online environment, it is very likely to become even slower due to issues such as CPU busyness, thus exceeding 5 seconds and triggering ANR. If we can monitor slow functions in the main thread online and manage them, we can achieve significant optimization effects on ANR.

There are many solutions for monitoring slow functions, such as using the Looper message queue of the main thread to complete the time-consuming statistics of each main thread task. The ByteDance bytecode instrumentation technology learned earlier can also be used to perform time-consuming statistics on each main thread function. The bytecode instrumentation solution is more flexible and powerful, so I mainly introduce this solution.

## 7.6.1 Slow Function Detection Method

In Chapter 3, "Practical Memory Optimization", I explained how to insert the logging output capability of "hello world" into each method through bytecode. In addition to this simple capability, we usually insert a defined function into a function through bytecode to more flexibly expand the function's capabilities. Therefore, we can first define the logical function for slow function monitoring, and then insert it into the main thread's function through bytecode instrumentation.&#x20;

The detection logic of the slow function is not complicated, and the code implementation is as follows. In the code, two functions, recordMethodStart and recordMethodEnd, are defined. recordMethodStart records the start time of the main thread method and needs to be inserted before the execution of each method. recordMethodEnd needs to be inserted after the execution of each method, calculates the time taken by the method based on the previous start time, and if the time taken exceeds the threshold, it can further print logs or report.&#x20;

```c++
public class MethodTracer {
    // A method whose execution time exceeds 3 seconds is considered a slow method
    public static int slowMethodThreshold = 3000;

    // Logic before method execution
    public static void recordMethodStart() {
        if (Thread.currentThread().name == Looper.getMainLooper().thread.name) {
            methodStartTime = System.currentTimeMillis();
        }
    }

    // Logic after method execution
    public static void recordMethodEnd(String name) {
        if (Thread.currentThread().name == Looper.getMainLooper().thread.name) {
            // Calculate method execution time
            int cost = System.currentTimeMillis() - methodStartTime;
            if (cost > slowMethodThreshold) {
                // Record if it exceeds the threshold
                printOrReoprt(name, time);
            }
        }
    }
}
```

## 7.6.2 Main Thread Method Instrumentation

After defining the detection method for slow functions, the above-defined function can be inserted before and after each method through ASM bytecode instrumentation. The previous chapter has already explained in detail the process of ASM instrumentation, so I will directly introduce the last step of the operation, which is to implement the instrumentation through the MethodVisitor object and in its two callback methods, onMethodEnter and onMethodExit. The implementation of the solution is as follows: in the code, simply insert the above-defined function through the MethodVisitor object during the onMethodEnter and onMethodExit phases respectively.&#x20;

```java
class MethodCostMethodVisitor(
    api: Int,
    methodVisitor: MethodVisitor?,
    access: Int,
    name: String?,
    descriptor: String?,
    val methodNameParams: String
) : AdviceAdapter(api, methodVisitor, access, name, descriptor) {

    override fun onMethodEnter() {
        super.onMethodEnter()
        mv.visitMethodInsn(
            Opcodes.INVOKESTATIC, 
            "com/example/android_performance/speed/MethodTracer",
            "recordMethodStart",
            "()V",
            false
        )
    }

    override fun onMethodExit(opcode: Int) {
        mv.visitLdcInsn(methodNameParams)
        mv.visitMethodInsn(
            Opcodes.INVOKESTATIC, "com/example/android_performance/speed/MethodTracer",
            "recordMethodEnd",
            "(Ljava/lang/String;)V",
            false
        )
    }
}
```

Optimizing the detected slow functions can effectively reduce the probability of ANR occurrence. However, instrumenting each method will inevitably affect the performance of the program, so we should avoid enabling slow function monitoring for all users and only enable it for a small number of test users.&#x20;



| breakpad：https://github.com/google/breakpad<br />tailor: https://github.com/bytedance/tailor<br />signal\_catcher.cc: <br />https://cs.android.com/android/platform/superproject/+/android-14.0.0\_r9:art/runtime/signal\_catcher.cc<br />hprof.cc: <br />https://cs.android.com/android/platform/superproject/+/android-14.0.0\_r9:art/runtime/hprof/hprof.cc |
| -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

