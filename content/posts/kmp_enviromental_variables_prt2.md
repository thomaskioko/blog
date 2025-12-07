---
title: "KMP Environment Variables: Part 2 - Simplifying with BuildConfig"
date: "2025-11-18"
draft: false
hideToc: true
tags: ["KMP", "Gradle Plugin"]
series: "Tv Maniac Journey"
---

![Photo by Scott Webb](https://www.thomaskioko.com/wp-content/uploads/2023/05/Environment-Variables-1000x564.jpeg)

In [Part 1](https://thomaskioko.me/posts/kmp_enviromental_variables_prt1/), I walked through different approaches for handling environment variables in Kotlin Multiplatform projects. Fast forward a couple of years, and I've learned quite a bit about what works at scale and what doesn't. In this article, I'll share how I evolved my configuration approach in [Tv-Maniac](https://github.com/thomaskioko/tv-maniac) and why I eventually moved to a custom BuildConfig plugin.

## The Problem with YAML Files

The YAML based approach I described in Part 1 worked well initially, but as the project grew, some pain points became apparent:

1. **Platform Duplication**: I had a separate YAML files for Android (`core/util/src/androidMain/resources/`) and iOS (`ios/ios/Resources/`). This meant maintaining the same configuration in two places.

2. **Type Safety**: YAML parsing required serialization/deserialization, adding another layer where things could go wrong. A typo in the YAML file wouldn't be caught until runtime.

3. **Developer Experience**: New developers had to remember to copy template files for both platforms, edit them in multiple locations, and ensure they're in sync. This friction adds up.

5. **CI/CD Complexity**: I had to manage separate configuration files for different environments, which made our CI/CD setup more complex than it needed to be.

## Enter: Custom BuildConfig Plugin

After evaluating options, I decided to build a custom Gradle plugin that generates BuildConfig files from `local.properties` or environment variables. This approach gives us:

- **Single source of truth**: One place to define all configuration
- **Compile-time constants**: Values embedded during build time
- **CI/CD friendly**: Falls back to environment variables seamlessly
- **Cross platform**: Works identically for Android and iOS

## Implementation

### 1. The BuildConfig Extension

First, we need a Gradle extension that provides a clean DSL for defining configuration fields:

```kotlin
public abstract class BuildConfigExtension(private val project: Project) {

    public abstract val packageName: Property<String>
    public abstract val stringFields: MapProperty<String, String>
    public abstract val booleanFields: MapProperty<String, Boolean>
    public abstract val intFields: MapProperty<String, Int>

    private val localProperties: Properties by lazy {
        val props = Properties()
        val localPropertiesFile = project.rootProject.file("local.properties")
        if (localPropertiesFile.exists()) {
            localPropertiesFile.inputStream().use { props.load(it) }
        }
        props
    }

    public fun stringField(name: String, value: String) {
        stringFields.put(name, value)
    }

    public fun booleanField(name: String, value: Boolean) {
        booleanFields.put(name, value)
    }

    public fun intField(name: String, value: Int) {
        intFields.put(name, value)
    }

    public fun buildConfigField(name: String) {
        val value = localProperties.getProperty(name) ?: System.getenv(name)
        requireNotNull(value) { "$name not found in local.properties or environment variables" }

        stringFields.put(name, value)
    }
}
```

The `buildConfigField()` function reads from `local.properties` first (for local development), then falls back to environment variables (for CI/CD). This gives us the best of both worlds. I might make this more dynamic in future and have the consumer specify the path.

### 2. The Code Generation Task

Next, we create a Gradle task that generates the actual Kotlin code:

```kotlin
public abstract class GenerateBuildConfigTask : DefaultTask() {

    @get:Input
    public abstract val packageName: Property<String>

    @get:Input
    public abstract val intFields: MapProperty<String, Int>

    @get:OutputDirectory
    public abstract val outputDir: DirectoryProperty

    @TaskAction
    public fun generate() {
        val packageNameValue = packageName.get()
        val outputDirectory = outputDir.get().asFile

        outputDirectory.deleteRecursively()
        outputDirectory.mkdirs()

        val fileContent = buildString {
            appendLine("package $packageNameValue")
            appendLine()
            appendLine("public object BuildConfig {")

            // Generate string fields
            stringFields.get().forEach { (name, value) ->
                appendLine("    public const val $name: String = \"$value\"")
            }

            // Generate boolean fields
            booleanFields.get().forEach { (name, value) ->
                appendLine("    public const val $name: Boolean = $value")
            }

            // Generate int fields
            intFields.get().forEach { (name, value) ->
                appendLine("    public const val $name: Int = $value")
            }

            appendLine("}")
        }

        val packagePath = packageNameValue.replace('.', '/')
        val targetDir = File(outputDirectory, packagePath)
        targetDir.mkdirs()

        File(targetDir, "BuildConfig.kt").writeText(fileContent)
    }
}
```

### 3. The Plugin

The plugin ties everything together and integrates with the Kotlin Multiplatform setup:

```kotlin
public class BuildConfigPlugin : Plugin<Project> {
    override fun apply(target: Project) {
        val extension = target.extensions.create("buildConfig", BuildConfigExtension::class.java, target)

        target.plugins.withId("org.jetbrains.kotlin.multiplatform") {
            val kotlin = target.extensions.getByType<KotlinMultiplatformExtension>()

            val generateTask = target.tasks.register("generateBuildConfig", GenerateBuildConfigTask::class.java) {
                packageName.set(extension.packageName)
                stringFields.set(extension.stringFields)
                booleanFields.set(extension.booleanFields)
                intFields.set(extension.intFields)
                outputDir.set(target.layout.buildDirectory.dir("generated/buildconfig/commonMain"))
            }

            kotlin.sourceSets.getByName("commonMain") {
                kotlin.srcDir(generateTask.map { it.outputDir })
            }

            target.tasks.named("compileKotlinMetadata") {
                dependsOn(generateTask)
            }
        }
    }
}
```

### 4. Usage in build.gradle.kts

With the plugin in place, configuration becomes beautifully simple:

```kotlin
plugins {
    alias(libs.plugins.app.kmp)
    id("io.github.thomaskioko.gradle.plugins.buildconfig")
}

buildConfig {

    booleanField("IS_DEBUG", true)
    stringField("TMDB_BASE_URL", "https://api.themoviedb.org/3")
    stringField("TRAKT_BASE_URL", "https://api.trakt.tv")

    buildConfigField("TMDB_API_KEY")
    buildConfigField("TRAKT_CLIENT_ID")
    buildConfigField("TRAKT_CLIENT_SECRET")
    buildConfigField("TRAKT_REDIRECT_URI")
}
```

### 5. Accessing Configuration

On Android, it's straightforward:

```kotlin
class TraktAuthAndroidComponent {
    @Provides
    fun provideAuthRequest(
        configuration: AuthorizationServiceConfiguration,
    ): AuthorizationRequest = AuthorizationRequest.Builder(
        configuration,
        BuildConfig.TRAKT_CLIENT_ID,
        ResponseTypeValues.CODE,
        BuildConfig.TRAKT_REDIRECT_URI.toUri(),
    ).build()
}
```

Then on iOS:

```swift
init(presenter: SettingsPresenter, authRepository: TraktAuthRepository, logger: Logger) {
    self.presenter = presenter
    _uiState = .init(presenter.state)

    // Read configuration from BuildConfig (shared KMP code)
    guard let redirectURL = URL(string: BuildConfig.shared.TRAKT_REDIRECT_URI) else {
        fatalError("Invalid Trakt redirect URI in BuildConfig")
    }

    authCoordinator = TraktAuthCoordinator(
        authRepository: authRepository,
        logger: logger,
        clientId: BuildConfig.shared.TRAKT_CLIENT_ID,
        clientSecret: BuildConfig.shared.TRAKT_CLIENT_SECRET,
        redirectURL: redirectURL
    )
}
```

## Lessons Learned

**1. Simplicity Wins**: The YAML approach was flexible but overengineered for our needs. The BuildConfig approach does one thing well.

**2. Developer Experience Matters**: New contributors can now get the app running in minutes instead of struggling with YAML file placement and configuration.

**3. Publishing Gradle Plugins**: This project pushed me to learn how to publish convention plugins to Maven Central. It's been invaluable for sharing build logic across projects. I wrote about this in [Publishing Gradle Convention Plugins](https://thomaskioko.me/posts/publishing_gradle_plugins/).


## Trade-offs

No solution is perfect. Here are the trade-offs I accepted:

**Pros:**
- Single source of truth for configuration
- Compile time safety
- Better CI/CD integration
- Simpler onboarding
- Works identically across platforms and everything is configured on the KMM side.

**Cons:**
- Requires custom plugin (though it's reusable)
- Changes require rebuild (vs runtime YAML reload)
- Need to understand Gradle plugin development
- Keys are not secure and can be accessed if the App is reverse engineered.


The current solution is working so well that I'm hesitant to add complexity unless there's a real need. The security trade-offs are acceptable for this use case, as this is a pet project.
## Conclusion

Migrating away from the YAML configuration reduced the cognitive load and things are way easier now. As always, there's probably a better approach but I went with the easiest one for this project. Time will tell if I will have a part 3. 

Until next time, happy coding! ✌️

## Resources

- [Gradle Plugin Development Guide](https://docs.gradle.org/current/userguide/custom_plugins.html)
- [KMP Documentation](https://kotlinlang.org/docs/multiplatform.html)
