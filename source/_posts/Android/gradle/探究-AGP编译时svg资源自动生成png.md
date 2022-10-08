---
title: '探究: AGP编译时svg资源自动生成png'
date: 2022-08-28 14:21:50
tags:
---

## 背景：
当安卓工程的minSdk<24时，项目中的svg资源如果使用到了有兼容问题的属性，同时module的 vectorDrawables.useSupportLibrary 开关为false，会自动在编译打包过程中生成png图片来解决兼容性问题。
```gradle
android {
    defaultConfig {
        vectorDrawables.useSupportLibrary = false 
        vectorDrawables.generatedDensities = ["xxhdpi"] // 设置只在drawable-xxhdpi目录下的生成png图片，如果不设置，最终的apk会在每一个drawable目录下有一个png
    }
}
```
这样兼容性的问题就解决了，但是，生成了很多冗余的png图片。本来我们使用svg就是为了缩小包体积，现在反而包体积增大了。需要去探究一下，冗余的png图片是在哪里生成的，什么情况下额外生成png图片

## MergeResources
基于AGP 4.1.0 版本

通过ide的全局文本查找 useSupportLibrary 出现的地方，可以找到一个明显有嫌疑的类 MergeResources 。
这是编译打包过程中的一个task类，需要注意的是，在子module和app，这个task有所不同。它在子module的task-name是packageDebugResource ，在app的task-name是mergeDebugResource。同样，在子module和app，表现也有所不同。

先从doFullTaskAction方法看这个task的主流程：
伪代码
```java
@Override
protected void doFullTaskAction() throws IOException, JAXBException {
    ResourcePreprocessor preprocessor = getPreprocessor(); // 关键1

    File destinationDir = getOutputDir().get().getAsFile(); 
    // 资源merge的最终目录
    // build/intermediates/package_res
    
    ... //省略
    // 因为是全量，删除destinationDir的所有文件

    // 获取所有资源目录
    List<ResourceSet> resourceSets =
            getConfiguredResourceSets(preprocessor, getAaptEnv().getOrNull());

    // task 的核心类
    ResourceMerger merger = new ResourceMerger(getMinSdk().get());
    
    try (
            // 创建资源编译服务类
            ResourceCompilationService resourceCompiler =
                    getResourceProcessor(
                            getProjectName(),
                            getPath(),
                            getAapt2FromMaven(),
                            workerExecutorFacade,
                            errorFormatMode,
                            flags,
                            processResources,
                            useJvmResourceCompiler,
                            getLogger(),
                            getAapt2DaemonBuildService().get())) {

        for (ResourceSet resourceSet : resourceSets) {
            resourceSet.loadFromFiles(new LoggerWrapper(getLogger())); // 关键2
            merger.addDataSet(resourceSet);
        }

        File publicFile =
                getPublicFile().isPresent() ? getPublicFile().get().getAsFile() : null;
        MergedResourceWriter writer =
                new MergedResourceWriter(
                        workerExecutorFacade,
                        destinationDir,
                        publicFile,
                        mergingLog,
                        preprocessor,
                        resourceCompiler,
                        getIncrementalFolder(),
                        dataBindingLayoutProcessor,
                        mergedNotCompiledResourcesOutputDirectory,
                        pseudoLocalesEnabled,
                        getCrunchPng());

        merger.mergeData(writer, false) // 关键3

    } catch (Exception e) {
        ...
    } finally {
        cleanup();
    }
}
```

### 关键1 ResourcePreprocessor
getPreprocessor() 返回的是一个接口
```java
public interface ResourcePreprocessor extends Serializable {
    // 返回预处理需要生成的file集合
    @NonNull
    Collection<File> getFilesToBeGenerated(@NonNull File original) throws IOException;

    // 生成 getFilesToBeGenerated 方法返回的file集合文件
    void generateFile(@NonNull File toBeGenerated, @NonNull File original) throws IOException;
}
```
它的实现类是 VectorDrawableRenderer ，看名字可以判定，这就是 svg->png 的实现类（罪魁祸首）。。。先寻找它被调用的地方。

