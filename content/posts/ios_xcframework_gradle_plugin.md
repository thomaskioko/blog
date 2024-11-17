---
title: "Streamlining iOS Development with Custom XCFramework Gradle Plugin in KMM"
date: "2024-11-17"
draft: false
hideToc: true
tags: ["KMM",  "XCFramework", "Gradle", "GradlePlugin"]
series: "Tv Maniac Journey"
---

# Intro

Howdy folks! The journey continues. In my [previous article](https://thomaskioko.me/posts/ios_previews_with_kmm/), I discussed how modularizing the iOS codebase using Swift packages improved development efficiency and code organization. In this article, we'll explore how we can create an new package for the shared code and automate XCFramework generation using a custom Gradle plugin.

## Background

After I was done moving UI related code to Swift packages, I realized I could create a specific package for the shared code that would be used in the iOS app. This package would contain utilities that are shared between the app and the shared code. e.g 
1. **StateFlow**: A property wrapper in Swift that bridges Kotlin's StateFlow to SwiftUI's state management system.
2. **NavigationStack**: A custom SwiftUI navigation implementation that bridges Decompose's (Kotlin Multiplatform navigation library) ChildStack to SwiftUI's navigation system.

Let's look at how we can streamline the distribution of these utilities by creating a new package, moving the classes to the package and creating a custom Gradle plugin to handle the XCFramework generation.


## The XCFramework Plugin

Let's look at our custom Gradle plugin that handles XCFramework generation and distribution. The plugin consists of two main components:

### 1. The Plugin 

The main plugin class handles the framework generation and distribution logic:

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
}
```

### 2. The XCFramework Interface

The Interface defines the configuration options for the plugin:

```kotlin
interface XCFrameworkExtension {
    val frameworkName: Property<String>
    val outputPath: Property<String>
    val cleanIntermediate: Property<Boolean>
}
```

This is completely optional, but it allows us to configure the plugin with custom values if we need to like. We can do this in our `build.gradle.kts` file:

```kotlin
xcframework {
    frameworkName.set("CoolName.xcframework")
    outputPath.set("ios/Modules/CoolNameKit")
    cleanIntermediate.set(true)
}
```


### Key Features

The plugin provides several key features:

1. **Automated Task Registration**
   - Generates tasks for different build configurations (Debug/Release)
   - Supports multiple target types (Device/Simulator). We can also add support for other targets like WatchOS, etc.
   - Creates assembly and copy tasks automatically

2. **Framework Management**
   - Cleans existing frameworks before copying. We do this to ensure we don't have any old frameworks laying around as they could be corrupt or have conflicts.
   - Handles intermediate file cleanup
   - Maintains proper framework directory structure

3. **Build Configuration Support**
   - Validates project setup
   - Integrates with Kotlin Multiplatform configuration
   - Supports custom build types and targets

4. **Error Handling**
   - Validates required plugins and directories
   - Provides clear error messages for missing configurations
   - Ensures proper setup before execution

### Usage

We first need to register the XCFramework plugin in our build-plugins `build.gradle.kts` file:

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

With that done, we can apply the plugin in our shared module`build.gradle.kts` file:

```kotlin
plugins {
    alias(libs.plugins.tvmaniac.xcframework)
}

```

The plugin will create several Gradle tasks:
- `assembleXCFramework`: Assembles the framework for default configuration
- `copyXCFramework`: Copies the assembled framework to the specified location
- Additional tasks for specific build types (e.g., `assembleDebugDeviceXCFramework`)

## Integration with Xcode

To ensure our XCFramework is always up-to-date, we add it to Xcode's pre-build phase. In your Xcode project:

1. Select your target and click on "Edit Scheme"
2. Go to "Build" -> **"Pre-actions"**
3. Click "+" to add a new phase
4. Select "Run Script"
5. Add the following script:

``` bash
cd "$SRCROOT/.."
./gradlew :shared:copyXCFramework
```

This automatically rebuilds and copies the framework before each Xcode build.

## CI Integration with Fastlane

For CI environments, we simply need to add a gradle task to the Fastlane lane job. We need to build the XCFramework before building the app as the XCFramework is used in the app.

``` ruby
desc "Build iOS App"
lane :build_tvmaniac do

    gradle(
        task: ":shared:copyXCFramework"
    )

   # Previous steps to build the app
end
```

# Alternative Approach: KMMBridge

While we've implemented a custom solution for XCFramework generation, it's worth mentioning [KMMBridge](https://github.com/touchlab/KMMBridge), a powerful tool developed by Touchlab that solves similar problems and more. KMMBridge is designed to streamline the distribution of Kotlin Multiplatform Mobile libraries to iOS developers, offering features like:

- Automated XCFramework generation and publishing
- SPM (Swift Package Manager) integration
- CocoaPods support
- Remote artifact hosting
- Version management
- Build configuration management

## Why Not KMMBridge?
While KMMBridge is an excellent solution, especially for larger projects or teams, I chose to implement a simpler custom plugin for several reasons:

1. **Project Scale**: My project is relatively small, with a manageable number of modules. KMMBridge's comprehensive feature set would be overkill for my needs.
2. **No Binary Publishing**: I don't need to publish my frameworks to external repositories or share them across multiple projects. Everything stays within the same monorepo.
3. **Build System Control**: The custom plugin gives me precise control over the build process and integrates seamlessly with my existing gradle tasks.
4. **Learning Opportunity**: Building a custom solution provided valuable insights into Gradle plugin development and KMM build processes.

If your project requires any of the following, consider using KMMBridge instead:

- Publishing frameworks to external repositories
- Managing multiple framework versions
- Supporting multiple teams or external consumers
- Complex dependency management
- CocoaPods or SPM integration


## Conclusion

Creating a custom Gradle plugin for XCFramework generation has significantly improved our development workflow and we can now have a dedicated package for the shared code that is used in the iOS app. This was a great experience getting to dance with Gradle. I got a lot of inspiration from the [KMMBridge](https://github.com/touchlab/KMMBridge) project. You should check it out!

The complete implementation can be found in the [TV Maniac repository](https://github.com/thomaskioko/tv-maniac).

Until we meet again, folks. Happy coding! ✌️

---

### References
- [KMMBridge Documentation](https://github.com/touchlab/KMMBridge)
- [Kotlin Multiplatform Documentation](https://kotlinlang.org/docs/multiplatform.html)