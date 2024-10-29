---
date: 2024-10-17
authors:
    - pablobaxter
draft: true
---

# Detecting File Leak in the Kotlin Daemon

File handle leaks are notoriously difficult to debug, so much so that most of the “fixes” for them are “increase the file descriptor limit”. However, this is not a fix. This is as close as you can get to covering your ears and closing your eyes, then screaming “I DON’T HEAR YOU” to a bug. The file handle leak will still exist, but now it’s ignored.

<figure markdown="span">
  ![img](../../assets/gifs/file-leak.gif){ loading=lazy }
  <figcaption>Seriously, it's exactly this</figcaption>
</figure>

In this case study, I decided to take the approach of finding out why the leak was occurring, and finding the best fix for it.

<!-- more -->

## Signs of a File Handle Leak

The hardest part of debugging an issue is noticing it to begin with and file handle leaks are very hard to notice until an error is thrown, and even then it may not be noticed since the error could be handled by some logic that changes the exception cause or swallows it altogether. So how did I notice the one in the Kotlin daemon? Dumb luck. I had been tracking another file handle leak in the Gradle daemon, and just got curious about how many files remained open in the Kotlin daemon after compilation.

There are some signs that your application may have a file handle leak, but even these signs are easily missed. A few common signs are:

* Errors relating to files unable to be created
* Any error about a file being deleted (especially on Windows)
* Unable to open files for any reason
* Files having garbage data written or being corrupted

These issues don't always appear on every run and could be intermittent as well. 

## Noticing the Kotlin Daemon File Handle Leak

Going back to how I found the Kotlin daemon file handle leak, as I had mentioned previously, it was due to curiosity and dumb luck. When Paul Klauser [reported a metaspace leak](https://youtrack.jetbrains.com/issue/KT-72169/Kotlin-Daemon-Metaspace-leak) occuring in the Kotlin daemon, I was curious if this metaspace leak was potentially due to a file handle leak, as I had been tracking one in the Gradle daemon (which I still haven't found).

At a high-level (technical details below), I attached Jenkin's [file-leak-detector](https://github.com/jenkinsci/lib-file-leak-detector) to the Kotlin daemon, and ran `./gradlew clean assembleDebug --rerun-tasks` several times and capturing the output of the file leak tool and using a post-processor (see [https://github.com/centic9/file-leak-postprocess](https://github.com/centic9/file-leak-postprocess)) to make the logs easier to read. I ended up with a file containing many stacktraces that show where the file was opened. Many of them looked like the following (shortened for easy reading):

``` text title="Actual file leak ouput" linenums="1"
// Several hundred other open files up here with the same stacktrace
...
#1295 /home/pablo/.gradle/caches/modules-2/files-2.1/org.jetbrains.kotlin/kotlin-stdlib-jdk8/1.9.0/e000bd084353d84c9e888f6fb341dc1f5b79d948/kotlin-stdlib-jdk8-1.9.0.jar by thread:RMI TCP Connection(10)-127.0.0.1 on Wed Oct 09 18:35:43 PDT 2024
#542 /home/pablo/.gradle/caches/modules-2/files-2.1/org.checkerframework/checker-qual/3.41.0/8be6df7f1e9bccb19f8f351b3651f0bac2f5e0c/checker-qual-3.41.0.jar by thread:RMI TCP Connection(21)-127.0.0.1 on Wed Oct 09 18:35:52 PDT 2024
#341 /home/pablo/.gradle/caches/modules-2/files-2.1/com.google.guava/listenablefuture/9999.0-empty-to-avoid-conflict-with-guava/b421526c5f297295adef1c886e5246c39d4ac629/listenablefuture-9999.0-empty-to-avoid-conflict-with-guava.jar by thread:RMI TCP Connection(140)-127.0.0.1 on Wed Oct 09 18:36:13 PDT 2024
	at java.base/java.util.zip.ZipFile.<init>(ZipFile.java:181)
	at java.base/java.util.jar.JarFile.<init>(JarFile.java:346)
	at java.base/jdk.internal.loader.URLClassPath$JarLoader.getJarFile(URLClassPath.java:825)
	at java.base/jdk.internal.loader.URLClassPath$JarLoader$1.run(URLClassPath.java:769)
	at java.base/jdk.internal.loader.URLClassPath$JarLoader$1.run(URLClassPath.java:762)
    ...
	at java.base/java.util.ServiceLoader$2.hasNext(ServiceLoader.java:1309)
	at java.base/java.util.ServiceLoader$3.hasNext(ServiceLoader.java:1393)
	at com.google.common.collect.ImmutableSet.copyOf(ImmutableSet.java:280)
	at com.google.common.collect.ImmutableSet.copyOf(ImmutableSet.java:265)
	at dagger.internal.codegen.ServiceLoaders.loadServices(ServiceLoaders.java:35)
	at dagger.internal.codegen.DelegateComponentProcessor.lambda$initialize$0(DelegateComponentProcessor.java:89)
    ...
	at dagger.internal.codegen.DelegateComponentProcessor.initialize(DelegateComponentProcessor.java:89)
	at dagger.internal.codegen.KspComponentProcessor.initialize(KspComponentProcessor.java:49)
	at dagger.spi.internal.shaded.androidx.room.compiler.processing.ksp.KspBasicAnnotationProcessor.process(KspBasicAnnotationProcessor.kt:57)
	at com.google.devtools.ksp.AbstractKotlinSymbolProcessingExtension$doAnalysis$8$1.invoke(KotlinSymbolProcessingExtension.kt:310)
	at com.google.devtools.ksp.AbstractKotlinSymbolProcessingExtension$doAnalysis$8$1.invoke(KotlinSymbolProcessingExtension.kt:308)
	at com.google.devtools.ksp.AbstractKotlinSymbolProcessingExtension.handleException(KotlinSymbolProcessingExtension.kt:414)
	at com.google.devtools.ksp.AbstractKotlinSymbolProcessingExtension.doAnalysis(KotlinSymbolProcessingExtension.kt:308)
	at org.jetbrains.kotlin.cli.jvm.compiler.TopDownAnalyzerFacadeForJVM.analyzeFilesWithJavaIntegration(TopDownAnalyzerFacadeForJVM.kt:112)
	at org.jetbrains.kotlin.cli.jvm.compiler.TopDownAnalyzerFacadeForJVM.analyzeFilesWithJavaIntegration$default(TopDownAnalyzerFacadeForJVM.kt:75)
    ...
```

> So what's going on here and how does this tell the story of a file leak?

Well, this stacktrace actually represents 417 files were opened and never closed, but for brevity, I only listed the final three files left open (lines 3-5). These were 

So I performed the following setup in the [NowInAndroid](https://github.com/android/nowinandroid) repo, which is what Paul Klauser used to report the metaspace leak.

### File Handle Leak Tooling




```properties title="gradle.properties"

kotlin.daemon.jvmargs=-Dfile.encoding=UTF-8 -XX:+UseG1GC -XX:SoftRefLRUPolicyMSPerMB=1 -XX:ReservedCodeCacheSize=320m -XX:+HeapDumpOnOutOfMemoryError -Xmx4g -Xms4g -javaagent:/path/to/

```