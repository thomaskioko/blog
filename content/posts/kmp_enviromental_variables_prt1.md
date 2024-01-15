---
title: "KMP Environment Variables: Part 1"
date: 2023-07-10
draft: false
hideToc: false
tags: ["KMP"]
series: ["Tv Maniac Journey"]
---

![Photo by Scott Webb](https://www.thomaskioko.com/wp-content/uploads/2023/05/Environment-Variables-1000x564.jpeg)

In this article, I will talk about my experience working with environment variables on Kmm (Kotlin Multiplatform) and how I am currently using it in my project [Tv-Maniac.](https://github.com/thomaskioko/tv-maniac)

### Environment Variables
During the development lifecycle of a mobile app, you probably may be create apps that use API keys or passwords. It is best practise to store such sensitive info in a secure place. In Android, you'd ideally use `local.properties` or `gradle.properties` to set this up and on iOS, configuration files commonly known as `xcconfig`. The good thing about this is you can configure it to support multiple builds/targets: eg, `development`, `QA` and `production` depending on your needs.

Now that we have that in mind, how do we go about this for a KMM project? There are multiple ways you can do this but I will only talk about the first two since it's what I have worked with. 
1. Custom gradle plugin
2. [BuildKonfig](https://github.com/yshrsmz/BuildKonfig)
3. Config file reader
4. [gradle-buildconfig-plugin](https://github.com/gmazzo/gradle-buildconfig-plugin) I recently stumbled on this.


### 1. Custom Gradle Task
I will not go in depth on this but just give an overview. We can use some Gradle utils to generate code from build properties. There are three main concepts that we will need to make this possible. 
1. Creating a gradle task.
2. Defining the generated file.
3. Setting the source set

With that in mind we will need to create a task that produces a file as output. We can use [Sync](https://docs.gradle.org/current/dsl/org.gradle.api.tasks.Sync.html)for this. According to Gradle's documentation, it synchronizes the contents of a destination directory with some source directories and files. It's more like a copy task. Once we have that in place we can use [`TestResourceFactory`](https://docs.gradle.org/current/dsl/org.gradle.api.resources.TextResourceFactory.html) to create files dynamically. If you have a custom source set, you will need to define it otherwise gradle won't recognise it.

You can check out [this answer](https://stackoverflow.com/a/74771876) by @aSemy on StackOverflow.

### 2. [BuildKonfig](https://github.com/yshrsmz/BuildKonfig)
This is a library that basically embeds values from a gradle file. If you are an Android dev, this will feel more like home. It's pretty simple

``` groovy
buildkonfig {
    packageName = 'com.thomaskioko.tvmaniac.shared'

     defaultConfigs {
        buildConfigField 'STRING', 'name', 'value'
        buildConfigField 'STRING', 'nullableField', null, nullable: true
    }
}
```

To read from `gradle.properties`, I created a small function

``` kotlin
fun <T : Any> propOrDef(propertyName: String, defaultValue: T): T {  
    @Suppress("UNCHECKED_CAST")  
    val propertyValue = project.properties[propertyName] as T?  
    return propertyValue ?: defaultValue  
}
```

Below is how we use the custom function and buildConfig to read from `gradle.properies`

``` groovy
buildkonfig {  
    packageName = "com.example.app"  
  
    defaultConfigs {  
        buildConfigField(  
            com.codingfeline.buildkonfig.compiler.FieldSpec.Type.STRING,  
            "TMDB_API_KEY",  
            "\"" + propOrDef("TMDB_API_KEY", "") + "\""  
        )  
        buildConfigField(  
            com.codingfeline.buildkonfig.compiler.FieldSpec.Type.STRING,  
            "TRAKT_CLIENT_ID",  
            "\"" + propOrDef("TRAKT_CLIENT_ID", "") + "\""  
        )   
    }  
}
```

To generate the files, run `./gradlew generateBuildKonfig`. 

``` kotlin
// commonMain
package com.example.app

internal object BuildKonfig {
    val TMDB_API_KEY: String = "value"
    val TRAKT_CLIENT_ID: String = "value"
}
```


### 3. Config File Reader
We can create a resource reader to help up read from `yaml` configuration files. The good thing about this is we can easily create multiple environment files for different flavours. Builkonfig also supports this but I has a hard time while switching between environments in a different project.

This implementation is from Touchlab’s [Droidcon KMM App](https://github.com/touchlab/DroidconKotlin) which has a neat way or reading files from both iOS and Android. I'd say this is the meat of it. `YamlResourceReader` helps us serialize the yaml file into a data class object. `readAndDecodeResource` takes in two params:
1. `name`: Name of the yaml file. We have this as a parameter so that you can pass in different files based on the build target you want.
2. `strategy`: Serialization strategy. We will be using `kotlinx.serialization` but you could also use `Gson` or `Jakcson`

``` kotlin
class YamlResourceReader(  
    private val resourceReader: ResourceReader,  
) {  
    internal fun <T> readAndDecodeResource(name: String, strategy: DeserializationStrategy<T>): T {  
        val text = resourceReader.readResource(name)  
        return Yaml.decodeFromString(strategy, text)  
    }  
}
```

``` kotlin
import kotlinx.serialization.Serializable  
  
@Serializable  
data class Configs(  
    val isDebug: Boolean,  
    val tmdbApiKey: String,  
    val traktClientId: String,
    val baseUrl: String
    ...
)
```


#### Android Platform Implementation
We could also use Android's `AssetManager` to read from the bundled assets `context.assets.open(name)` but this example we are using `javaClass.classLoader` 

``` kotlin
// Android
class ClasspathResourceReader : ResourceReader {  
    override fun readResource(name: String): String {  
        return javaClass.classLoader?.getResourceAsStream(name).use { stream ->  
            InputStreamReader(stream).use { reader ->  
                reader.readText()  
            }  
        }    
    }  
}
```


#### iOS Platform Implementation

``` kotlin
class BundleResourceReader(  
    private val bundle: NSBundle = NSBundle.bundleForClass(BundleMarker),  
) : ResourceReader {  
  
    override fun readResource(name: String): String {   
        val (filename, type) = when (val lastPeriodIndex = name.lastIndexOf('.')) {  
            0 -> {  
                null to name.drop(1)  
            }  
            in 1..Int.MAX_VALUE -> {  
                name.take(lastPeriodIndex) to name.drop(lastPeriodIndex + 1)  
            }  
            else -> {  
                name to null  
            }  
        }  
        val path = bundle.pathForResource(filename, type) ?: error("Couldn't get path of $name (parsed as: ${listOfNotNull(filename, type).joinToString(".")})")  
  
        return memScoped {  
            val errorPtr = alloc<ObjCObjectVar<NSError?>>()  
  
            NSString.stringWithContentsOfFile(path, encoding = NSUTF8StringEncoding, error = errorPtr.ptr) ?: run {  
                error("Couldn't load resource: $name. Error: ${errorPtr.value?.localizedDescription} - ${errorPtr.value}")  
            }  
        }    }  
  
    private class BundleMarker : NSObject() {  
        companion object : NSObjectMeta()  
    }  
}  
  
data class BundleProvider(val bundle: NSBundle)
```


#### Resources Folder
We then need to add our configuration files. Kmm libraries have a structure folder for resources. You can name them whatever you want. `qa.yaml` `release.yaml` depending on your needs. In this case, `config.yaml`.

![[Kmm Resource Folder.png]](https://miro.medium.com/v2/resize:fit:640/format:webp/0*lIt5FXXky-LbbhEa.png)

We need to drag or add the file on iOS so it's accessible. For uniformity, I have created a directory called Resources and added the file there.

![[iOS Structure.png]](https://miro.medium.com/v2/resize:fit:466/format:webp/0*1AsI1TMolqzwpePs.png)

One extra step that I did was to create a symlink in the root directory so we can easily access it. (This is optional)

#### Reader Implementation
Tying things up together. We can now use the resource reader and pass the intend file

``` kotlin
val configObject : Config = resourceReader.readAndDecodeResource("config.yaml", Configs.serializer())

configs.baseUrl
configs.isDebug
configs.tmdbApiKey
```

### Conclusion
There are two things we need to do:
1. Remember to add the files on iOS.
2. Depending on your application, you might want to ignore the file so that you don't end up publishing sensitive info. 

With that, all our resources are in one directory. We could definitely take this further and implement different build targets. I'll do this in a follow-up article. 

There might be a better way but this is what has worked of me. I'm always up to learn from you so feel free to poke holes in my implementation. 😅

### Resources
- [DroidconKotlin repo](https://github.dev/touchlab/DroidconKotlin)