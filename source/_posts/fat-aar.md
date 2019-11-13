---
title: 合并AAR踩坑之旅
date: 2018-9-12 11:21:06
tags:
  - Android
  - Gradle
---

### 背景

在输出Android模块时，有时会因为个别原因（比如来自业务的不可抗力），要求将模块打包成一个文件提供给接入方。这就意味着在输出模块由多个子模块组成的情况下，我们需要把多个AAR（或JAR）合并成一个大AAR输出，这个合并过程涉及到了很多有用的知识和难点，这篇文章就详细解析下其中的内容。
### 认识AAR

AAR文件是一种Android归档包（类比Jar：Java Archive），这种归档包是由Gradle构建库的Android Library插件产出的。它是一个压缩包，里面的内容可以总结为5个目录和5个文件：

![AAR文件内容](https://i.loli.net/2019/11/13/oJGSOwfF1QDBl8W.png))

`*：卖个关子，在上面10个内容中，其中有一个是已经合并后的结果，它默认已经包含了所有子模块的内容，你能猜出来是那个么？`

图里文字已经基本解释清楚了各个内容，不再赘述，现在我们根据现有的了解设想下合并AAR的大致思路：AAR里无非是几个目录和文件，所以最终AAR的5个目录下，必然包含了所有子模块的对应内容，而5个文件也肯定是各个子文件合并的结果。那么如何实现包含和合并呢？第一反应是写个脚本对AAR们做解压、整合，再压缩的思路，但稍微推演下就知道不现实（理论上也不可行，AAR和普通压缩文件还是有区别的），再思考下，既然直接拿产物做合并不可行，能不能在构建主模块AAR时，就顺便将子模块内容纳入进来呢？这无疑是个优雅的思路，理论上是否可行呢？答案是可以的，别忘了AAR是Android Gradle Plugin构建产出的，而Gradle强大的拓展支持刚好能实现我们的需求。所以合并AAR的方法，其实就是修改Gradle构建流程，在默认构建的基础上，插入我们自己的操作，最终产出一个包含子模块内容的大AAR。

在入题之前，有必要先理解下Gradle是如何支持构建拓展的，下图是Gradle官网截图：

<!--more-->

![Gradle描述及生命周期](https://i.loli.net/2019/11/13/aduKx84RSBE7fHz.png)

Gradle中定义了`Project`和`Task`两个概念，同时也将构建流程`面向对象`化（即构建是针对Project执行的一系列Task）。对Gradle的描述关键在于`基于依赖编程`，具体解释就是可以用Gradle来定义Task和Task间的`依赖关系`，最终得出一个Task的[有向无环图](https://zh.wikipedia.org/wiki/%E6%9C%89%E5%90%91%E6%97%A0%E7%8E%AF%E5%9B%BE)来描述构建。这意味着，我们需要将合并的操作封装为Task，并在合适的生命周期内，将自定义Task插入到任务图的依赖关系中去，进而得出一个新的Task图，最终构建出我们要的产物。

我们在主工程下执行./gradlew assembleRelease，看一下默认构建中都执行了哪些Task：

![默认构建](https://i.loli.net/2019/11/13/5GYqshztyo2vDNS.png)

可以看到依次执行了收集依赖，合并资源、Javac编译、打包文件等Task。同时，通过观察`/build/intermediates`目录可以发现，不同任务会产出各自的构建中间结果，最终的AAR就是由这些中间结果打包而来的。所以理论上，我们需要在主模块执行打包Task前将子模块内容混入到中间结果中，而由于构建Task间存在依赖关系，混入操作需要分阶段、找时机执行。

合并前需要先确定到底需要合并哪些子模块。我们通过定义一个dependency configuration: emeded来标记需要被合并的模块，然后在gradle构建的afterEvaluate阶段收集被emeded依赖的模块信息：
```gradle
afterEvaluate {
    def dependencies = new ArrayList(configurations.embedded.resolvedConfiguration.firstLevelModuleDependencies)
    dependencies.reverseEach {
        ...
        it.moduleArtifacts.each {
            artifact ->
                if (artifact.type == 'aar') {
                    if (!embeddedAarFiles.contains(artifact)) {
                        //要合并的AAR文件
                        embeddedAarFiles.add(artifact)
                    }
                    if (!embeddedAarDirs.contains(modulePath)) {
                        ...
                        //每个AAR的解压目录
                        embeddedAarDirs.add(modulePath)
                    }
                } else if (artifact.type == 'jar') {
                    ...
                    //要合并的JAR文件
                    embeddedJars.add(artifactPath)
                } 
                ...
        }
    }
}
```
可以看出，我们收集了三个集合：
- 要合并的AAR文件
- 每个AAR的解压目录
- 要合并的JAR文件

AAR和JAR分开收集是因为合并这两种文件的操作不同，JAR只需纳入将其Class文件，而AAR需要合并更多内容。

下面针对AAR中的5和目录和5个文件，逐个介绍合并Task，从最简单的开始：
### /assets目录
合并assets内容的Task很简单，只需将子模块解压后的assets目录添加到主工程的assets.srcDirs中即可
```
task embedAssets << {
    embeddedAarDirs.each { aarPath ->
        android.sourceSets.main.assets.srcDirs += file("$aarPath/assets")
    }
}
```
### /res目录
通过conventionMapping，修改构建流程`task packageRecourses`的输入集合，将收集到的子模块/res目录追加到Task输入参数中
```gradle
task embedLibraryResources << {
    def oldInputResourceSet = task_packageResources.inputResourceSets
    task_packageResources.conventionMapping.map("inputResourceSets") {
        getMergedInputResourceSets(oldInputResourceSet)
    }
}
private List getMergedInputResourceSets(List inputResourceSet) {
    ...
    List newInputResourceSet = new ArrayList(inputResourceSet)
    embeddedAarDirs.each { aarPath ->
        ...
        rs.addSource(file("$aarPath/res"))
        newInputResourceSet += rs
    }
    return newInputResourceSet
}
```
### /jni目录
遍历子模块解压目录，将其中的so文件copy到主工程build目录下
```gradle
task embedJniLibs << {
    embeddedAarDirs.each { aarPath ->
        copy {
            from fileTree(dir: "$aarPath/jni")
            into file("$bundle_release_dir/jni")
        }
    }
}
```
### proguard.txt
`注意：这个文件与AAR构建自身时的混淆操作无关，只作用于接入到宿主后的混淆操作`
```gradle
task embedProguard << {
    //bundle_release_dir = "build/intermediates/bundles/release"
    def proguardFile = file("$bundle_release_dir/proguard.txt")
    //遍历aar解压目录，将子模块的proguard.txt文件内容追加到主工程build目录内的proguard.txt中
    embeddedAarDirs.each { aarPath ->
         ...
         def proguardLibFile = file("$aarPath/proguard.txt")
         if (proguardLibFile.exists())
            //调用file.append()方法可以在txt文件里追加内容
            proguardFile.append("\n" + proguardLibFile.text)
         ...
    }
}
```
### AndroidManifest.xml
合并AndroidManifest使用官方库manifest-merger即可，这里遇到了第一个难点：想要把子模块AndroidManifest合并进来必须使用MergeType为`APPLICATION`的ManifestMerger对象，但在这种MergeType下会替换定义在AndroidManifest里的PlaceHolder，比如`${applicationId}`。然而矛盾的是，此时的构建还没有接入到宿主App，也就拿不到最终的ApplicationId，怎么办呢？通过阅读源码发现，官方仁慈的给我们开放了一个功能位，通过在初始化ManifestMerger对象时传入一个`NO_PLACEHOLDER_REPLACEMENT`功能位，即可禁止在APPLICATION模式下替换PlaceHolder。
```gradle
buildscript {
    repositories {
        jcenter()
    }
    dependencies {
        classpath 'com.android.tools.build:manifest-merger:25.3.2'
    }
}
...
task embedManifests << {
  ...
  embeddedAarDirs.each { aarPath ->
      File dependencyManifest = file("$aarPath/AndroidManifest.xml")
      if (!libraryManifests.contains(aarPath) && dependencyManifest.exists()) {
          //先收集需要合并的子模块AndroidManifest
          libraryManifests.add(dependencyManifest)
      }
  }
  ...
  Invoker manifestMergerInvoker = ManifestMerger2.newMerger(origManifest, mLogger, MergeType.APPLICATION)
        //通过Invoker.Feature.NO_PLACEHOLDER_REPLACEMENT这个Flag禁止执行PlaceHolder替换
        .withFeatures(Invoker.Feature.NO_PLACEHOLDER_REPLACEMENT)
  manifestMergerInvoker.addLibraryManifests(libraryManifests.toArray(new File[libraryManifests.size()]))
  manifestMergerInvoker.setMergeReportFile(reportFile);
  //执行合并
  MergingReport mergingReport = manifestMergerInvoker.merge();
  ...
}
```
### classes.jar

这似乎是最重要的文件，但是合并方法却不难：对于AAR类型的子模块，我们需要提取两样东西：一是解压子模块内部classes.jar得到的Class文件，二是看看子模块内部有没有自身携带的JAR文件。对于Class文件，将它们copy到主工程`build/intermediates/classes/release`目录下，随后的构建任务会从这个目录打包出主模块的classes.jar；对于子模块内部的JAR文件，将它们和JAR类型的子模块一起，放入主工程的`build/intermediates/bundles/release/libs`目录下，作为依赖包携带即可
```gradle
task embedClassesAndJars(dependsOn: embedRJar) << {
    embeddedAarDirs.each { aarPath ->
        ...
        embeddedAarFiles.each {
                artifact ->
                    ...
                    //找到每个aar里的clasess.jar
                    def aarFile = aarFileTree.files.find { it.name.contains("classes.jar") }
                    // 解压clasess.jar，将classes放入build/intermediates/classes/release
                    copy {
                        from zipTree(aarFile)
                        into classes_dir
                    }
            }
        ...
        //找到每个aar里携带的额外jar包
        FileTree jars = fileTree(dir: jar_dir, include: '*.jar', exclude: 'classes.jar')
        ...
        //放入build/intermediates/bundles/release/libs
        copy {
            from jars
            into file("$bundle_release_dir/libs")
        }
    }
    //主工程直接依赖的jar包，也放入build/intermediates/bundles/release/libs
    copy {
        from embeddedJars
        into file("$bundle_release_dir/libs")
    }
}
```
### R文件

至此，我们已经合并了三个文件和四个目录：
![已合并的内容](https://i.loli.net/2019/11/13/SfeVAYCzpFXbGjU.png)

剩下的内容里，/aidl目录和annotation.zip不常用到，我们暂不理会，乍一看好像已经合并完成了，只剩下个看似无害的R.txt，它是干嘛用的？我们当真完成了么？答案是没有，假如按照目前的改造构建出一个AAR并接入到App中运行，结果是App会在每一处使用到AAR子模块资源文件的时候崩溃，堆栈日志很简单：`NoClassDefFoundError` - 没有子模块包名下的R文件。啊~忘了R文件这个东西了！可R文件不是构建时自动生成的么？没错，是自动生成的，但由于我们合并了模块，子模块将丢失它们的R文件，仔细梳理下App构建流程就会发现原因：
![构建历程对比](https://i.loli.net/2019/11/13/zLiJ5lXuIsjxroM.png)
如图上部所示，正常情况下，不管是主模块还是子模块，都是作为一个个独立的dependence引入到宿主工程的，App构建时会给每一个dependence生成它们包名下的R文件；而当我们自主的将一群AAR合并为一个AAR后（图下半部），对于宿主来说最后只接入了一个dependence（主模块），子模块本该对应的dependences不存在了，所以App构建时只给主模块生成了R文件！知道原因了，怎么解决呢？我们知道，构建主模块时，会在`/build/generated/source/r`目录下生成所有包名的R文件，当然也包括子模块R文件，那么我们把这个目录下的子模块R文件打成Jar包，放入主工程`build/intermediates/bundles/release/libs`目录下，这样R.jar会作为依赖库被AAR携带，不就可以了吗？我们兴冲冲地改了脚本，再次构建运行，结果还是崩溃！看下堆栈信息，提示资源找不到：`Resources$NotFoundException`，这又是为什么？R文件可是自动生成的呀？为什么生成的文件内容（也就是资源ID）找不到对应资源？
在解释原因之前，先抛几个问题：
- AAR里默认有R文件么？
- 为什么Android Library工程生成的R.java里的域不是final修饰的？
- 生成R文件流程的输出无疑是R.java，那么输入是什么？

在回答这些问题之前，先复习下Java基础知识，下面是一段简单的Java代码：
```java
public class Test {
    private static final int SOME_ID = 0x07111111;
    private int getID(){
        return SOME_ID;
    }
}
```
假如我们将SOME_ID的`final`修饰符去掉，那么编译后的字节码跟不去掉`final`相比，会有什么不同呢？我们反编译看下结果：
```java
//带final修饰符
public class Test {
    private static final int SOME_ID = 118558993;

    public Test() {
    }

    private int getID() {
        return 118558993;
    }
}
```
```java
//不带final修饰符
public class Test {
    private static int SOME_ID = 118558993;

    public Test() {
    }

    private int getID() {
        return SOME_ID;
    }
}
```
对比发现，带`final`时，getID()方法直接返回了数值；而不带`final`时，getID()方法返回的是SOME_ID，即依然保留着对变量的符号引用。这个看似不起眼的差别却暗藏玄机，看下图：
![构建历程](https://i.loli.net/2019/11/13/7RNFpBMxoKGOhly.png)
如图，每个AAR被接入后，都会跟随App工程再经历一次构建，即宿主工程的构建。而我们知道，R文件是跟随每次构建重新生成的（`不管是AAR的构建还是App的构建`），而R.java中每个域的值是由当前工程的资源集合做排列得出的，这意味着，假如工程的资源集合发生了变化，那么R.java中域的值都可能发生变化。对于AAR文件来说，自身工程的资源集合必然和宿主工程的资源集合不一样，或者可以这样理解，R.java的值是`一次性`的，它只保证在当前构建结果下有效，这次生成的R文件，不保证在下次构建后可用，这也是每次构建都重新生成R文件的原因。这样我们就找到了上面的崩溃原因和抛出的前两个问题的答案：AAR接入到App后跟随App又经历了一次构建，资源发生了重排列，所以手动打到AAR中的R文件ID值全部失效，无法再索引到资源，所以运行时崩溃。AAR中默认没有R文件，因为带上也完全不能用。同样因为重排列，AAR无法预知自身资源接入到App后的ID值，所以Library工程生成的R.java中的域都不能用final修饰。
`*：这里的逻辑很刁钻，但却很重要，是理解后续操作的前提，务必要反复品味...`

那要怎么办呢？其实关键在主模块身上：合并子模块后，主模块成了所有子模块的代言人，App只需接入主模块就等于接入了所有模块。能力越大责任越大，我们赋予了主模块如此特殊的地位，那么它也应该起到足够特殊的作用，比如桥接子模块到宿主的资源索引：

```java
//com.sub.library
public final class R {
  public static final class string {
    public static int some_string = com.main.library.R.string.some_string;
    }
}
```
如代码所示，我们在把子模块R文件放入classes.jar之前，手动将其中的域指向主模块R文件相同的域（`正是因为这里没有final修饰符，才能保留对主模块域的符号引用，编译后才不会变成数值而无法被更改`），因为主模块R文件会跟随每一次App构建而重新生成，所以它的ID值总是新鲜可靠的，这样子模块就可以通过这个桥接拿到正确的资源索引了。这时你可能会拍案而起，不对啊，主模块没有子模块里的资源文件，为什么R文件中会有跟子模块相同的域呢？！这其实就是上面提到的第三个问题：生成R文件流程的输出无疑是R.java，那么输入是什么？这个问题也牵扯到本文刚开始卖的关子：那个默认包含子模块内容的文件是谁？没错，就是`R.txt`（这个看似老实的东西其实坏的很 XD）。
我们知道，APK或AAR构建时会用Aapt编译资源文件，这个R.txt正是Aapt的产物，通过在`aapt package`命令后面跟上`--output-text-symbols`参数得到。R.txt中会记录下`-S`参数传入的资源目录下所有的资源文件信息，一个资源占一行，格式为`int type name id`，比如: int string some_string 0x7f050000（和R文件格式一致，所以猜到了吧，R.txt就是R.java的种子）。在AAR的构建中，R.txt自然记录了Library库内的资源文件信息，不过一个很不自然的事情是，构建时`aapt package`命令的`-S`参数传入的不是本工程/src/res目录，而是build/intermediates/res/`merged`/release目录，也就是合并子模块资源后的/res目录！所以，默认生成的R.txt中就已经包含了所有子模块的资源文件，而这个R.txt恰恰就是构建生成R.java的种子文件，脚本会根据R.txt中每一行的内容生成R.java中对应的域。这也就解释了为什么主模块的R.java会包含所有子模块的资源索引。终于，我们知道了如何处理R文件：
```gradle
task redirectRJava << {
  ...
  embeddedAarDirs.each { aarPath ->
      ...
      def sb = "package $subPackageName;" << '\n' << '\n'
       //将ID赋值为主模块包名下对应的值
      sb << "    public static $type $name = ${mainPackageName}.R.${subclass}.${name};" << '\n'
      ...
      //generated_rsrc_dir = "build/generated/source/r/release"
      mkdir("$generated_rsrc_dir/$packagePath")
      file("$generated_rsrc_dir/$packagePath/R.java").write(sb.toString())
  }
}

//将R文件打成Jar包放入libs目录
task embedRClass(type: org.gradle.jvm.tasks.Jar, dependsOn: collectRClass) {
    ...
    baseName "EmbedR"
    destinationDir file("$bundle_release_dir/libs")
    from base_r2x_dir
}
```
解决完R文件的难题后，我们就基本完成了AAR的合并工作，但是在实际业务中，会遇到另一个棘手的问题：Support库兼容问题。上面已经分析过，一个库的R文件中会包含它依赖的所有子模块的资源文件（还记得原因么），假如我们的模块依赖了Support库（实际业务中很常见，比如用到了v4的Fragment或者RecyclerView），那么R文件或R.txt中就会包含Support库内所有资源文件的信息，而当模块被接入后，在App的构建流程中，会根据最终的Support库版本（App依赖的Support库版本不可知）对各个R.txt里的内容做过滤，只保留App工程中确实存在的资源文件，生成最终的R文件（源代码见`com.android.builder.symbols.RGeneration`）。举个例子，假如模块使用的A版本的Support库中有资源文件res1，而接入方App使用了更高版本的Support库中没有res1这个资源，那么App构建生成的R文件中将没有res1这个资源索引，可是我们打入AAR中的R文件还保留着对res1的符号引用，结果就是在运行时类初始化失败（R.class的<cinit>错误）：
```java
public final class R {
  public static final class string {
    //这里的com.main.library.R.string.res1被过滤掉不再存在，符号引用失效
    public static int res1 = com.main.library.R.string.res1;
    }
}
```
怎么办呢？第一反应是保证support库版本一致（必须精确到小版本号），即针对每个接入方构建出跟其Support库一致的产物，这就带来了很多隐藏成本（还需要手动做R.txt的过滤），更何况假如接入方某天更新了Support库版本，直接就Runtime Crash了，这属于对接事故，显然不能接受。那么，应该怎么做呢？其实很简单：禁止Support库资源写入R.txt即可（其实官方也明确提示过不要自己使用Support库里的资源），还记得R.txt是根据什么生成的么？没错，是build/intermediates/res/merged/release目录下的资源合集，而这些文件又是根据遍历依赖合并进来的，假如我们能在合并时过滤掉Support库的依赖项，就在源头上过滤掉了Support库资源。所以我们只需要对合并Resources的Task稍做手脚：
```gradle
task filterMergeResource << {
    def oldInputResourceSet = task_mergeResources.inputResourceSets
    List<ResourceSet> newInputResourceSet = new ArrayList(oldInputResourceSet)
    newInputResourceSet.removeAll {
        //过滤com.android.support包名下的依赖，避免support库资源打入R.txt
        (null != it.libraryName) && (it.libraryName.contains("com.android.support"))
    }
    task_mergeResources.conventionMapping.map("inputResourceSets") {
        newInputResourceSet
    }
}
```
### 总结

![总结](https://i.loli.net/2019/11/13/l5iHQ3CTKvNg94n.png)
最终我们插入了以上几个自定义任务到不同的构建节点中，完成了合并AAR的构建改造，但仍然有拓展空间，比如支持buildType和productFlavor，有兴趣的同学可以自己尝试下（其实就是找到对应的build目录）。
> 参考：[https://github.com/adwiv/android-fat-aar](https://github.com/adwiv/android-fat-aar)