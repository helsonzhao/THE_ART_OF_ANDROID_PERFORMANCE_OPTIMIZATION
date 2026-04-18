The size of an APK package is mainly composed of resource files, dex files, and so library files. The optimization of the size of these three types of files is carried out based on the three methodologies of streamlining, compression, and dynamicization. In this chapter, I will lead you to learn multiple APK size optimization solutions based on these three methodologies.&#x20;

Regarding the methodology of streamlining, this chapter will comprehensively introduce solutions such as "streamlining resources", "streamlining dex files", and "streamlining so libraries" around resource files, dex files, and so library files. In the direction of compression, this chapter will introduce cases such as "compressing dex files" and "compressing so libraries".In the direction of dynamicization, this chapter will delve into the plug-in dynamicization solution. Therefore, technologies related to plug-inization, including "dynamic loading of resource files", "dynamic loading of class files", "dynamic loading of so library files", "dynamic loading of the four major components", etc., will all be comprehensively explained.&#x20;

Although for many small and medium-sized programs, package size optimization is not the top priority, the techniques and knowledge points explained in this chapter regarding package size optimization are all highly valuable. They are not only used in package size optimization but also help us gain a deeper understanding of the Android system and provide more ideas and inspiration for technology implementation in our project development.

# 9.1 Streamline Resources

When it comes to streamlining resources, we naturally think of methods such as deleting resources that are no longer in use and removing duplicate images. However, to achieve better results, we cannot rely on manual methods to check and delete them one by one; instead, we need more automated approaches. Therefore, this section will introduce how to more efficiently delete unused resources and duplicate images. In addition, it will also cover solutions for reducing the size of string resources by obfuscating file names, etc., which will be explained together in this section.

## 9.1.1 Delete Unused Resources

The first way to delete unused resources is to perform an unused resource scan through Lint (a static code analysis tool). In the Analyze menu of Android Studio, click "Run Inspection by Name" and enter "unused resource" in the dialog box to execute the unused resource scan, as shown in Figure 9-1.&#x20;

![Figure 9-1 AndroidStudio scans for useless resource entrances](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter9_img_1.png)

The scanned results are shown in Figure 9-2, and then you can manually delete them.&#x20;

![Figure 9-2 Android Studio useless resource scanning results](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter9_img_2.png)

The second way to remove unused resources is to let the compiler do it during the compilation and packaging process. You only need to enable shrinkResources in the buildTypes configuration item of the gradle configuration file in the app directory of the project. The configuration code is as follows. If the dontshrink field is enabled in the proguard obfuscation configuration file, it needs to be disabled; otherwise, even if unused resource files are scanned, they will not be automatically deleted.&#x20;

```groovy
buildTypes {
    release {
        shrinkResources true
        ……        
    }
    debug {
        ……
    }
}
```

shrinkResources has two modes, safe and strict, with the default mode being safe. In Android, in addition to directly obtaining resources based on resource IDs, resources can also be obtained by dynamically concatenating IDs through the Resources.getIdentifier interface. In safe mode, all resources that can match the name in the dynamic concatenation mode will be marked as used, so these resources cannot be optimized.For example, the approach shown in the following code will cause all images with the suffix "img_xxx" to be marked as used. However, in strict mode, the rules of the getIdentifier interface will be ignored, meaning that a resource will only be marked as used if it is actually retrieved and used via its ID in the code.

```java
String name = String.format("img_%d", index + 1);
res = getResources().getIdentifier(name, "drawable", getPackageName());
```

We can configure the mode of shrinkResources in the "res/raw/keep.xml" file. In the project, we should try to avoid using Resources.getIdentifier interface to obtain resources, and try to set shrinkResources to strict mode.If the project really needs to use this interface, and the strict mode causes some resource files to be accidentally deleted, we can configure the files to be retained in keep.xml.&#x20;

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources xmlns:tools="http://schemas.android.com/tools"
    tools:shrinkMode="strict"
    tools:keep="@drawable/ic_get_by_identifier"/>
```

When shrinkResources is enabled, the apk will execute [ShrinkResourcesTransform](https://android.googlesource.com/platform/tools/base/+/gradle_2.0.0/build-system/gradle-core/src/main/groovy/com/android/build/gradle/internal/transforms/ShrinkResourcesTransform.java) this Gradle task during the build process to scan and optimize unused resources. Here is a brief introduction to the process of this Gradle task, with the main steps as follows:&#x20;

1. Traverse all class files through bytecode operations, analyze whether the id in the R file is used in the files, and use this to detect whether the resource files are used by the code&#x20;

2. Traverse the Mainfest configuration file and non-file resources under the res directory to detect whether resource files are used by the Mainfest configuration file&#x20;

3. Determine whether to process the resource referenced by getIdentifier based on shrinkMode

4. Finally, replace the unused resources with an empty resource of the same name. It should be noted here that the unused resources will not be deleted, because this is to avoid potential exceptions in the program caused by the deletion of resource files. However, starting from AGP 7.1, support forcomplete deletion of unused resourceshas been added, which can be enabled by adding the configuration "android.experimental.enableNewResourceShrinker.preciseShrinking=true" to the "gradle.properties" file in the root directory

## 9.1.2 Delete Duplicate Images

In small projects, it is easy to find and delete duplicate images in the project by manually checking resource files. However, as the project grows larger, the number of images in the project will increase. Especially for large-scale applications with a multi-repository architecture, image resources are scattered across various repositories. At this point, manually detecting duplicate images is no longer feasible. Instead, an automated approach is needed to scan the program and detect and optimize duplicate images. There are usually two solutions for the automated approach.&#x20;

* The first method is to scan the images in the res resource directory through a custom Gradle script, then determine whether the images are duplicate through md5, delete the duplicate images, and scan the code that uses the images, and perform replacement through byte modification.&#x20;

* The second method is to scan the images in the res resource directory, then use MD5 to determine whether the images are duplicate, delete the duplicate images and record their addresses, and simultaneously replace the index address of the image in the resources.arsc file.&#x20;

The first solution requires traversing the code of the entire project, which will increase the compilation time significantly and is also relatively complex to implement. Therefore, I mainly introduces the second solution, which does not lead to deterioration of compilation time and is also the currently mainstream solution.&#x20;

In Android code, to access image resources under the res file directory, one must use the resource manager (AssertManager) to look up the address of the corresponding image in the resources.arsc file based on the id of the image resource. Therefore, if the index of duplicate images can be directly replaced with the index of the same image in the resources.arsc file, the optimization of duplicate images can be completed, and the process is shown in Figure 9-3.&#x20;

![Figure 9-3 resources.arsc duplicate image optimization scheme](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter9_img_3.png)

### 1. resouces.arsc format

The core steps of this solution are the parsing and modification of the resources.arsc file. We already have experience in modifying so files and Hprof files, so it is easy to get started when dealing with the resources.arsc file. The first step in implementing optimization is still to have a certain familiarity with the structure of this file, whose format is shown in Figure 9-4.&#x20;

![Figure 9-4 resouces.arsc file format](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter9_img_4.png)

It is mainly divided into 6 data segments, and the explanations for each data segment from top to bottom are as follows:

1\) Header Information (RES_TABLE_TYPE): This data segment records the type and location information of resources, and its data structure is shown in Figure 9-5

![Figure 9-5 header information data structure](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter9_img_5.png)

The header of each data segment will have a block of data with a ResChunk_header structure, which is used to record information such as the type and size of the current data segment. Its data structure is shown in Figure 9-6.&#x20;

![Figure 9-6 header information data structure](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter9_img_6.png)

For ease of understanding, I use Java code to define the data format of the resouces.arsc file, so the data structure of the header information and the explanation of each data item are shown in the following code.&#x20;

```c
class ResTableType
{  
    ResChunk_header header; // Represents the header information of the resource table type, including type and size
    byte id;   // ID of the current data segment, one byte 
    byte res0; // Reserved field, no specific meaning
    short res1; // Reserved field, no specific meaning   
    short entryCount; // Indicates the number of resource entries under this resource type    
    int entriesStart; // Indicates the starting position of resource entries in the resource table   
};

// Common header structure used to represent the header information of each data segment in the resource table, total 8 bytes
struct ResChunk_header 
{
    short type; // Indicates the type of data segment. For example, RES_TABLE_TYPE, RES_STRING_POOL_TYPE, etc.
    short headerSize; // Indicates the size of this header
    int size; // Indicates the length of this data segment, i.e., the length including header and content
};
```

2\) String Constant Pool (RES_STRING_POOL_TYPE): This data segment stores information about all string resources, and its data structure is shown in Figure 9-7, where stringCount and stringsStart represent the number of strings and the starting position of strings in the data segment, and sytesCount and stylesStart represent the number and starting position of style strings (style strings are a special type of string resource that contains additional style information, such as font style, color, and size, etc.).Style strings are generally represented by SpannableString objects. Immediately following are the offset array of strings and style strings, as well as the data content. The offset array of strings records the position of each string, through which the specific data can be found in the string data content.&#x20;

![Figure 9-7 string constant pool data structure](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter9_img_7.png)

Its data structure is defined as follows.&#x20;

```java
class ResStringPool
{
    ResChunk_header header;
    int stringCount; // Indicates the number of strings
    int styleCount; // Indicates the number of styled strings
    int flags; // Indicates the flags of the string pool
    int stringsStart; // Indicates the starting position of strings
    int stylesStart; // Indicates the starting position of styled strings        
    int[] stringsOffset; // String offset array
    String[] strings;  // String data
    int[] stylesOffset; // Styled string offset array
    String[] styles;  // Styled string data
};
```

3\) Resource Package Information (RES_TABLE_PACKAGE_TYPE): Contains basic information about the application package, such as package name, version number, resource type, and resource ID mapping, etc.

4\) Resource Type String Pool (RES_STRING_POOL_TYPE): Used to store resource types defined in the application, such as layout, string, color, etc., and uses the ResStringPool data structure.

5\) Resource item name string pool (RES_STRING_POOL_TYPE): Used to store the names of resource items, also using the ResStringPool data structure.

6\) Type Specification Data Block (RES_TABLE_TYPE_SPEC_TYPE): Used to store the configuration information of resource items, and the system can load different resource items based on the configuration of different devices.

7\) Resource Type Item Data Block (RES_TABLE_TYPE_TYPE): Used to store the name, type, value, configuration, etc. of resource items.

In the data segment above, the second item, the string constant pool, the fourth item, the resource type string pool, and the fifth item, the resource item name string pool, all use the same ResStringPool data structure. Here, a simple string resource "\<string name="tip"> hello world \</string>" is used as an example to illustrate their differences. In this string resource, "hello world" is the string resource, which is stored in the string constant pool; "string" is a resource type, stored in the resource type string constant pool; "tip" is the resource item name, stored in the resource item name string pool.&#x20;

### 2. Modify resources.arsc

Once you understand the format of the resources.arsc file, you can start making modifications. The modification method still involves operating through the file stream, that is, reading the resources.arsc file stream, finding the position corresponding to the target data and making modifications, and then writing the data stream back to a new file. This is similar to the method for the Hprof file introduced earlier. I will take parsing and modifying a certain field in the string constant pool of resources.arsc as an example to explain the operation method again.

1\) First, read the file stream of resources.arsc, and then read the data corresponding to type, headerSize, and size in the file stream header respectively. The code implementation is as follows:

```java
public static void readTable() {
    try {
        // 1. Read arsc file，and transfrom to byte stream
        FileInputStream stream = new FileInputStream("resources.arsc");
        DataInputStream dataStream = new DataInputStream(stream);
        // 2. Read short, short, and int respectively to construct the ResChunk_header data format
        ResChunk_header resChunk_header = new ResChunk_header();
        resChunk_header.type = dataStream.readShort();
        resChunk_header.headerSize = dataStream.readShort();
        resChunk_header.size = dataStream.readInt();     
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

2\) Through the size data in the ResChunk_header read above, we can determine the position of the string constant pool in ByteFlow. Then, we only need to move the position of the file stream forward by ResChunk_header.size - 8 byte to locate the data segment of the string constant pool. The position here needs to be subtracted by 8 byte because after the operations of readShort, readShort, and readInt, the position of the file stream is already at the 8th byte.The code implementation is as follows.&#x20;

```java
dataStream.skipBytes(resChunk_header.size - 8);
```

3\) After the file stream is positioned at the string constant pool, it parses the content in this data stream sequentially according to the data structure ResStringPool of the string constant pool mastered earlier, and the code implementation is as follows:

```java
// Skip the ResChunk_header data segment, which is 8 bytes
dataStream.skipBytes(8);
// The mark is called here to allow the stream's read position to be reset back to this point later.
dataStream.mark(dataStream.available());
// Create the data structure for the string constant pool
ResStringPool resStringPool = new ResStringPool(); 
resStringPool.stringCount = dataStream.readInt();
resStringPool.styleCount = dataStream.readInt();
resStringPool.flags = dataStream.readInt();
resStringPool.stringsStart = dataStream.readInt();
resStringPool.stylesStart = dataStream.readInt();
```

4\) Among the data parsed above, the most critical are the two offset data items, strings and styles. Based on the values of this data, the starting positions of strings and style strings in the file stream can be located. After locating the starting positions, the specific string content can be parsed sequentially according to the offset array of the strings. The code implementation is as follows.

```java
public void parseStringPool(ResStringPool resStringPool) {

    int stringCount = resStringPool.stringCount;
    resStringPool.stringsOffset = new int[stringCount];
    // Read the string offset array
    for (int i = 0; i < stringCount; i++) {
        stringOffsets[i] = dataStream.readInt();
    }
    
    // Reset the stream position back to the beginning of ResStringPoolHeader
    dataStream.reset();
    
    resStringPool.strings = new String[stringCount];
    int stringStartOffset = resStringPoolHeader.stringsStart;
    
    // At this point, skip forward by stringStartOffset bytes to reach the start position of the strings
    dataStream.skip(stringStartOffset);
    for (int i = 0; i < stringCount; i++) {
        int size;
        if (i + 1 < stringCount) {
            size = stringOffsets[i + 1] - stringOffsets[i];
        } else {
            /* For the last string, since only its start position is available and the exact data size cannot be directly known,
               the length of the last string is calculated as: the start position of the styled strings
               minus the size of the styled string offset array, minus the start position of the last string */
            size = resStringPool.stylesStart - 
                    resStringPool.styleCount * 4 - 
                    stringOffsets[i];
        }
        // Read the string        
        strings[i] = dataStream.readFully(new byte[size]);;
    }
}
```

Here we have parsed out all the strings in the string constant pool. After modifying a certain string data, we can write it back to a new file through the DataOutputStream stream. The operation of writing back the ByteFlow has been explained in the previous chapter, so I will not repeat the usage of DataOutputStream here.

### 3. Implementation of Image Deduplication

Once you understand how to modify the resources.arsc file, you can proceed to implement the optimization plan for image deduplication. We can achieve image deduplication by reading the APK package and parsing the resources.arsc file, or by using a custom Gradle task during the packaging process. The former method requires re-signing the APK package, which involves relatively cumbersome steps. Therefore, I mainly introduces here how to implement this plan through a Gradle task.&#x20;

Through the previous chapter, we already know that the APK compilation and packaging process has many stages, and we can place custom Gradle tasks into these stages for execution. If we want to modify the resources.arsc file, we need to find a suitable stage. If the timing is too early, the relevant files may not have been generated yet; if it is too late, it may affect the execution of other scripts. Here, it is recommended to execute it after the processDebug/ReleaseResources task.&#x20;

This stage will package res resources, assets resources, and the resources.arsc file into a zip Compressed Packet with a.ap_ suffix, so the resources.arsc file can be read at this stage. The code implementation of the process is as follows.&#x20;

```java
class AnnotationExecutorPlugin implements Plugin<Project> {
    @Override
    void apply(Project project) {
        project.afterEvaluate {
            //1. Find the ProcessResources task
            def processResSet = project.tasks.findAll {
                boolean isProcessResourcesTask = false
                android.applicationVariants.all {
                    variant -> 
                    if (it.name == 'process' + variant.getName() + 'Resources') {
                        isProcessResourcesTask = true
                    }
                }
                return isProcessResourcesTask
            }
            if(!isProcessResourcesTask){
                return
            }
            //2. Execute the image deduplication logic for the resources.arsc file after the ProcessResources task
            for (def processRes in processResSet){
                processRes.doLast {
                    File[] fileList = getResPackageOutputFolder().listFiles()
                    for (def i = 0; i < fileList.length; i++) {
                        //3. Find the .ap_ file
                        if (fileList[i].isFile() && fileList[i].path.endsWith(".ap_")) {
                            File packageOutputFile = fileList[i];
                            //4. Configure the extraction path and extract the .ap_ file
                            String unzipPath = packageOutputFile.path.substring(
                                    0, packageOutputFile.path.lastIndexOf("."))
                            unZip(packageOutputFile, unzipPath)
                            //5. Parse the resources.arsc file and perform image deduplication
                            imageOptimize(unzipPath)                          
                            //6. Repackage the extracted files into an .ap_ zip archive
                            zipFolder(unzipPath, packageOutputFile.path)
                        }
                    }
                }
            }
        }
    }
}
```

Through the custom Gradle script above, we have understood the main process of this solution. Next, we will mainly understand the implementation details of the imageOptimize method in the code above for image deduplication.

