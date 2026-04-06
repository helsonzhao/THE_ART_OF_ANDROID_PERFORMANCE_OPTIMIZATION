Before we start this chapter, I will first lead you to explore why we need to optimize the APK package size. The value of package size optimization is reflected in two directions: promotion and performance experience, mainly as follows.

<table><colgroup><col width="114"><col width="706"></colgroup>
<thead>
<tr>
<th><strong>Promote</strong></th>
<th><ul>
<li>The smaller the installation package, the higher the download conversion rate </li>
<li>The smaller the installation package, the lower the channel promotion cost and the unit cost of manufacturer pre-installation </li>
<li>Google Play has a clear limit on the size of installation packages, and packages exceeding the limit cannot be listed on the platform</li>
</ul></th>
</tr>
</thead>
<tbody>
<tr>
<td><strong>Performance</strong></td>
<td><ul>
<li>During the APK installation process, operations such as file copying, decompression, and dex compilation are required. Therefore, the smaller the package size, the shorter the installation process.</li>
<li>After the application is launched, files such as dex, resource, and lib in the application package need to be mapped into memory. Therefore, the smaller the package size, the less memory it occupies. </li>
</ul></td>
</tr>
</tbody>
</table>

Among them, the value in the direction of promotion is even immeasurable, so large companies often invest a lot of resources in APK size optimization to reduce the APK size as much as possible. Like other optimizations, to do a good job in APK size optimization, it is still necessary to first master the underlying principles, so as to carry out optimization in a more systematic and comprehensive manner. Therefore, in this chapter, we will mainly learn about the composition of APK packages and the APK package building process, and based on this basic knowledge, derive the optimization methodology for APK size.

# 8.1 APK Composition Analysis

The essence of an APK is a Compressed Packet, so after we decompress an APK package via ZIP or directly drag it into Android Studio, we can directly see the composition of the APK, as shown in Figure 8-1.&#x20;

![Figure 8-1 APK Package Composition](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter8_img_1.png)

The explanation of the APK package composition is as follows:&#x20;

| File Type           | Instructions                                                                                                                                                                              |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| classes.dex file    | Bytecode file generated after Java code compilation                                                                                                                                       |
| lib Directory       | Store so library files                                                                                                                                                                    |
| res Directory       | Store resource files                                                                                                                                                                      |
| resources.arsc file | Stores the index of file-type resources under the res directory, as well as the values of non-file-type resources, such as name, type information, configuration information, and values  |
| AndroidManifest.xml | Program Global Configuration File                                                                                                                                                         |
| META-INF Directory  | A file that describes package information, which also stores the manifest file                                                                                                            |
| assets Directory    | Stores the original file, which will not be compiled and will be directly packaged into the APK package                                                                                   |

Among the directories and files above, the ones that account for the bulk of the package size are mainly the dex files, the so library files under the lib directory, and the resource files under the resources.arsc, res, and assets directories. Our optimization of the package size also mainly starts with these files.

## 8.1.1 dex File

Java code generates class bytecode files after compilation, while the dex file is actually just a further integration of class bytecode files, placing multiple class files into a single dex file. The advantage of doing this is that it reduces redundancy, as duplicate data in multiple class files will be merged into one. According to official data, the size of the same Java code compiled into a dex file is only about 50% of the size of the class file.

From Figure 8-2, we can understand the differences between class files and dex files as well as the composition of their data segments. It should be noted that the data segments in class files do not have a one-to-one correspondence with those in dex files. For example, class files have a constant pool, but dex files do not;For example, the data in the method table (Method) that stores data such as method names, descriptors, and bytecode in a class file is actually scattered across multiple data segments in the dex file, such as Methods\_ids, Data, and String\_ids.&#x20;

![Figure 8-2 Comparison of class files and dex files](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter8_img_2.png)

Understanding which data exists in each data segment of a file can help us better optimize the file size. As shown in the following table, it is the meaning of the data segments in a dex file.&#x20;

