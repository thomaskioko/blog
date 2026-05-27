------
title: "XCFramework Generation Using A Custom Gradle Plugin"
date: "2024-11-17"
draft: false
hideToc: true
tags: ["KMM",  "XCFramework", "Gradle", "GradlePlugin"]
series: "Tv Maniac Journey"
---

Howdy folks! In my [previous article](https://thomaskioko.me/posts/ios_previews_with_kmm/), I discussed how modularizing the iOS codebase using Swift packages improved development efficiency and code organization. In this article, we'll explore how to create a new package for the shared code and automate XCFramework generation using a custom Gradle plugin.

I subsequently created a dedicated package for shared code and automated XCFramework generation using a custom Gradle plugin. This package (TvManiacKit) contains utilities used across the application such as `StateFlow` property wrappers and `NavigationStack` implementations.

## Custom XCFramework plugin implementation

I developed a custom Gradle plugin to manage XCFramework assembly and copying. The plugin validates project configuration, establishes default output paths, and registers tasks for various build types.

```kotlin
abstract class XCFrameworkPlugin : Plugin<Project> {
    override fun apply(project: Project) = with(project) {
        val extension = extensions.create<XCFrameworkExtension>("xcframework").apply {
            frameworkName.convention("TvManiac.xcframework")
            outputPath.convention("ios/Modules/TvManiacKit")
            cleanIntermediate.convention(true)
        }

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

    val assembleXCFrameworkTask = tasks.register<XCFrameworkTask>("assemble${buildInfix}XCFramework") {
      group = GROUP_NAME
      buildType = nativeBuildType
      val frameworks = multiplatformExtension.nativeFrameworks(nativeBuildType, targetType.targets)
      from(*frameworks.toTypedArray())
    }

    return tasks.register<Copy>("copy${buildInfix}XCFramework") {
      group = GROUP_NAME
      description = "Copies the $buildInfix XCFramework to ${extension.outputPath.get()}"
      val outputDir = project.rootProject.projectDir.resolve(
        "${extension.outputPath.get()}/${extension.frameworkName.get()}",
      )
      dependsOn(assembleXCFrameworkTask)
      from(assembleXCFrameworkTask.map { it.outputDir.resolve(nativeBuildType.getName()).resolve("${project.name}.xcframework") })
      into(outputDir)
      doFirst { outputDir.deleteRecursively() }
    }
  }
}
```

The plugin automates assembling the framework for specific configurations (e.g., debug or device) and copying the result to the target directory. It also manages cleanup of intermediate build data to prevent conflicts.

### Plugin registration and application

Register the plugin in the project's build-plugins configuration:

```kotlin
gradlePlugin {
  plugins {
    register("xcframework") {
      id = "plugin.tvmaniac.xcframework"
      implementationClass = "com.thomaskioko.tvmaniac.plugins.XCFrameworkPlugin"
    }
  }
}
```

Apply the plugin to the shared module:

```kotlin
plugins {
    alias(libs.plugins.tvmaniac.xcframework)
}
```

## Integration with Xcode build phases

I configured Xcode to generate the XCFramework before building the application. I added a pre-action script to the build scheme:

``` bash
cd "$SRCROOT/.."
./gradlew :shared:copyXCFramework
```

## Automate CI builds with Fastlane

For continuous integration, I added a Gradle task to the Fastlane lane to ensure the framework is generated prior to the build.

``` ruby
desc "Build iOS App"
lane :build_tvmaniac do
    gradle(task: ":shared:copyXCFramework")
end
```

## Final considerations

This custom plugin improves the development workflow by automating framework generation and distribution. While projects with complex requirements might benefit from KMMBridge, this solution provides a lightweight alternative for localized framework management. The complete implementation is available in the [Tv Maniac repository](https://github.com/thomaskioko/tv-maniac).

Until we meet again, folks. Happy coding! ✌️