The process of parsing and modifying the resources.arsc file is rather cumbersome and error-prone. Therefore, I uses the third-party open-source tool [android-chunk-utils](https://github.com/madisp/android-chunk-utils) to implement the parsing and modification of the resources.arsc file. The following is the code flow for implementing the solution using this tool.&#x20;

```java
void imageOpitmize(String resourcePath) {
    // 1. Traverse the images in the res directory, find duplicate images by md5, and record them in a map
    HashMap<String, ArrayList <DuplicatedEntry>> duplicatedResources 
        = findDuplicatedResources(resourcePath);
    // Open the resources.arsc file
    File arscFile = new File(resourcePath + 'resources.arsc')
    if (arscFile.exists()) {
        FileInputStream arscStream = null;      
        /* ResourceFile is a data structure defined in android-chunk-utils,
           corresponding to the file structure of resources.arsc */
        ResourceFile resourceFile = null;
        try {
            arscStream = new FileInputStream(arscFile);
            resourceFile = ResourceFile.fromInputStream(arscStream);
            // 2. Call the getChunks method of ResourceFile to convert the arsc stream into a Chunk object tree
            List<Chunk> chunks = resourceFile.getChunks();
            HashMap<String, String> toBeReplacedResourceMap 
                    = new HashMap<String, String>(1024);
            Iterator<Map.Entry<String, ArrayList<DuplicatedEntry>>> iterator 
                    = duplicatedResources.entrySet().iterator();
            // 3. Iterate over the duplicate images recorded in duplicatedResources and delete them
            while (iterator.hasNext()) {
                Map.Entry<String, ArrayList<DuplicatedEntry>> duplicatedEntry = 
                        iterator.next();
                // Keep only the first resource (index 0), delete others starting from index 1
                for (def index = 1; index < duplicatedEntry.value.size(); ++index) {
                    // Delete the image and save the deleted image info in toBeReplacedResourceMap
                    removeZipEntry(apFile, duplicatedEntry.value.get(index).name);
                    toBeReplacedResourceMap.put(duplicatedEntry.value.get(index).name, 
                            duplicatedEntry.value.get(0).name);
                }
            }

            // 4. Update the data in resources.arsc
            for (def index = 0; index < chunks.size(); ++index) {
                Chunk chunk = chunks.get(index);
                if (chunk instanceof ResourceTableChunk) {
                    ResourceTableChunk resourceTableChunk = (ResourceTableChunk) chunk;
                    /* Find the string constant pool. Just call the getStringPool method directly.
                       StringPoolChunk is also a data structure defined in the android-chunk-utils tool */
                    StringPoolChunk stringPoolChunk = 
                            resourceTableChunk.getStringPool();
                    for (def i = 0; i < stringPoolChunk.stringCount; ++i) {
                        /* Traverse the values in the string constant pool. If a value is present in toBeReplacedResourceMap,
                           replace it */
                        def key = stringPoolChunk.getString(i);
                        if (toBeReplacedResourceMap.containsKey(key)) {
                            stringPoolChunk.setString(i, 
                                toBeReplacedResourceMap.get(key));
                        }
                    }
                }
            }
        } catch (IOException|FileNotFoundException ignore) {
        } finally {
            if (arscStream != null) {
                IOUtils.closeQuietly(arscStream);
            }
        }
    }
}
```

The main processes in the above code are explained as follows:&#x20;

1. Traverse the res file directory, find duplicate images based on md5, and record the paths in the duplicatedResources container&#x20;

2. Then call android-chunk-utils's getChunks method to parse the data segment of the resources.arsc file into the Chunk structure defined by this tool&#x20;

3. Begin traversing the duplicatedResources container and deleting the duplicate images recorded inside, retaining only one available image. Then save the information of the deleted images to the toBeReplacedResourceMap container

4. Finally, update the data in resources.arsc. The update method is to iterate through the previously parsed Chunk data segments, find the data segment ResourceTableChunk corresponding to the string constant pool, and then call the getStringPool method provided by android-chunk-utils to parse and obtain all the string data in the string constant pool. Then iterate through and compare the data recorded in toBeReplacedResourceMap, and if a match is found, replace it with the data of the first reserved image resource.

It can be seen that parsing and modifying the resources.arsc file through the android-chunk-utils tool can greatly simplify the process of this solution, making it easier to implement. In project development, we should be good at leveraging these mature open-source tools to make project development more efficient and reliable.&#x20;

## 9.1.3 Obfuscate File Name

The strings of resources in the project are all stored in the resources.arsc file. Therefore, for an APK package, the size of the resources.arsc file is also relatively large. As shown in Figure 9-8, the example program installation package has resources.arsc taking up more than 700 kb, which is even larger than the res file. So, streamlining strings is also an important means of optimizing resource files, and obfuscating file names is one of the most commonly used ways to streamline strings.&#x20;

![Figure 9-8 Size of resources.arsc in sample program](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter9_img_8.png)

After obfuscating the file names, long file names can be converted into short ones, so the data recorded in resources.arsc will be less, and the file size of resources.arsc will naturally decrease. For an application with a large number of resource files, the package size reduction brought about by resource name file obfuscation is quite significant.&#x20;

```java
res/anim/abc_fade_in.xml -> r/a/a.xml
```

Android provides code obfuscation, but does not provide obfuscation for resource file names. Therefore, we can implement this optimization solution ourselves. Implementing the solution requires parsing and modifying the resources.arsc file, and the process is similar to the implementation of image deduplication. So, with an understanding of the previous solution, implementing file name obfuscation becomes very easy.&#x20;

We can also perform operations in the custom Gradle plugin of the image deduplication solution. We only need to iterate through the resource files under the res directory, then rename the files in the order of incrementing strings a, b, c, d..., while finding the index address of the file in the string constant pool of the resources.arsc file and then modifying it. The code process is actually not much different from the image deduplication solution, so I will not go into details, and readers can implement it themselves. It should be noted that starting from AGP 4.2, official support forresource obfuscationhas been provided, which can be enabled by adding "android.enableResourceOptimizations = true" to thegradle.propertiesfile in the project root directory. Therefore, if the AGP version of our project is 4.2 or higher, there is no need to implement this solution anymore.

## 9.1.4 Use open source tools

There is still much that can be done with resources.arsc. For example, the shrinkResources configuration mentioned earlier only empties unused resource files but does not delete them. Therefore, we can further delete these resources and their related strings in the resources.arsc file.The principles of these optimizations are not complicated, but their implementation is relatively cumbersome, requiring familiarity with the format of the resources.arsc file, as well as parsing and modification. We can use tools such as android-chunk-utils to more conveniently implement these technical solutions.&#x20;

For the functions mentioned above, such as image deduplication and filename simplification, implementing them on your own would involve a large amount of work and be complex. Fortunately, these optimization solutions are all very mature, so these optimizations can be completed using open-source libraries, and there is no need for us to reinvent the wheel. Whether it is WeChat's [Andresguard](https://github.com/shwenzhang/AndResGuard) framework or byte's [AabResGuard](https://github.com/bytedance/AabResGuard), they are all very powerful and easy to use. You can refer to the official documentation for integration, and readers can check it out themselves.&#x20;

# 9.2 Streamline dex Files

Reducing code volume is one of the most cost-effective volume optimization strategies, among which the simplest approach is to delete business code that is no longer in use. This optimization is not complicated; rather, the complexity lies in how we can identify unused code in the project. Therefore, in this section, I will introduce strategies for detecting unused code, as well as some general optimization strategies for code streamlining.

## 9.2.1 Remove unnecessary code

The most crucial step in deleting useless code is how to identify it. As long as these useless codes can be identified, they can be directly deleted. Here, two methods for identifying useless code are introduced, namely using Lint detection and code coverage checking.&#x20;

Similar to detecting unused resources, Lint can also detect unused code. The operation method is the same, which can be done through Android Studio. Click Run Inspection by Name in the Analyze menu, and enter unused declaration in the dialog box to scan for unused methods, fields, and classes. For the scanned results, simply click safe delete. Code executed through dynamic means such as reflection may also be scanned as dead code, so we also need to check it and not simply delete it all at once.

The modules scanned by Lint are all those that depend on local source code. For modules that are dependent in the form of AAR or SDK, Lint cannot scan them. In this case, we can check which code has been executed and which has not during runtime. Here, we introduce Android's built-in code coverage detection tool: [jacoco](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Fjacoco%2Fjacoco).&#x20;

Code coverage detection is mainly used when testers or unit tests are testing new features to detect which code has been covered in the test and which code has not, thus fully ensuring the usability of the function before going live. In addition to being used for testing, we can also use this function in online code to assist developers in checking unused code. The following introduces the use of jacoco.

1\) Apply the jacoco script in gradle and enable it in buildTypes. I uses version 0.8.5, which is a relatively stable version.

```groovy
apply plugin: 'jacoco'

jacoco {
    toolVersion = "0.8.5"
}

android {
    ……
    buildTypes {
        release {
            ……
            testCoverageEnabled = true 
        }
        debug {
            ……
            testCoverageEnabled = true
        }
    } 
}
```

2\) Through the simple configuration above, we have enabled code coverage checking. We only need to find a suitable time, such as when the user moves to the background, to write the data collected by Jacoco to the local storage or upload it to the server. Next, let's take a look at the specific implementation code.

```java
private void generateCoverageReport() {
    File file = new File(Environment.getExternalStorageDirectory(),  "/coverage.ec")
    // Create a file for writing data
    if (!file.exists()) {
        try {
            file.createNewFile();
        } catch (IOException e) {
            
        }
    }
    try {
        // We need to use reflection and call the getExecutionData method to obtain the data and write it to the file
        OutputStream out = new FileOutputStream(file, false);
        Object agent = Class.forName("org.jacoco.agent.rt.RT")
                .getMethod("getAgent")
                .invoke(null);
        // Call the getExecutionData function to obtain coverage data
        out.write((byte[]) agent.getClass().getMethod("getExecutionData", boolean.class)
                .invoke(agent, false));
        out.close();
    } catch (Exception e) {
       
    }
    // Upload the file to the server
    ……
}
```

3\) After we obtain the code coverage data file generated by Jacoco on the server, we can go to the official Jacoco website to download the Jacoco JAR package, find the jacococli JAR file as shown in Figure 9-9, and execute the command "java -jar jacococli.jar report coverage.ec --classfiles xxx (the class directory generated by the project) --sourcefiles xxx (the source code directory of the project) --html report" to convert this file into an HTML file.

![Figure 9-9 Jacococli file in jacoco](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter9_img_9.png)

The parameter explanations in the script are as follows:&#x20;

* &#x20;coverage.ec is the code coverage file we obtained from the server;

* \--classfiles needs to specify the file path of the class files generated by our project. In an Android project, it is generally the debug or release directory under the /app/build/intermediates/javac/ path generated after compilation;

* \--sourcefiles needs to specify the source code path, which is the src/main/java directory of the project.&#x20;

Finally, the output format is specified as HTML, named "report". After we generate the HTML file, we can clearly see the usage rate of the object, as shown in Figure 9-10. After clicking in, we can also see the code coverage.

![Figure 9-10 Usage of code generated by Jacoco](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter9_img_10.png)

The implementation principle of jacoco is also through bytecode instrumentation. Inserting detection code into each method increases the amount of code, which not only leads to an increase in package size but also causes a certain amount of performance loss. Therefore, we should try not to enable this feature on a large scale; it is sufficient to enable it for a small number of test users. Currently, the main use case of jacoco is still in testing, and its use in online scenarios is relatively rare. If we need to use it online, I suggests that we trim jacoco to reduce the amount of instrumentation.&#x20;

## 9.2.2 Enable Compile Optimization

During the process of compiling Java into bytecode and bytecode into dex files, we also have many solutions to optimize the size, among which the most commonly used ones are obfuscation, code trimming, and code optimization. Let's take a look at them separately below.&#x20;

1\) First is obfuscation. Obfuscation was also mentioned when optimizing the size of resource files. Code obfuscation goes a step further than resource file obfuscation. In addition to shortening names, the code itself will also be obfuscated, such as method names, variable names, etc. After obfuscation, the size of class files will be smaller and they will also be more secure. After the code is obfuscated, the correspondence between obfuscated names and source code can be viewed through the mapping file under the path "/build/outputs/mapping/release/", as shown below.

```java
androidx.appcompat.app.ActionBarDrawerToggle$DelegateProvider -> a.a.a.b:
androidx.appcompat.app.AlertController -> androidx.appcompat.app.AlertController:
    android.content.Context mContext -> a
    int mListItemLayout -> O
    int mViewSpacingRight -> l
    android.widget.Button mButtonNeutral -> w
    int mMultiChoiceItemLayout -> M
    boolean mShowTitle -> P
    int mViewSpacingLeft -> j
    int mButtonPanelSideLayout -> K
```

We can enable code obfuscation through the minifyEnabled field in the buildTypes configuration of the build.gradle file under the project app directory. After obfuscation is enabled, we need to specify the configuration file for obfuscation rules (proguardFiles). The configuration code is as follows:

```java
android {
    buildTypes {
        release {
            minifyEnabled true
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
}
```

The configuration file for the obfuscation rules specified in the above code is "proguard-rules.pro", so we need to further configure the obfuscation rules for the code in this obfuscation configuration file. Common obfuscation rules are as follows:&#x20;

```plain&#x20;text
# Code obfuscation compression ratio, between 0 and 7, default is 5, generally no need to modify
-optimizationpasses 5

# Do not use mixed case class names during obfuscation, obfuscated class names will be lowercase
-dontusemixedcaseclassnames

# Specify not to ignore non-public library classes
-dontskipnonpubliclibraryclasses

# Specify not to ignore members of non-public library classes
-dontskipnonpubliclibraryclassmembers

# Do not preverify
-dontpreverify

# Whether to generate obfuscation mapping file, must keep, otherwise release builds cannot analyze exceptions
-verbose

# Specify the algorithm used for obfuscation, the following parameter is a filter, this filter is recommended by Google, generally unchanged
-optimizations !code/simplification/artithmetic,!field/*,!class/merging/*

# Keep annotations in code from being obfuscated, important for JSON entity mapping, e.g., fastJson would be unusable if obfuscated
-keepattributes *Annotation*

# Avoid obfuscating generics
-keepattributes Signature

# Keep line numbers when exceptions are thrown
-keepattributes SourceFile,LineNumberTable
```

Of course, there are also many classes in the project that cannot be obfuscated, otherwise the program may behave abnormally, such as objects obtained through string reflection, etc. I will also explain the keep rules of the file here, and the main keep rules are shown in the following table.

| **Command**                      | **Function**                                                                          |
| -------------------------------- | ------------------------------------------------------------------------------------- |
| -keep                            | Prevent classes and members from being removed or renamed                             |
| -keepnames                       | Prevent classes and members from being renamed                                        |
| -keepclassmembers                | Prevent members from being removed or renamed                                         |
| -keepclassmembersname            | Prevent members from being renamed                                                    |
| -keepclasseswithmembers          | Prevent the class and member that owns this member from being removed or renamed      |
| -keepclasseswithmembernames      | Prevent the class and member that owns this member from being renamed                 |
| Wildcard \*                      | Matches characters of any length, but does not contain the package name separator (.) |
| Class Wildcard                   | Matches characters of any length and includes the package name separator (.)          |
| Class extends                    | Matches a keep class that inherits from a certain base class                          |
| Class implements                 | Matches keep classes that implement a certain interface                               |
| Class $                          | Inner Class                                                                           |
| Member (Method) Wildcard \*      | Matches characters of any length, but does not contain the package name separator (.) |
| Member (Method) Wildcard \*\*    | Matches characters of any length and includes the package name separator (.)          |
| Member (Method) Wildcard \*\*\*  | Matches any parameter type                                                            |
| Member (Method) Wildcard...      | Matches parameters of any type and any length                                         |
| Member (Method) Wildcard <>      | Match Method Name                                                                     |

2\) Next is code pruning. During R8 compilation, the compiler will detect whether a method is used through reachability analysis of the method. If it is not used, it will be pruned during the compilation process. Enabling R8 is very simple; you only need to set the minifyEnabled property to true in build.gradle.

3\) Finally, there is code optimization. Code optimization is actually quite complex, as it requires going through three steps: lexical analysis, syntax analysis, and semantic analysis, to detect which code can be optimized. The most common example is code inlining, where methods that are only called once, or methods that are called multiple times but have an assembly code length less than 8, will be merged into the calling method. Other optimizations include deleting an if-else statement if it has an empty implementation, and so on.

When we turn on obfuscation, options such as code trimming and code optimization will also be enabled together, so these three optimizations are all performed simultaneously during the compilation process.&#x20;

## 9.2.3 dex rearrangement

In Chapter 5, "Optimization Practice of Speed and Fluency", we learned about the optimization solution of accelerating the startup speed by rearranging dex files through Facebook's redex. After rearranging the dex files, there is actually an additional optimization that also has a certain optimization effect on the size of the dex files.&#x20;

Why can optimizing the size be achieved after rearranging dex class files? By default, our class files are out of order in dex files, which results in many cross-dex references because objects in the current dex file use a lot of data such as objects, methods, or variables from other dex files. To support cross-dex references, the current dex file needs to hold the IDs of the data referenced by other dex files.As shown in Figure 9-12, if we call a method of a class in the dex1 file from a class in dex0, then dex0 needs to retain a reference to the class in dex1.&#x20;

![Figure 9-12 Rearrangement Optimization](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter9_img_11.png)

Therefore, when the class files in dex are rearranged, the cases of cross-dex calls will be greatly reduced, and the related reference data will also decrease, so the size of the dex file will naturally decrease. As can be seen from Figure 9-12, after rearrangement, the index of Class B has decreased from two to one. The optimization effect of package size after dex rearrangement is not exactly the same for applications of different sizes; the more dex files an application has, the more obvious the effect will be. If an application has only one dex file, this optimization will have no effect. According to official data, a medium-sized app can reduce its size by about 5% after dex class file rearrangement.

