---
title: "Using Roborazzi for Android Screenshot Testing"
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

Next, add the Roborazzi Gradle plugin to your project's `build.gradle`:

```kotlin
plugins {
    id("io.github.takahirom.roborazzi") version "1.5.0"
}
```

## Writing Our First Screenshot Test

Let's create a screenshot test for one of our screens. We'll use ShowDetailsScreen as an example.

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

Let's break down the annotations:

- `@RunWith(RobolectricTestRunner::class)`: This tells JUnit to use Robolectric to run the tests
- `@Config(sdk = [33])`: Specifies the Android SDK version to emulate
- `@GraphicsMode(GraphicsMode.Mode.NATIVE)`: Enables hardware-accelerated rendering for more accurate screenshots
- `@LooperMode(LooperMode.Mode.PAUSED)`: Controls how Robolectric handles the main thread's Looper

### Customizing Screenshot Capture

Roborazzi allows you to customize various aspects of screenshot capture. Here's an example of how to set up custom options:

```kotlin
val DefaultRoborazziOptions = RoborazziOptions(
  compareOptions = CompareOptions(changeThreshold = 0f),
  recordOptions = RecordOptions(resizeScale = 0.5),
)
```

These options set up pixel-perfect matching `changeThreshold = 0f` and reduce the size of the PNG files `resizeScale = 0.5` to save storage space.

Testing Across Multiple Devices and Themes
Roborazzi makes it easy to test across different device configurations and themes:

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

The `DefaultTestDevices` enum defines the device configurations to test (in this case, just a Pixel 7). The `captureMultiDevice` function uses this to capture screenshots for each defined device, and also captures both light and dark themes using the captureMultiTheme function and specifies the directory to store the screenshots.

```kotlin
fun <A : ComponentActivity> AndroidComposeTestRule<ActivityScenarioRule<A>, A>.captureMultiTheme(
  name: String,
  deviceSpec: String,
  overrideFileName: String? = null,
  shouldCompareDarkMode: Boolean = true,
  content: @Composable () -> Unit,
) {

  // Set qualifiers from specs
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

These utility functions provide several benefits:
- **Multi-device testing:** The captureMultiDevice function allows you to capture screenshots for multiple device configurations (in this case, a Pixel 7).
- **Theme testing:** The captureMultiTheme function captures screenshots in both light and dark themes.
- **Customizable options:** The DefaultRoborazziOptions set up pixel-perfect matching and reduce the size of the PNG files.
- **Flexible naming:** The functions allow for custom naming of the screenshot files.

## Running Tests and Reviewing Results

To run the tests, use the following Gradle command:

```
./gradlew verifyRoborazziDebug
```

The first time you run this, Roborazzi will generate reference images. On subsequent runs, it will compare the new screenshots against these references.
If there are differences, you can review them using the following command:

```
./gradlew compareRoborazziDebug
```

This command opens a web interface where you can see side-by-side comparisons of the reference images and the new screenshots.

![ShowDetailsLoadedState_dark_compare](https://github.com/user-attachments/assets/241cd73c-242a-40bb-a454-641851106359)


### My experience with Roborazzi

My experience with Roborazzi was very smooth and easy to setup.


Pros: Very fast, produces vector output for high-quality diffs, easy Gradle integration
Cons: Doesn't use actual Android framework code, limited to view-based tests


- Easy setup and integration with Gradle
- Uses Robolectric, providing a good balance of speed and accuracy
- Flexible API allowing for customization of device configs and other parameters
- Active development and community support

**Cons (Not really buut yeah ...):**
- Relatively newer compared to some other options, so the ecosystem is still growing 🤓


## Conclusion

Roborazzi provides a powerful and flexible way to implement screenshot testing in your Android projects. By catching unintended UI changes early, you can maintain a consistent user experience and reduce the risk of visual regressions. 

There are some improvements that can be done, especially uploading reports as a comment when there's a change in the pull request. But this will be done in a different post (Maybe 🫣)

That's all folks Until we meet again. ✌️