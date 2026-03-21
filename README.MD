# THE ART OF ANDROID PERFORMANCE OPTIMIZATION

This is the English translation of the highly acclaimed Chinese book **《Android性能优化之道》**.

## About this Book

Original Title: **《Android性能优化之道》**

English Title: **The Art of Android Performance Optimization**

This book has received significant praise and is widely recognized in China as a definitive guide for Android performance optimization.

You can also find more articles and insights about this book on the author's personal blog: [Helson's Blog - Android Performance](https://blog.helson.uk/tags/androidperformance)

This repository aims to bring these valuable insights to the global Android developer community through a clear and accurate translation.

## Table of Contents

- [Chapter 1 - Preface](Chapter%201%20-%20Preface.md)
  - [1.1 Why This Book Was Written](Chapter%201%20-%20Preface.md#11-why-this-book-was-written)
  - [1.2 How to do a good job in Performance optimization](Chapter%201%20-%20Preface.md#12-how-to-do-a-good-job-in-performance-optimization)
  - [1.3 Features and Main Content of This Book](Chapter%201%20-%20Preface.md#13-features-and-main-content-of-this-book)
  - [1.4 Target Audience of This Book](Chapter%201%20-%20Preface.md#14-target-audience-of-this-book)
  - [1.5 How to Read This Book](Chapter%201%20-%20Preface.md#15-how-to-read-this-book)
- [Chapter 2 - Memory Optimization Principles](Chapter%202%20-%20Memory%20Optimization%20Principles.md)
  - [2.1 Virtual Memory](Chapter%202%20-%20Memory%20Optimization%20Principles.md#21-virtual-memory)
    - [2.1.1 Why Virtual Memory is Needed](Chapter%202%20-%20Memory%20Optimization%20Principles.md#211-why-virtual-memory-is-needed)
    - [2.1.2 What is virtual memory?](Chapter%202%20-%20Memory%20Optimization%20Principles.md#212-what-is-virtual-memory)
    - [2.1.3 ELF Files](Chapter%202%20-%20Memory%20Optimization%20Principles.md#213-elf-files)
    - [2.1.4 Virtual Memory Allocation and Deallocation](Chapter%202%20-%20Memory%20Optimization%20Principles.md#214-virtual-memory-allocation-and-deallocation)
    - [2.1.5 Virtual Memory to Physical Memory](Chapter%202%20-%20Memory%20Optimization%20Principles.md#215-virtual-memory-to-physical-memory)
  - [2.2 Composition of Memory Data](Chapter%202%20-%20Memory%20Optimization%20Principles.md#22-composition-of-memory-data)
    - [2.2.1 maps File](Chapter%202%20-%20Memory%20Optimization%20Principles.md#221-maps-file)
    - [2.2.2 Java Heap Memory](Chapter%202%20-%20Memory%20Optimization%20Principles.md#222-java-heap-memory)
    - [2.2.3 Native Memory](Chapter%202%20-%20Memory%20Optimization%20Principles.md#223-native-memory)
  - [2.3 Methodology for Memory Optimization](Chapter%202%20-%20Memory%20Optimization%20Principles.md#24-methodology-for-memory-optimization)
    - [2.3.1 Clean up data in a timely manner](Chapter%202%20-%20Memory%20Optimization%20Principles.md#241-clean-up-data-in-a-timely-manner)
    - [2.3.2 Reduce data loading](Chapter%202%20-%20Memory%20Optimization%20Principles.md#242-reduce-data-loading)
    - [2.3.3 Increase Memory Space Size](Chapter%202%20-%20Memory%20Optimization%20Principles.md#243-increase-memory-space-size)
- [Chapter 3 - Practical Memory Optimization](Chapter%203%20-%20Practical%20Memory%20Optimization.md)
  - [3.1 Java Memory Leak Detection](Chapter%203%20-%20Practical%20Memory%20Optimization.md#31-java-memory-leak-detection)
    - [3.1.1 Manual Analysis](Chapter%203%20-%20Practical%20Memory%20Optimization.md#311-manual-analysis)
    - [3.1.2 Automatic Analysis](Chapter%203%20-%20Practical%20Memory%20Optimization.md#312-automatic-analysis)
  - [3.2 Native Memory Leak Detection](Chapter%203%20-%20Practical%20Memory%20Optimization.md#32-native-memory-leak-detection)
    - [3.2.1 Intercept the malloc and free functions](Chapter%203%20-%20Practical%20Memory%20Optimization.md#321-intercept-the-malloc-and-free-functions)
    - [3.2.2 Obtain Native stack](Chapter%203%20-%20Practical%20Memory%20Optimization.md#322-obtain-native-stack)
    - [3.2.3 Native Stack Information Restoration](Chapter%203%20-%20Practical%20Memory%20Optimization.md#323-native-stack-information-restoration)
    - [3.2.4 Introduction to Open Source Tools](Chapter%203%20-%20Practical%20Memory%20Optimization.md#324-introduction-to-open-source-tools)
  - [3.3 Bitmap Governance](Chapter%203%20-%20Practical%20Memory%20Optimization.md#33-bitmap-governance)
    - [3.3.1 Bytecode Operations](Chapter%203%20-%20Practical%20Memory%20Optimization.md#331-bytecode-operations)
    - [3.3.2 Optimization of Super Large Bitmaps](Chapter%203%20-%20Practical%20Memory%20Optimization.md#332-optimization-of-super-large-bitmaps)
    - [3.3.3 Bitmap Leak Optimization](Chapter%203%20-%20Practical%20Memory%20Optimization.md#333-bitmap-leak-optimization)
  - [3.4 Thread Stack Optimization](Chapter%203%20-%20Practical%20Memory%20Optimization.md#34-thread-stack-optimization)
    - [3.4.1 Thread Creation Process](Chapter%203%20-%20Practical%20Memory%20Optimization.md#341-thread-creation-process)
    - [3.4.2 Reduce the number of threads](Chapter%203%20-%20Practical%20Memory%20Optimization.md#342-reduce-the-number-of-threads)
    - [3.4.3 Reduce the default stack memory of threads](Chapter%203%20-%20Practical%20Memory%20Optimization.md#343-reduce-the-default-stack-memory-of-threads)
  - [3.5 Default webview memory release](Chapter%203%20-%20Practical%20Memory%20Optimization.md#35-default-webview-memory-release)
    - [3.5.1 Find the address through the maps file](Chapter%203%20-%20Practical%20Memory%20Optimization.md#351-find-the-address-through-the-maps-file)
    - [3.5.2 Find the address through system variables](Chapter%203%20-%20Practical%20Memory%20Optimization.md#352-find-the-address-through-system-variables)
- (More chapters coming soon...)