## 9.2.4 Remove line number information

As shown in Figure 9-13, this is a common Crash log. It can be seen that some information in the Crash log has an accurate line number, such as "android.os.Handler.dispatchMessage (Handler.java:109)", which occurs on line 109. However, "androidx.window.embedding.h.\<init> (Unknown Source:0)" has no line number information.&#x20;

![Figure 9-13 Crash Log](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter9_img_12.png)

The line number information of the class file is stored in the "debug_items" data section of the data area of the dex file. By default, the line number information is retained, so after an exception occurs, the crash information can locate which line of which object has an error.However, we can also remove line number information. We only need to add the configuration "-keepattributes LineNumberTable" to the Proguard obfuscation configuration file. According to Google's official statistics, after removing line number information, the size of the dex file can be reduced by approximately 5.5%.

Removing line number information is not difficult; the challenge lies in how we can effectively analyze the stack after removing it. I will introduce here the ideas of two current mainstream solutions. Let's look at the first solution

1. Option 1

The overall idea of Solution 1 is to first compile a package without removing line number information and extract the "debug_items" data from the dex file. However, the package released online still has the line numbers removed. When a crash occurs in the online package, the stack-related information is captured in the global exception handling Handler and uploaded to the server. The server then uses the captured information and the previously extracted "debug_items" data to restore the line numbers.&#x20;

The idea is easy to understand, but to implement it, we first need to know the format of the data stored in "debug_items". In fact, each method has a corresponding data structure of "debug_info_item" stored in the "debug_items" area, and the data structure of debug_info_item is as follows:&#x20;

```c++
struct debug_info_item {
    uint32_t line_start;        // Starting line number
    uint32_t parameters_size;   // Number of parameters
    uint32_t* parameter_names;  // Parameter name offset array
    uint16_t* line_code;        // Line number instruction array
};
```

The explanations for each field in this structure are as follows:&#x20;

* line_start: indicates the starting line number of the code.

* parameters_size: Represents the number of parameters of the method.

* parameter_names: is an offset array used to point to the string list storing parameter names. Through the offsets in the offset array, the corresponding parameter name strings can be found.

* line_code: An instruction array used to represent the line number of each line of code. Each instruction is a 16-bit unsigned integer representing the offset relative to the starting line number.

Based on the line_code instruction line number array, we can know the line number offset information of each instruction in the method. Then, by adding the offset value to the line_start starting line number value, we can know the actual line number of each instruction. Therefore, if we can obtain the stack instructions in the crash capture handler, we can restore the line numbers.

But how do we obtain this data? When we don't know how to implement a technical solution, we can actually first refer to how the Android system does it. When we get the stack through the Throwable object, the object actually obtains the stack information through the nativeGetStackTrace method. Its source code implementation is as follows, and we can see that the method will actually execute the InternalStackTraceToStackTraceElementArray function in the [thread.cc](https://cs.android.com/android/platform/superproject/+/master:art/runtime/thread.cc)file.

```c++
static jobjectArray Throwable_nativeGetStackTrace(JNIEnv *env, jclass, 
                                                jobject javaStackState) {
    ...
    ScopedFastNativeObjectAccess soa(env);
    return Thread::InternalStackTraceToStackTraceElementArray(soa, javaStackState);
}
```

The InternalStackTraceToStackTraceElementArray function traverses the stack of the current thread based on its depth. It then retrieves this method (artMethod) and the offset of the method instruction in the dex (dex_pc) through the code "decoded_traces->Get(0)". With this offset, it is actually possible to look up the line number in the debug_info_item.

```c++
objectArray Thread::InternalStackTraceToStackTraceElementArray(const
                                       ScopedObjectAccessAlreadyRunnable &soa,
                                       jobject internal, jobjectArray output_array,
                                       int *
                                       stack_depth) {
    // 1. Get the stack depth
    int32_t depth = soa.Decode<mirror::Array>(internal)->GetLength() - 1;
    ...   
    // Traverse the stack information   
    for (uint32_t i = 0; i < static_cast<uint32_t>(depth); ++i) 
    {
        // 2. Decode the internal stack information 
        ObjPtr <mirror::ObjectArray<mirror::Object>> decoded_traces = 
                soa.Decode<mirror::Object>(internal)->AsObjectArray<mirror::Object>();
                
        // Get the method information of the stack       
        const ObjPtr <mirror::PointerArray> method_trace = 
                ObjPtr<mirror::PointerArray>::DownCast(decoded_traces->Get(0));  
                      
        // 3. From the stack trace, obtain the corresponding ArtMethod object and the offset in the Dex file
        ArtMethod *method = 
                method_trace->GetElementPtrSize<ArtMethod *>(i, kRuntimePointerSize);      
        uint32_t dex_pc = method_trace->GetElementPtrSize<uint32_t>(
                i + static_cast<uint32_t> (method_trace->GetLength()) / 2,
                kRuntimePointerSize);       
        // 4. Create the StackTraceElement object      
        const ObjPtr <mirror::StackTraceElement> obj = 
                CreateStackTraceElement(soa, method, dex_pc);
    }
    return result;
}
```

Partial explanations of the above code are as follows

1. First, decode the incoming internal object through the soa object to obtain the stack depth&#x20;

2. Start using a loop to iterate through the stack trace information. In each iteration, decode the stack information and obtain the method information of the stack.&#x20;

3. Use the method_trace object to obtain the ArtMethod object of the current method and the offset (dex_pc) in the Dex file from the stack trace.&#x20;

4. Finally, use the CreateStackTraceElement function to create a StackTraceElement object, which encapsulates information about the method and offset.

Next, let's take a look at how the CreateStackTraceElement function restores the line number. The method code is as follows. It can be seen that this method will use the offset (dex_pc) and call the[GetLineNumForPc function in the code_item_accessors.h](https://cs.android.com/android/platform/superproject/+/master:art/libdexfile/dex/code_item_accessors.h) file to complete the restoration of the line number. In the logic of this function, it will iterate through the "debug_info_item" corresponding to this method to find the line number.

```c++
static ObjPtr <mirror::StackTraceElement> CreateStackTraceElement(const
                              ScopedObjectAccessAlreadyRunnable &soa,
                              ArtMethod *method,
                              uint32_t dex_pc)REQUIRES_SHARED(Locks::mutator_lock_){
    ...      
    int32_t line_number;
    // Get the code line number corresponding to the pc  
    line_number = method->GetLineNumFromDexPC(dex_pc);
    ...
}


inline bool CodeItemDebugInfoAccessor::GetLineNumForPc(const uint32_t address, 
                                                    uint32_t *line_num) const {
    return DecodeDebugPositionInfo([&](const DexFile::PositionInfo &entry) {
        if (entry.address_ > address) {
            return true;
        }
        *line_num = entry.line_;
        return entry.address_ == address;
    });
}

bool DexFile::DecodeDebugPositionInfo(const uint8_t *stream, const
IndexToStringData &index_to_string_data, const
                                      DexDebugNewPosition &position_functor) {
    // Traverse the corresponding debugInfo in the dex
    PositionInfo entry;
    entry.line_ = DecodeDebugInfoParameterNames(&stream, VoidFunctor());
    for (;;) {
        uint8_t opcode = *stream++;
        switch (opcode) {
            case DBG_END_SEQUENCE:
                return true;  
            case DBG_ADVANCE_PC:
                entry.address_ += DecodeUnsignedLeb128(&stream);
                break;
            case DBG_ADVANCE_LINE:
                entry.line_ += DecodeSignedLeb128(&stream);
                break;
             ...       // Handling of other event types, related to local variables and source files
            default: {
                int adjopcode = opcode - DBG_FIRST_SPECIAL;
                entry.address_ += adjopcode / DBG_LINE_RANGE;
                // Calculate the line number based on the offset of debug_info_item.
                entry.line_ += DBG_LINE_BASE + (adjopcode % DBG_LINE_RANGE);
                break;
            }
        }
    }
}
```

The key data that appears in the above process are the artMethod object and the dex_pc offset value. Both of these data are stored in method_trace, which is the method information of the stack. Therefore, we only need to obtain the data of method_trace to complete the calculation of the line number according to the above logic.&#x20;

We can obtain method_trace through Native Hook technology, and there are many points that can be hooked, such as [VMStack_getThreadStackTrace](https://cs.android.com/android/platform/superproject/+/master:art/runtime/native/dalvik_system_VMStack.cc) function, which is a signed function and therefore easy to intercept. After obtaining the instruction set and dex_pc offset value in ArtMethod and reporting them to the server, the server can then use the "debug_info_item" data to implement the above logic again, thus restoring the corresponding line number. Here, I only introduces the idea and general process of the solution. If readers are interested, they can further refine the solution details through the Native Hook technology we have learned.&#x20;

* Option 2

Each method has a corresponding "debug_info_item" data. Therefore, Option 2 is to have all methods share a single "debug_info_item" data and remove all other data from the "debug_items" data segment, so that the data volume of "debug_items" can be significantly reduced.

We can operate and modify dex files during the stage of generating dex files in the Gradle compilation process, such as the transformDexArchiveWithDexMerger stage, merge debug_info_item into one, and point all references to debug_info_item in dex methods to the same one.This method does not require server-side cooperation and can directly restore line numbers locally. However, this solution is very complex and cumbersome, requiring a very in-depth understanding of the data segment of the dex file. Interested readers can refer to byte's open source [ByteX](https://github.com/bytedance/ByteX/blob/master/SourceFileKiller/src/main/java/com/ss/android/ugc/bytex/sourcefilekiller/SourceFileExtension.java) open source library, which uses this solution to optimize line numbers.&#x20;

# 9.3 Streamline the so library

The optimization of so files can be summarized into three categories: the first is to reduce the code in so libraries; the second is to delete redundant so files; the third is to streamline the data content within so files.&#x20;

## 9.3.1 Delete unused code

During the compilation process, there are many configuration items that can help us remove useless resources or Java code. Similarly, during the compilation process of the so library, there are also many compilation configuration items that can help us streamline Native code.&#x20;

During the process of compiling Native code in the project, such as.cpp,.c, or.h files, into.so files, they all go through the processes of preprocessing, compilation, assembly, and linking. In this process, it is also possible to optimize the file size by enabling various configurations. I will introduce the two most commonly used methods here.&#x20;

1\) First, enable --gc-sections. After --gc-sections is enabled, it can remove unused code during the compilation of Native code. If we compile Native code through Android.mk, we can add the following configuration to the Android.mk file:

```plain&#x20;text
LOCAL_CPPFLAGS += -ffunction-sections -fdata-sections 
LOCAL_CFLAGS += -ffunction-sections -fdata-sections 
LOCAL_LDFLAGS += -Wl,--gc-sections
```

If you are compiling C++ code using CMake, you can add the following configuration to CMakeLists.txt:&#x20;

```plain&#x20;text
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -ffunction-sections -fdata-sections -Wl,--gc-sections")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -ffunction-sections -fdata-sections -Wl,--gc-sections")
set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -Wl,--gc-sections")
```

Actually, the two configuration methods above are the same, both enabling the configurations of "-ffunction-sections -fdata-sections" and "-Wl, --gc-sections".

* Among them, the two parameters -ffunction-sections and -fdata-sections indicate that each function or symbol is created as an independent section. Without using this parameter, after Native code compilation, there is only one monolithic.text section that records all the code. Using this parameter will make each function become a separate section. For example, the function func will be compiled into the.text.func section.

* In the parameter -Wl, --gc-sections, -Wl indicates the linking stage, and --gc-sections is a parameter following -Wl, which means deleting unused sections. Therefore, when these three configurations are used together, unused C or C++ code can be removed during the compilation process.&#x20;

2\) Next is enabling LTO. LTO is an abbreviation for Link Time Optimization, which refers to optimization during the linking phase. LTO can detect and remove invalid code when linking object files, thereby reducing the size of the compiled output. So, what is invalid code? For example, if a certain if condition is always false, then the Code Block under the true branch of the if statement is invalid code, and it will be removed after enabling LTO. The methods for enabling the configuration of the two compilation types, Cmake and Android.mk, are as follows respectively.

* Configuration method of Android.mk:

```plain&#x20;text
LOCAL_CFLAGS += -flto
OCAL_CPPFLAGS += -flto
LOCAL_LDFLAGS += -O3 -flto
```

* Configuration method of Cmake:

```plain&#x20;text
set(CMAKE_CXX_FLAGS "${CMAKE_C_FLAGS} -flto")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -flto")
set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -O3 -flto")
```

In the above configuration, -O3 represents the optimization level, with three levels: O1, O2, and O3, where O3 has the best effect. -flto indicates enabling LTO.&#x20;

## 9.3.2 Delete redundant so files

There are mainly the following 5 types of CPU processors in mobile phones.&#x20;

* armeabi-v7a: 7th generation 32-bit ARM processor.

* arm64-v8a: 8th generation 64-bit ARM processor.

* armeabi: 32-bit ARM processors of the 5th and 6th generations, which were used in early mobile phones and are now rarely used.

* x86: Intel 32-bit processor, which is now rarely used in mobile phones.

* x86_64: Intel 64-bit processor, which is now rarely used in mobile phones.

To ensure our application can run on devices with different CPU platforms, we may include a copy of the so file for each platform in the project, which results in multiple copies of the so file and a significantly larger size.&#x20;

Actually, the only two platforms we need to consider are armeabi-v7a and arm64-v8a, and there is no need to include the corresponding types of so libraries for compatibility for the others.Moreover, the arm64-v8a platform, which refers to 64-bit mobile phones, supports both 32-bit and 64-bit applications. However, armeabi-v7a, that is, 32-bit mobile phones, cannot run 64-bit applications. Therefore, to further reduce the package size, we can uniformly include only 32-bit so files in the project. Since 64-bit mobile phones can run 32-bit so files in a compatible manner, the program can run normally, but the performance will be slightly worse. We can then download the 64-bit so files on 64-bit devices through a dynamic solution.

However, with the increasing popularity of 64-bit machines, 32-bit machines have become fewer and fewer. So, it won't be long before we may no longer have to worry about the large size caused by including multiple so files to be compatible with different platforms. Before the widespread adoption of 64-bit machines, we can also count the proportion of 32-bit machines among the devices using our application. If it is very small, we can simply stop supporting 32-bit machines and uniformly use 64-bit machines instead.&#x20;

## 9.3.3 Delete Symbol Information

The symbol table has been discussed many times. It mainly stores symbol data for functions or variables. After removing the symbol table, the size of the so file can be significantly reduced, usually by more than 50%. When building an apk or aar, Android will automatically generate a symbol table-free so file for us. If our project has Native code, after compilation, we can see that the compiled product file intermediates contains a merged_native_libs file, which stores the so files without symbol tables removed, and a stripred_native_libs file, which stores the so files with symbol tables removed. For the release package, we need to ensure that the so files with symbol tables removed are used.&#x20;

When compiling C or C++ code in a project into a shared object (so), although Android will automatically remove symbols for us, if these C or C++ code introduce methods, functions, or variables from other static so files, their symbols will be automatically introduced, and Android will not remove these symbols during the compilation process. However, we can remove the symbols introduced by using these static so files by adding -Wl, --exclude-libs, ALL to the compilation options.&#x20;

# 9.4 Compress dex files

In terms of dex compression, Facebook is an app that does it relatively well. Download and extract the Facebook APK package, as shown in the figure, you can see that it has only one dex file. How is this achieved?&#x20;

![Figure 9-14 APK installation package with only one dex file](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter9_img_13.png)

In fact, except for the first dex, which remains unchanged, all other dex files have been compressed, named secondary-program-dex-jars, and placed in the assets directory, as shown in Figure 9-15&#x20;

![Figure 9-15 Other dex files are placed in secondary-program-dex-jars files](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter9_img_14.png)

Here, I will briefly introduce the principle of this solution: During the packaging process, first, through dex sorting, the class files required for startup and initial use are placed in the first dex file and retained. Then, all the remaining dex files are compressed using 7z or other compression algorithms and placed in the assets directory. Facebook uses its self-developed zstd algorithm, which has a higher compression ratio than 7z and is also open source. After the application starts, we extract the dex files from the assets directory as early as possible, and take over the system's class loader (ClassLoader) to load classes from the extracted dex files.&#x20;

Let's take a look at the key processes of this solution.