### 关键2 ResourceSet#loadFromFiles
```java
for (ResourceSet resourceSet : resourceSets) {
    resourceSet.loadFromFiles(new LoggerWrapper(getLogger())); // 关键2
    merger.addDataSet(resourceSet);
}
```

resourceSets 是一个res目录的集合。 
在子module，会包含工程里module的src/main/res目录；（断点看，似乎有用的也就只有这个）
在app，会包含 app 的 src/main/res ，所有子module的 build/intermediates/package_res ，以及一些安卓官方库的资源目录
从 resourceSet.loadFromFiles 一直追，会到 ResourceSet#getResourceMergerItemsForGeneratedFiles 方法
```java
@NonNull
private List<ResourceMergerItem> getResourceMergerItemsForGeneratedFiles(@NonNull File file)
        throws MergingException {
    Collection<File> filesToBeGenerated;
    try {
        filesToBeGenerated = mPreprocessor.getFilesToBeGenerated(file); // 关键
    } catch (IOException e) {
        throw new MergingException(e);
    }
    List<ResourceMergerItem> resourceItems = new ArrayList<>(filesToBeGenerated.size());
    for (File generatedFile : filesToBeGenerated) {
        FolderData generatedFileFolderData =
                getFolderData(generatedFile.getParentFile());
        resourceItems.add(
                new GeneratedResourceMergerItem(
                        getNameForFile(generatedFile),
                        mNamespace,
                        generatedFile,
                        generatedFileFolderData.type,
                        generatedFileFolderData.folderConfiguration.getQualifierString(),
                        mLibraryName)
                );
    }
    return resourceItems;
}
```

这里的 mPreprocessor 就是 VectorDrawableRenderer ，`mPreprocessor.getFilesToBeGenerated(file)` 会返回 svg 资源文件和png兼容资源文件的集合。里面不详细看了，主要是根据useSupportLibrary 和generatedDensities 这两个gradle参数生成png文件的绝对路径。 
这里的png路径是 build/generated/res/pngs/debug 。 这和 MergeResources task 的目标路径并不相同。
getResourceMergerItemsForGeneratedFiles 方法最终会将每一个资源文件抽象为ResourceMergerItem ，并返回他们的集合。

### 关键3 merger#mergeData
ResourceMerger#mergeData 会调用到父类的 DataMerger#mergeData
```java
public void mergeData(@NonNull MergeConsumer<I> consumer, boolean doCleanUp){
    consumer.start(mFactory);
    for(...) {
        ... // 忽略 merge 的过程
        consumer.addItem(toWrite)
    }
    consumer.end();
}
```
这里的 consumer 是 MergedResourceWriter 实例。 

```java
@Override
public void addItem(@NonNull final ResourceMergerItem item) throws ConsumerException {
    final ResourceFile.FileType type = item.getSourceType();

    if (type == ResourceFile.FileType.XML_VALUES) {
        mValuesResMap.put(item.getQualifiers(), item);
    } else {
        if (item.isTouched()) {
            File file = item.getFile();
            String folderName = getFolderName(item);
            
            if (type == DataFile.FileType.GENERATED_FILES) {
                try {
                    FileGenerationParameters workItem =
                            new FileGenerationParameters(item, mPreprocessor);
                    if (workItem.resourceItem.getSourceFile() != null) {
                        // 这个action是生成png文件的地方
                        getExecutor().submit(FileGenerationWorkAction.class, workItem);
                    }
                } catch (Exception e) {
                    throw new ConsumerException(e, item.getSourceFile().getFile());
                }
            }

            // 添加资源编译 Request
            mCompileResourceRequests.add(
                    new CompileResourceRequest(
                            file, getRootFolder(), folderName, item.mIsFromDependency));
        }
    }
}
```
这里有一个 ResourceFile.FileType 枚举值

```java
public enum FileType {
    SINGLE_FILE,   // 表示项目中工程目录 src/main/res 原本就有的资源文件
    GENERATED_FILES,  // 需要生成的资源文件，生成的目录在 build/generated/res/pngs/debug 
    XML_VALUES  // values文件
}
```
如果是 GENERATED_FILES 会生成新文件

