---
title: "Background Tasks in Kotlin Multiplatform"
date: "2025-11-30"
draft: false
hideToc: true
tags: ["KMP", "WorkManager", "BGTaskScheduler", "background-tasks"]
series: "Tv Maniac Journey"
---

# Intro

If you've built apps that rely on authentication tokens, you've likely dealt with the challenge of keeping those tokens fresh. Tokens expire, and if your user opens the app after being away for a while, they might get hit with an unexpected logout. Not a great experience.

Background tasks solve this problem. They allow your app to do work even when it's not in the foreground—refreshing tokens, syncing data, or fetching updates. The challenge? Android and iOS handle background work very differently.

In this post, I'll walk through how I set up background tasks in [my pet project](https://github.com/thomaskioko/tv-maniac) to keep authentication tokens fresh. The same approach works for any periodic background work like data synchronization.


## The Problem

OAuth tokens have a limited lifespan. In my case, Trakt tokens expire after a set period. If the token expires while the user is away, they'd need to re-authenticate when they return. That means navigating through the OAuth flow again—not a great experience, especially if they were just trying to quickly check what's next on their watchlist.

The solution is to refresh tokens proactively in the background before they expire. This keeps the user logged in seamlessly.


## The Approach

Since Android and iOS have different background task APIs, we need platform-specific implementations. The good news is that all of this can be done in Kotlin Multiplatform—including the iOS implementation using Kotlin/Native interop with Apple's frameworks. Here's how everything fits together:

```
          ┌─────────────────────────────────────────────────────────────┐
          │                   Background Token Refresh                  │
          └─────────────────────────────────────────────────────────────┘

          ┌─────────────────────────┐         ┌─────────────────────────┐
          │   Android Platform      │         │    iOS Platform         │
          │                         │         │                         │
          │  ┌──────────────────┐   │         │  ┌──────────────────┐   │
          │  │ TokenRefresh     │   │         │  │ TokenRefresh     │   │
          │  │ Worker           │   │         │  │ Service          │   │
          │  │                  │   │         │  │                  │   │
          │  │ - WorkManager    │   │         │  │ - BGTaskScheduler│   │
          │  │ - Periodic 6h    │   │         │  │ - Earliest 6h    │   │
          │  │ - Constraints    │   │         │  │ - BackgroundTask │   │
          │  └────────┬─────────┘   │         │  └────────┬─────────┘   │
          │           │             │         │           │             │
          └───────────┼─────────────┘         └───────────┼─────────────┘
                      │                                   │
                      └───────────────┬───────────────────┘
                                      ▼
                          ┌───────────────────────┐
                          │ Shared KMP Logic      │
                          │                       │
                          │ TraktAuthRepository   │
                          │                       │
                          │ - Check expiry        │
                          │ - Call refresh API    │
                          │ - Save new tokens     │
                          │ - Handle errors       │
                          └───────────────────────┘
```

The structure breaks down into three parts:

1. **Common interface** - Defines what background tasks should do
2. **Platform implementations** - Android uses WorkManager, iOS uses BGTaskScheduler
3. **Common initializer** - Manages when to schedule or cancel tasks based on auth state


## Defining the Common Interface

We start with a simple interface:

```kotlin
interface TraktAuthTasks {
    fun setup() = Unit
    fun scheduleTokenRefresh()
    fun cancelTokenRefresh()
}
```

The `setup()` method has a default implementation because only iOS needs it for task registration. Android's WorkManager doesn't require upfront registration.


## Android Implementation

