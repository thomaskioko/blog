---
title: "Centralizing Background Tasks Implementation"
date: "2026-02-15"
draft: false
hideToc: true
tags: ["KMP", "WorkManager", "BGTaskScheduler", "background-tasks", "architecture"]
series: "Tv Maniac Journey"
---

# Intro

In a [previous post](/posts/kmp_background_tasks/), I walked through setting up background tasks in KMP for token refresh. After that post, I added two more background tasks: library sync and episode notifications. Same pattern, same structure. That's when I noticed a gap in my implementation..

Each new task meant copy pasting the same registration, scheduling, and execution boilerplate with slight variations. Three tasks across two platforms, and I was already seeing inconsistencies that made debugging harder than it needed to be.

This post covers how I centralized all background task logic behind a shared implementation. 


## What Went Wrong

The original pattern was sound in isolation. But once I had three tasks on each platform, the duplication became a problem.

On **iOS**, each task repeated the same six steps: register with `BGTaskScheduler`, build a `BGAppRefreshTaskRequest`, submit it, filter by identifier in the handler, manage the execution window with an expiration callback, and reschedule after completion. That's a lot of surface area to get wrong, and each task implemented it slightly differently.

On **Android**, each feature required *two* classes: a `*Tasks` class for scheduling and a `*Worker` class for execution. Three features meant six classes, plus a factory that had to manually map every worker class name.

Three tasks, three different approaches to concurrency and error handling. None of them were wrong individually. But, when you're debugging a background task that didn't fire at 2am, the last thing you want is to remember which concurrency pattern *this particular task* uses.


## The Inconsistency

When background tasks fail, they fail silently. Of course you can add logs and crash reporting for that, but I am yet to implement that in my project. So debugging had to happen manually for me. After debugging and finding what the issue, I need to make sure all tasks apply the fix.

There's also the onboarding cost. If another engineer decided to contribite to the project and they need to add a background task, which of the existing implementations do they follow? They'll pick one, probably the most recent, and introduce yet another slight variation.

Why I was introducing background tasks, the architecture worked. But as I contunied adding more features, I saw the issue; Inconsistency!


## The Centralization

The fix was straightforward: extract the platform boilerplate into a shared library in `core/tasks/` and have each task declare only what's unique to it.

On **iOS**, each task now implements a `BackgroundTask` interface:

```kotlin
interface BackgroundTask {
    val taskId: String
    val interval: Double
    suspend fun execute()
}
```

A `BGTaskRegistry` handles all the registration, scheduling, execution window management, and rescheduling.

On **Android**, each task implements `BackgroundWorker`:

```kotlin
interface BackgroundWorker {
    val workerName: String
    val interval: Duration
    val constraints: WorkerConstraints
    suspend fun execute(): WorkerResult
}
```

A single `DispatchingWorker` replaces all three `CoroutineWorker` subclasses. It looks up the registered worker by name and delegates execution. The worker factory went from a manual switch statement to a single line.

The before/after on a task implementation tells the story. Here's the iOS token refresh task:

**Before:**

```kotlin
class IosTraktAuthTasks(
    private val traktAuthRepository: TraktAuthRepository,
) : TraktAuthTasks {
    private val taskScheduler by lazy { BGTaskScheduler.sharedScheduler }

    override fun setup() {
        taskScheduler.registerForTaskWithIdentifier(
            TASK_ID, usingQueue = null, launchHandler = ::handleTask,
        )
    }

    override fun scheduleTokenRefresh() {
        val request = BGAppRefreshTaskRequest(TASK_ID).apply {
            earliestBeginDate = NSDate.dateWithTimeIntervalSinceNow(5.0 * 24 * 60 * 60)
        }
        taskScheduler.submitTaskRequest(request, error = null)
    }

    override fun cancelTokenRefresh() = taskScheduler.cancelTaskRequestWithIdentifier(TASK_ID)

    private fun handleTask(task: BGTask?) {
       //Task logic
    }

    private suspend fun performRefresh(): Boolean { /* ... */ }
}
```

Registration, scheduling, execution window management, expiration handling, rescheduling are all inlined. Now multiply that by three tasks.

**After:**

