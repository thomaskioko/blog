---
title: "Internationalization (I18n) in Kotlin Multiplatform"
date: "2025-05-25"
draft: false
hideToc: true
tags: ["KMM", "i18n", "Internationalization"]
series: "Tv Maniac Journey"
---

Welcome back! In this article, we'll explore my approach to implementing Internationalization (I18n) in a Kotlin Multiplatform project. I'll share how I've structured the solution to enable resource sharing between Android and iOS platforms. While this implementation might be more complex than necessary for some use cases, it demonstrates a modular approach to handling internationalization. Let's dive in and see how it works.

# Internationalization (I18n) in Kotlin Multiplatform

Internationalization (I18n) is the process of designing and developing applications that can be adapted to different languages and regions. This allows your app to "customize" the content based on the user's region. 

For our Kotlin Multiplatform project, we'll be using [MokoResources](https://github.com/icerockdev/moko-resources/). It provides a unified way to handle resources across Android and iOS platforms. Here's why I chose it:

- **Shared Resources**: MokoResources allows us to maintain a single source of truth for all our strings, images, and other resources
- **Type Safety**: The library generates type-safe accessors for all resources, reducing the chance of runtime errors
- **Platform Integration**: Seamless integration with both Android and iOS native resource systems

While I won't cover the setup process in detail (as the [official documentation](https://github.com/icerockdev/moko-resources/) provides excellent guidance), I'll focus on how we've structured our localization strategy and implemented it in [TvManiac](https://github.com/thomaskioko/tv-maniac)

## Our Implementation Approach

In my previous implementation, I had a single module (`:i18n`) that contained all the localization logic. This module was then added as a dependency to other modules that needed string resources. While this approach worked, I wanted to improve it by:

1. Moving string handling from the UI layer to the presentation layer (State Objects)
2. Making the resources more testable
3. Ensuring consistent behavior across all platforms.

To achieve this, I modularized the `:i18n` module, which allows us to:
- Test resources independently
- Maintain consistent behavior across platforms
- Better separate concerns between UI and business logic
- Make the codebase more maintainable
- Optimize iOS framework generation by keeping resources in a single module, avoiding multiple framework generation and maintaining a clean dependency hierarchy

This has been broken down to 3 main steps
1. Gradle Task & Plugin.
2. Modularizing `:i18n`
3. Testing

## 1. The Generator Task

The `MokoResourceGeneratorTask` is a custom Gradle task that automates the generation of type-safe resource accessors. Here's what it does:

1. **Resource Processing**:
   - Reads the generated `MR.kt` file from MokoResources
   - Extracts string and plural resource keys
   - Generates type-safe sealed classes for resource access
   - Writes the generated files to the specified output directory

