---
title: "Breaking Up a Complex Screen with Decompose"
date: "2026-06-21"
draft: false
hideToc: true
tags: ["KMP", "Decompose", "Architecture", "Jetpack Compose", "SwiftUI", "Android", "iOS", "Metro"]
series: "Tv Maniac Journey"
---

This is the first post in a multiple part series on building Tv Maniac's screens and navigation with Decompose. The next two posts will cover navigation and codegen. I'll walk you through how I use Decompose to navigate between screens while keeping screens and features free of navigation logic. I'll also walk you through how I use code generation to write all the DI bindings. This reduces the boilerplate code one needs to write when adding a new feature.

For some context, [Tv Maniac](https://github.com/thomaskioko/tv-maniac) is a Kotlin Multiplatform app that runs on Android and iOS. [Decompose](https://arkivanov.github.io/Decompose/) is the library I use to handle navigation and state on the shared side. Each screen is a component that owns its state and survives configuration changes, much like an Android ViewModel, except the same component drives both the Android and iOS UI.

So, let's take a look at how I reworked a complex screen and made it easier to work with, thanks to [Decompose child components](https://arkivanov.github.io/Decompose/component/child-components/).

## The Discover Screen

The Discover screen is the most complex screen in the app. It is the first screen that is rendered when the user opens the app. It fetches a lot of information: featured shows, upcoming shows, continue watching episodes, and the full show catalog. All this was being managed by a single presenter. It grew and became a pain to manage.

![Discover screen components](https://github.com/user-attachments/assets/e04e89e9-8589-4252-b16f-ec8c6346a57b)

## The Overloaded Presenter

A presenter here plays the same role a ViewModel or state machine does: it holds the screen's state, handles UI actions, and survives configuration changes. With that in mind, the old `DiscoverShowsPresenter` accepted 17 injected dependencies. Here is a trimmed view of its constructor:

##### Before

```kotlin
public class DiscoverShowsPresenter(
    componentContext: ComponentContext,
    private val navigator: Navigator,
    private val discoverShowsInteractor: DiscoverShowsInteractor,
    private val followShowInteractor: FollowShowInteractor,
    private val unfollowShowInteractor: UnfollowShowInteractor,
    private val featuredShowsInteractor: FeaturedShowsInteractor,
    private val topRatedShowsInteractor: TopRatedShowsInteractor,
    private val popularShowsInteractor: PopularShowsInteractor,
    private val trendingShowsInteractor: TrendingShowsInteractor,
    private val upcomingShowsInteractor: UpcomingShowsInteractor,
    private val genreShowsInteractor: GenreShowsInteractor,
    private val markEpisodeWatchedInteractor: MarkEpisodeWatchedInteractor,
    private val observeStartWatchingInteractor: ObserveStartWatchingInteractor,
    private val observeUpNextInteractor: ObserveUpNextInteractor,
    private val accountManager: AccountManager,
    private val localizer: Localizer,
    private val errorToStringMapper: ErrorToStringMapper,
    private val logger: Logger,
) : ComponentContext by componentContext {
    public val presenterInstance: PresenterInstance = instanceKeeper.getOrCreate { PresenterInstance() }

    public inner class PresenterInstance : InstanceKeeper.Instance {
        public val state: StateFlow<DiscoverViewState> = combine(
            upNextActionLoadingState.observable,
            featuredLoadingState.observable,
            topRatedLoadingState.observable,
            popularLoadingState.observable,
            trendingLoadingState.observable,
            upComingLoadingState.observable,
            discoverShowsInteractor.flow,
            uiMessageManager.message,
            _state,
            observeStartWatchingInteractor.flow,
            observeUpNextInteractor.flow,
        ) { ... }.stateIn(...)
    }
}
```

The inner `PresenterInstance` combined 11 flows into a single state object. The action sealed interface had 19 entries covering featured show clicks, catalog navigation, episode marking, follow and unfollow, and Up Next interactions.

The same problem showed up in the data layer. `DiscoverShowsInteractor` was an aggregate that combined six repository streams (featured, top rated, popular, trending, upcoming, genres) into a single `DiscoverShowsData` bundle. Changing anything in the featured section meant reading the entire interactor, the entire presenter, and its entire test suite. Every section's state update could affect every other section's rendering, and there was no clean boundary between them.

## Choosing childContext over childStack

I had already used Decompose's `childStack` to manage navigation between screens. `childStack` is built for navigation: it maintains a back stack and keeps only the topmost component fully alive. That's exactly right for moving from screen to screen.

Inside a single screen, I wanted the opposite. All sections alive at the same time, each managing its own lifecycle, each owning its own state. Decompose has a function for that too: `childContext(key = "...")`. Calling it on a `ComponentContext` creates an independent child context with its own lifecycle, its own `InstanceKeeper`, and its own coroutine scope. The parent stays alive, and so do all the children, simultaneously.

## Splitting the Presenter

The screen now has four child presenters: `DiscoverFeaturedPresenter`, `DiscoverCatalogPresenter`, `DiscoverUpNextPresenter`, and `DiscoverStartWatchingPresenter`. They live in subpackages under `features/discover/presenter/`, each scoped to a custom `DiscoverChildScope`:

```kotlin
public abstract class DiscoverChildScope private constructor()
```

The scope is just a marker class. It doesn't do anything on its own. It exists so the DI graph can tell which components belong to the same child lifecycle tier, separate from the activity scope that owns the host. I'll talk about this more in another article.

The host presenter is now thin:

##### After

```kotlin
@Inject
@NavDestination(
    route = DiscoverRoot::class,
    parentScope = ActivityScope::class,
    kind = DestinationKind.TAB_ROOT,
)
public class DiscoverShowsPresenter(
    componentContext: ComponentContext,
    featuredGraphFactory: DiscoverFeaturedChildGraph.Factory,
    catalogGraphFactory: DiscoverCatalogChildGraph.Factory,
    upNextGraphFactory: DiscoverUpNextChildGraph.Factory,
    startWatchingGraphFactory: DiscoverStartWatchingChildGraph.Factory,
    private val navigator: Navigator,
) : ComponentContext by componentContext {

    private val coroutineScope = coroutineScope()

    public val featuredPresenter: DiscoverFeaturedPresenter =
        featuredGraphFactory.createDiscoverFeaturedGraph(childContext(key = "Featured")).discoverFeaturedPresenter

    public val catalogPresenter: DiscoverCatalogPresenter =
        catalogGraphFactory.createDiscoverCatalogGraph(childContext(key = "Catalog")).discoverCatalogPresenter

    public val upNextPresenter: DiscoverUpNextPresenter =
        upNextGraphFactory.createDiscoverUpNextGraph(childContext(key = "UpNext")).discoverUpNextPresenter

    public val startWatchingPresenter: DiscoverStartWatchingPresenter =
        startWatchingGraphFactory.createDiscoverStartWatchingGraph(childContext(key = "StartWatching"))
            .discoverStartWatchingPresenter

    public val state: StateFlow<DiscoverViewState> = combine(
        featuredPresenter.state,
        catalogPresenter.state,
    ) { featured, catalog ->
        val isRefreshing = featured.isRefreshing || catalog.isRefreshing
        val isEmpty = featured.isEmpty && catalog.isEmpty
        val message = featured.message ?: catalog.message
        DiscoverViewState(
            isRefreshing = isRefreshing,
            isLoading = isRefreshing && isEmpty,
            isEmpty = !isRefreshing && isEmpty,
            showError = message != null && !isRefreshing && isEmpty,
            message = message,
        )
    }.stateIn(
        scope = coroutineScope,
        started = SharingStarted.WhileSubscribed(),
        initialValue = DiscoverViewState.Empty,
    )

    public fun dispatch(action: DiscoverShowAction) {
        when (action) {
            SearchIconClicked -> navigator.navigateTo(SearchRoute)
            RefreshData -> {
                featuredPresenter.refresh()
                catalogPresenter.refresh()
            }
            is MessageShown -> {
                featuredPresenter.clearMessage(action.id)
                catalogPresenter.clearMessage(action.id)
            }
        }
    }
}
```

From 17 dependencies down to 5. From an 11 flow combine down to 2. From 19 actions down to 3. The host knows only what it needs to coordinate: the two sections that contribute to screen level refresh state, navigation to search, and clearing messages on both sections.

One thing worth pointing out: no presenter holds a reference to another presenter. The host exposes each child as a public property and nothing more; the children don't know the host exists, and they definitely don't know about each other. When `RefreshData` is invoked, the host calls `refresh()` on the two sections that care about it, because it's the only place that knows both of them need to reset together. Anything that concerns only one section, like navigating to a show's detail page or expanding a catalog row, is handled inside that child's own `dispatch` function. The host never sees it.

## Declaring a Child Presenter

Each child presenter is annotated with `@ChildPresenter`:

```kotlin
@ChildPresenter(scope = DiscoverChildScope::class, parentScope = DiscoverRoot::class)
@Inject
public class DiscoverFeaturedPresenter(
    componentContext: ComponentContext,
    private val navigator: Navigator,
    private val observeFeaturedShowsInteractor: ObserveFeaturedShowsInteractor,
    private val featuredShowsInteractor: FeaturedShowsInteractor,
    private val accountManager: AccountManager,
    private val errorToStringMapper: ErrorToStringMapper,
    private val logger: Logger,
) : ComponentContext by componentContext { ... }
```

That annotation is what triggers code generation to produce the `DiscoverFeaturedChildGraph.Factory` the host injects, and it contributes that graph into both the Android and iOS dependency graphs. This gets rid of boilerplate code I would have to write manually. How that code generation works, and how I use the same annotations to wire up navigation, is a bigger topic that deserves its own post. The part that matters here is just the annotation itself: it marks a class as a child component and lets the host create it through its own `childContext`.

## What Each Child Owns

`DiscoverFeaturedPresenter` is the clearest example of what a child gets to own. It manages its own coroutine scope, its own `ObservableLoadingCounter`, its own `UiMessageManager`, and its own auth observation:

```kotlin
init {
    observeFeaturedShowsInteractor(Unit)
    fetchFeaturedShows()
    observeAuthState()
}

public val state: StateFlow<DiscoverFeaturedState> = combine(
    loadingState.observable,
    observeFeaturedShowsInteractor.flow,
    uiMessageManager.message,
    _state,
) { isLoading, shows, message, currentState ->
    currentState.copy(
        isInitial = currentState.isInitial && !isLoading && shows.isEmpty() && message == null,
        loading = isLoading,
        featuredShows = shows.toShowList(),
        message = message,
    )
}.stateIn(
    scope = coroutineScope,
    started = SharingStarted.WhileSubscribed(),
    initialValue = _state.value,
)

private fun observeAuthState() {
    coroutineScope.launch {
        accountManager.isConnected
            .drop(1)
            .distinctUntilChanged()
            .filter { it }
            .collect { fetchFeaturedShows(forceRefresh = true) }
    }
}
```

When the user logs in, the Featured section triggers a refresh on its own. The Catalog section does the same for its own data. Neither knows the other exists, and the host doesn't need to manage auth driven refreshes at all. Each child just responds to the signal directly.

The data layer split mirrors the presenter split. The aggregate `DiscoverShowsInteractor` is gone, replaced by five focused interactors that each wrap a single repository call. A focused interactor is trivial to test and trivial to trace when something goes wrong. The old aggregate hid which repository triggered a recomposition; the per category interactors make that obvious just from the name.

## Rendering on Both Platforms

On **Android**, `DiscoverScreen` passes each child presenter straight to its section composable:

```kotlin
@TabUi(presenter = DiscoverShowsPresenter::class, parentScope = ActivityScope::class)
@Composable
public fun DiscoverScreen(presenter: DiscoverShowsPresenter) {
    val hostState by presenter.state.collectAsState()
    // ...
    DiscoverLazyColumn(...) {
        item(key = DiscoverTestTags.FEATURED_PAGER_TEST_TAG) {
            DiscoverFeaturedSection(presenter = presenter.featuredPresenter)
        }
        item(key = DiscoverTestTags.UP_NEXT_SECTION_TEST_TAG) {
            DiscoverUpNextSection(presenter = presenter.upNextPresenter)
        }
        item(key = DiscoverTestTags.ROW_KEY_START_WATCHING) {
            DiscoverStartWatchingSection(presenter = presenter.startWatchingPresenter)
        }
        item(key = DiscoverTestTags.CATALOG_SECTION_TEST_TAG) {
            DiscoverCatalogSection(presenter = presenter.catalogPresenter)
        }
    }
}
```

Each section composable calls `collectAsState()` on its own presenter's `StateFlow`. The host state only drives screen level concerns, like the loading indicator and the error view.

```kotlin
@Composable
public fun DiscoverFeaturedSection(presenter: DiscoverFeaturedPresenter) {
    val state by presenter.state.collectAsState()
    val pagerState = rememberPagerState(pageCount = { state.featuredShows.size })
    DiscoverFeaturedSection(state = state, pagerState = pagerState, onAction = presenter::dispatch)
}
```

On **iOS**, the pattern is structurally identical. `DiscoverScreen` composes `DiscoverFeaturedSection` and a `DiscoverSectionsContent` view that holds the remaining three children:

```swift
private var scrollViewContent: some View {
    ScrollView(showsIndicators: false) {
        VStack(spacing: 0) {
            DiscoverFeaturedSection(presenter: presenter.featuredPresenter)
            DiscoverSectionsContent(presenter: presenter)
        }
    }
}

private struct DiscoverSectionsContent: View {
    let presenter: DiscoverShowsPresenter

    var body: some View {
        VStack {
            DiscoverUpNextSection(presenter: presenter.upNextPresenter)
            DiscoverStartWatchingSection(presenter: presenter.startWatchingPresenter)
            DiscoverCatalogSection(presenter: presenter.catalogPresenter)
        }
    }
}
```

Each iOS section view subscribes to its own presenter's state through `@StateValue`:

```swift
struct DiscoverFeaturedSection: View {
    private let presenter: DiscoverFeaturedPresenter
    @StateValue private var state: DiscoverFeaturedState

    init(presenter: DiscoverFeaturedPresenter) {
        self.presenter = presenter
        _state = .init(presenter.stateValue)
    }

    var body: some View {
        DiscoverFeaturedContent(
            shows: state.featuredShowsSwift,
            // ...
            onShowClicked: { id in
                presenter.dispatch(action: FeaturedShowClicked(showId: id))
            }
        )
    }
}
```

## The Result

One `DiscoverShowsPresenter` with 17 dependencies became a host with 5, plus four children with 5 to 7 dependencies each, scoped to their own concern. The 11 flow combine became four independent combines inside their respective children. The 19 action sealed interface was retired in favor of 3 host actions plus per child action types. The aggregate `DiscoverShowsInteractor`, which combined six repositories, became five single repository interactors.

Unit tests for the featured section no longer require constructing interactors for trending, popular, or catalog behavior. Each child's test stands up only what that child needs.

## When to Use Child Contexts

Use child contexts when a screen has independent blocks that are all on screen at the same time and each manage their own state. If the blocks pull from separate data sources, track separate loading states, or handle actions that only make sense for one section, each one is a candidate for its own child presenter.

Splitting a complex screen into smaller units means more files, but each unit becomes simpler to test, debug, and update. The other costs are a few more generated graph interfaces and a handful of actions the host forwards to its children by hand. For a screen like Discover, I'll pay that price every time. For a simple form with a single submit button, I wouldn't bother.

The test I use now: could I write a focused unit test for one block without knowing anything about the others? If not, the coupling is already there, and it belongs somewhere explicit.

## Final Thoughts

The big presenter wasn't a decision I ever made. It just grew. Each feature I added to Discover was one more dependency and one more branch in dispatch, and no single one felt like a big deal at the time. The cost only became clear once the file crossed 400 lines and setting up a test took longer than writing it.

`childContext` was the right tool for this, and the codegen made the per child DI graph automatic. The result is a screen where I can read, test, and change one section without touching any of the others.

## Resources

- [TvManiac Github Project](https://github.com/thomaskioko/tv-maniac)
- [Decompose on GitHub](https://github.com/arkivanov/Decompose)
- [Decompose documentation](https://arkivanov.github.io/Decompose/)

Until we meet again, folks. Happy coding! ✌️
