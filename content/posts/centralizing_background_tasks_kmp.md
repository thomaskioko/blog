---
title: "Centralizing Background Tasks Implementation"
date: "2026-02-15"
draft: false
hideToc: true
tags: ["KMP", "WorkManager", "BGTaskScheduler", "background-tasks", "architecture"]
series: "Tv Maniac Journey"
---

# Intro

In a [previous post](/posts/kmp_background_tasks/), I walked through setting up background tasks in KMP for token refresh. After that post, I added two more background tasks: library sync and episode notifications. Same pattern, same structure. That's when I noticed a gap in my implementation.

Each new task meant copy pasting the same registration, scheduling, and execution boilerplate with slight variations. Three tasks across two platforms, and I was already seeing inconsistencies that made debugging harder than it needed to be.

This post covers how I centralized all background task logic, first by extracting platform boilerplate into shared registries, and then by going a step further and unifying everything into a single common API.


## What Went Wrong

The original pattern was sound in isolation. But once I had three tasks on each platform, the duplication became a problem.

On **iOS**, each task repeated the same six steps: register with `BGTaskScheduler`, build a `BGAppRefreshTaskRequest`, submit it, filter by identifier in the handler, manage the execution window with an expiration callback, and reschedule after completion. That's a lot of surface area to get wrong, and each task implemented it slightly differently.

On **Android**, each feature required *two* classes: a `*Tasks` class for scheduling and a `*Worker` class for execution. Three features meant six classes, plus a factory that had to manually map every worker class name.

Three tasks, three different approaches to concurrency and error handling. None of them were wrong individually. But when you're debugging a background task that didn't fire at 2am, the last thing you want is to remember which concurrency pattern *this particular task* uses.


## The Inconsistency

When background tasks fail, they fail silently. Of course you can add logs and crash reporting for that, but I am yet to implement that in my project. So debugging had to happen manually for me. After debugging and finding what the issue, I need to make sure all tasks apply the fix.

There's also the onboarding cost. If another engineer decided to contribite to the project and they need to add a background task, which of the existing implementations do they follow? They'll pick one, probably the most recent, and introduce yet another slight variation.

Why I was introducing background tasks, the architecture worked. But as I contunied adding more features, I saw the issue; Inconsistency!


## The First Approach

The first fix was straightforward: extract the platform boilerplate into a shared library in `core/tasks/` and have each task declare only what's unique to it.

On **iOS**, each task implemented a `BackgroundTask` interface:

```kotlin
interface BackgroundTask {
    val taskId: String
    val interval: Double
    suspend fun execute()
}
```

A `BGTaskRegistry` handled all the registration, scheduling, execution window management, and rescheduling.

On **Android**, each task implemented `BackgroundWorker`:

```kotlin
interface BackgroundWorker {
    val workerName: String
    val interval: Duration
    val constraints: WorkerConstraints
    suspend fun execute(): WorkerResult
}
```

A single `DispatchingWorker` replaced all three `CoroutineWorker` subclasses. It looked up the registered worker by name and delegated execution. The worker factory went from a manual switch statement to a single line.

The before/after on a task implementation tells the story. Here's the iOS token refresh task:

##### Before

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

##### After

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

The task only contains the *what*. The platform machinery lives in one place.

This worked. Debugging got easier, adding new tasks got simpler. But there was still a problem I'd intentionally left on the table.


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

After centralizing, each feature became a single class, and the factory collapsed to one line:

```kotlin
return when (workerClassName) {
    name<DispatchingWorker>() -> dispatchingWorker(appContext, workerParameters)
    else -> null
}
```

Three worker classes deleted. Factory no longer needs to know about individual features.


## The Problem with Two Interfaces

At the end of that first centralization, I wrote a section in my notes titled "Why Not a Common API?" and made the case for keeping iOS and Android on separate interfaces. The reasoning was: the task implementations are already platform-specific, the business logic lives in shared repositories, and the small differences between platforms are easier to model when each platform owns its contract.

That reasoning was valid when the only thing I was centralizing was platform boilerplate. But it missed something that became obvious once I started maintaining the code.

The *workers themselves* were duplicated across platforms.

Token refresh? Same logic in `AndroidTraktAuthTasks.execute()` and `IosTraktAuthTasks.execute()`. Library sync? Same calls to the same shared repositories. Episode notifications? Same three-step pipeline. The business logic was identical. Only the class name and the source set were different.