```java
public static class FileGenerationWorkAction implements Runnable {

    private final FileGenerationParameters workItem;

    @Inject
    public FileGenerationWorkAction(FileGenerationParameters workItem) {
        this.workItem = workItem;
    }

    @Override
    public void run() {
        try {
            workItem.resourcePreprocessor.generateFile(
                    workItem.resourceItem.getFile(),
                    workItem.resourceItem.getSourceFile().getFile());
        } catch (Exception e) {
            ...
        }
    }
}
```
workItem.resourcePreprocessor 是 VectorDrawableRenderer 对象，这样生成了png文件的地方也找到了。
```java
@Override
public void generateFile(@NonNull File toBeGenerated, @NonNull File original)
        throws IOException {
    Files.createParentDirs(toBeGenerated);
    if (isXml(toBeGenerated)) {
        // 目标文件如果是xml文件，直接拷贝即可
        Files.copy(original, toBeGenerated);
    } else {
        // 在对应 density 的目录下生成png文件
        FolderConfiguration folderConfiguration = getFolderConfiguration(toBeGenerated);
        Density density = folderConfiguration.getDensityQualifier().getValue();
        float scaleFactor = density.getDpiValue() / (float) Density.MEDIUM.getDpiValue();
        if (scaleFactor <= 0) {
            scaleFactor = 1.0f;
        }
        VdPreview.TargetSize imageSize = VdPreview.TargetSize.createFromScale(scaleFactor);
        String xmlContent = Files.asCharSource(original, StandardCharsets.UTF_8).read();
        BufferedImage image = VdPreview.getPreviewFromVectorXml(imageSize, xmlContent, null);
        ImageIO.write(image, "png", toBeGenerated);
    }
}
```
- 生成png文件的效果
工程module下有两个资源图片文件，一个是svg，一个是png。 svg图片会在 build/generated/res/pngs 下生成多个资源，工程原有的png则不会

### 编译资源
再来看 ResourceMerger#mergeData，

```java
public void mergeData(@NonNull MergeConsumer<I> consumer, boolean doCleanUp){
    consumer.start(mFactory);
    for(...) {
        ... // 忽略 merge 的过程
        consumer.addItem(toWrite)
    }
    consumer.end();
}
```

最后有一个consumer.end() 

```java
public void end() throws ConsumerException {
    while (!mCompileResourceRequests.isEmpty()) {
        mResourceCompiler.submitCompile(
            new CompileResourceRequest(
                    fileToCompile,
                    request.getOutputDirectory(),
                    request.getInputDirectoryName(),
                    request.getInputFileIsFromDependency(),
                    pseudoLocalesEnabled,
                    crunchPng,
                    ImmutableMap.of(),
                    request.getInputFile()));
        mCompiledFileMap.put(
            fileToCompile.getAbsolutePath(),
            mResourceCompiler.compileOutputFor(request).getAbsolutePath());
    }
}
```

mResourceCompiler.submitCompile 正式提交编译。
mResourceCompiler 的实现在子module和app有所不同，
在module，实例是CopyToOutputDirectoryResourceCompilationService 
在app，实例是 WorkerExecutorResourceCompilationService

- 子module的资源 "编译"
```kotlin
object CopyToOutputDirectoryResourceCompilationService : ResourceCompilationService {
    override fun submitCompile(request: CompileResourceRequest) {
        val out = compileOutputFor(request)
        FileUtils.mkdirs(out.parentFile)
        FileUtils.copyFile(request.inputFile, out)
    }

    override fun compileOutputFor(request: CompileResourceRequest): File {
        val parentDir = File(request.outputDirectory, request.inputDirectoryName)
        return File(parentDir, request.inputFile.name)
    }

    override fun close() {
    }
}
```
在子module比较简单，就是把 request 里的 inputFile 拷贝到目标目录。
如果是svg生成png的case，是 build/generated/res/pngs/ 文件拷贝到 build/intermediates/package_res/
如果是工程中的一般资源文件，是`src/main/res` 文件拷贝到 build/intermediates/package_res/