Android uses [WorkManager](https://developer.android.com/topic/libraries/architecture/workmanager) for background tasks. It handles constraints like network availability and survives app restarts.

The worker is straightforward—check if the token needs refreshing and refresh it:

```kotlin
class TokenRefreshWorker(
    context: Context,
    params: WorkerParameters,
    private val traktAuthRepository: Lazy<TraktAuthRepository>,
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        val authState = traktAuthRepository.value.getAuthState() ?: return Result.success()
        if (!authState.isExpiringSoon()) return Result.success()
        return if (traktAuthRepository.value.refreshTokens() != null) Result.success() else Result.failure()
    }
}
```

The task scheduler uses WorkManager's periodic work API:

```kotlin
class AndroidTraktAuthTasks(workManager: Lazy<WorkManager>) : TraktAuthTasks {

    override fun scheduleTokenRefresh() {
        val refreshWork = PeriodicWorkRequestBuilder<TokenRefreshWorker>(5L, TimeUnit.DAYS)
            .setConstraints(Constraints.Builder().setRequiredNetworkType(NetworkType.CONNECTED).build())
            .build()

        workManager.enqueueUniquePeriodicWork("token_refresh_work", ExistingPeriodicWorkPolicy.UPDATE, refreshWork)
    }

    override fun cancelTokenRefresh() = workManager.cancelUniqueWork("token_refresh_work")
}
```


## iOS Implementation

iOS uses [BGTaskScheduler](https://developer.apple.com/documentation/backgroundtasks) for background work. Unlike WorkManager, you must register task identifiers before the app finishes launching.

Here's how we add the iOS implementation via KMM

```kotlin
class IosTraktAuthTasks(
    private val traktAuthRepository: TraktAuthRepository,
) : TraktAuthTasks {
    private val taskScheduler by lazy { BGTaskScheduler.sharedScheduler }

    override fun setup() {
        taskScheduler.registerForTaskWithIdentifier(TASK_ID, usingQueue = null, launchHandler = ::handleTask)
    }

    override fun scheduleTokenRefresh() {
        val request = BGAppRefreshTaskRequest(TASK_ID).apply {
            earliestBeginDate = NSDate.dateWithTimeIntervalSinceNow(5.0 * 24.0 * 60.0 * 60.0) // 5 days
        }
        taskScheduler.submitTaskRequest(request, error = null)
    }

    override fun cancelTokenRefresh() = taskScheduler.cancelTaskRequestWithIdentifier(TASK_ID)

    private fun handleTask(task: BGTask?) {
        task?.runTask { performRefresh() }
        scheduleTokenRefresh() // Reschedule for next run
    }

    private suspend fun performRefresh(): Boolean {
        val authState = traktAuthRepository.getAuthState() ?: return true
        if (!authState.isExpiringSoon()) return true
        return traktAuthRepository.refreshTokens() != null
    }

    companion object {
        private const val TASK_ID = "com.thomaskioko.tvmaniac.tokenrefresh"
    }
}
```

One important detail: iOS requires configuration in your `Info.plist`. You need both the task identifier and the correct background mode:

```xml
<key>UIBackgroundModes</key>
<array>
    <string>fetch</string>
</array>
<key>BGTaskSchedulerPermittedIdentifiers</key>
<array>
    <string>com.thomaskioko.tvmaniac.tokenrefresh</string>
</array>
```

A common point of confusion: `BGAppRefreshTask` uses the `fetch` background mode, not `processing`. Here's the distinction:

| Task Type | UIBackgroundModes | Use Case |
|-----------|-------------------|----------|
| `BGAppRefreshTask` | `fetch` | Short tasks (~30 seconds) like token refresh, checking for updates |
| `BGProcessingTask` | `processing` | Longer tasks when device is idle/charging, like database cleanup |

Since token refresh is a quick operation, `BGAppRefreshTask` with `fetch` is the right choice.


## Orchestrating with a Common Initializer

Now we need something to coordinate when tasks should run. In this case, we want to schedule the task only when the user logs in on Trakt:

```kotlin
class TokenRefreshInitializer(
    private val tasks: TraktAuthTasks,
    private val traktAuthRepository: TraktAuthRepository,
) : AppInitializer {

    override fun init() {
        tasks.setup()
        scope.launch {
            traktAuthRepository.state.collectLatest { state ->
                when (state) {
                    TraktAuthState.LOGGED_OUT -> tasks.cancelTokenRefresh()
                    TraktAuthState.LOGGED_IN -> tasks.scheduleTokenRefresh()
                }
            }
        }
    }
}
```

This keeps the platform implementations simple—they just schedule and cancel tasks. The decision of *when* to do so lives in shared code.


## Testing Background Tasks

### iOS Testing

Testing iOS background tasks locally can be tricky since the system controls when they run. Here's how to trigger them manually:

**Simulate via LLDB**

While debugging in Xcode:
1. Run the app on simulator or device
2. Login with Trakt
3. Pause execution (Debug → Pause)
4. In LLDB console, run:

```lldb
e -l objc -- (void)[[BGTaskScheduler sharedScheduler] _simulateLaunchForTaskWithIdentifier:@"com.thomaskioko.tvmaniac.tokenrefresh"]
```

5. Resume execution (Debug → Continue)
6. The background task will execute immediately

### Android Testing

Use App Inspection in Android Studio to see scheduled WorkManager tasks and their status.


## Things to Consider

A few things worth keeping in mind.

**iOS doesn't guarantee execution time.** The `earliestBeginDate` is a hint, not a promise. The system decides when your task actually runs based on battery, network conditions, and app usage patterns. If the user rarely opens your app, iOS might deprioritize your background tasks. For token refresh, this is usually fine—you have a buffer before expiry.

**Task registration timing matters on iOS.** You must register your task identifier before `applicationDidFinishLaunching` returns. If you're using lazy initialization or dependency injection, make sure the registration happens early enough.

**Think about failure scenarios.** The current implementation returns `Result.failure()` when refresh fails, which tells WorkManager or the Service to retry with backoff. But what if the refresh token itself is invalid? Detect 401 responses and clear the auth state rather than retrying multiple times.

**Battery impact is minimal, but not zero.** A quick network call every 5 days is negligible, but it's worth being intentional about the interval. For a 7 day token, refreshing at day 5 gives us a 2 day buffer for retries if something fails.


## When to Skip Background Refresh

Background tasks aren't always the right tool. For short lived tokens (minutes to hours), refreshing on app launch is simpler and more reliable. The complexity of background tasks makes sense when:

- Tokens have multiday or longer lifespans
- You're syncing data that should be ready when the user opens the app
- The user expects fresh content immediately on launch

For my use case, Trakt tokens that expire in 7 days. The background refresh makes sense. For a token that expires in minutes, I'd just refresh when the app becomes active.


## Final Thoughts

Background tasks in KMP require platform specific implementations, but that's okay. The key is keeping the implementations focused on *how* to run background work while sharing the *what* and *when* in common code.

This same pattern works for other background work like:
- Syncing local data with a server
- Prefetching content for offline use

The platform APIs are different, but the orchestration logic can be shared.

If you want to dig into the implementation, check out the [pull request](https://github.com/thomaskioko/tv-maniac/pull/683).

Until next time, happy coding! ✌️


### Resources

- [WorkManager Documentation](https://developer.android.com/topic/libraries/architecture/workmanager)
- [BGTaskScheduler Documentation](https://developer.apple.com/documentation/backgroundtasks)
- [UIBackgroundModes - Apple Developer Documentation](https://developer.apple.com/documentation/bundleresources/information-property-list/uibackgroundmodes)
- [Background Modes Tutorial: Getting Started - Kodeco](https://www.kodeco.com/34269507-background-modes-tutorial-getting-started)