| **Data Segment** | **Explanation**                                                                                                                              |
| ---------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| Header           | dex file header, which records descriptive data such as file information                                                                     |
| String\_ids      | String data index, which records the offset of each string in the data area                                                                  |
| Type\_ids        | Type data index, which records the string index of each type                                                                                 |
| Proto\_ids       | Prototype data index, which records the string of the method declaration, the return type string, and the parameter list                     |
| Field\_ids       | Field data index, which records the class it belongs to, type, and method name                                                               |
| Method\_ids      | Class method index, recording the class name to which the method belongs                                                                     |
| Class\_def       | Class definition data, which records various information of the specified class, including interfaces, superclasses, and class data offsets  |
| Data             | Data area, the area where data is actually stored                                                                                            |

The index area does not store the actual data; the actual data is stored in the Data data area. During the program's operation, it will use the index to search for the actual data in the data area.&#x20;

## 8.1.2 Resources and so Library Files

The resource files in the APK package include those under the res directory, the resources. resource file, and those under the assets directory. Let's take a look at them separately.&#x20;

### 1. resources  file

Files under the res directory mainly contain the following types of resources:

* res/anim/ Directory: This directory stores XML files that define animation attributes. Accessed via the R.anim class in the code.

* res/color/ Directory: This directory stores XML files that define the color state list. Accessed via the R.color class in the code.

* res/drawable/ Directory: This directory stores image files, such as.png,.jpg,.gif, or XML files, which are compiled into bitmaps, state lists, shapes, and animated images. Accessed via the R.drawable class in code.

* res/layout/ Directory: This directory stores XML files that define User Interface layouts, which are accessed through the R.layout class in the code.

* res/menu/ Directory: This directory stores XML files that define application menus, such as option menus, context menus, submenus, etc. Accessed via the R.menu class in code.

* res/raw/ Directory: This directory stores source files and will not be compiled. You need to call resource.openRawResource() in the code based on the resource ID named R.raw.filename to open the raw file

* res/values/ Directory: This directory stores XML files containing simple values (such as strings, integers, colors, etc.).

* res/xml/ Directory: This directory stores various configuration files used during runtime

Among these resources, except for the files under the raw directory and image files such as png, jpg, gif, etc., all other files have been compiled into binary format.

### 2. resources.arsc&#x20;

The resources.arsc file stores the indices of file-type resources under the res directory, as well as the values of non-file-type resources. In the code, by calling the getResource() interface and passing in the id of the corresponding resource, you can obtain the resource in the res directory. The logic behind this is actually that the resource manager will look for the real resource in the resources.arsc file based on the resource id. If it is a file-type resource, the resources.arsc file records the corresponding path of the file; if it is a non-file-type resource, the resources.arsc file records the corresponding value.&#x20;

Drag the resources.arsc file directly into Android Studio, and Android Studio will help us parse resources.arsc and display it in an intuitive way. As shown in Figure 8-3, you can see the content of this file, which contains data such as the corresponding IDs, names, and resource paths of resources in the res directory, as well as the values of non-file resources such as strings.

![Figure 8-3 Resources.arsc data](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter8_img_3.png)

### 3. assets files

Files in the res/raw and assets directories are all source files. Any files placed in these directories will be directly included in the APK package without being compiled or compressed. Therefore, we need to be more cautious with the data in these two directories to avoid placing large files in them.

Since the files in these two directories will not be compiled or compressed, why do we need the files in these two directories? In fact, the assets or raw directories are often used to store files such as text files, audio files, video files, HTML web pages, JS scripts, etc. By including these original files, we can enable the program to perform better in certain business operations, such as improving the opening speed of web pages and enhancing the playback speed of audio, etc.However, these resource files can actually be obtained through network downloads, so we can try our best to download these files via the network and reduce the interference of resource downloads on the program experience through some strategy optimizations. For example, resources can be downloaded in the background immediately after the application starts, and if the resource is still being downloaded when the user uses it, a download reminder can be provided to the user through a Progress Bar or other means.&#x20;

### 4. so library file

As a type of ELF format file, we are already quite familiar with so libraries from previous articles. In actual projects, it is easy for so files to become the main factor affecting the size of the APK package due to the use of multiple second-party and third-party libraries, or the need to support multiple processor architectures and thus include so libraries for multiple platforms. For these so files, we can still optimize the package size by downloading them, just like the resource files mentioned above.

# 8.2 APK Package Build Process