- app的资源编译
```kotlin
class WorkerExecutorResourceCompilationService(...) : ResourceCompilationService {

  private val requests: MutableList<CompileResourceRequest> = ArrayList()

  override fun submitCompile(request: CompileResourceRequest) {
    requests.add(request)
  }
  ...
  override fun close() {
    for (...) {
      val bucketRequests = requests.filterIndexed { i, _ ->
        i.rem(buckets) == bucket
      }
      workerExecutor.submit(
        Aapt2CompileRunnable::class.java,
        Aapt2CompileRunnable.Params(aapt2ServiceKey, bucketRequests, errorFormatMode, true)
      )
    }
  }
}
```
在close方法遍历所有request，然后在Aapt2CompileRunnable进行资源编译。 输出路径是app/build/intermediates/res/merged/debug 
对于appt2编译，我不是很了解，这里不详细说了。

## SVG -> PNG 的条件
```java
public class VectorDrawableRenderer implements ResourcePreprocessor {
    @Override
    @NonNull
    public Collection<File> getFilesToBeGenerated(@NonNull File inputXmlFile) throws IOException {
        FolderConfiguration originalConfiguration = getFolderConfiguration(inputXmlFile);
        PreprocessingReason reason = getReasonForPreprocessing(inputXmlFile, originalConfiguration);
        if (reason == null) {
            return Collections.emptyList();
        }
    
        Collection<File> filesToBeGenerated = new ArrayList<>();
    
        DensityQualifier densityQualifier = originalConfiguration.getDensityQualifier();
        boolean validDensityQualifier = ResourceQualifier.isValid(densityQualifier);
    
        if (mMinSdk < reason.getSdkThreshold() && mDensities.size() > 0) {
           for (Density density : mDensities) {
               FolderConfiguration newConfiguration =
                                FolderConfiguration.copyOf(originalConfiguration);
               newConfiguration.setDensityQualifier(new DensityQualifier(density));
               filesToBeGenerated.add(
                   new File(
                       getDirectory(newConfiguration),
                       inputXmlFile.getName().replace(".xml", ".png"))
               );
           }
        }
    
        return filesToBeGenerated;
    }
}
```
上面的代码省略了诸多逻辑，我们只看关心的部分。
首先去获取一个reason，如果reason为空，直接返回空集合。 
如果 mMinSdk < reason.getSdkThreshold() ，项目的minSdk小于 reason的sdk阈值，则往filesToBeGenerated添加一个png路径。所以要去看如何得到reason的

```java
@Nullable
private PreprocessingReason getReasonForPreprocessing(
        @NonNull File resourceFile, @NonNull FolderConfiguration folderConfig)
        throws IOException {
    // 如果开启了 supportLibraryIsUsed 直接返回null。 和「背景」中说的一致
    if (mSupportLibraryIsUsed) return null;
    // 如果 minSdk 大于 GRADIENT_SUPPORT 的版本号，也直接返回null
    // PreprocessingReason.GRADIENT_SUPPORT 版本号是24。 
    // 即到了24版本开始就没有兼容性问题了。 minSdk > 24 即无需处理
    if (mMinSdk >= PreprocessingReason.GRADIENT_SUPPORT.getSdkThreshold()) return null;
    if (!isXml(resourceFile) || !isInDrawable(resourceFile)) return null;
    
    try (InputStream stream = new BufferedInputStream(new FileInputStream(resourceFile))) {
        XMLInputFactory factory = XMLInputFactory.newFactory();
        XMLStreamReader xmlReader = factory.createXMLStreamReader(stream);

        // 开始读取svg xml文件的内容，check每一个元素和属性
        boolean beforeFirstTag = true;
        while (xmlReader.hasNext()) {
            int event = xmlReader.next();
            if (event == XMLStreamReader.START_ELEMENT) {
                if (beforeFirstTag) {
                    if (!TAG_VECTOR.equals(xmlReader.getLocalName())) {
                        // 如果不是 vector 直接不用check了
                        return null;
                    }
                    beforeFirstTag = false;
                } else {
                    if (TAG_GRADIENT.equals(xmlReader.getLocalName())) {
                        // 找到 gradient 这个元素，返回一个reason
                        // GRADIENT_SUPPORT 的版本阈值是24
                        return PreprocessingReason.GRADIENT_SUPPORT;
                    }
                    int n = xmlReader.getAttributeCount();
                    for (int i = 0; i < n; i++) {
                        // 找到 fillType 这个属性，返回一个reason 。 
                        // FILLTYPE_SUPPORT 的版本阈值是 24
                        if ("fillType".equals(xmlReader.getAttributeLocalName(i))
                                && NS_RESOURCES.equals(xmlReader.getAttributeNamespace(i))) {
                            return PreprocessingReason.FILLTYPE_SUPPORT;
                        }
                    }
                }
            }
        }
        if (!beforeFirstTag && mMinSdk < PreprocessingReason.VECTOR_SUPPORT.getSdkThreshold()) {
            // 判断是minSdk否是压根就不支持 vector ，返回一个reason
            // VECTOR_SUPPORT 的版本阈值是 21
            return PreprocessingReason.VECTOR_SUPPORT;
        }
    } catch (XMLStreamException e) {
        throw new IOException(
                "Failed to parse resource file " + resourceFile.getAbsolutePath(), e);
    }

    return null;
}
```