```kotlin
class IosTraktAuthTasks(
    private val registry: BGTaskRegistry,
    private val traktAuthRepository: TraktAuthRepository,
) : TraktAuthTasks, BackgroundTask {

    override val taskId = "com.thomaskioko.tvmaniac.tokenrefresh"
    override val interval = 5.0 * 24 * 60 * 60

    override fun setup() = registry.register(this)
    override fun scheduleTokenRefresh() = registry.schedule(taskId)
    override fun cancelTokenRefresh() = registry.cancel(taskId)

    override suspend fun execute() {
        val authState = traktAuthRepository.getAuthState() ?: return
        if (!authState.isExpiringSoon()) return
        traktAuthRepository.refreshTokens()
    }
}
```

The task only contains the *what*. The implementation lives in one place.


### Android: Collapsing the Worker/Task Pair

On Android, the duplication had a different shape. Each feature required two classes: A `*Tasks` class that built constraints and enqueued work, and a `*Worker` that extended `CoroutineWorker` to do the actual job. Three features meant six classes. On top of that, `TvManiacWorkerFactory` had a manual switch statement mapping every worker class name to its factory method:

```kotlin
return when (workerClassName) {
    name<TokenRefreshWorker>() -> tokenRefreshWorker(appContext, workerParameters)
    name<LibrarySyncWorker>() -> librarySyncWorker(appContext, workerParameters)
    name<EpisodeNotificationWorker>() -> episodeNotificationWorker(appContext, workerParameters)
    else -> null
}
```

Every new background task meant adding a worker class, a tasks class, and updating this factory in the app module. Three files for one feature.

Here's what the token refresh looked like before two classes for one concern:

```kotlin
// Class 1: Scheduling
class AndroidTraktAuthTasks(
    private val workManager: Lazy<WorkManager>,
) : TraktAuthTasks {

    override fun scheduleTokenRefresh() {
        val constraints = Constraints.Builder()
            .setRequiredNetworkType(NetworkType.CONNECTED)
            .build()
        val refreshWork = PeriodicWorkRequestBuilder<TokenRefreshWorker>(120L, TimeUnit.HOURS)
            .setConstraints(constraints)
            .build()
        workManager.value.enqueueUniquePeriodicWork(
            "token_refresh_work", ExistingPeriodicWorkPolicy.UPDATE, refreshWork,
        )
    }

    override fun cancelTokenRefresh() =
        workManager.value.cancelUniqueWork("token_refresh_work")
}

// Class 2: Execution
class TokenRefreshWorker(
    context: Context,
    params: WorkerParameters,
    private val traktAuthRepository: Lazy<TraktAuthRepository>,
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        val authState = traktAuthRepository.value.getAuthState() ?: return Result.success()
        if (!authState.isExpiringSoon()) return Result.success()
        return when (traktAuthRepository.value.refreshTokens()) {
            is TokenRefreshResult.Success -> Result.success()
            is TokenRefreshResult.NetworkError -> Result.retry()
            else -> Result.failure()
        }
    }
}
```

After centralizing, each feature is a single class:

```kotlin
class AndroidTraktAuthTasks(
    private val scheduler: BackgroundWorkerScheduler,
    private val traktAuthRepository: Lazy<TraktAuthRepository>,
) : TraktAuthTasks, BackgroundWorker {

    override val workerName = "token_refresh_work"
    override val interval = 120.hours
    override val constraints = WorkerConstraints(NetworkRequirement.CONNECTED)

    override fun setup() = scheduler.register(this)
    override fun scheduleTokenRefresh() = scheduler.schedulePeriodic(workerName)
    override fun cancelTokenRefresh() = scheduler.cancel(workerName)

    override suspend fun execute(): WorkerResult {
        val authState = traktAuthRepository.value.getAuthState() ?: return WorkerResult.Success
        if (!authState.isExpiringSoon()) return WorkerResult.Success
        return when (traktAuthRepository.value.refreshTokens()) {
            is TokenRefreshResult.Success -> WorkerResult.Success
            is TokenRefreshResult.NetworkError -> WorkerResult.Retry
            else -> WorkerResult.Failure
        }
    }
}
```