After understanding the internal composition of the APK package, let's take a look at the APK package build process. The files mentioned above are all products of the APK package build process, so only when we are familiar with the build process can we identify optimization points in this process that can reduce the size of the product files.&#x20;

The construction of an APK package is mainly divided into two processes: compilation and packaging, as shown in Figure 8-3. The compilation process converts source code into dex files and all other files into compiled files; the packaging process combines, signs, and aligns the dex files and compiled resources to generate the APK installation package file. Next, let's take a detailed look at these two processes.&#x20;

![Figure 8-3 APK Build Flow](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter8_img_4.png)

## 8.2.1 Compilation and Packaging Process

The compilation process mainly involves the compilation of resource files and Java source code. In fact, besides Java code, there is also Kotlin code, but to simplify the process, it will not be introduced in this chapter. Let's first take a look at the resource files.&#x20;

### 1. Resource file compilation

The Android Asset Packaging Tool, abbreviated as aapt, generates the R.java file and the resource.arsc file based on the files in the res directory. These two files are actually index files. When we want to use resources in the code, we first need to know the Int-type ID value of the resource in the R.java file, and then use this ID value through the resource manager to find the actual resource in the resource.arsc file.&#x20;

In addition to generating the above two files, aapt will also compress the XML resource files in the res directory and generate binary files. As for the files in the res/raw directory and the assets directory, they will be directly included in the APK package without any compilation operations.&#x20;

### 2. dex file compilation

Next, let's continue to understand the compilation process of dex files. Java files first need to be compiled into class files, and this process can be accomplished using the javac command of the JDK, which belongs to the Java process and does not involve Android knowledge. However, the further compilation of class files into dex files belongs to the Android packaging process, in which Android does quite a lot of things, and will be introduced in detail next.

The process of compiling class files into dex files has, so far, gone through the following three versions.&#x20;

* Proguard + DX Compile

* Proguard + D8 Compile

* R8 Compile&#x20;

Next, let's take a look at the processes of several versions one by one.

**1) Proguard + DX Compile**

Through Flowchart 8-4, we can understand the process of compiling Java files into dex files.

![Figure 8-4 Proguard + dx compile process](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter8_img_5.png)

The main steps involved in the compilation process are as follows:&#x20;

1. Generate class bytecode files via javac

2. The class bytecode file then goes through desugar, which is the desugaring process, mainly to be compatible with the new syntactic sugar features in JDK8, such as lambda expressions&#x20;

3. Then the bytecode file goes through some third-party script processing procedures, such as the bytecode manipulation procedures introduced earlier

4. Then the Proguard script will shrink and optimize the bytecode files

5. Finally, the class bytecode file is compiled into a runnable dex file by the DX compiler

**2) Proguard + D8 Compile**

Performance optimization is a never-ending process. D8 emerged as an optimized version of DX, and its process is basically the same as the above, except that the DX compiler has been replaced with the D8 compiler, and the desugar process has also been integrated into D8 instead of being carried out through third-party scripts. The process is shown in Figure 8-5.

![Figure 8-5 Proguard + D8 compile process](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter8_img_6.png)

Compared to DX, R8 has significantly improved performance. According to official Android data, when compiling dex files with D8, the compilation time has been reduced by 20%, and the size has also decreased by 4%. Starting from Android Studio 3.1, D8 has become the default dex compiler.

**3) R8 Compile**

Due to the emergence of the new languag&#x65;**&#x20;**&#x4B;otlin, Android needed to improve and optimize the D8 compiler again, thus giving rise to the currently most widely used compiler, R8. Starting from Android Studio 3.4, the R8 compiler has been used by default.

R8 integrates Proguard scripts, but still uses the same obfuscation and keep rules as Proguard, and supports the compilation of Kotlin. Its process is shown in Figure 8-6. According to the official documentation, R8's compilation speed has been significantly improved, and the size of the dex file has also been optimized to some extent.&#x20;

![Figure 8-6 R8 compile flow](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter8_img_7.png)

Android's compilation tools have been upgraded by several versions, from DX to D8 and then to R8, with increasingly stronger optimizations for package size and performance during the compilation process. According to the introduction in[the official documentation](https://developer.android.com/studio/build/shrink-code?hl=zh-cn), the following optimizations are mainly included.