It leverages [KotlinPoet](https://square.github.io/kotlinpoet/) to create type-safe resource key classes. Here's an example of the generated wrapper class:

```kotlin
sealed class PluralsResourceKey(
    public val resourceId: PluralsResource,
) {
    data object SeasonCount : PluralsResourceKey(MR.plurals.season_count)
    data object EpisodeCount : PluralsResourceKey(MR.plurals.episode_count)
    // ...
}
```

Here's a snippet of the implementation:

```kotlin
@CacheableTask
public abstract class MokoResourceGeneratorTask : DefaultTask() {
    @get:OutputDirectory
    public val commonMainOutput: DirectoryProperty = objectFactory.directoryProperty()
        .convention(layout.buildDirectory.dir("generated/resources"))

    @TaskAction
    public fun generate() {
        // Read MR.kt file
        val mrFile = project.file("build/generated/moko-resources/commonMain/src/com/thomaskioko/tvmaniac/i18n/MR.kt")
        
        // Extract resource keys
        val (stringKeys, pluralKeys) = readKeysFromMRFile(mrFile)
        
        // Generate StringResourceKey sealed class
        stringResourceKeyFileSpec(
            stringKeys = stringKeys,
            mrClass = ClassName("com.thomaskioko.tvmaniac.i18n", "MR")
        ).writeTo(outputDir)
        
        // Generate PluralsResourceKey sealed class
        pluralsResourceKeyFileSpec(
            pluralKeys = pluralKeys,
            mrClass = ClassName("com.thomaskioko.tvmaniac.i18n", "MR")
        ).writeTo(outputDir)
    }
}
```

It's a bit rough around the edges and could be improved. For instance, how we read MokoResource's generated file. This is something I will improve in the next iteration as this is a bit brittle. If MokoResources decides to change the location of the generated file, this will break.

# 2. The Module Trinity: A Three-Module Architecture

I've broken down the localization module into three modules:

  - Generator Module (`:i18n:generator`)
  - API Module (`:i18n:api`)
  - Implementation Module (`:i18n:implementation`)

## a.) Generator Module (`:i18n:generator`)

This module is responsible for Moko Resources configuration and resource generation. We also apply the plugin in this module allowing us to run the task that generates the classes.

```kotlin
plugins {
  alias(libs.plugins.tvmaniac.kmp)
  alias(libs.plugins.tvmaniac.resource.generator) //<- Apply the plugin
}
```

## b.) API Module (`:i18n:api`)

This module defines the contract for resource access through a clean interface:

```kotlin
interface Localizer {
    fun getString(key: StringResourceKey): String
    fun getString(key: StringResourceKey, vararg args: Any): String
    fun getPlural(key: PluralsResourceKey, quantity: Int): String
    fun getPlural(key: PluralsResourceKey, quantity: Int, vararg args: Any): String
}
```

## c.) Implementation Module (`:i18n:implementation`)

This module provides the concrete implementation of the `Localizer` interface. It handles platform-specific implementations internally, so consumers don't need to worry about the underlying details.

Using the expect/actual pattern, each platform provides its own implementation for string resolution. For example:

### Android Implementation
```kotlin
@Inject
actual class PlatformLocalizer(
    private val context: Context,
) {
    actual fun localized(stringDesc: StringDesc): String {
        return stringDesc.toString(context)
    }
}
```

### iOS Implementation
```kotlin
@Inject
actual class PlatformLocalizer {
    actual fun localized(stringDesc: StringDesc): String {
        return stringDesc.localized()
    }
}
```

This architecture provides several benefits:
- Clear separation of concerns
- Type-safe resource access
- Platform-specific implementations hidden from consumers
- Easy testing and maintenance

In the next section, we'll explore how to inject the Locale to provide the correct strings based on the user's region.


# 3. Testing

With our modular architecture in place, we can now implement some tests tests. 🥳. We'll use a base test class approach to share test logic across platforms while allowing platform-specific implementations.


### Base Test Class
We create an abstract base class that defines our test cases. This class resides in the `commonMainTest` directory:

```kotlin
abstract class MokoLocalizerTest {
    abstract val localizer: Localizer

    @Test
    fun `should return localized string for valid key`() {
        val result = localizer.getString(StringResourceKey.ButtonErrorRetry)
        result shouldBe "Retry"
    }

    // Additional test cases...
}
```

### Platform-Specific Implementations

#### Android Tests
```kotlin
@RunWith(AndroidJUnit4::class)
@Config(sdk = [33])
internal class MokoLocalizerAndroidTest : MokoLocalizerTest() {
    override lateinit var localizer: Localizer

    @Before
    fun setup() {
        val context = ApplicationProvider.getApplicationContext<Context>()
        localizer = MokoResourcesLocalizer(PlatformLocalizer(context))
    }
}
```

#### iOS and JVM Tests
```kotlin
internal class MokoLocalizerJvmTest : MokoLocalizerTest() {
    override lateinit var localizer: Localizer

    @BeforeTest
    fun setup() {
        localizer = MokoResourcesLocalizer(PlatformLocalizer())
    }
}
```

With this in place, we can ensure we always get the correct string and is formatted as expected.

# Summary

While this implementation might seem complex at first, it provides a flexible and testable solution for sharing resources across platforms. I've also explored [Lyricist](https://github.com/adrielcafe/lyricist) as a potential alternative, but for now, this solution meets my need. But, time will tell.

In the next article, we'll explore how to inject the Locale and implement the localizer in the presenter module.

### References
- [Moko Resources GitHub Repository](https://github.com/icerockdev/moko-resources)
- [Kotlin Multiplatform Documentation](https://kotlinlang.org/docs/multiplatform.html)