1\) First, we need to sort the dex files and compress the non-primary dex files. This step can still be performed using redex provided by Facebook. After sorting the dex files, we can compress the remaining dex files using the open-source zstd tool, place them in the assets directory, and then sign them. We can also incorporate this process into a Gradle task to avoid repackaging.The implementation of dex rearrangement can refer to the open source code of redex [InterDexPass.cpp](https://github.com/facebook/redex/blob/089e7925c88a73485098b2efb9aed5117814a691/opt/interdex/InterDexPass.cpp) file, then write a gradle script and place the script in the stage after dex generation, such as mergeReleaseAssets or packageRelease.&#x20;

2\) Next, we need to compress all dex files except the first one and place them in the assets directory. We can still handle this through a custom Gradle task during the previously mentioned mergeReleaseAssets or packageRelease phases. For better compression efficiency, we need to introduce the zstd open-source library into the Gradle dependencies. It is recommended to directly use the open-source version that has already been adapted to Android, such as [luben](https://github.com/luben/zstd-jni) open-source library, etc., which can be directly used out of the box.&#x20;

```c++
implementation "com.github.luben:zstd-jni:1.4.5-6@aar"
```

The code for implementing the compression process using luben is as follows:&#x20;

```c++
public static void doCompress(File dexPath, File assetPath) {
    File[] seconddexs = dexPath.listFiles(new FilenameFilter() {
        @Override
        public boolean accept(File file, String s) {
            // Traverse files and handle dex files other than the first one
            return s.endsWith(".dex") && (s.indexOf("classes.dex") == -1);
        }
    });

    try {
        for (File f : seconddexs) {
            FileInputStream input = new FileInputStream(f);
            // Compress into a file with .zst suffix and place it under the asset directory
            FileOutputStream outputStream = 
                    new FileOutputStream(assetPath.getAbsolutePath() +
                    File.separator +
                    f.getName().substring(0, filename.indexOf('.')) + ".zst");
  
            // Call the zstd API for compression
            ZstdCompressorOutputStream output = 
                    new ZstdCompressorOutputStream(outputStream, 19);
            IOUtils.copy(input, output);

            output.flush(); 
            output.close();
            outputStream.close();
            input.close();
        }
    } catch (FileNotFoundException e) {
        e.printStackTrace();
    } catch (IOException e) {
        e.printStackTrace();
    }

    // Delete the original classes2.dex ~ classesN.dex
    Arrays.asList(seconddexs).stream().forEach(File::delete);    
}
```

3\) The work on the compilation process has been completed, and what follows is the process that needs to be done during the application startup. When the application starts, we need to decompress the dex files that were previously compressed and placed in the assets directory. Neither the decompression operation nor how to use the decompressed dex files is the difficult part of the solution. The difficult part of this solution lies in the fact that the dex files we decompressed have not undergone the dex2oat operation, so when we directly use the class files inside, the system will block the application to perform the dex2oat operation.Since this process takes a relatively long time, it is unacceptable in terms of user experience. So how does Facebook solve this problem? Facebook creates a fake odex file for the dex file. This odex file has the same format as the one generated by the dex2oat process, but it only contains the dex bytecode, and the rest of the content is empty.The purpose of doing this is to deceive the dex2oat process into thinking that the odex file has already been generated, so it will no longer execute this step. Facebook refers to this technical solution as oatmeal. The timing of oatmeal generating odex needs to be earlier than that of the system dex2oat, and this process needs to be very fast, so as to eliminate the negative impact brought about by the dex compression solution.&#x20;

The process of generating an odex file is relatively complex, requiring a very good understanding of the structure of the odex file, and then generating a similar odex file according to the logic in dex2oat. Moreover, even if the verification of dex2oat is bypassed, the decompression process will always have a certain impact on performance. If there are no strict requirements for package size, I does not recommend using this solution. Even Facebook has turned off this optimization in newer versions. Therefore, this article only introduces the principle and process of this solution, hoping to bring some new inspiration and ideas to readers, without delving into the detailed implementation. Readers interested can read the source code of [oatmeal](https://github.com/facebook/redex/tree/main/tools/oatmeal)open-sourced by Facebook.&#x20;

# 9.5 Compress so library

Having understood the compression of dex files, let's now take a look at the compression of so libraries. The following introduces two so compression schemes, one provided by the official and the other a custom compression scheme.&#x20;

## 9.5.1 Official Solution Compresses so

We can configure whether to enable compression of so files through the extractNativeLibs attribute in the AndroidManifest.xml file.&#x20;

```xml
<application android:extractNativeLibs="true"></application>
```

We introduced a large so file into the sample program, then packaged it with extractNativeLibs enabled and disabled respectively. As shown in the figure, the package size was reduced by half after enabling so compression.&#x20;

![Figure 9-16 Comparison of package size with SO compression turned on and off](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter9_img_15.png)

It should be noted that when minSdkVersion is less than 23 or Android Gradle plugin is less than 3.6, extractNativeLibs defaults to true; otherwise, it defaults to false. Although enabling so compression significantly reduces the package size, especially for applications with many so files, the installation time will increase when installing the APK because the so will be decompressed during the installation process. However, compared to the package size benefits obtained, this increase in installation time is acceptable.&#x20;

## 9.5.2 Customized Solution Compresses so

When extractNativeLibs is enabled, compression will be performed using zip. However, we also know that the compression ratio of zip is actually not high, and the compression algorithms of 7z or zstd have much higher compression ratios than zip. Therefore, if we need to further reduce the package size, we can customize the compression scheme for so files and use 7zip or zstd to compress them. The entire process mainly consists of 2 steps.&#x20;

1. Find a suitable timing during the APK build process, such as after the so processing is completed, and compress the so through a Gradle task.&#x20;

2. Unzip the so file at an appropriate time during app startup, such as when the CPU is idle, when the user is in the background, or when the user uses this so file.

**1) Compress the so file during packaging**

Let's first look at the first step. During the APK build process, if we want to compress the so files, we need to wait until all so-related tasks are completed before proceeding. Therefore, we can compress the so files before the packageRelease stage, which is the stage of generating the APK. The process implementation is as follows:

```java
project.afterEvaluate {
    project.plugins.withId("com.android.application") { AppPlugin p ->
        p.getVariantManager().getVariantScopes().each { VariantScope scope ->

            // 1. Find the package task
            def packageTask = 
                    project.tasks.findByName("package${scope.fullVariantName.capitalize()}")
            
            if(packageTask != null){
                // Execute SO compression before the packageTask
                packageTask.doFirst {
                 // Get SO files in the jni directory
                 FileCollection soFiles = scope
                        .getTransformManager()
                        .getPipelineOutputAsFileCollection(StreamFilter.NATIVE_LIBS)
                 // Get assets file to store the compressed SO files
                 File assetsFile = scope
                        .getArtifacts()
                        .getFinalArtifactFiles(InternalArtifactType.MERGED_ASSETS)
                        .getFiles().iterator().next();

                   //2. Compress SO files
                   doSoCompress(soFiles, assetsFile)                
                }
            }                   
        }
    }
}
```

Next, let's take a look at the operation of compressing the so file inside deSoCompress. The code here is actually the same as that in the dex compression in the previous chapter.

````c++
```java
private void doSoCompress(Set<File> soFiles, File assetsFile){
       // Traverse the SO files in the jni directory, compress and delete them
     try {
        for (File f : soFiles) {
            FileInputStream input = new FileInputStream(f);
            // Compress into a file with .so suffix and place it under the asset directory
            FileOutputStream outputStream = new FileOutputStream(assetsFile.getAbsolutePath() +
                    File.separator + f.getName().substring(0, filename.indexOf('.'))+".so");
  
            // Call the zstd API for compression
            ZstdOutputStream output = new ZstdOutputStream(outputStream, 19);
            IOUtils.copy(input, output);

            output.flush(); 
            output.close();
            outputStream.close();
            input.close();
        }
    } catch (FileNotFoundException e) {
        e.printStackTrace();
    } catch (IOException e) {
        e.printStackTrace();
    }

    // Delete the original SO files
    Arrays.asList(soFiles).stream().forEach(File::delete); 
}
```
````

By this point, the process of compressing so files using zstd during the packaging process is complete.&#x20;

2\) Unzip and use the so file

After compressing the so file, the next step is to figure out how to decompress it. We can perform batch decompression of the so file either during startup or when the CPU is idle, or we can decompress it when the so file is actually used. Here, we will focus on the decompression operation when the so file is used. If we can perfectly handle the decompression operation at this timing, the decompression operations at other timings will be much easier.&#x20;

When using a so in an application, you can call System.loadLibrary(String libName) to load it based on the name of the so, or call System.load(String pathName) to load it by passing in the path of the so library file. If this so cannot be found during the loading process, the program will throw an UnsatisfiedLinkError exception.&#x20;

After understanding this basic knowledge point, the solution is quite simple. We load the so file via System.loadLibrary, then catch the UnsatisfiedLinkError exception, and then check if the so file is compressed in the assets directory. If it is compressed, we can decompress it and then load it via System.load.To implement this solution, we first need to intercept System.loadLibrary, hook this method through bytecode manipulation, and the following is the code for intercepting loadLibrary via Lancet.

```c++
@TargetClass(value = "java.lang.System")
@Proxy(value = "loadLibrary", globalProxyClass = true)
public static void loadLibrary(String libName) {
    depressAndLoadSo(libName);
}
```

After intercepting the System.loadLibrary method, we can catch the UnsatisfiedLinkError exception of System.loadLibrary through try-catch. When an exception occurs, we can look for the corresponding so file in the assets directory for decompression, using the decompression interface provided by the zstd library. After decompression is completed, we can use the System.load method, passing in the path of the decompressed so library for loading.&#x20;

```java
void depressAndLoadSo(String libName){
    try{
        System.loadLibrary(libName);
    }catch(UnsatisfiedLinkError e){
        if(soDepress(libName)){
            System.load(context.getExternalFilesDir()+"soPath/"+soName+".so")
        }
    }
}

boolean soDepress(String soName){
    try {
        File outPutFile = new File(context.getExternalFilesDir()+"soPath/"+soName+".so");
        if(outPutFile.exists()){
            outPutFile.createNewFile();
        }
        ZstdInputStream input = new ZstdInputStream(context.getAssets().open(soName+".so"));
        FileOutputStream output = context.openFileOutput(outPutFile, Context.MODE_PRIVATE);
        IOUtils.copy(input, output);
        output.flush();
        output.close();
        input.close();
    } catch (IOException e) {
        outPutFile.delete();
        return false;
    }
    return true;

}
```

Based on the above process, the entire solution is not complicated, so it is relatively easy to implement. However, when used online, considering factors such as performance and stability, we need an allowlist mechanism. For example, the so libraries related to startup and the first screen need to be configured in the allowlist without compression, because decompressing so during startup will block the application and cause slower startup.

# 9.6 Dynamically Loading Resource Files

In Chinese Mainland, Google Play Store is unavailable, so the Dynamic Feature technology cannot be used. Therefore, major companies all use plug-in technologies similar to Dynamic Feature to achieve capabilities such as optimizing package size and dynamically updating code. Due to the restrictions of Google Play Store, the installation packages uploaded through Google Play Store cannot use these technologies. However, I will still introduce the technical principles of plug-in technology here, so that readers may gain an in-depth understanding of the underlying loading principles of Android in terms of resources and code.&#x20;

Pluginization technology can help us dynamically load APKs to achieve dynamic expansion of program functionality. Through pluginization technology, we can significantly reduce the package size. Dynamically loading APKs requires solving the problems of dynamically loading resources, class files, so libraries, and the four major components inside the APK. Let's first look at the first thing, dynamically loading the resources inside the APK.

When we load resource files such as strings and images under the res directory in the code, we need to know the resource id and call the "context.getResources().getXXX(int id)" method to obtain the corresponding resource. The underlying implementation of this method is that AssetsManager (resource manager) searches for the real resource in the resources.arsc file based on the id. Although we understand the general process of resource loading, to achieve dynamic resource loading through plug-inization, we still need to further delve into the code to understand the implementation details of resource loading.

## 9.6.1 Resource Loading Principle

Here, we take the getResources().getText method to obtain a string resource as an example to explain the principle of resource loading. This method will call the  getResourceText method of the AssetManager  object, and ultimately call the nativeGetResourceValue native method. The code is as follows.&#x20;

```java
public CharSequence getText(@StringRes int id) throws NotFoundException {
    CharSequence res = mResourcesImpl.getAssets().getResourceText(id);
    if (res != null) {
        return res;
    }
}

CharSequence getResourceText(@StringRes int resId) {
    synchronized (this) {
        final TypedValue outValue = mValue;
        if (getResourceValue(resId, 0, outValue, true)) {
            return outValue.coerceToString();
        }
        return null;
    }
}

boolean getResourceValue(int resId, int densityDpi,TypedValue outValue,
        boolean resolveRefs) {
    synchronized (this) {
        ensureValidLocked();
        final int cookie = nativeGetResourceValue(
                mObject, resId, (short) densityDpi, outValue, resolveRefs);
        if (cookie <= 0) {
            return false;
        }
        ……      
        return true;
    }
}
```

The nativeGetResourceValue function returns the result in  TypedValue  structure. We looked at the implementation of this function in [AssetManager.cpp](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_util_AssetManager.cpp) and found that it calls the GetResource method in the [AssetManager2.cpp](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/androidfw/AssetManager2.cpp) object, and the key step in this method is to find the data through the FindEntry method. The code implementation of the process is as follows.&#x20;

```c++
static jint NativeGetResourceValue(JNIEnv* env, jclass /*clazz*/, jlong ptr, jint resid,
                                   jshort density, jobject typed_value,
                                   jboolean resolve_references) {
  ScopedLock<AssetManager2> assetmanager(AssetManagerFromLong(ptr));
  auto value = assetmanager->GetResource(static_cast<uint32_t>(resid), false ,
                                         static_cast<uint16_t>(density));
  ……
  return CopyValue(env, *value, typed_value);
}

base::expected<AssetManager2::SelectedValue, NullOrIOError> AssetManager2::GetResource(
    uint32_t resid, bool may_be_bag, uint16_t density_override) const {
    //find resource
  auto result = FindEntry(resid, density_override, false, false);
  ……
  return SelectedValue(value.dataType, value.data, result->cookie, result->type_flags,
                       resid, result->config);
}
```

The FindEntry method will then search for the actual resource data in the resources.arsc file. The simplified process code for this method is as follows.&#x20;

```c++
base::expected<FindEntryResult, NullOrIOError> AssetManager2::FindEntry(
    uint32_t resid, uint16_t density_override, bool stop_at_first_match,
    bool ignore_configuration) const {
  ……

  // Get package id
  const uint32_t package_id = get_package_id(resid);
  // Get type id
  const uint8_t type_idx = get_type_id(resid) - 1;
  // Get entry id
  const uint16_t entry_idx = get_entry_id(resid);
  uint8_t package_idx = package_ids_[package_id];
 

  // Get PackageGroup
  const PackageGroup& package_group = package_groups_[package_idx];
  // Find data within the PackageGroup
  auto result = FindEntryInternal(package_group, type_idx, entry_idx, *desired_config,
                                  stop_at_first_match, ignore_configuration);
  ……

  return result;
}

base::expected<FindEntryResult, NullOrIOError> AssetManager2::FindEntryInternal(……)  {
  ……

  const size_t package_count = package_group.packages_.size();
  for (size_t pi = 0; pi < package_count; pi++) {
    const ConfiguredPackage& loaded_package_impl = package_group.packages_[pi];
    // Get LoadedPackage
    const LoadedPackage* loaded_package = loaded_package_impl.loaded_package_;
    const ApkAssetsCookie cookie = package_group.cookies_[pi];
    ……
    // Get the corresponding data segment by type id
    const TypeSpec* type_spec = loaded_package->GetTypeSpecByTypeIndex(type_idx);
    const size_t type_entry_count = (use_filtered) ? filtered_group.type_entries.size()
                                                   : type_spec->type_entries.size();
    for (size_t i = 0; i < type_entry_count; i++) {
      const TypeSpec::TypeEntry* type_entry = (use_filtered) ? filtered_group.type_entries[i]
                                                         : &type_spec->type_entries[i];
      // Further lookup
       ……    
    }
  }
```

The main processes in this method are explained as follows:&#x20;

1. Retrieve the package_id, type_id, and package_id of the resource based on the resource id, which are used to locate the specific data content in resources.arsc.

2. According to the id of the resource, call the FindEntryInternal method, which will packages_ collection, find the correct LoadedPackage, and then search for the real data from it.LoadedPackage is actually the corresponding data structure after the system parses the resources.arsc file,

From the above code process, we know that the resources.arsc file will be parsed and placed in the LoadedPackage data structure. If we want to dynamically load the resources of an APK, we need to ensure that the resources.arsc of the plugin APK is parsed and placed in the LoadedPackage data structure. So when is the resources.arsc parsed and encapsulated into LoadedPackage data?Actually, when the AssetsManager is created, the parsing of the application's resources.arsc file will be executed. Next, let's take a look at the creation process of the AssetsManager.

AssetsManager is created when the application cold starts and executes the performLaunchActivity method. During this method, the createActivityContext method is called to create the Context. While creating the Context, the createBaseTokenResources method is also used to create Resources and AssetManager. The process code is as follows.

```c++
static ContextImpl createActivityContext(ActivityThread mainThread,
        LoadedApk packageInfo, ActivityInfo activityInfo, IBinder activityToken, 
        int displayId,
        Configuration overrideConfiguration) {
  
    String[] splitDirs = packageInfo.getSplitResDirs();
    ClassLoader classLoader = packageInfo.getClassLoader();

   ……

    // create context
    ContextImpl context = new ContextImpl(……);
    context.mContextType = CONTEXT_TYPE_ACTIVITY;
    context.mIsConfigurationBasedContext = true;

   ……

    final ResourcesManager resourcesManager = ResourcesManager.getInstance();

    // create Resources 
    context.setResources(resourcesManager.createBaseTokenResources(……);
    return context;
}
```

We then follow the implementation of the createBaseTokenResources method. Tracing the internal call chain all the way down, it will ultimately call the createResourcesImpl method to create the AssetManager. The process code is as follows.&#x20;

