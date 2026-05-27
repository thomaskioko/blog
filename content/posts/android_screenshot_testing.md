---
title: "Android Screenshot Testing with Roborazzi"
date: "2024-05-03"
draft: false
hideToc: true
tags: ["Android", "Screenshot tests", "Roborazzi", "Jetpack Compose"]
series: "Tv Maniac Journey"
---

# Using Roborazzi for Android Screenshot Testing

Screenshot testing is a powerful tool for catching unintended UI changes in your Android app. Roborazzi is a modern screenshot testing library that makes it easy to implement and maintain these tests. It leverages Robolectric to provide fast, reliable tests that can run on your development machine without the need for emulators or physical devices.
Key benefits of Roborazzi include:

- Easy integration with existing Android projects
- Fast execution times
- Support for multiple device configurations and themes
- Compatibility with Jetpack Compose

In this post, we'll walk through setting up Roborazzi and creating some basic screenshot tests.

## Setting Up Roborazzi

First, add the necessary dependencies to your app's `build.gradle` file:

```kotlin
dependencies {
    testImplementation("org.robolectric:robolectric:4.10.3")
    testImplementation("io.github.takahirom.roborazzi:roborazzi:1.5.0")
    testImplementation("io.github.takahirom.roborazzi:roborazzi-junit-rule:1.5.0")
}
```

Apply the Roborazzi Gradle plugin in the project `build.gradle`:

```kotlin
plugins {
    id("io.github.takahirom.roborazzi") version "1.5.0"
}
```

## Implement screenshot tests

Create a screenshot test for a screen using `ShowDetailsScreen` as an example.

```kotlin
@RunWith(RobolectricTestRunner::class)
@Config(sdk = [33])
@GraphicsMode(GraphicsMode.Mode.NATIVE)
@LooperMode(LooperMode.Mode.PAUSED)
class ShowDetailsScreenScreenshotTest {

  @get:Rule val composeTestRule = createAndroidComposeRule<ComponentActivity>()

  @Test
  fun showDetailsLoadedState() {
    composeTestRule.captureMultiDevice("ShowDetailsLoadedState") {
      TvManiacBackground {
        ShowDetailsScreen(
          state = showDetailsContent,
          title = "",
          snackBarHostState = SnackbarHostState(),
          listState = LazyListState(),
          onAction = {},
        )
      }
    }
  }
}
```

Annotate the test class as follows:
- `@RunWith(RobolectricTestRunner::class)`: Instructs JUnit to use Robolectric.
- `@Config(sdk = [33])`: Specifies the Android SDK version.
- `@GraphicsMode(GraphicsMode.Mode.NATIVE)`: Enables hardware accelerated rendering for accurate screenshots.
- `@LooperMode(LooperMode.Mode.PAUSED)`: Manages the main thread Looper.

## Customize capture options

Roborazzi supports customization of screenshot capture aspects. Configure custom options as shown:

```kotlin
val DefaultRoborazziOptions = RoborazziOptions(
  compareOptions = CompareOptions(changeThreshold = 0f),
  recordOptions = RecordOptions(resizeScale = 0.5),
)
```

These options establish pixel perfect matching with `changeThreshold = 0f` and reduce PNG file size using `resizeScale = 0.5` to conserve storage.

## Verify multiple configurations

Test across different device configurations and themes using the following pattern:

```kotlin
enum class DefaultTestDevices(val spec: String) {
  Pixel7(RobolectricDeviceQualifiers.Pixel7),
}

fun <A : ComponentActivity> AndroidComposeTestRule<ActivityScenarioRule<A>, A>.captureMultiDevice(
  name: String,
  content: @Composable () -> Unit,
) {
  DefaultTestDevices.entries.forEach {
    this.captureMultiTheme(
      deviceSpec = it.spec,
      name = name,
      content = content,
    )
  }
}
```

The `DefaultTestDevices` enum defines configurations including the Pixel 7. The `captureMultiDevice` function captures screenshots for defined devices and themes.

```kotlin
fun <A : ComponentActivity> AndroidComposeTestRule<ActivityScenarioRule<A>, A>.captureMultiTheme(
  name: String,
  deviceSpec: String,
  overrideFileName: String? = null,
  shouldCompareDarkMode: Boolean = true,
  content: @Composable () -> Unit,
) {
  RuntimeEnvironment.setQualifiers(deviceSpec)

  val darkModeValues = if (shouldCompareDarkMode) listOf(true, false) else listOf(false)
  var darkMode by mutableStateOf(true)

  this.setContent {
    CompositionLocalProvider(
      LocalInspectionMode provides true,
    ) {
      TvManiacTheme(
        darkTheme = darkMode,
      ) {
        content()
      }
    }
  }

  darkModeValues.forEach { isDarkMode ->
    darkMode = isDarkMode
    val darkModeDesc = if (isDarkMode) "dark" else "light"
    val filename = overrideFileName ?: name

    this.onRoot()
      .captureRoboImage(
        "src/test/screenshots/" + filename + "_$darkModeDesc" + ".png",
        roborazziOptions = DefaultRoborazziOptions,
      )
  }
}
```

Utility functions provide multi device testing, theme verification, and flexible file naming.

## Execute tests and review results

Run tests using the following Gradle command:

```
./gradlew verifyRoborazziDebug
```

Roborazzi generates reference images on the first execution. Subsequent runs compare new screenshots against these references. Review differences using this command:

```
./gradlew compareRoborazziDebug
```

This command launches a web interface for side by side comparisons of reference and new images.

![ShowDetailsLoadedState_dark_compare](https://github.com/user-attachments/assets/241cd73c-242a-40bb-a454-641851106359)

## Evaluation

I found the Roborazzi setup process smooth and efficient.

Advantages include:
- Rapid execution and high quality vector output for diffs
- Integration with Gradle and Robolectric
- Flexible API for device configuration customization
- Active development and community support

The library is relatively new compared to other options, but the ecosystem continues to grow.

## Final considerations

Roborazzi provides a flexible method for implementing screenshot testing. It identifies unintended UI changes early, maintaining consistent user experiences and reducing visual regressions. Future improvements may include automatic report uploads to pull requests when changes occur.

Until we meet again, folks. Happy coding! ✌️