A single `DispatchingWorker` replaced all three `CoroutineWorker` subclasses. It receives the worker name via `inputData`, looks it up in the scheduler's registry, and calls `execute()`. The factory collapsed to one line:

```kotlin
return when (workerClassName) {
    name<DispatchingWorker>() -> dispatchingWorker(appContext, workerParameters)
    else -> null
}
```

Three worker classes deleted. Factory no longer needs to know about individual features. Adding a new background task is just one class that implements `BackgroundWorker` and calls `scheduler.register(this)` during setup.


## Why Not a Common API?

You might have noticed that iOS has `BackgroundTask` and Android has `BackgroundWorker`. Two separate interfaces in platform-specific source sets. It's a fair question: why not define a single `BackgroundTask` interface in `commonMain` and have both platforms implement it?

I considered it. A common interface would look something like:

```kotlin
// commonMain
interface BackgroundTask {
    val taskId: String
    val interval: Duration
    suspend fun execute(): TaskResult
}
```

It's doable. The interfaces are already similar in shape both have an identifier, an interval, and an `execute()` function. You could absolutely model this in `commonMain` and have each platform's registry accept the common type.

I went with separate platform interfaces for simplicity. The task implementations are already platform-specific:`AndroidTraktAuthTasks` lives in `androidMain`, `IosTraktAuthTasks` lives in `iosMain`. The business logic that matters (checking token expiry, calling the refresh API) already lives in shared repositories. A common interface would unify the *declaration*, but there's no shared code that needs to operate on tasks generically across platforms.

There are also small differences that are easier to model when each platform owns its contract. Android has typed results (success, retry, failure) where retry hooks into WorkManager's backoff. iOS uses a boolean completion callback. Android declares network constraints upfront. These aren't blockers and we can abstract over them, But doing so adds a layer that isn't needed for the current structure.

The symmetry between the two APIs is intentional. Same shape, same registration pattern, same `setup()`/`schedule()`/`cancel()` flow. It's a convention, not an abstraction. If this evolves to the point where shared code needs to schedule or manage tasks generically, a common interface is the natural next step. For now, keeping it simple means less to maintain and fewer decisions baked into a layer that's hard to change later.


## What Changed for Debugging

The centralization paid off in a few concrete ways.

**Single logging path.** All task submissions, registrations, and execution results now flow through the registry/scheduler. One place to add logging means consistent, complete logs for every task.

**Consistent error handling.** On Android, the `DispatchingWorker` maps `WorkerResult` to WorkManager's `Result` in one place. No more wondering whether a specific worker handles `CancellationException` or not.

**Predictable execution model.** On iOS, every task gets the same expiration handler behavior. When I was debugging a task that seemed to get killed mid-execution, I didn't have to check which concurrency pattern it was using.

**Easier simulation.** When testing via LLDB on iOS, I'm confident that the execution path through `BGTaskRegistry` is the same one that runs in production. The registry dispatches to `execute()` the same way regardless of which task triggered it.


## When to Centralize

Not every case of duplication warrants a library. With one or two background tasks, the original approach was fine. Here's how I'd think about it:

**Centralize when:**
- You have three or more tasks sharing the same platform boilerplate
- Inconsistencies are creeping in across implementations
- Debugging requires understanding task-specific patterns rather than a single common one
- New tasks require touching multiple files that have nothing to do with the task's business logic

**Don't centralize when:**
- You have one or two tasks and the duplication is trivial
- The platform APIs are genuinely different enough between tasks that a common abstraction would leak
- You'd be building infrastructure for hypothetical future tasks


## Final Thoughts

The first background task in a codebase is about getting it to work. The second one validates your pattern. The third one tells you whether that pattern scales.

In this case, it didn't because the pattern was wrong, but because the platform boilerplate was too heavy to copy reliably. Centralizing it into a library meant each new task is just a data declaration and an `execute()` function.

Until we meet again, folks. Happy coding! ✌️


### Resources

- [Previous post: Background Tasks in KMP](/posts/kmp_background_tasks/)
- [WorkManager Documentation](https://developer.android.com/topic/libraries/architecture/workmanager)
- [BGTaskScheduler Documentation](https://developer.apple.com/documentation/backgroundtasks)