```c++
public @Nullable Resources createBaseTokenResources(……) {
    try {
        final ResourcesKey key = new ResourcesKey(
                resDir,
                splitResDirs,
                combinedOverlayPaths(legacyOverlayDirs, overlayPaths),
                libDirs,
                displayId,
                overrideConfig,
                compatInfo,
                loaders == null ? null : loaders.toArray(new ResourcesLoader[0]));
        classLoader = classLoader != null ? classLoader : ClassLoader.getSystemClassLoader();

        ……

        
        return createResourcesForActivity(token, key,
                Configuration.EMPTY,  null,
                classLoader, null);
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_RESOURCES);
    }
}
    
private Resources createResourcesForActivity(……) {
    synchronized (mLock) {
        ResourcesImpl resourcesImpl = findOrCreateResourcesImplForKeyLocked(key, apkSupplier);
        if (resourcesImpl == null) {
            return null;
        }

        return createResourcesForActivityLocked(activityToken, initialOverrideConfig,
                overrideDisplayId, classLoader, resourcesImpl, key.mCompatInfo);
    }
}

private @Nullable ResourcesImpl findOrCreateResourcesImplForKeyLocked(
        @NonNull ResourcesKey key, @Nullable ApkAssetsSupplier apkSupplier) {
    ResourcesImpl impl = findResourcesImplForKeyLocked(key);
    if (impl == null) {
        impl = createResourcesImpl(key, apkSupplier);
        if (impl != null) {
            mResourceImpls.put(key, new WeakReference<>(impl));
        }
    }
    return impl;
}
```

Next, let's look at the implementation process of the createResourcesImpl function. The code is as follows.

```c++
 private @Nullable ResourcesImpl createResourcesImpl(@NonNull ResourcesKey key) {
    final DisplayAdjustments daj = new DisplayAdjustments(key.mOverrideConfiguration);
    daj.setCompatibilityInfo(key.mCompatInfo);

    final AssetManager assets = createAssetManager(key);
    ...
    return impl;
}
    
private @Nullable AssetManager createAssetManager(@NonNull final ResourcesKey key,
        @Nullable ApkAssetsSupplier apkSupplier) {
    final AssetManager.Builder builder = new AssetManager.Builder();

    final ArrayList<ApkKey> apkKeys = extractApkKeys(key);
    for (int i = 0, n = apkKeys.size(); i < n; i++) {
        final ApkKey apkKey = apkKeys.get(i);
        try {
            // 创建 ApkAssets
            builder.addApkAssets(
                    (apkSupplier != null) ? 
                    apkSupplier.load(apkKey) : loadApkAssets(apkKey));
        } catch (IOException e) {
            ……
        }
    }

    if (key.mLoaders != null) {
        for (final ResourcesLoader loader : key.mLoaders) {
            builder.addLoader(loader);
        }
    }
    // 通过建造者模式创建 AssetManager 
    return builder.build();
}
```

As can be seen from the above process, AssetManager will be created at this time, and the key member object of AssetManager is [ApkAssets](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/content/res/ApkAssets.java;l=46?q=ApkAssets), which is the class that actually stores the resources.arsc resources. During the process, the loadApkAssets function will be called, and ApkAssets will be created through ResourcesKey, which is the resource path, to load and parse the resources. loadApkAssets function's process code is as follows.&#x20;

```c++
private @NonNull ApkAssets loadApkAssets(@NonNull final ApkKey key) throws IOException {
    ApkAssets apkAssets;

    ……
    
    //Create ApkAssets，and load resource
    apkAssets = ApkAssets.loadFromPath(key.path, flags);

    synchronized (mLock) {
        mCachedApkAssets.put(key, new WeakReference<>(apkAssets));
    }

    return apkAssets;
}
    
public static @NonNull ApkAssets loadFromPath(String path, int flags)
    throws IOException {
    return new ApkAssets(FORMAT_APK, path, flags, null);
}
    
private ApkAssets(@FormatType int format, @NonNull String path, int flags,
        @Nullable AssetsProvider assets) throws IOException {
    Objects.requireNonNull(path, "path");
    mFlags = flags;
    mNativePtr = nativeLoad(format, path, flags, assets);
    mStringBlock = new StringBlock(nativeGetStringBlock(mNativePtr), true);
    mAssets = assets;
}
```

As can be seen from the above process, the loadApkAssets function calls the ApkAssets.loadFromPath method to create ApkAssets, and the constructor of ApkAssets will execute nativeLoad to load and parse resources. The implementation of this method is located in [ApkAssets.cpp](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_content_res_ApkAssets.cpp), and the code is as follows. The NativeLoad method can load resources of types such as APK, Overlay, ARSC, and directories.When the application starts, what is loaded here is of type FORMAT_DIRECTORY, that is, the resource files under the directory,

```c++
static jlong NativeLoad(JNIEnv* env, jclass /*clazz*/, const format_type_t format,
                        jstring java_path, const jint property_flags, jobject assets_provider) {
  ScopedUtfChars path(env, java_path);
  if (path.c_str() == nullptr) {
    return 0;
  }

  auto loader_assets = LoaderAssetsProvider::Create(env, assets_provider);
  std::unique_ptr<ApkAssets> apk_assets;
  switch (format) {
    case FORMAT_APK: {
        auto assets = MultiAssetsProvider::Create(std::move(loader_assets),
                                                  ZipAssetsProvider::Create(path.c_str(),
                                                                            property_flags));
        apk_assets = ApkAssets::Load(std::move(assets), property_flags);
        break;
    }
    case FORMAT_IDMAP:
      apk_assets = ApkAssets::LoadOverlay(path.c_str(), property_flags);
      break;
    case FORMAT_ARSC:
      apk_assets = ApkAssets::LoadTable(AssetsProvider::CreateAssetFromFile(path.c_str()),
                                        std::move(loader_assets),
                                        property_flags);
      break;
    case FORMAT_DIRECTORY: {
      auto assets = MultiAssetsProvider::Create(std::move(loader_assets),
                                                DirectoryAssetsProvider::Create(path.c_str()));
      apk_assets = ApkAssets::Load(std::move(assets), property_flags);
      break;
    }
    default:
      const std::string error_msg = base::StringPrintf("Unsupported format type %d", format);
      jniThrowException(env, "java/lang/IllegalArgumentException", error_msg.c_str());
      return 0;
  }

  return CreateGuardedApkAssets(std::move(apk_assets));
}
```

Continuing to look at the final implementation function LoadImpl of ApkAssets::Load, the code is as follows. It can be found that it calls LoadedArsc::Load to load and parse the resources.arsc resource file under the directory.

```java
std::unique_ptr<ApkAssets> ApkAssets::LoadImpl(std::unique_ptr<Asset> resources_asset,
                                               std::unique_ptr<AssetsProvider> assets,
                                               package_property_t property_flags,
                                               std::unique_ptr<Asset> idmap_asset,
                                               std::unique_ptr<LoadedIdmap> loaded_idmap) {
  if (assets == nullptr ) {
    return {};
  }

  std::unique_ptr<LoadedArsc> loaded_arsc;
  if (resources_asset != nullptr) {
    const auto data = resources_asset->getIncFsBuffer(true);
    const size_t length = resources_asset->getLength();

    loaded_arsc = LoadedArsc::Load(data, length, loaded_idmap.get(), property_flags);
  } else {
    loaded_arsc = LoadedArsc::CreateEmpty();
  }

  return std::unique_ptr<ApkAssets>(new ApkAssets(std::move(resources_asset),
                                                  std::move(loaded_arsc), std::move(assets),
                                                  property_flags, std::move(idmap_asset),
                                                  std::move(loaded_idmap)));
}
```

In the LoadImpl method, [the LoadedArsc::Load](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/androidfw/LoadedArsc.cpp) function parses the resources.arsc under the path into  LoadedPackage  structure and stores it in  packages_  collection container. At this point, the entire resource loading process is complete. Understanding the resource file loading process, we can further learn how to dynamically load resource files.&#x20;

## 9.6.2 Dynamic Loading Resource Solution

In the previous principle analysis of resource loading, we know that when searching for resources, the FindEntryInternal method will iterate through packages_ container, and then find and obtain the real resource data through the LoadedPackage stored in packages_ container. During the startup process, AssetsManager will be created, and the most critical part of AssetsManager is to create ApkAssets. ApkAssets will parse the resources.arsc file into LoadedPackage and store it in packages_ container. Now that we know this process, we have our solution. Can we also create an ApkAssets for our plugin APK and load and parse the plugin's resource files into this ApkAssets? The answer is yes, and this is the implementation principle of the dynamic resource loading solution.&#x20;

This solution can be implemented through the addAssetPath method of AssetManager. From the following code, it can be seen that the implementation of the addAssetPath method calls the ApkAssets.loadFromPath method, which was also introduced earlier. This method creates ApkAssets and loads and parses the resource.arsc file at the corresponding path.&#x20;

```c++
@Deprecated
@UnsupportedAppUsage
public int addAssetPath(String path) {
    return addAssetPathInternal(path, false /*overlay*/, false /*appAsLib*/);
}

private int addAssetPathInternal(String path, boolean overlay, boolean appAsLib) {
    Objects.requireNonNull(path, "path");
    synchronized (this) {
        ensureOpenLocked();
        ……

        final ApkAssets assets;
        try {
                ……
                //Create ApkAssets，and load resource.arsc
                assets = ApkAssets.loadFromPath(path,
                        appAsLib ? ApkAssets.PROPERTY_DYNAMIC : 0);              
        } catch (IOException e) {
            return 0;
        }

        mApkAssets = Arrays.copyOf(mApkAssets, count + 1);
        mApkAssets[count] = assets;
        nativeSetApkAssets(mObject, mApkAssets, true);
        invalidateCachesLocked(-1);
        return count + 1;
    }
}
```

addAssetPath is a hidden method that cannot be directly used in the application, but we can call it through reflection. Once we know how to load the resources of the plugin APK, we can continue to complete the solution. At this point, we will face two choices:

1. Load the resources of the plugin APK into the AssetManager of the host application

2. Create an independent Resources and AssetManager to load the resources of the plugin APK

Both of these two solutions have their own advantages and disadvantages. For the first solution, if the resources of the plugin APK are loaded into the AssetManager of the main application, resource IDs may conflict. Therefore, when generating the R.java file during Gradle build, we need to regenerate resource IDs that do not conflict with those of the host. However, if we have multiple plugins, it is difficult to ensure non-conflict.However, one advantage of this solution is that it will be much easier for us to obtain resources in the plugin, which can be directly retrieved through the "context.getResources().getXXX()" method.

In the second scenario, when the code in the plugin uses resources, it can no longer directly obtain resources through "getContext().getResources().getXXX()", but instead must obtain resources through the "pluginResource().getXXX()" method. However, I still recommends using this approach, as it can isolate the plugin resources from the host, enabling better management and control after isolation, without the need to worry about resource ID conflicts. The code implementation is as follows.&#x20;

```c++
public Resources createPluginResources(String pluginPath){
    //1. Create AssetManager Instance
    AssetManager assetManager = AssetManager.class.newInstance();  

    //2. Set resource loading path via reflection
    Method method = AssetManager.class.getMethod("addAssetPath", String.class);
    method.invoke(assetManager, pluginPath);  
    
    //3. Create a new Resource
    Resources pluginResource = new Resources(assetManager, 
            mContext.getResources().getDisplayMetrics(),
            mContext.getResources().getConfiguration());   
    
    return pluginResource;
}

```

Previously, a long passage was spent introducing the principle of resource loading, but ultimately the solution for dynamic resource loading is not complicated. It is precisely because of our in-depth understanding of the resource loading process that we can confirm that the final solution is feasible. For any technology, we should not only know how to do it, but also understand why it is done this way. Only in this way can we go further on the path of technological growth.

# 9.7 Dynamically Loading Class Files

Having mastered the dynamic loading of resource files, we will now proceed to understand the dynamic loading of class files.&#x20;

## 9.7.1 Class Loading Principle

To use a class in project code, we first need to create this class. We have two ways to create class objects: one is to create an object using the new keyword in the code. When the ART virtual machine executes this code, it will automatically help us find and create this object. This method is also called implicit loading; the second way is to create an object through reflection using Class.forName or ClassLoader.loadClass in the code. This method is called explicit loading.

### 1. Implicitly Loaded Class

Let's first take a look at implicit loading. When the ART interpreterencounters the new keyword during the process of interpreting and executing code, it will use the following logic to find and create the object, where ResolveVerifyAndClinit is the key method for finding this class.

```c++
HANDLER_ATTRIBUTES bool NEW_INSTANCE() {
    ObjPtr<mirror::Object> obj = nullptr;
    /*1. Find and create the class object, where dex::TypeIndex(B()) is the index of the object to be created.
     shadow_frame_ is the current method stack, GetMethod returns the current method*/
    ObjPtr<mirror::Class> c = ResolveVerifyAndClinit(dex::TypeIndex(B()),
                                                     shadow_frame_.GetMethod(),
                                                     Self(),
                                                     false,
                                                     do_access_check);
    if (LIKELY(c != nullptr)) {
    
      // 2. Allocate space for this class in the virtual machine
      gc::AllocatorType allocator_type = 
          Runtime::Current()->GetHeap()->GetCurrentAllocator();
      if (UNLIKELY(c->IsStringClass())) {
        obj = mirror::String::AllocEmptyString(Self(), allocator_type);
      } else {
        obj = AllocObjectFromCode(c, Self(), allocator_type);
      }
    }
    if (UNLIKELY(obj == nullptr)) {
      return false;  
    }
    obj->GetClass()->AssertInitializedOrInitializingInThread(Self());
    SetVRegReference(A(), obj);
    return true;
  }
```

In the above code,  the ResolveVerifyAndClinit  method will perform class lookup and creation, and the code implementation of this method is as follows.&#x20;

```c++
inline ObjPtr<mirror::Class> ResolveVerifyAndClinit(dex::TypeIndex type_idx,
                                                    ArtMethod* referrer,
                                                    Thread* self,
                                                    bool can_run_clinit,
                                                    bool verify_access) {
  ClassLinker* class_linker = Runtime::Current()->GetClassLinker();
  // Lookup and load the class
  ObjPtr<mirror::Class> klass = class_linker->ResolveType(type_idx, referrer);
  ……
  return h_class.Get();
}

inline ObjPtr<mirror::Class> ClassLinker::ResolveType(dex::TypeIndex type_idx,
                                                      ArtMethod* referrer) {
  ……
  resolved_type = DoResolveType(type_idx, referrer);
  ……   
  return resolved_type;
}

template <typename RefType>
ObjPtr<mirror::Class> ClassLinker::DoResolveType(dex::TypeIndex type_idx, RefType referrer) {
  StackHandleScope<2> hs(Thread::Current());
  // Get the DexCache of the calling method
  Handle<mirror::DexCache> dex_cache(hs.NewHandle(referrer->GetDexCache()));
  // Get the ClassLoader of the calling method
  Handle<mirror::ClassLoader> class_loader(hs.NewHandle(referrer->GetClassLoader()));
  return DoResolveType(type_idx, dex_cache, class_loader);
}

ObjPtr<mirror::Class> ClassLinker::DoResolveType(dex::TypeIndex type_idx,
                                                 Handle<mirror::DexCache> dex_cache,
                                                 Handle<mirror::ClassLoader> class_loader) {
  Thread* self = Thread::Current();
  // Get the fully qualified name of the object to be created based on its type index
  const char* descriptor = dex_cache->GetDexFile()->StringByTypeIdx(type_idx);
  // Find the object by its fully qualified name and class loader
  ObjPtr<mirror::Class> resolved = FindClass(self, descriptor, class_loader);
  if (resolved != nullptr) {
    
    dex_cache->SetResolvedType(type_idx, resolved);
  } else {
    ……
  }
  return resolved;
}
```

The  ClassLinker  that appears in the above code logic is a global object specifically designed to handle processes such as class lookup and creation. The entire process is as follows:&#x20;

1. Get the classloader of the calling function, which is the function that will execute new to create an object;

2. Get the fully qualified name (in the format of package name + class name) of the object based on the id of the object to be created;

3. Find the object through the  FindClass  function of ClassLinker.&#x20;

Next, let's look at the logic of the FindClass function in the ClassLinker object. The code logic is as follows.

```c++
ObjPtr<mirror::Class> ClassLinker::FindClass(Thread* self,
                                             const char* descriptor,
                                             Handle<mirror::ClassLoader> class_loader) {

  const size_t hash = ComputeModifiedUtf8Hash(descriptor);
  // 1. Call LookupClass to find the Class. If the class has already been loaded, the corresponding class can be obtained through this method
  ObjPtr<mirror::Class> klass = LookupClass(self, descriptor, hash, class_loader.Get());
  if (klass != nullptr) {
    return EnsureResolved(self, descriptor, klass);
  }

  if (descriptor[0] != '[' && class_loader == nullptr) {
    // Find this class in bootClassLoader
    ClassPathEntry pair = FindInClassPath(descriptor, hash, boot_class_path_);
    if (pair.second != nullptr) {
      return DefineClass(self,
                         descriptor,
                         hash,
                         ScopedNullHandle<mirror::ClassLoader>(),
                         *pair.first,
                         *pair.second);
    } else {
        
      return nullptr;
    }
  }
  ObjPtr<mirror::Class> result_ptr;
  bool descriptor_equals;
  if (descriptor[0] == '[') {
   ……
  } else {
    ScopedObjectAccessUnchecked soa(self);
    // Find class in BaseDexClassLoader
    bool known_hierarchy =
        FindClassInBaseDexClassLoader(self, descriptor, hash, class_loader, &result_ptr);
    if (result_ptr != nullptr) {
      descriptor_equals = true;
    } else if (!self->IsExceptionPending()) {
      // Throw class not found error
      ……
    }
  }
  ……
  
  // Success.
  return result_ptr;
}
```

The above logic mainly does the following.&#x20;

1. Execute LookupClass to find the Class in the class_table of the corresponding ClassLoader, where class_table is a container used to store currently loaded classes.

2. Determine the type of the class to be created. If it starts with "\[", then look up this class in the bootClassLoader. Since the class names in the application project do not start with "\[", they will all go through FindClassInBaseDexClassLoader to look up the object.