* Code shrinking: Detect and safely remove unused classes, fields, methods, and properties from the app and its library dependencies. For example, if only a few APIs of a library dependency are used, the shrinking feature can identify the library code that the app does not use and remove only this part of the code from the app.

* Resource shrinking: Remove unused resources from the packaged app, including those in the app's library dependencies. This feature can be used in conjunction with code shrinking, so that after removing unused code, all resources that are no longer referenced can also be safely removed.

* Obfuscation: Shorten the names of classes and members to reduce the size of DEX files.

* Optimization: Check and rewrite the code to further reduce the size of the application's DEX file.

All the above optimizations need to be enabled through configuration, and I will also provide a detailed introduction in the subsequent practical chapters.

### 3. Compile.so files

When writing and building C++ code, the code needs to go through several processes, including preprocessing, compilation, assembly, and linking, as shown in Figure 8-7. The detailed flow of each process is as follows:&#x20;

1. Preprocessing
   During the preprocessing phase, the preprocessor processes the source code files, executes a series of preprocessing directives, such as performing text replacement, file inclusion, conditional compilation, etc., on source code like #include, #define, #if, etc., to generate a new source code file without macro definitions, comments, or preprocessing directives, which typically ends with the.i file extension.

2. Compiling (Compiling)
   The compiler performs lexical analysis, syntax analysis, semantic analysis, code optimization and other operations on the preprocessed source code, generating a text file containing assembly language instructions, usually with the extension .s.

3. Assembling
   The assembler translates assembly language instructions into machine language instructions, generating an object file containing binary code, typically with the.o extension

4. Linking
   During the linking phase, the linker combines the object files with the required library files, resolves symbol references, and generates an executable file or a shared object file. The linker processes the referenced functions and variables, matching them with their corresponding definitions. If external libraries are used in the code, the linker will search for and combine them with the object files. Finally, the linker will generate an executable file or a shared library file (so)

### 4. APK Packaging

The packaging process is much simpler than the compilation process. The apkbuilder tool compresses the files generated after compilation, as well as files such as assets and raw that do not need to be compiled, in zip format, thus generating an apk installation package. After generating the APK package, for security reasons, signing is also required. Meanwhile, for performance reasons, byte alignment is also necessary. The Linux system reads data in pages, and byte alignment means aligning the starting offset address of each file in the package to an integer multiple of the page size (4KB), so that the speed of accessing the APK file through memory mapping will be faster.&#x20;

## 8.2.2 Gradle Tasks

The logic of the above compilation and packaging process is all completed through individual Gradle tasks. We can view the compilation information of each Gradle task, including its name, logs, and other details, in the "Build Output" of Android Studio, as shown in Figure 8-8.&#x20;

![Figure 8-8 Gradle task log executed during Android Studio packaging](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter8_img_8.png)

Here, the author takes the Debug package of the sample program as an example to list the functions of some core Gradle tasks during the build process. The Gradle tasks for the Release package are the same as those for the Debug package, with the difference being that the names of tasks containing the word "Debug" will all be changed to "Release".&#x20;

* compileDebugAidl: This task compiles AIDL files into Java interface files

* checkDebugManifest: This task verifies whether the configurations in the Manifest file, such as permission declarations, activity registrations, services, etc., meet the requirements of the build

* generateDebugBuildConfig: This task generates the BuildConfig.java file, which contains some configuration constants at build time, such as version name, version code, build type, etc. These constants can be used in the application's code

* generateDebugResValues: This task reads the resource values configured in the build.gradle file, such as version number, build type, etc., and creates a generate.xml file in the /build/generated/res/resValues/debug directory. The resource values contained in this file can be directly used in XML files and code, facilitating some dynamic configurations

* mergeDebugResources: This task is responsible for merging all resource files, including layouts, images, values, etc., into a specified directory for packaging into the final APK file

* processDebugManifest: This task is responsible for processing and merging AndroidManifest.xml files from different modules and all dependencies to generate a single Manifest file used in the final APK file

* processDebugResources: This task packages resources through aapt, including converting source formats into the binary format required by the Android runtime, generating the R.java file, handling resource conflicts, and other operations.

