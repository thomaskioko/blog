---
title: "XCFramework Generation Using A Custom Gradle Plugin"
date: "2024-11-17"
draft: false
hideToc: true
tags: ["KMM",  "XCFramework", "Gradle", "GradlePlugin"]
series: "Tv Maniac Journey"
---

# Intro

Howdy folks! In my [previous article](https://thomaskioko.me/posts/ios_previews_with_kmm/), I discussed how modularizing the iOS codebase using Swift packages improved development efficiency and code organization. In this article, we'll explore how to create a new package for the shared code and automate XCFramework generation using a custom Gradle plugin.

## Background

After moving most of the reused UI-related code to Swift packages, I realized I could create a specific package for the shared code that can be used in the iOS app. This package would contain utilities that are used across the app. e.g 

1. **StateFlow**: A property wrapper in Swift that bridges Kotlin's StateFlow to SwiftUI's state management system.
2. **NavigationStack**: A custom SwiftUI navigation implementation that bridges Decompose's (Kotlin Multiplatform navigation library) ChildStack to SwiftUI's navigation system.

With that said, let's strap in and get this journey on the road.


## Creating a Custom XCFramework Gradle Plugin

Some of the classes we want to move depend on the shared code. We need to generate the XCFramework and copy that to our new package, TvManiacKit. We can then use this package in our app. Let's look at our custom Gradle plugin.

```kotlin
abstract class XCFrameworkPlugin : Plugin<Project> {
    override fun apply(project: Project) = with(project) {
        // Validate project configuration
        validateConfiguration()

        // Configure extension with defaults
        val extension = extensions.create<XCFrameworkExtension>("xcframework").apply {
            frameworkName.convention("TvManiac.xcframework")
            outputPath.convention("ios/Modules/TvManiacKit")
            cleanIntermediate.convention(true)
        }

        // Register tasks for different build configurations
        registerCopyXCFrameworkTask(
            nativeBuildType = nativeBuildType,
            targetType = nativeTargetType,
            extension = extension
        )

        // Register tasks for each build type/target combination
        nativeBuildTargetTypes.forEach { (buildType, targetType) ->
            registerCopyXCFrameworkTask(
                nativeBuildType = buildType,
                targetType = targetType,
                buildInfix = "${buildType.capitalizedName}${targetType.capitalizedName}",
                extension = extension
            )
        }
    }

  private fun Project.registerCopyXCFrameworkTask(
    nativeBuildType: NativeBuildType,
    targetType: NativeTargetType,
    buildInfix: String = "",
    extension: XCFrameworkExtension,
    ): TaskProvider<Copy> {

    // 1.
    val assembleXCFrameworkTask = tasks.register<XCFrameworkTask>("assemble${buildInfix}XCFramework") {
      group = GROUP_NAME
      buildType = nativeBuildType

      val frameworks = multiplatformExtension.nativeFrameworks(nativeBuildType, targetType.targets)
      from(*frameworks.toTypedArray())
    }

    // 2.
    return tasks.register<Copy>("copy${buildInfix}XCFramework") {
      group = GROUP_NAME
      description = "Copies the $buildInfix XCFramework to ${extension.outputPath.get()}"

      val outputDir = project.rootProject.projectDir.resolve(
        "${extension.outputPath.get()}/${extension.frameworkName.get()}",
      )

      // 3.
      dependsOn(assembleXCFrameworkTask)

      // 4.
      from(
        assembleXCFrameworkTask.map {
          it.outputDir.resolve(nativeBuildType.getName())
            .resolve("${project.name}.xcframework")
        },
      )

      // 5.
      into(outputDir)

      // 6.
      doFirst {
        outputDir.deleteRecursively()
      }
    }
  }
}
```

Let's break down the plugin.

1. **Register the assemble task**: This task is responsible for assembling the XCFramework. The buildInfix param allow us to create different build for different configurations. We will have `assembleDebugDeviceXCFramework` and `assembleReleaseXCFramework`.
2. **Register the copy task**: This task is responsible for copying the assembled XCFramework to the output directory. 
3. **Require the aassemble task**: This ensures that the XCFramework is assembled before it is copied.
4. **Copy the assembled XCFramework to the output directory**: This copies the XCFramework to the output directory.
5. **Clean the DerivedData directory**: We need to clean this up to ensure we have no conflicts or corrupt data from the previous builds. 

`XCFrameworkExtension` interface defines the configuration options for the plugin. This is completely optional, but it allows us to configure the plugin with custom values if we need to use the pluginin another module. We simply need to add this in our `build.gradle.kts` file:


```kotlin
xcframework {
    frameworkName.set("CoolName.xcframework")
    outputPath.set("ios/Modules/CoolNameKit")
    cleanIntermediate.set(true)
}
```

### Adding The Plugin

We are almost there. We just nedd to do two things on the Kotlin side of things and we are done. 

1. We first need to register the XCFramework plugin in our build-plugins `build.gradle.kts` file:

```kotlin
gradlePlugin {
  plugins {
    ...

    register("xcframework") {
      id = "plugin.tvmaniac.xcframework"
      implementationClass = "com.thomaskioko.tvmaniac.plugins.XCFrameworkPlugin"
    }
  }
}
```

2. With that done, we can apply the plugin in our shared module`build.gradle.kts` file:

```kotlin
plugins {
    alias(libs.plugins.tvmaniac.xcframework)
}

```

## Integration with Xcode

Now that our plugin is ready, we need to add it to the Xcode build phase in oder to generated the XCFramework before we build the app. It's pretty straightforward:

1. Select your target and click on "Edit Scheme"
2. Go to "Build" -> **"Pre-actions"**
3. Click "+" to add a new phase
4. Select "Run Script"
5. Add the following script:

``` bash
cd "$SRCROOT/.."
./gradlew :shared:copyXCFramework
```

## Fastlane Build

For CI environments, we must add a gradle task to the Fastlane lane job so that the XCFramework is generated before building the app.

``` ruby
desc "Build iOS App"
lane :build_tvmaniac do

    gradle(
        task: ":shared:copyXCFramework"
    )

   # Previous steps to build the app
end
```

## KMMBridge Shoutout
I got a lot of inspiration from the [KMMBridge](https://github.com/touchlab/KMMBridge) project. You should check it out! I tried it a while back and it's pretty neat. However, I felt like it was an overkill for my project. I was also looking for an opportunity to do some Gradle scripting.

If your project requires any of the following, consider using KMMBridge instead:

- Publishing frameworks to external repositories
- Managing multiple framework versions
- Supporting multiple teams or external consumers
- Complex dependency management
- CocoaPods or SPM integration


## Conclusion

Creating a custom Gradle plugin for XCFramework generation has significantly improved our development workflow. We can now have a dedicated package for the shared code used in the iOS app. It was a great experience getting to dance with Gradle.

You can find the complete project here. [TV Maniac repository](https://github.com/thomaskioko/tv-maniac).

Until we meet again, folks. Happy coding! ✌️

---

### References
- [KMMBridge Documentation](https://github.com/touchlab/KMMBridge)
- [Building an XCFramework on Kotlin Multiplatform from Kotlin 1.5.30](https://www.marcogomiero.com/posts/2021/kmp-xcframework-official-support/)
- [Build final native binaries](https://kotlinlang.org/docs/multiplatform-build-native-binaries.html)