Continue with the process and take a look at the process of the FindClassInBaseDexClassLoader method for finding objects. The code is as follows. Through the code process, it can be seen that the class_loader->GetParent method will first be called to start the search from the parent class, which is the commonly known parent delegation mechanism, and finally the FindClassInBaseDexClassLoaderClassPath method will be executed.

```c++
bool ClassLinker::FindClassInBaseDexClassLoader(Thread* self,
                                                const char* descriptor,
                                                size_t hash,
                                                Handle<mirror::ClassLoader> class_loader,
                                                ObjPtr<mirror::Class>* result) {
    ……
  if (IsPathOrDexClassLoader(class_loader) || IsInMemoryDexClassLoader(class_loader)) {
    StackHandleScope<1> hs(self);
    // First, look up in the parent of the class_loader
    Handle<mirror::ClassLoader> h_parent(hs.NewHandle(class_loader->GetParent()));
    ……
    RETURN_IF_UNRECOGNIZED_OR_FOUND_OR_EXCEPTION(
        FindClassInBaseDexClassLoaderClassPath(self, 
                                                descriptor, 
                                                hash, 
                                                class_loader, 
                                                result),*result,self);
    ……
    return true;
  }

  ……
  return false;
}
```

The FindClassInBaseDexClassLoaderClassPath method traverses the dex files in the class_loader and searches for the class based on the class name. The code implementation is as follows.&#x20;

```c++
bool ClassLinker::FindClassInBaseDexClassLoaderClassPath(
    Thread* self,
    const char* descriptor,
    size_t hash,
    Handle<mirror::ClassLoader> class_loader,
    /*out*/ ObjPtr<mirror::Class>* result) {


  const DexFile* dex_file = nullptr;
  const dex::ClassDef* class_def = nullptr;
  ObjPtr<mirror::Class> ret;
  // Traverse the dex files of the current class loader and find the class by its fully qualified name
  auto find_class_def = [&](const DexFile* cp_dex_file) REQUIRES_SHARED(Locks::mutator_lock_) {
    const dex::ClassDef* cp_class_def = OatDexFile::FindClassDef(*cp_dex_file, descriptor, hash);
    if (cp_class_def != nullptr) {
      dex_file = cp_dex_file;
      class_def = cp_class_def;
      return false;  
    }
    return true;  
  };
  VisitClassLoaderDexFiles(self, class_loader, find_class_def);

  if (class_def != nullptr) {
    *result = DefineClass(self, descriptor, hash, class_loader, *dex_file, *class_def);
    if (UNLIKELY(*result == nullptr)) {
      CHECK(self->IsExceptionPending()) << descriptor;
      FilterDexFileCaughtExceptions(self, this);
    } else {
      DCHECK(!self->IsExceptionPending());
    }
  }
  return true;
}
```

By this point, we have a clear understanding of the process of implicit addition and lookup of classes. After finding this class, ClassLinker will perform subsequent tasks such as linking, loading, and initialization.&#x20;

### 2. Show Loading Class

The mechanism for explicitly loading classes and implicitly loading classes is the same, except that the entry points are different. The entry points for explicitly loading classes are Class.forName or ClassLoader.loadClass, which is not much different from the above mechanism, as both call [ClassLinker](https://cs.android.com/android/platform/superproject/+/master:art/runtime/class_linker-inl.h) to find and load classes. We will not repeat the explanation here, and readers can study it on their own.&#x20;

## 9.7.2 Dynamically Loaded Classes

Once you understand the principle of how the ART virtual machine loads classes, it becomes easy to implement dynamic class loading. We still have two options: the first is to insert the dex file of the plugin APK package into the dex list of the host's ClassLoader through reflection technology; the second is to create a DexClassLoader and load the dex file from the plugin APK into it, with all classes in the plugin using this DexClassLoader. Considering decoupling and stability, I recommends using the second approach.The implementation is also relatively simple, and the code is as follows.&#x20;

```java
public DexClassLoader createDexClassLoader(String dexPath, 
        String mNativeLibDir, 
        Context mContext) {
    // Path where the parsed odex will be placed
    File dexOutputDir = mContext.getDir("dex", Context.MODE_PRIVATE);
    dexOutputPath = dexOutputDir.getAbsolutePath();
    // mNativeLibDir is the path where the lib files in the plugin are placed
    DexClassLoader loader = new DexClassLoader(dexPath, 
            dexOutputPath,
            mNativeLibDir, 
            mContext.getClassLoader()); 
    // Here the context is applicationContext, passing the application's classloader as the parent classLoader
    return loader;
}
```

# 9.8 Dynamically Load so Library Files

Next, let's take a look at the loading principle of the so library file. In the project, we have two ways to load the so file. The first is to pass the name of the so library through the System.loadLibrary(String libName) method;The second method is to pass the path of the so library file through the System.load(String pathName) method. This approach only requires passing the path of the so library, which can directly enable us to achieve the ability to dynamically load the so library. However, the method of passing the full path every time when loading the so library is not very convenient in use. Therefore, we can delve into the implementation principle of the first method to see if we can achieve the ability to dynamically load the so library through this concise first method.&#x20;

## 9.8.1 Principle of so Library Loading

The implementation of the System.loadLibrary(libName) method is in Runtime.java, and its process code is as follows.

```c++
private synchronized void loadLibrary0(ClassLoader loader, Class<?> callerClass, String libname) {
    
    String libraryName = libname;    
    if (loader != null && !(loader instanceof BootClassLoader)) {
        String filename = loader.findLibrary(libraryName);
        if (filename == null &&
                (loader.getClass() == PathClassLoader.class ||
                 loader.getClass() == DelegateLastClassLoader.class)) {           
            // Add the lib prefix and .so suffix to libname
            filename = System.mapLibraryName(libraryName);
        }
        if (filename == null) {         
            throw new UnsatisfiedLinkError(loader + " couldn't find \"" +
                                           System.mapLibraryName(libraryName) + "\"");
        }
        // Load the SO file
        String error = nativeLoad(filename, loader);
        if (error != null) {
            throw new UnsatisfiedLinkError(error);
        }
        return;
    }

    ……
}
```

In the above method, the nativeLoad native method will be called, which will ultimately execute the LoadNativeLibrary method under the java_vm_ext.cc file to load the so library. The code is as follows.&#x20;

```c++
JNIEXPORT jstring JNICALL
Runtime_nativeLoad(JNIEnv* env, jclass ignored, jstring javaFilename,
                   jobject javaLoader, jclass caller)
{
    return JVM_NativeLoad(env, javaFilename, javaLoader, caller);
}

JNIEXPORT jstring JVM_NativeLoad(JNIEnv* env,
                                 jstring javaFilename,
                                 jobject javaLoader,
                                 jclass caller) {
  ScopedUtfChars filename(env, javaFilename);
  if (filename.c_str() == nullptr) {
    return nullptr;
  }

  std::string error_msg;
  {
    art::JavaVMExt* vm = art::Runtime::Current()->GetJavaVM();
    //load so library
    bool success = vm->LoadNativeLibrary(env,
                                         filename.c_str(),
                                         javaLoader,
                                         caller,
                                         &error_msg);
    if (success) {
      return nullptr;
    }
  }

  env->ExceptionClear();
  return env->NewStringUTF(error_msg.c_str());
}
```

Let's take a look at the implementation of the LoadNativeLibrary method. Its code flow is very long, and we only need to focus on how to find the so file in the flow. The simplified flow code is as follows.

```c++
bool JavaVMExt::LoadNativeLibrary(JNIEnv* env,
                                  const std::string& path,
                                  jobject class_loader,
                                  jclass caller_class,
                                  std::string* error_msg) {

  // Check from the libraries_ cache container whether this SO has already been loaded
  ……

 
 // Get the path of the SO
  ScopedLocalRef<jstring> library_path(env, GetLibrarySearchPath(env, class_loader));

  // Load the SO based on the path
  void* handle = android::OpenNativeLibrary(
      env,
      runtime_->GetTargetSdkVersion(),
      path_str,
      class_loader,
      (caller_location.empty() ? nullptr : caller_location.c_str()),
      library_path.get(),
      &needs_native_bridge,
      &nativeloader_error_msg);
  ……
  return was_successful;
}

jstring JavaVMExt::GetLibrarySearchPath(JNIEnv* env, jobject class_loader) {
  ……
  return soa.AddLocalReference<jstring>(
      WellKnownClasses::dalvik_system_BaseDexClassLoader_getLdLibraryPath->InvokeVirtual<'L'>(
          soa.Self(), mirror_class_loader));
}
```

As can be seen, here the getLdLibraryPath method in the Java layer's ClassLoader will be called to obtain the path of the so file. Return to the Java layer's  BaseDexClassLoader.java  object to see the implementation of this method, the code is as follows.&#x20;

```c++
public @NonNull String getLdLibraryPath() {
    StringBuilder result = new StringBuilder();
    for (File directory : pathList.getNativeLibraryDirectories()) {
        if (result.length() > 0) {
            result.append(':');
        }
        result.append(directory);
    }

    return result.toString();
}

public List<File> getNativeLibraryDirectories() {
    return nativeLibraryDirectories;
}
```

By now, we know that for the way of loading so files like System.loadLibrary, we only need to put the path of the so file into the nativeLibraryDirectories collection, and it can be used normally.&#x20;

## 9.8.2 Dynamically Loading so Libraries

After understanding the above principle, the solution for dynamically loading the so library becomes obvious, and there are multiple ways to implement it. I lists the ways here.&#x20;

1. Modify the collection data of nativeLibraryDirectories in the array through reflection, and add the so path of the plugin apk to it.

2. Specify the path of the plugin so in the DexClassLoader created by oneself.

3. Load by specifying the address of the specific so path through the System.load method in the plugin

4. Using the method of System.loadLibrary, but replacing it with the method of System.load through bytecode manipulation

Considering decoupling and isolation in the plugin scenario, I suggests using the second approach. The implementation mainly involves setting the so path of the plugin APK as an input parameter when creating the DexClassLoader in the previous dynamic class file loading solution. The code is as follows.&#x20;

```c++
public DexClassLoader createDexClassLoader(String dexPath, 
        String dexOutputPath, 
        String mNativeLibDir, 
        Context mContext) {
    // Path where the parsed odex will be placed
    File dexOutputDir = mContext.getDir("dex", Context.MODE_PRIVATE);
    dexOutputPath = dexOutputDir.getAbsolutePath();
    // mNativeLibDir is the path where the lib files in the plugin are placed
    DexClassLoader loader = new DexClassLoader(dexPath, 
            dexOutputPath, 
            mNativeLibDir, 
            mContext.getClassLoader()); 
    // Here the context is applicationContext, passing the application's classloader as the parent classLoader
    return loader;
}
```

However, for scenarios where plugins are not used, such as simply making the so dynamic or the so compression mentioned earlier, the second approach is not recommended. In this case, the first approach is the best, as decoupling and isolation need to be reduced in such scenarios.

# 9.9 Dynamically Load Four Major Components

We have been able to dynamically load resource files, so library files, and class files in the plugin package, but this does not mean that we can successfully start the four major components in the plugin package. During the startup process of the four major components, ActivityManagerService will perform a series of verifications. If the component to be started is not configured in the Mainfest file of the program, it will not pass the verification. Since the plugin and the program body (also known as the host) are separated, the components in the plugin cannot be pre-configured in the host's Mainfest file. If they are configured, it means losing the ability of dynamic loading. Therefore, to start the components in the plugin normally, we need to find a way to bypass the verification of this AMS.&#x20;

I will take the Activity component as an example to explain how to bypass the AMS check, and currently there are mainly 2 solutions.&#x20;

* The first method is to start a pre-configured Activity in the host (hereinafter collectively referred to as ProxyActivity). After completing the AMS verification and process, Hook is performed at a subsequent process point, and this ProxyActivity is replaced with the Activity to be started in the plugin (hereinafter collectively referred to as PluginActivity) through a bait-and-switch approach.&#x20;

* The second method also first starts a pre-configured Activity in the host, and then in all lifecycle methods of this Activity, retargets the calls to the corresponding lifecycle methods of the Activity to be started in the plugin.&#x20;

Since both of these two solutions require a certain understanding of the Activity startup process, this section will first introduce the general process of Activity startup.

## 9.9.1 Activity Launch Process

The startup process of Activity is very long. We don't need to be familiar with every method call in the process, because even if we spend a lot of effort to get familiar with them, it's easy to forget. Therefore, we only need to be familiar with the key processes and nodes. For better understanding, the process will be divided into two parts for explanation here, namely the application side and the AMS side.&#x20;

### 1. Application Side&#x20;

Let's first take a look at what we've done on the application side.When we start an Activity through the startActivity method of Content in an application, the Instrumentation object will notify the AMS to execute the startActivity method via Binder communication. After the ASM side goes through a series of processes, including the verification of the target Activity, it will return to the application side, where the application side will create the target Activity and execute the lifecycle of the target Activity, etc. The sequence diagram of this process is shown in Figure 9-17.&#x20;

![Figure 9-17 Application Sequence Diagram at Activity Startup Flow](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter9_img_16.png)

Following the process of the Sequence Diagram, let's continue to look at the code logic inside. The code implementation of startActivity is as follows.

```groovy
public void startActivity(Intent intent, @Nullable Bundle options) {
    getAutofillClientController().onStartActivity(intent, mIntent);
    if (options != null) {
        startActivityForResult(intent, -1, options);
    } else {
        startActivityForResult(intent, -1);
    }
}
    
public void startActivityForResult(
        String who, Intent intent, int requestCode, @Nullable Bundle options) {
    Uri referrer = onProvideReferrer();
    if (referrer != null) {
        intent.putExtra(Intent.EXTRA_REFERRER, referrer);
    }
    options = transferSpringboardActivityOptions(options);
    //start activity by Instrumentation 
    Instrumentation.ActivityResult ar =
        mInstrumentation.execStartActivity(
            this, mMainThread.getApplicationThread(), mToken, who,
            intent, requestCode, options);
    if (ar != null) {
        mMainThread.sendActivityResult(
            mToken, who, requestCode,
            ar.getResultCode(), ar.getResultData());
    }
    cancelInputsAndStartExitTransition(options);
}
```

As can be seen from the above code, when starting an Activity, it actually executes the execStartActivity method of the [Instrumentation](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r9:frameworks/base/core/java/android/app/Instrumentation.java) object.Starting from Android 10, the execStartActivity method will be called via the Binder proxy object IActivityTaskManagerSingleton [ActivityTaskManagerService](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/services/core/java/com/android/server/wm/ActivityTaskManagerService.java) 's startActivity method, and the code implementation is as follows&#x20;

```groovy
public ActivityResult execStartActivity(
    Context who, IBinder contextThread, IBinder token, String target,
    Intent intent, int requestCode, Bundle options) {
    ……
    try {
        intent.migrateExtraStreamToClipData(who);
        intent.prepareToLeaveProcess(who);
        //调用 ActivityTaskManagerService 来 startActivity
        int result = ActivityTaskManager.getService().startActivity(whoThread,
                who.getOpPackageName(), who.getAttributionTag(), intent,
                intent.resolveTypeIfNeeded(who.getContentResolver()), token, target,
                requestCode, 0, null, options);
        checkStartActivityResult(result, intent);
    } catch (RemoteException e) {
        throw new RuntimeException("Failure from system", e);
    }
    return null;
}
```

Before Android 10, the startActivity method of ActivityManagerService was directly called through the Binder proxy object IActivityManagerSingleton.In the startActivity method within ActivityManagerService, the startActivity method of ActivityTaskManagerService is still called, so the differences in their processes are also quite similar.&#x20;

```java
public ActivityResult execStartActivity(
        Context who, IBinder contextThread, IBinder token, Activity target,
        Intent intent, int requestCode, Bundle options) {
    ……
    try {
        intent.migrateExtraStreamToClipData();
        intent.prepareToLeaveProcess(who);
        // 调用 ActivityManagerService 来 startActivity
        int result = ActivityManager.getService()
            .startActivity(whoThread, who.getBasePackageName(), intent,
                    intent.resolveTypeIfNeeded(who.getContentResolver()),
                    token, target != null ? target.mEmbeddedID : null,
                    requestCode, 0, null, options);
        checkStartActivityResult(result, intent);
    } catch (RemoteException e) {
        throw new RuntimeException("Failure from system", e);
    }
    return null;
}
```

When the AMS side receives a call from the application side to start an Activity, it will initiate a series of processes, such as permission verification, stack processing, etc. After AMS completes these processes, the process will return to the application side, at which point the application side will continue to handle the subsequent tasks:&#x20;

1\) The inner class Handler named H in the main thread (ActivityThread) on the application side receives the call notification from AMS and processes the EXECUTE_TRANSACTION task logic. In this logic, after being processed by objects such as TransactionExecutor, ClientTransactionItem, etc., it will ultimately execute the handleLaunchActivity method of ActivityThread.

2\) The handleLaunchActivity method calls performLaunchActivity, which is a critical method that creates corresponding Activity, Context, Window, etc., based on the Activity information passed from ASM. After creating these critical objects, the onCreate lifecycle method of the created Activity will be executed.

Next, we will further familiarize ourselves with the above process through code. The EXECUTE_TRANSACTION in the inner class H object of ActivityThread  is the starting point of the process after the application side receives the AMS notification to start an Activity. The code process is as follows, and the key process is to execute the execute method of the TransactionExecutor object&#x20;