很明显，AGP 只会判断svg文件里的 gradient 和 fillType 这两个东西。
gradient是颜色渐变
fillType 专业性太强，搜了一圈看不懂，不知道是做什么的。。。
他们的版本号阈值都是24，svg文件中出现这两个东西，并且 minSdk < 24 时就会触发 svg->png 的生成

## 如何处理
在我们的项目里，svg自动生成png的case太多了，大概接近300个资源图片。 需要去治理一波。 
生成的png直接导致我们使用svg去减小包体积的初衷失去意义。
但是不能一个一个改，太费事。

有一个开源的 McImage 插件，作用是把项目中的所有资源转换成webp，我们可以模仿它

AGP提供了 variant.getAllRawAndroidResources().files 这个api，在app下使用，可以获取到所有的resource目录。 打印所有的目录
```
// 子module的 build/intermediates/packaged_res 目录
/Users/xx/PluginX/module_1/build/intermediates/packaged_res/debug
// 依赖库的res目录
/Users/xx/.gradle/caches/transforms-2/files-2.1/414fc23eb49f389f342a6e17218892be/appcompat-1.2.0/res
/Users/xx/.gradle/caches/transforms-2/files-2.1/72202874f0ee490d85b35e8bc8155d2b/constraintlayout-1.1.3/res
/Users/xx/.gradle/caches/transforms-2/files-2.1/52e4a4d01e3d8c6a6c3d516d66f6acc9/recyclerview-1.0.0/res
/Users/xx/.gradle/caches/transforms-2/files-2.1/83248d60fee84d269b4d5ae691d6e421/jetified-appcompat-resources-1.2.0/res
/Users/xx/.gradle/caches/transforms-2/files-2.1/5fddbd55bd0ad4a84e0959052d0c417d/coordinatorlayout-1.0.0/res
/Users/xx/.gradle/caches/transforms-2/files-2.1/05736e5c1eb0ab8976aa5868fca67ffe/core-1.3.1/res

```

结合上面 MergeResources 的分析。
子module会在 packageDebugResources 任务，把module所有资源汇总拷贝到 build/intermediates/packaged_res/
在app也能拿到所有子module汇总拷贝之后的这个目录。
可以在app的 mergeDebugResources 任务之前，插入一个我们自己的任务。 
作用是收集 getAllRawAndroidResources().files 目录里的所有资源文件，查找删除同名的多余资源图片，如果存在xml后缀的svg资源和多个同名的png资源，只保留 xxhdpi 目录下的png图片资源。 
实践来看，可行，没有报错，打出来的apk文件安装使用也都正常。 

TODO：是否可以在 mergeDebugResources 之后进行 hook ？


## 参考:
[Android中Gradle原理以及机制深入分析](http://www.youkmi.cn/2020/01/01/android-zhong-gradle-yuan-li-yi-ji-ji-zhi-shen-ru-fen-xi/)
[gradle编译打包过程分析之ProcessAndroidResources](https://juejin.cn/post/7060337237761720334)
[McImage插件解析（旧版本）](https://smallsoho.com/android/2017/04/07/McImage%E6%8F%92%E4%BB%B6%E8%A7%A3%E6%9E%90/)