I had six worker classes doing the same thing. When I fixed a bug in the Android token refresh worker, I had to remember to apply the same fix to the iOS one. That's the exact problem the first centralization was supposed to solve, and it did for platform boilerplate, but not for the workers themselves.


## The Unified Approach

Around this time, I came across [Meeseeks](https://docs.meeseeks.mattramotar.dev/introduction), a KMP library for background task scheduling. It takes the exact approach I'd been avoiding: a single common API with platform implementations behind the scenes. Seeing how they modeled the worker interface and scheduler gave me the push to stop maintaining two copies of the same logic. I didn't end up using the library directly since my use case is simple enough and I already had a lot of the infrastructure in place, but it validated the direction and inspired the shape of the API.

The answer was the common API I'd talked myself out of the first time. One `BackgroundWorker` interface in `commonMain`, one `BackgroundTaskScheduler` interface in `commonMain`, and workers that live in shared code.

Here's the shared API:

```kotlin
interface BackgroundWorker {
    val workerName: String
    suspend fun doWork(): WorkerResult
}

interface BackgroundTaskScheduler {
    fun schedulePeriodic(request: PeriodicTaskRequest)
    fun scheduleAndExecute(request: PeriodicTaskRequest)
    fun cancel(id: String)
    fun cancelAll()
}

data class PeriodicTaskRequest(
    val id: String,
    val intervalMs: Long,
    val constraints: TaskConstraints = TaskConstraints(),
)
```

Workers now live in `commonMain`. Here's what the token refresh worker looks like:

```kotlin
@Inject
@SingleIn(AppScope::class)
@ContributesBinding(AppScope::class, boundType = BackgroundWorker::class, multibinding = true)
class TokenRefreshWorker(
    private val traktAuthRepository: Lazy<TraktAuthRepository>,
    private val logger: Logger,
) : BackgroundWorker {

    override val workerName: String = WORKER_NAME

    override suspend fun doWork(): WorkerResult {
        val authState = traktAuthRepository.value.getAuthState() ?: return WorkerResult.Success
        if (!authState.isExpiringSoon()) return WorkerResult.Success

        return when (traktAuthRepository.value.refreshTokens()) {
            is TokenRefreshResult.Success -> WorkerResult.Success
            is TokenRefreshResult.NetworkError -> WorkerResult.Retry("Network error during token refresh")
            else -> WorkerResult.Failure("Token refresh failed")
        }
    }

    internal companion object {
        internal const val WORKER_NAME = "com.thomaskioko.tvmaniac.tokenrefresh"
        private const val FIVE_DAYS_MS = 5L * 24 * 60 * 60 * 1000

        internal val REQUEST = PeriodicTaskRequest(
            id = WORKER_NAME,
            intervalMs = FIVE_DAYS_MS,
            constraints = TaskConstraints(requiresNetwork = true),
        )
    }
}
```

One class. Shared code. Runs on both platforms. The scheduling request is a companion on the worker itself. No separate `*Tasks` class, no platform-specific wrapper.

Workers register themselves via kotlin-inject multibinding. A `DefaultWorkerFactory` collects all `BackgroundWorker` implementations and maps them by name:

```kotlin
class DefaultWorkerFactory(
    workers: Set<BackgroundWorker>,
) : WorkerFactory {
    private val registry: Map<String, BackgroundWorker> = workers.associateBy { it.workerName }

    override fun createWorker(workerName: String): BackgroundWorker? = registry[workerName]
}
```

Adding a new worker means implementing `BackgroundWorker` with the multibinding annotation. No factory updates, no manual registration. The DI framework handles discovery.

The platform-specific code that remains is the scheduler, the thin wrapper that actually talks to WorkManager or `BGTaskScheduler`. These don't know anything about specific tasks. They receive a `PeriodicTaskRequest` and a `workerName`. Workers are completely decoupled from the platform.


### Initializers Tie It Together

Each feature has an initializer that reacts to app state and schedules work accordingly:

```kotlin
class TokenRefreshInitializer(
    private val scheduler: BackgroundTaskScheduler,
    private val traktAuthRepository: TraktAuthRepository,
    dispatchers: AppCoroutineDispatchers,
) : AppInitializer {
    private val scope = CoroutineScope(SupervisorJob() + dispatchers.io)

    override fun init() {
        scope.launch {
            traktAuthRepository.state.distinctUntilChanged().collectLatest { state ->
                when (state) {
                    TraktAuthState.LOGGED_IN -> scheduler.schedulePeriodic(TokenRefreshWorker.REQUEST)
                    TraktAuthState.LOGGED_OUT -> scheduler.cancel(TokenRefreshWorker.WORKER_NAME)
                }
            }
        }
    }
}
```

The initializer observes auth state and tells the scheduler what to do. The scheduler talks to the OS. The worker does the work. Clean separation.


## The Cleanup

Eighteen files deleted across the codebase. Six platform-specific worker classes, three domain-level `*Tasks` interfaces, six platform-specific `*Tasks` implementations, and three old infrastructure classes. Replaced by three shared workers, a shared scheduler interface, a worker factory, and two platform schedulers.

Six worker classes collapsed into three. The domain layer no longer needs to define task interfaces at all. It just implements `BackgroundWorker` and declares a `REQUEST`.



## What Changed for Debugging

All task submissions, registrations, and execution results now flow through one scheduler per platform. One place to add logging means consistent, complete logs for every task. Before, I had to check each task's implementation to understand how it handled errors or what it logged. Now the scheduler handles that uniformly.

On Android, `SchedulerDispatchWorker` maps `WorkerResult` to WorkManager's `Result` in one place. On iOS, `IosTaskScheduler` handles the expiration window the same way for every task. When something goes wrong, I'm not jumping between three different implementations trying to figure out which concurrency pattern this particular task uses. There's one path, one set of logs, one place to fix things.

That's the real win of centralizing. It's not just less code, it's fewer places to look when something breaks.


## A Lesson in Timing

In the first pass, I argued against a common API because "there's no shared code that needs to operate on tasks generically across platforms." That was true at the time. The registries and schedulers were the pain point, and centralizing those per-platform was the right call.

But the workers were always going to converge. The business logic was shared from day one; it lived in shared repositories. The only reason the workers were platform-specific was because they had to conform to platform-specific interfaces. Once I unified the interface, the workers naturally moved to shared code.

I don't think I should have built the common API from the start. With one task, separate platform interfaces were simpler. With three tasks and identical business logic on both sides, the common API became obvious. The right abstraction reveals itself when the duplication does.


## When to Centralize

Not every case of duplication warrants a library. With one or two background tasks, the original approach was fine. Here's how I'd think about it:

**Centralize** when you have three or more tasks sharing the same platform boilerplate, when inconsistencies are creeping in across implementations, or when new tasks require touching multiple files that have nothing to do with the task's business logic.

**Unify** when the business logic in your workers is identical across platforms and you're fixing the same bug in two places. If adding a new task means creating two classes with the same body in different source sets, you've outgrown separate interfaces.

**Don't centralize** when you have one or two tasks and the duplication is trivial, or when you'd be building infrastructure for hypothetical future tasks.


## Pull Requests

If you want to dig into the implementation, here are the pull requests for each iteration:

- [First approach: Centralizing platform boilerplate](https://github.com/thomaskioko/tv-maniac/pull/759)
- [Unified approach: Shared workers and scheduler](https://github.com/thomaskioko/tv-maniac/pull/774)


## Final Thoughts

The first background task in a codebase is about getting it to work. The second one validates your pattern. The third one tells you whether that pattern scales.

I went through two iterations here. The first centralization extracted platform boilerplate into shared registries and kept workers platform-specific. That was the right call at the time. The second iteration unified the workers themselves into shared code, because the duplication had shifted from platform machinery to business logic.

Neither iteration was wasted. The first one taught me the shape of the problem. The second one solved the part I'd left on the table. Build for what you know. Refactor when the code tells you to.

Until we meet again, folks. Happy coding! ✌️


### Resources

- [Previous post: Background Tasks in KMP](/posts/kmp_background_tasks/)
- [Meeseeks](https://docs.meeseeks.mattramotar.dev/introduction)
- [WorkManager Documentation](https://developer.android.com/topic/libraries/architecture/workmanager)
- [BGTaskScheduler Documentation](https://developer.apple.com/documentation/backgroundtasks)