```java
class H extends Handler {
    ……
    public void handleMessage(Message msg) {
        switch (msg.what) {
            ……
            case EXECUTE_TRANSACTION:
                final ClientTransaction transaction = (ClientTransaction) msg.obj;
                mTransactionExecutor.execute(transaction);
                break;
            ……
        }
        Object obj = msg.obj;
        if (obj instanceof SomeArgs) {
            ((SomeArgs) obj).recycle();
        }
    }
}
```

Next, look at the execute method of the TransactionExecutor object, which will then execute the executeCallbacks method. The code is shown below.

```java
public void execute(ClientTransaction transaction) {
   ……
   executeCallbacks(transaction);
   ……
}

public void executeCallbacks(ClientTransaction transaction) {
    // Retrieve callbacks from ClientTransaction
    final List<ClientTransactionItem> callbacks = transaction.getCallbacks();
    
    final IBinder token = transaction.getActivityToken();
    ActivityClientRecord r = mTransactionHandler.getActivityClient(token);
    ……
    final int size = callbacks.size();
    for (int i = 0; i < size; ++i) {
        // The actual instance object of ClientTransactionItem here is LaunchActivityItem
        final ClientTransactionItem item = callbacks.get(i);
        ……
        // Call the execute method
        item.execute(mTransactionHandler, token, mPendingActions);
        item.postExecute(mTransactionHandler, token, mPendingActions);
       
        if (postExecutionState != UNDEFINED && r != null) {
            
            final boolean shouldExcludeLastTransition =
                    i == lastCallbackRequestingState && finalState == postExecutionState;
            // Execute subsequent onStart, onResume flow
            cycleToPath(r, postExecutionState, shouldExcludeLastTransition, transaction);
        }
    }
}
```

In the executeCallbacks method, the execute and postExecute methods of ClientTransactionItem in callbacks will be called, where ClientTransactionItem is an abstract class, and its implementation class is LaunchActivityItem object. From the above code, we can see that callbacks are retrieved from the ClientTransaction data, which is passed from the AMS side. We can see from the last method executed on the AMS side realStartActivityLocked that the LaunchActivityItem object instance is created in the clientTransaction.addCallback method.And at the end of the method, execute mService.getLifecycleManager().scheduleTransaction to notify the application side to start the Activity.&#x20;

```java
boolean realStartActivityLocked(ActivityRecord r, WindowProcessController proc,
        boolean andResume, boolean checkConfig) throws RemoteException {
        ……
        clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent),
                System.identityHashCode(r), r.info,
                mergedConfiguration.getGlobalConfiguration(),
                mergedConfiguration.getOverrideConfiguration(), r.compat,
                r.getFilteredReferrer(r.launchedFromPackage), task.voiceInteractor,
                proc.getReportedProcState(), r.getSavedState(), r.getPersistentSavedState(),
                results, newIntents, r.takeOptions(), isTransitionForward,
                proc.createProfilerInfoIfNeeded(), r.assistToken, activityClientController,
                r.shareableActivityToken, r.getLaunchedFromBubble(), fragmentToken));

            ……
            // Notify the H Handler in ActivithThread
            mService.getLifecycleManager().scheduleTransaction(clientTransaction);
            ……

    return true;
}
```

Next, let's look at the execute method and postExecute method of the LaunchActivityItem object. The execute method calls the handleLaunchActivity method of the client, where the client is an instance of ActivityThread. postExecute sets a flag in ActivityThread.

```java
public void execute(ClientTransactionHandler client, IBinder token,
        PendingTransactionActions pendingActions) {
    Trace.traceBegin(TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
    ActivityClientRecord r = new ActivityClientRecord(token, mIntent, mIdent, mInfo,
            mOverrideConfig, mCompatInfo, mReferrer, mVoiceInteractor, mState, mPersistentState,
            mPendingResults, mPendingNewIntents, mActivityOptions, mIsForward, mProfilerInfo,
            client, mAssistToken, mShareableActivityToken, mLaunchedFromBubble,
            mTaskFragmentToken);
    //Call handleLaunchActivity method of ActvityThread
    client.handleLaunchActivity(r, pendingActions, null);
    Trace.traceEnd(TRACE_TAG_ACTIVITY_MANAGER);
}

@Override
public void postExecute(ClientTransactionHandler client, IBinder token,
        PendingTransactionActions pendingActions) {
    client.countLaunchingActivities(-1);
}
```

It should be noted that the code for Android versions below 9 differs from the above process. For Android versions below 9, the LAUNCH_ACTIVITY case condition of the H object will be executed. In this case, the handleLaunchActivity method will be directly executed, so the code process will be much shorter than the above. Since this is also a hook point in the Hook solution, we need to understand the differences between different system versions.&#x20;

```java
private class H extends Handler {
    ……
    public void handleMessage(Message msg) {
        switch (msg.what) {
            case LAUNCH_ACTIVITY: {
                final ActivityClientRecord r = (ActivityClientRecord) msg.obj;

                r.packageInfo = getPackageInfoNoCheck(
                        r.activityInfo.applicationInfo, r.compatInfo);
                handleLaunchActivity(r, null, "LAUNCH_ACTIVITY");
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            } break;
           ……
        }
        Object obj = msg.obj;
        if (obj instanceof SomeArgs) {
            ((SomeArgs) obj).recycle();
        }
    }
}
```

Let's continue to look at the handleLaunchActivity method in ActivityThread. The simplified source code is as follows.

```java
public Activity handleLaunchActivity(ActivityClientRecord r,
        PendingTransactionActions pendingActions, Intent customIntent) {
    ……
    final Activity a = performLaunchActivity(r, customIntent);
    ……
    return a;
}


```

The performLaunchActivity method is called in handleLaunchActivity, and the simplified source code is as follows.&#x20;

```java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    ActivityInfo aInfo = r.activityInfo;  
    //1. Create Context
    ContextImpl appContext = createBaseContextForActivity(r);
    Activity activity = null;
    try {
        java.lang.ClassLoader cl = appContext.getClassLoader();
        // Create Activity
        activity = mInstrumentation.newActivity(
                cl, component.getClassName(), r.intent);
        ……
    } catch (Exception e) {
       
    }

    try {
       ……

        if (activity != null) {
            ……          
            Window window = null;           
            ……
            //2. Initialize Activity, create Window object (PhoneWindow) and associate Activity with Window
            activity.attach(appContext, this, getInstrumentation(), r.token,
                    r.ident, app, r.intent, r.activityInfo, title, r.parent,
                    r.embeddedID, r.lastNonConfigurationInstances, config,
                    r.referrer, r.voiceInteractor, window, r.activityConfigCallback,
                    r.assistToken, r.shareableActivityToken);
            ……          
            r.activity = activity;
            if (r.isPersistable()) {
                //3. Execute the Activity's onCreate callback
                mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
            } else {
                mInstrumentation.callActivityOnCreate(activity, r.state);
            }
           ……
        }


    } catch (SuperNotCalledException e) {
       ……
    } catch (Exception e) {
      ……
    }

    return activity;
}
```

performLaunchActivity is a very critical method, and the main tasks it performs are as follows:

1. Create Context and Activity

2. Call the attach method of Activity to initialize Activity

3. Callback the OnCreate lifecycle function of Activity through the Instrumentation object&#x20;

After the onCreate lifecycle of the Activity is executed, the executeCallbacks method will then call cycleToPath to execute the subsequent lifecycle of the Activity. The logic executed in the cycleToPath method has little to do with the plug-in-related technology, so it will not be covered here. By now, we have understood the startup process on the application side. The process is not complicated, and readers can refer to the Sequence Diagram to read the code process to prevent getting lost in the lengthy code.&#x20;

### 2. AMS Side

Having understood what has been done on the application side, let's now take a look at what has been done on the AMS side. The process on the AMS side is very long, but we don't need to be familiar with every step; we only need to focus on the key processes and paths. Its sequence process is shown in the figure

![Figure 9-18 AMS side timing flow chart](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter9_img_17.png)

In the key steps on the AMS side, the following tasks are mainly performed.&#x20;

1\) First, inspection and verification work will be carried out, such as caller permission check, target Intent information and integrity check, etc.

2\) Create an ActivityRecord object, which is the record of the Activity in the AMS

3\) Handle the logic of Activity launch modes, such as calculating whether there is an available task stack, whether it is allowed to start an activity on a given task or a new task, reusing or creating a stack, etc.

4\) Finally, in the startSpecificActivity method, check whether the process where the target Activity resides exists. If it does not exist, notify to create the target process, and then re-execute this method. If the process already exists, call  realStartActivityLocked  method to notify the application side to proceed with the subsequent process.&#x20;

We start from the entry point on the AMS side, which is the startActivity function of the ActivityTaskManagerService object, and its code is as follows.

```java
public final int startActivity(IApplicationThread caller, String callingPackage,
        String callingFeatureId, Intent intent, String resolvedType, IBinder resultTo,
        String resultWho, int requestCode, int startFlags, ProfilerInfo profilerInfo,
        Bundle bOptions) {
    return startActivityAsUser(……);
}
    
public int startActivityAsUser(……) {
    return startActivityAsUser(……);
}

private int startActivityAsUser(……) {
    ……
    return getActivityStartController().obtainStarter(intent, "startActivityAsUser")
            .setCaller(caller)
            .setCallingPackage(callingPackage)
            .setCallingFeatureId(callingFeatureId)
            .setResolvedType(resolvedType)
            .setResultTo(resultTo)
            .setResultWho(resultWho)
            .setRequestCode(requestCode)
            .setStartFlags(startFlags)
            .setProfilerInfo(profilerInfo)
            .setActivityOptions(opts)
            .setUserId(userId)
            .execute();

}
```

After performing preliminary verification processes such as checking the caller's uid permissions, startActivityAsUser will obtain the ActivityStarter object through the obtainStarter method and execute the execute method of this object. The code is as follows. After further checks and verifications in this method, these verifications include checking whether the target Activity is configured in the Mainfest configuration file. For Activities in plugins, since they are not configured in the host's Mainfest, they cannot pass the verification at this step.After all validations are completed, the executeRequest method will be executed next.&#x20;

```java
int execute() {
    try {
        ……
        /* PackageManager queries all Activities in the system that meet the requirements.
         If there are multiple Activities that meet the criteria, a dialog will pop up for the user to choose.
         If none is found, it means it is not configured in the manifest, and an error will be reported.
        */
        if (mRequest.activityInfo == null) {
            mRequest.resolveActivity(mSupervisor);
        }
        ……  
        res = executeRequest(mRequest);
        ……
}
```

In executeRequest, a series of checks and validations are continued, such as validating the permissions of the target Activity. After the validation is successful, an ActivityRecord will be created for the Activity to be launched. Each launched Activity needs to have a record in AMS, and the ActivityRecord object is the record of this Activity, storing data information such as the callerApp and callingPid of this Activity. The code implementation is as follows.

```java
private int executeRequest(Request request) {
    ……
  
    final ActivityRecord r = new ActivityRecord.Builder(mService)
            .setCaller(callerApp)
            .setLaunchedFromPid(callingPid)
            .setLaunchedFromUid(callingUid)
            .setLaunchedFromPackage(callingPackage)
            .setLaunchedFromFeature(callingFeatureId)
            .setIntent(intent)
            .setResolvedType(resolvedType)
            .setActivityInfo(aInfo)
            .setConfiguration(mService.getGlobalConfiguration())
            .setResultTo(resultRecord)
            .setResultWho(resultWho)
            .setRequestCode(requestCode)
            .setComponentSpecified(request.componentSpecified)
            .setRootVoiceInteraction(voiceSession != null)
            .setActivityOptions(checkedOptions)
            .setSourceRecord(sourceRecord)
            .build();

    mLastStartActivityRecord = r;

    ……

    mLastStartActivityResult = startActivityUnchecked(r, sourceRecord, voiceSession,
            request.voiceInteractor, startFlags, true /* doResume */, checkedOptions,
            inTask, inTaskFragment, restrictedBgActivity, intentGrants);

   

    return mLastStartActivityResult;
}
```

After creating the ActivityRecord, the startActivityUnchecked method will be executed. The code is as follows. This method will in turn execute the startActivityInner method, which mainly contains the logic for setting the launch mode of the Activity.&#x20;

```java
private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        int startFlags, boolean doResume, ActivityOptions options, Task inTask,
        TaskFragment inTaskFragment, boolean restrictedBgActivity,
        NeededUriGrants intentGrants) {
        ……
        result = startActivityInner(r, sourceRecord, voiceSession, voiceInteractor,
                startFlags, doResume, options, inTask, inTaskFragment, restrictedBgActivity,
                intentGrants);
        ……

    return result;
}
    
int startActivityInner(final ActivityRecord r, ActivityRecord sourceRecord,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        int startFlags, boolean doResume, ActivityOptions options, Task inTask,
        TaskFragment inTaskFragment, boolean restrictedBgActivity,
        NeededUriGrants intentGrants) {
    
    ...

    // Get TargetRootTask
    if (mTargetRootTask == null) {
        mTargetRootTask = 
                getOrCreateRootTask(mStartActivity, mLaunchFlags, targetTask, mOptions);
    }

    ...
    
    if (mDoResume) {
        final ActivityRecord topTaskActivity = startedTask.topRunningActivityLocked();
        if (!mTargetRootTask.isTopActivityFocusable() 
            || (topTaskActivity != null 
            && topTaskActivity.isTaskOverlay() 
            && mStartActivity != topTaskActivity)) {
          ……
        } else {
           ……
            /* RootWindowContainer is a window container that holds all windows and tasks in the system.
               When an Activity is started or regains focus, this method ensures that the Activity at the top of the task stack
               that currently has user focus is properly resumed to the active state. */ 
            mRootWindowContainer.resumeFocusedTasksTopActivities(mTargetRootTask, 
                    mStartActivity, 
                    mOptions, 
                    mTransientLaunch);
        }
    }
    mRootWindowContainer.updateUserRootTask(mStartActivity.mUserId, mTargetRootTask);

    // Update the recent tasks list after Activity starts
    mSupervisor.mRecentTasks.add(startedTask);
    mSupervisor.handleNonResizableTaskIfNeeded(startedTask, 
            mPreferredWindowingMode, mPreferredTaskDisplayArea, mTargetRootTask);
    ...

    return START_SUCCESS;
}
```

When the start mode of the Activity to be started is handled, it is another long process. First, call the resumeFocusedTasksTopActivities method of the RootWindowContainer object, then call the resumeTopActivityUncheckedLocked and resumeTopActivityInnerLocked methods of the Task object, and then call the resumeTopActivity method of the TaskFragment object.The methods of these processes are very long, but their main function is to manage the states of other Activities when a new Activity is launched, ensuring that they are correctly paused, resumed, or stopped. I will not elaborate further; we only need to know the general process, and the process code is as follows.&#x20;

```java
boolean resumeFocusedTasksTopActivities(
        Task targetRootTask, ActivityRecord target, ActivityOptions targetOptions,
        boolean deferPause) {
    ……
    result = targetRootTask.resumeTopActivityUncheckedLocked(target, targetOptions,
                deferPause); 
    ……
    return result;
}
boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options,
        boolean deferPause) {
    ……
    someActivityResumed = resumeTopActivityInnerLocked(prev, options, deferPause);
    ……
        
    return someActivityResumed;
}

private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options,
        boolean deferPause) {
    ……

    final boolean[] resumed = new boolean[1];
    final TaskFragment topFragment = topActivity.getTaskFragment();
    // Call the resumeTopActivity method of TaskFragment
    resumed[0] = topFragment.resumeTopActivity(prev, options, deferPause);
    ……
    return resumed[0];
}

 final boolean resumeTopActivity(ActivityRecord prev, ActivityOptions options,
            boolean deferPause) {
   ……
   mTaskSupervisor.startSpecificActivity(next, true, true);
   ……
    return true;
}
```

At the end of the resumeTopActivity method, it calls ActivityTaskSupervisor's startSpecificActivity method. The code for this method is as follows. During the startSpecificActivity process, it will determine whether the process of the target Activity exists. If it does not exist, it will synchronously create the process. After the process creation and startup are completed, the startSpecificActivity method will be re-executed.If the target process exists, call the realStartActivityLocked method.&#x20;

```java
void startSpecificActivity(ActivityRecord r, boolean andResume, boolean checkConfig) {
    final WindowProcessController wpc =
            mService.getProcessController(r.processName, r.info.applicationInfo.uid);

    boolean knownToBeDead = false;
    if (wpc != null && wpc.hasThread()) {
        try {
            // If the process of the target activity exists, execute the realStartActivityLocked method
            realStartActivityLocked(r, wpc, andResume, checkConfig);
            return;
        } catch (RemoteException e) {
          
        }
        ……
    }

    // If the process of the target activity does not exist, start the target process
    mService.startProcessAsync(r, knownToBeDead, isTop,
            isTop ? HostingRecord.HOSTING_TYPE_TOP_ACTIVITY
                    : HostingRecord.HOSTING_TYPE_ACTIVITY);
}

boolean realStartActivityLocked(ActivityRecord r, WindowProcessController proc,
    boolean andResume, boolean checkConfig) throws RemoteException {
    ……
    clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent),
            System.identityHashCode(r), r.info,       
            mergedConfiguration.getGlobalConfiguration(),
            mergedConfiguration.getOverrideConfiguration(), r.compat,
            r.getFilteredReferrer(r.launchedFromPackage), task.voiceInteractor,
            proc.getReportedProcState(), r.getSavedState(), r.getPersistentSavedState(),
            results, newIntents, r.takeOptions(), isTransitionForward,
            proc.createProfilerInfoIfNeeded(), r.assistToken, activityClientController,
            r.shareableActivityToken, r.getLaunchedFromBubble(), fragmentToken));
    ……
    // Notify the application-side MainThread to continue the subsequent flow
    mService.getLifecycleManager().scheduleTransaction(clientTransaction);
    ……

    return true;
}
```

