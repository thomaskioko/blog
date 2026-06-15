---
title: "Adding Firebase Crashlytics to a KMP Project Using Dependency Inversion"
date: "2026-02-28"
draft: false
hideToc: true
tags: ["KMP", "Firebase", "Crashlytics", "dependency-inversion", "logging"]
series: "Tv Maniac Journey"
---

Crash reporting is one of those things you don't think about until your app is in production and users are hitting issues you can't reproduce. I recently added Firebase Crashlytics to [TvManiac](https://github.com/thomaskioko/tv-maniac), a Kotlin Multiplatform project that targets both Android and iOS. The project already had a `Logger` interface backed by [Kermit](https://github.com/touchlab/Kermit), used across the app for console debugging. The goal was to layer crash reporting on top of that without touching any of the existing consumers.

In this article, I'll walk through how I wired Firebase Crashlytics into the shared logging layer using dependency inversion, a composite pattern for multiple logger destinations, and a bridge pattern to call Swift-only SDKs from Kotlin.


## The Existing Logger Setup

Before integrating Firebase, the project had a single logger implementation: `KermitLogger`. It extended a public `Logger` interface that defines standard logging functions.

```kotlin
public interface Logger {

    public fun setup(debugMode: Boolean): Unit = Unit

    public fun debug(message: String): Unit = Unit

    public fun error(tag: String, message: String)

    public fun info(message: String, throwable: Throwable): Unit = Unit
    ...

}
```

Every presenter and interactor in the project depends on this interface, not on Kermit directly. This is the key detail that made everything else possible.


## Why Firebase?

There are various crash reporting services: [Sentry](https://docs.sentry.io/platforms/kotlin/guides/kotlin-multiplatform/), [Datadog](https://www.datadoghq.com/), [BugSnag](https://www.bugsnag.com/), and others. I went with Firebase because it requires native setup on each platform, and I wanted to wire it myself rather than pull in a third-party KMP wrapper like [firebase-kotlin-sdk](https://github.com/nicegram/nicegram-android). In the future, I'll explore Sentry's KMP SDK, but for now, doing it natively was a good exercise in understanding the platform boundaries.


## Handling Missing Config Files

When working with Firebase, each platform needs a config file: `google-services.json` for Android and `GoogleService-Info.plist` for iOS. These shouldn't be committed to the repo, but if they're missing, builds break. We need to make sure:

1. The app builds and runs without the files.
2. CI injects them from secrets.

On Android, I conditionally apply the google-services plugin only when the file exists:

```kotlin
if (file("google-services.json").exists()) {
    apply(plugin = libs.plugins.google.services.get().pluginId)
    apply(plugin = libs.plugins.firebase.crashlytics.gradle.get().pluginId)
}
```

The DI layer returns nullable types like `FirebaseApp?` and `FirebaseCrashlytics?`, so when the plugin isn't applied, everything degrades to no-ops.

iOS needed more work. The plist is a bundle resource referenced in `project.pbxproj`. The build succeeds without it, but Firebase crashes at runtime. I wrapped the initialization in `AppDelegate` with an existence check:

```swift
if Bundle.main.path(forResource: "GoogleService-Info", ofType: "plist") != nil {
    FirebaseApp.configure()
    CrashReportingBridgeHolder.shared.bridge = FirebaseCrashlyticsBridge()
}
```

No plist, no Firebase. A `NoOpCrashReportingBridge` fallback handles the rest. More on `CrashReportingBridgeHolder` later.

There's also a Crashlytics build phase that uploads dSYM files after every build. That script expects the plist to exist, so I added an early exit:

```sh
GSP_CHECK="${TARGET_BUILD_DIR}/${UNLOCALIZED_RESOURCES_FOLDER_PATH}/GoogleService-Info.plist"
if [ ! -f "$GSP_CHECK" ]; then
  echo "warning: GoogleService-Info.plist not found - skipping dSYM upload"
  exit 0
fi
```

For CI, I store both config files as base64-encoded GitHub secrets and decode them before the build step. If someone clones the repo and runs the build, everything works, just without Firebase.


## Adding a Firebase Logger

With the config out of the way, let's look at how I wired the actual logging. The approach is a composite pattern with multibinding. Here's how the pieces fit together:

- **KermitLogger** handles console output, enabled only for debug builds.
- **FirebaseCrashLogger** is a second `Logger` implementation that only cares about crash-relevant methods like `error()`, `recordException()`, and `setUserId()`. It delegates to a `CrashReporter` interface for the actual Firebase calls. Methods like `debug()` and `info()` use the default empty bodies from the interface. We don't want to send debug logs to Crashlytics.
- **CompositeLogger** receives the full `Set<Logger>` and fans out every call. When a presenter calls `logger.error(...)`, it dispatches to both `KermitLogger` (prints to console) and `FirebaseCrashLogger` (records to Crashlytics). The call site doesn't know either of those exist.

The DI wiring connects it all. `KermitLogger` and `FirebaseCrashLogger` use `@ContributesBinding(AppScope::class, multibinding = true)`, which means they contribute to the set. `CompositeLogger` uses `@ContributesBinding(AppScope::class)` without multibinding, making it the *single* `Logger` binding that consumers inject.

On Android, the `CrashReporter` implementation is straightforward: `AndroidCrashReporter` wraps the Firebase Crashlytics SDK directly. The SDK is available as a Gradle dependency in `androidMain`, so Kotlin can call it without any indirection.


## What About iOS?

This is where we need to do some extra work. The Firebase iOS SDK is distributed as a Swift Package. Kotlin/Native can interop with Objective-C frameworks, but not with SPM packages directly. There's no way for `iosMain` Kotlin code to import `FirebaseCrashlytics` and call `Crashlytics.crashlytics().record(error:)`. The compiler simply doesn't see it.

I looked at [firebase-kotlin-sdk](https://github.com/nicegram/nicegram-android) which wraps Firebase for KMP using CocoaPods interop. That works, but it pulls in a third-party wrapper with its own release cadence. For a handful of crash reporting methods, that felt like overkill.

So I went with a bridge pattern. Kotlin defines the contract, Swift provides the implementation.

On the Kotlin side, a `CrashReportingBridge` interface in `iosMain` defines the methods the crash reporter needs:

```kotlin
public interface CrashReportingBridge {
    public fun setCollectionEnabled(enabled: Boolean)
    public fun recordException(throwable: Throwable)
    public fun recordException(throwable: Throwable, tag: String)
    public fun setCustomKey(key: String, value: String)
    public fun setUserId(userId: String)
    public fun log(message: String)
}
```

`IosCrashReporter` receives this bridge via constructor injection and delegates every call to it. It never imports Firebase directly.

On the Swift side, `FirebaseCrashlyticsBridge` implements that interface and wraps the real SDK. Since it lives in a Swift package, it can depend on `FirebaseCrashlytics` via SPM without any issue:

```swift
public class FirebaseCrashlyticsBridge: CrashReportingBridge {

    public func recordException(throwable: KotlinThrowable) {
        let error = NSError(
            domain: String(describing: type(of: throwable)),
            code: 0,
            userInfo: [NSLocalizedDescriptionKey: throwable.message ?? "Unknown error"]
        )
        Crashlytics.crashlytics().record(error: error)
    }

    public func setUserId(userId: String) {
        Crashlytics.crashlytics().setUserID(userId)
    }

    // ... other methods follow the same pattern
}
```

Notice that Kotlin's `Throwable` becomes `KotlinThrowable` on the Swift side. The bridge wraps it into an `NSError` since that's what the Crashlytics SDK expects.

The connection happens in `AppDelegate` with the same config check shown earlier. Swift sets the bridge on a singleton holder that lives in Kotlin's `iosMain`, and the DI graph picks it up from there. If the plist is missing, the holder stays `nil` and a `NoOpCrashReportingBridge` kicks in as the fallback. No crash, no Firebase dependency at runtime, just silent no-ops.

## What About Adding More Frameworks?

This is where the setup pays off. Say I want to add [Sentry](https://sentry.io/) tomorrow. All I need is a new `SentryLogger` that implements `Logger` and overrides the methods I care about, like `error()`, `recordException()`, and maybe `setUserId()`. The multibinding annotation ensures the DI framework discovers it automatically and adds it to the set. No consumer changes. Errors flow to both Firebase and Sentry simultaneously.

## Wrapping Up

Every file in the project depends on the `Logger` interface, not on Kermit, not on Firebase, and not on any concrete implementation. Because of that, I was able to completely change what happens behind that interface without any consumer knowing or caring about the implementation.

The multibinding and composite pattern make the wiring clean, but they only work because the dependency points the right way. Consumers depend on the abstraction. Implementations depend on the abstraction. Nothing depends on the concrete  implementation. That's dependency inversion, and it's what let me add Crashlytics across the entire codebase with zero changes to any consumer.

Until we meet again, folks. Happy coding! ✌️


### References
- [Firebase Android Setup](https://firebase.google.com/docs/android/setup)
- [Firebase iOS Setup](https://firebase.google.com/docs/ios/setup)
- [Firebase Crashlytics](https://firebase.google.com/docs/crashlytics)
- [KMP Firebase Setup](https://funkymuse.dev/posts/kmp-firebase/)
- [Firebase Crashlytics iOS: dSYM Uploading](https://firebase.google.com/docs/crashlytics/ios/get-started?authuser=0#set-up-dsym-uploading)