* javaPreCompileDebug: This task performs pre-compilation work such as processing annotation processors

* compileDebugJavaWithJavac: Compile the Java source code of the project

* compileDebugNdk: This task invokes the NDK toolchain to compile Native code such as C or C++

* mergeDebugAssets: Merges the assets directories of all modules and libraries into a specified directory for packaging into the final APK file

* transformClassesWithDexBuilderForDebug: This task converts Java bytecode into dex files

* transformDexArchiveWithExternalLibsDexMergerForDebug: Merge all external library Dex files into a single Dex file

* mergeDebugJniLibFolders: Merge the contents of all JNI library folders

* transformNativeLibsWithMergeJniLibsForDebug: Merges all JNI libraries into a single folder so that they can be packaged into the APK

* transformNativeLibsWithStripDebugSymbolForDebug: This task removes the symbol table from the so library

* processDebugJavaRes: Process Java resource files and merge them into the final APK resources

* validateSigningDebug: Signature Verification

* packageDebug: Package all resources, Dex files, and other components into an APK file.

# 8.3 Package Volume Optimization Methodology

For files, three methodologies, namely streamlining, compression, and dynamization, can be adopted to optimize their size. From the previous knowledge, we know the composition of files in an APK installation package. Therefore, for each type of file in an APK package, optimization solutions can be sought from these three directions.&#x20;

## 8.3.1 Streamlining

Minimization, which refers to reducing the data of files, is the most readily conceivable approach in file size optimization. The most straightforward way to reduce the size of dex files or so libraries is by deleting the useless code in the project. The difficulty of this optimization solution lies not in deleting the useless code, but in how to identify it.

Of course, streamlining and optimization are not limited to a single approach like deleting code; there are also many more complex optimization strategies. For example, having an in-depth understanding of the data format of files and reducing some unused data segments based on the characteristics of the data format. Common strategies include removing the symbol table from so files, removing debug information from the Data data area of dex files, etc. Although these strategies for streamlining and optimizing based on file data formats are much more complex, they can also bring us more inspiration and directions for optimization.

## 8.3.2 Compression

The dex, so, and resource files in the APK package will ultimately be compressed using the ZIP format. However, ZIP is not the compression format with the highest compression ratio. Therefore, we can attempt to replace it with a more optimal compression format to optimize the package size. For both dex files and so files, the compression algorithm can be changed during the packaging process. Correspondingly, when the application starts, it is also necessary to take over the system's decompression process and use the new compression algorithm to decompress the dex and so files. For audio, video, html, jss and other resource files that need to be placed in the asset directory, they can first be compressed using a format with higher compression ratio such as 7z, and then placed in the asset directory. Correspondingly, when we use them in the code, we also need to first decompress the files through decompression algorithms such as 7z before we can use them.&#x20;

In addition to replacing the general compression algorithm, we can also use compression techniques tailored to specific files, the most common of which is image compression technology. There are many image compression techniques available on the market that can significantly reduce the size of images without any perceptible difference to the human eye. A common example is [tinypng](https://tinypng.com/), a tool that can achieve lossless compression, reducing the size of image files to 30% to 50% of their original size. As shown in Figure 8-9, tinypng can compress an original 57 KB image to just 15 KB. Using TinyPNG is also very simple. You can directly import the image into the conversion address provided by the official website, or download the TinyPNG plugin in Android Studio to complete the compression. Of course, if the image format in our program does not need to be PNG, JPEG, or other formats, then directly converting the image to WebP format is the best solution, as WebP's compression ratio is more powerful than TinyPNG.

![Figure 8-9 tinypng optimization comparison chart](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter8_img_9.png)

## 8.3.3 Dynamization

There are many dynamic solutions in Android, such as Webview-based solutions, plug-in-based solutions, cross-platform solutions based on RN, and mini-program-based solutions. Through dynamic solutions, we can continuously and dynamically expand the functionality of the program without affecting its package size. Therefore, these dynamic solutions are also widely used in programs on the market. We need to choose the appropriate dynamic solution based on the actual business scenario. Most dynamic technologies have relatively high complexity and also require a certain learning cost. In the subsequent practical section, I will also specifically target the plug-in-based dynamic technology and delve into its technology implementation and principles.