In the realStartActivityLocked method, communication is carried out through the Binder proxy on the application side, and the process is handed over to the application side. By this point, we have gone through all the operations performed on the AMS side. The process inside is very long. If the reader is new to the Activity startup process, there is actually no need to pay too much attention to the code details inside, as it is easy to get lost and unable to find the key points. Just grasp the key processes and paths.&#x20;

## 9.9.2 Initiate Interception

Once we understand the startup process of Activity, we can implement the solution to start the Activity component in the plugin. Here, we assume that we want to start the Activity component named PluginActvity. When PluginActvity is an Activity in the host, the normal startup process is as follows:&#x20;

```java
startActivity(new Intent(this, PluginActvity.class));
```

However, after we move PluginActvity into the plugin, we can no longer directly use the above method to start this Activity, because the object cannot be found in the code. At this time, we can use the plugin's context and start PluginActvity in the plugin based on the package name and class name.&#x20;

```java
Intent intent = new Intent();
intent.setClassName("com.example.test","PluginActvity");
pluginContext.startActivity(intent);
```

Previously, we've already learned how to load classes from a plugin. Therefore, through pluginContext, we can now properly load the PluginActivity class from the plugin, but we're unable to start this Activity properly. The reason is also known: it's because PluginActivity fails to pass the verification of AMS.

Let's first look at how to bypass the verification of AMS through the Hook solution. As can be specified from the previous process explanation, although the verification of Activity is performed on the AMS side, the creation of Activity is carried out in the handleLaunchActivity method of the ActivityThread class on the application side. Understanding this, the solution is easy to understand and mainly consists of 3 steps.

1\) Pre-embed a ProxyActivity in the host's Mainfest configuration file.

2\) Find a timing point before the process enters AMS, and replace the PluginActivity to be launched with the ProxyActivity pre-embedded in the host.

3\) After the AMS side completes the verification of ProxyActivity and related processes and returns to the application side, find another timing point to replace ProxyActivity with PluginActivity in the plugin.

Let's take a detailed look at the implementation of these three steps below.

### 1. Pre-embedded ProxyActivity

Pre-embedding ProxyActivity is very simple. You just need to directly create an empty Activity in the host and configure it in the Manifest file. In mature plug-in frameworks, ProxyActivity is automatically generated through Gradle scripts, which saves a lot of tedious manual operation processes. To simplify the solution process, I will not elaborate on this method here.

### 2. Activity Replacement

Before entering AMS, we replace PluginActivity with ProxyActivity in the host that can start normally. Once it enters the AMS side, replacement is no longer possible, and the program will also experience exceptions. According to the startup process, the best timing for hooking is when the Binder proxy object on the application side calls the startActivity method of AMS.

Starting from Android 10, the Binder proxy object of AMS on the client side is the IActivityTaskManagerSingleton object in IActivityTaskManager; from Android 10 to Android 8, the Binder proxy is the IActivityManagerSingleton of ActivityManager; For Android versions below 8, it is gDefault in ActivityManagerNative. After knowing the method and object to be hooked, we can proceed with the hooking. We can obtain the object to be hooked through reflection, and then modify the method in this object through Java's Proxy dynamic proxy mechanism.

Below is the complete implementation of the solution. After we hook the startActivity method, we replace the PluginActivity to be launched with ProxyActivity and store the PluginActivity in the extra data segment for later use.

1\) Select different Hook schemes based on system judgment

```java
public static void hookAMSBinderProxy(){
    if(Build.VERSION.SDK_INT > Build.VERSION_CODES.P){
        hookIActivityTaskManager();
    }else{
        hookIActivityManager();
    }
}
```

2\) For the Hook solution starting from Android 10, we need to obtain the IActivityTaskManagerSingleton in IActivityTaskManager through reflection, and use Java's Proxy technology to intercept the startActivity method. In the startActivity method, we complete the replacement of PluginActivity. The code implementation is as follows.

```java
public static void hookIActivityTaskManager(){
    try{
        Field singletonField = null;
        Class<?> actvityManager = Class.forName("android.app.ActivityTaskManager");
        singletonField = actvityManager.getDeclaredField("IActivityTaskManagerSingleton");
        singletonField.setAccessible(true);
        Object singleton = singletonField.get(null);
        // Get the IActivityTaskManagerSingleton object
        Class<?> singletonClass = Class.forName("android.util.Singleton");
        Field mInstanceField = singletonClass.getDeclaredField("mInstance");
        mInstanceField.setAccessible(true);
        final Object IActivityTaskManager = mInstanceField.get(singleton);

        // Create a dynamic proxy object
        Object proxy = Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader()
                , new Class[]{Class.forName("android.app.IActivityTaskManager")}
                , new InvocationHandler() {
                    @Override
                    public Object invoke(final Object proxy, 
                            final Method method, final Object[] args) throws Throwable {    
                        Intent raw = null;
                        int index = -1;
                        if ("startActivity".equals(method.getName())) {
                            for (int i = 0; i < args.length; i++) {
                                if(args[i] instanceof Intent){
                                    raw = (Intent)args[i];
                                    index = i;
                                }
                            }
                            // Replace PluginActivity with ProxyActivity
                            Intent newIntent = new Intent();
                            newIntent.setComponent(new ComponentName("com.example.test", 
                                    ProxyActivity.class.getName()));
                            newIntent.putExtra(TARGET_INTENT,raw);
                            args[index] = newIntent;
                        }

                        return method.invoke(IActivityTaskManager, args);
                    }
                });
        // Replace ActivityManager.getService() with the proxyInstance
        mInstanceField.set(singleton, proxy);

    }catch (Exception e){
        e.printStackTrace();
    }
}
```

2\) In the Hook solution for Android versions below 10, further system judgment is made to determine whether to obtain the gDefault object or the IActivityManagerSingleton object, and then similarly proxy the startActivity method within the object. The code implementation is as follows.

```java
public static void hookIActivityManager() {
    try {
        // get singleton object
        Field singletonField = null;
        if (Build.VERSION.SDK_INT < Build.VERSION_CODES.O) { // 小于8.0
            Class<?> clazz = Class.forName("android.app.ActivityManagerNative");
            singletonField = clazz.getDeclaredField("gDefault");
        } else {
            Class<?> clazz = Class.forName("android.app.ActivityManager");
            singletonField = clazz.getDeclaredField("IActivityManagerSingleton");
        }

        singletonField.setAccessible(true);
        Object singleton = singletonField.get(null);

        Class<?> singletonClass = Class.forName("android.util.Singleton");
        Field mInstanceField = singletonClass.getDeclaredField("mInstance");
        mInstanceField.setAccessible(true);
        final Object mInstance = mInstanceField.get(singleton);

        Class<?> iActivityManagerClass = Class.forName("android.app.IActivityManager");

        Object proxyInstance = 
                Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(),
                new Class[]{iActivityManagerClass}, new InvocationHandler() {
                    @Override
                    public Object invoke(Object proxy, 
                            Method method, Object[] args) throws Throwable {
                        if ("startActivity".equals(method.getName())) {
                            int index = -1;

                            for (int i = 0; i < args.length; i++) {
                                if (args[i] instanceof Intent) {
                                    index = i;
                                    break;
                                }
                            }
                            Intent intent = (Intent) args[index];

                            Intent proxyIntent = new Intent();
                            proxyIntent.setComponent("com.example.test", 
                                    ProxyActivity.class.getName()));
                                    
                            proxyIntent.putExtra(TARGET_INTENT, intent);
                            args[index] = proxyIntent;
                        }
                        return method.invoke(mInstance, args);
                    }
                });

        mInstanceField.set(singleton, proxyInstance);

    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

### &#x20;3. Activity Restoration&#x20;

In the above process, we have already replaced PluginActivity with ProxyActivity in the host, so we can successfully pass the verification process in AMS, and then return to the application side to carry out subsequent processes such as Activity creation. At this time, we need to find an opportunity to restore ProxyActivity to PluginActivity again, so that PluginActivity can be started normally.&#x20;

According to the startup process, the optimal timing point is when the H object in ActivityThread processes the AMS notification to start an Activity. We can obtain the H object through reflection, and since the H object is a Handler object, we can modify the original logic by adding our own logic to the Handler's callback. In our own logic, we simply need to retrieve the PluginActivity stored in the extra and replace it.In different Android versions, the type of Message received in the H object varies. Starting from Android 9, the received message is the EXECUTE_TRANSACTION message, with a value of 159, but before Android 9, the received message is the LAUNCH_ACTIVITY message, with a value of 100. Therefore, for compatibility reasons, we also need to handle different system versions separately. The code implementation is as follows.&#x20;

```java
public static void hookHandler() {
    try {
        // Get the Class object of the ActivityThread class
        Class<?> clazz = Class.forName("android.app.ActivityThread");
        Field activityThreadField = clazz.getDeclaredField("sCurrentActivityThread");
        activityThreadField.setAccessible(true);
        Object activityThread = activityThreadField.get(null);
        // Get the mH object
        Field mHField = clazz.getDeclaredField("mH");
        mHField.setAccessible(true);
        final Handler mH = (Handler) mHField.get(activityThread);

        // Create a new callback
        Field mCallbackField = Handler.class.getDeclaredField("mCallback");
        mCallbackField.setAccessible(true);
        Handler.Callback callback = new Handler.Callback() {
            @Override
            public boolean handleMessage(@NonNull Message msg) {
                switch (msg.what) {
                    case 100: 
                        // LAUNCH_ACTIVITY, for versions below Android 9
                        replaceBelow9(msg);
                        break;
                    case 159: 
                        // EXECUTE_TRANSACTION, starting from Android 9
                        replaceAftert9(msg);
                        break;
                }
                return false;
            }
        };
        // Replace the system callback
        mCallbackField.set(mH, callback);
    } catch (
            Exception e) {
        e.printStackTrace();
    }
}
```

* Replacement logic for Android versions below 9

```java
void replaceBelow9(Message msg){
    try {
        // Use reflection to get the Intent, extract the Intent from the extra, and replace it 
        Field intentField = msg.obj.getClass().getDeclaredField("intent");
        intentField.setAccessible(true);
        Intent proxyIntent = (Intent) intentField.get(msg.obj);
        Intent intent = proxyIntent.getParcelableExtra(TARGET_INTENT);
        if (intent != null) {
            intentField.set(msg.obj, intent);
        }
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

* Replacement logic for Android 9 and above

```java
void replaceAftert9 (Message msg){
    try {
        // Get the mActivityCallbacks object,
        Field mActivityCallbacksField =  
                msg.obj.getClass().getDeclaredField("mActivityCallbacks");
        mActivityCallbacksField.setAccessible(true);
        List mActivityCallbacks = (List) mActivityCallbacksField.get(msg.obj);
        
        for (int i = 0; i < mActivityCallbacks.size(); i++) {
            // Extract the Intent from LaunchActivityItem and replace it
            if (mActivityCallbacks.get(i).getClass().getName().
                        equals("android.app.servertransaction.LaunchActivityItem")){
                // Get the Intent of the proxy startup
                Object launchActivityItem = mActivityCallbacks.get(i);
                Field mIntentField = 
                launchActivityItem.getClass().getDeclaredField("mIntent");
                mIntentField.setAccessible(true);
                // Replace proxyIntent with the target intent
                Intent proxyIntent = (Intent) mIntentField.get(launchActivityItem);
                Intent intent = proxyIntent.getParcelableExtra(TARGET_INTENT);
                if (intent != null) {
                    mIntentField.set(launchActivityItem, intent);
                }
            }
        }
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

The above is the entire process of the Hook solution. As long as we have a certain understanding of the Activity startup process, we can easily understand and implement this solution. The principles and solutions of other components such as Service and Broadcast are similar to those of the Activity component, so I will not explain them one by one.&#x20;

## 9.9.3 Method Retargeting

The Retargeting method is much easier than the Hook method. We only need to retarget the lifecycle methods of ProxyActivity to the corresponding lifecycle methods of the plugin PluginActivity, and at the same time retarget the methods overridden from Activity in the plugin PluginActivity to the corresponding methods in ProxyActivity for execution. The process is shown in Figure 9-19.&#x20;

![Figure 9-19 Method Retargeting Flow](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter9_img_18.png)

Since there are many methods involved, performing retargeting operations on each method is a rather cumbersome process. Therefore, we usually need to inherit a base class that has already handled these retargeting relationships. For example, PluginActivity has already handled these callback relationships, so it can be used as a base class, and the Activity components in the plugin can directly inherit from PluginActivity.The principle and technical points of this solution are also very simple, only requiring proper handling of the retargeting relationships between functions. However, it is not flexible enough, so the code implementation will not be elaborated here.&#x20;

## 9.9.4 Open Source Plug-in Framework

Although we have already understood the principles and implementation solutions of plug-in technology above, considering aspects such as stability and performance, I does not recommend that everyone reinvent the wheel to implement a plug-in framework. Plug-in technology is already very mature, and there are also many open-source plug-in frameworks available for us to use online. Common plug-in frameworks include [VirtualApp](https://github.com/asLody/VirtualApp), [Small](https://github.com/wequick/Small), Ctrip's [DynamicApk](https://github.com/CtripMobile/DynamicAPK), 360's [RePlugin](https://github.com/Qihoo360/RePlugin), Tencent's [Shadow](https://github.com/Tencent/Shadow), etc.

There are numerous open-source pluggable frameworks, so how should we make a choice in application development? We primarily base our selection on these two points.&#x20;

1. Try to choose frameworks with a large user base, high update frequency, and few Hook points. For example, frameworks like VirtualApp or Small are outdated and have not been updated for a long time, so they are not recommended for use.&#x20;

2. According to the Technology Implementation plan of the plug-in framework, select a framework that aligns with the application's characteristics. If the capabilities of the plug-in are part of the host and require strong coupling with the host, then we need to choose a solution that merges the plug-in's resources, dex, and so into the host's context. If the capabilities of the plug-in are merely an extension of the host and do not require strong coupling with the host, then we should try to choose a solution with an independent context.

Although there are many excellent open-source plug-in frameworks available for us to use, we still need to have a deep understanding of the principles of plug-in technology, because only in this way can we use these open-source frameworks more reasonably and confidently, and further modify the open-source frameworks to better adapt them to our own project scenarios.

| Source code appearing in this chapter:<br />ShrinkResourcesTransform: <br /><https://android.googlesource.com/platform/tools/base/+/gradle_2.0.0/build-system/gradle-core/src/main/groovy/com/android/build/gradle/internal/transforms/ShrinkResourcesTransform.java><br />ResourceTypes.h: <br /><https://cs.android.com/android/platform/superproject/+/android-14.0.0_r9:frameworks/base/libs/androidfw/include/androidfw/ResourceTypes.h><br />AssetManager.java: <br /><https://cs.android.com/android/platform/superproject/+/android-14.0.0_r9:frameworks/base/core/java/android/content/res/AssetManager.java><br />AssetManager.cpp:<br /><https://cs.android.com/android/platform/superproject/+/android-14.0.0_r9:frameworks/base/core/jni/android_util_AssetManager.cpp><br />Instrumentation.java: <br /><https://cs.android.com/android/platform/superproject/+/android-14.0.0_r9:frameworks/base/core/java/android/app/Instrumentation.java><br />ActivityManagerService.java:<br /><https://cs.android.com/android/platform/superproject/+/android-14.0.0_r9:frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java><br />ActivityThread.java:<br /><https://cs.android.com/android/platform/superproject/+/android-14.0.0_r9:frameworks/base/core/java/android/app/ActivityThread.java><br />LaunchActivityItem.java:<br /><https://cs.android.com/android/platform/superproject/+/android-14.0.0_r9:frameworks/base/core/java/android/app/servertransaction/LaunchActivityItem.java><br />ActivityTaskManagerService.java:<br /><https://cs.android.com/android/platform/superproject/+/android-14.0.0_r9:frameworks/base/services/core/java/com/android/server/wm/ActivityTaskManagerService.java><br />ActivityStarter.java:<br /><https://cs.android.com/android/platform/superproject/+/android-14.0.0_r9:frameworks/base/services/core/java/com/android/server/wm/ActivityStarter.java><br />RootWindowContainer.java:<br /><https://cs.android.com/android/platform/superproject/+/android-14.0.0_r9:frameworks/base/services/core/java/com/android/server/wm/RootWindowContainer.java><br />ActivityTaskSupervisor.java:<br /><https://cs.android.com/android/platform/superproject/+/android-14.0.0_r9:frameworks/base/services/core/java/com/android/server/wm/ActivityTaskSupervisor.java><br />InterDexPass.cpp:<br /><https://github.com/facebook/redex/blob/main/opt/interdex/InterDexPass.cpp><br />oatmeal:<br /><https://github.com/facebook/redex/tree/main/tools/oatmeal><br />ClassLinker:<br /><https://cs.android.com/android/platform/superproject/+/android-14.0.0_r9:art/runtime/class_linker-inl.h><br />Runtime.java:<br /><https://cs.android.com/android/platform/superproject/+/android-14.0.0_r9:libcore/ojluni/src/main/java/java/lang/Runtime.java><br />java_vm_ext.cc：<br /><https://cs.android.com/android/platform/superproject/+/android-14.0.0_r9:art/runtime/jni/java_vm_ext.cc><br />BaseDexClassLoader.java:<br /><https://cs.android.com/android/platform/superproject/+/android-14.0.0_r9:libcore/dalvik/src/main/java/dalvik/system/BaseDexClassLoader.java> |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
