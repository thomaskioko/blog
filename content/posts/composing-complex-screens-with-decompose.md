---
title: "Composing Complex Screens with Decompose"
date: "2026-06-21"
draft: false
hideToc: true
tags: ["KMP", "Decompose", "Architecture", "Jetpack Compose", "SwiftUI", "Android", "iOS", "Metro"]
series: "Tv Maniac Journey"
---

The three-part Navigation Refactor series covered how Decompose drives screen-to-screen movement in Tv Maniac: decentralized routes, codegen-wired bindings, and platform-specific renderers. That story was about what happens between screens. This post is about what happens inside one.

The Discover screen is the most complex screen in the app. It is the first thing a user sees, it holds featured shows, upcoming picks, a continue-watching row, and the full show catalog, and for a long time all of that lived inside a single presenter. I kept telling myself the file was manageable. It was not.

## When one presenter does too much

The old `DiscoverShowsPresenter` accepted 17 injected dependencies. Here is a trimmed view of its constructor:

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

The inner `PresenterInstance` combined 11 flows into a single state object. The action sealed interface had 19 entries covering featured show clicks, catalog navigation, episode marking, follow and unfollow, Up Next interactions, and everything in between. A unit test had to construct all 17 dependencies just to exercise one piece of behavior.

The data layer mirrored the problem. `DiscoverShowsInteractor` was an aggregate: one `SubjectInteractor` that combined six repository streams (featured, top-rated, popular, trending, upcoming, genres) and emitted a `DiscoverShowsData` bundle.

```kotlin
// The aggregate interactor that got retired
public class DiscoverShowsInteractor(
    private val featuredShowsRepository: FeaturedShowsRepository,
    private val topRatedShowsRepository: TopRatedShowsRepository,
    private val popularShowsRepository: PopularShowsRepository,
    private val trendingShowsRepository: TrendingShowsRepository,
    private val upcomingShowsRepository: UpcomingShowsRepository,
    private val genreRepository: GenreRepository,
) : SubjectInteractor<Unit, DiscoverShowsData>() {

    override fun createObservable(params: Unit): Flow<DiscoverShowsData> = combine(
        genreRepository.observeGenresWithShows(),
        featuredShowsRepository.observeFeaturedShows(),
        topRatedShowsRepository.observeTopRatedShows(),
        popularShowsRepository.observePopularShows(),
        trendingShowsRepository.observeTrendingShows(),
        upcomingShowsRepository.observeUpcomingShows(),
    ) { ... }
}
```

Changing anything in the featured section meant reading the entire interactor, the entire presenter, and its entire test suite. Every section's state update could affect every other section's rendering. There was no seam.

## What Decompose offers inside a screen

I had already used Decompose's `childStack` to manage navigation between screens. `childStack` is designed for navigation: it maintains a back stack and only keeps the top-most component fully alive. That works perfectly for going from screen to screen.

Inside a single screen, you want the opposite: all sections alive at the same time, each managing its own lifecycle, each owning its own state. Decompose provides `childContext(key = "...")` for exactly this. Calling `childContext` on a `ComponentContext` creates an independent child context with its own lifecycle, `InstanceKeeper`, and coroutine scope. The parent stays alive; all children stay alive simultaneously.

I first encountered a systematic write-up of this pattern in Artur Artikov's "Component-based Approach" series on ITNEXT, which describes organizing screens into independent functional blocks. The pattern fits naturally onto what Decompose already gives you.

The key distinction: `childStack` for navigation (one active child at a time), `childContext` for composition (all children alive together).

## Splitting Discover into a host and four children

The screen now has four child presenters: `DiscoverFeaturedPresenter`, `DiscoverCatalogPresenter`, `DiscoverUpNextPresenter`, and `DiscoverStartWatchingPresenter`. They live in subpackages under `features/discover/presenter/`.

Each child is scoped to a custom `DiscoverChildScope`:

```kotlin
// features/discover/nav/src/commonMain/kotlin/.../discover/nav/scope/DiscoverChildScope.kt
public abstract class DiscoverChildScope private constructor()
```

The scope is just a marker. It tells the DI graph which components belong to the same child lifecycle tier, separate from the activity scope that owns the host.

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

From 17 dependencies down to 5. From an 11-flow combine down to 2. From 19 actions down to 3. The host knows only what it needs to coordinate: the two sections that contribute to screen-level refresh state, navigation to search, and message clearing fanout.

## How codegen wires the child graphs

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

KSP picks this up and generates a `DiscoverFeaturedChildGraph` interface at build time:

```kotlin
// Generated — do not edit
@GraphExtension(DiscoverChildScope::class)
public interface DiscoverFeaturedChildGraph {
    public val discoverFeaturedPresenter: DiscoverFeaturedPresenter

    @ContributesTo(DiscoverRoot::class)
    @GraphExtension.Factory
    public interface Factory {
        public fun createDiscoverFeaturedGraph(
            @Provides componentContext: ComponentContext,
        ): DiscoverFeaturedChildGraph
    }
}
```

The `@ContributesTo(DiscoverRoot::class)` on the factory tells Metro to contribute this factory into any graph that has `DiscoverRoot` as its scope. No manual `build.gradle.kts` edits are needed, and no binding module to hand-write. The Android graph picks it up and so does the iOS framework graph. Both platforms get the same child presenter instances resolved through the same DI chain.

## A child owns its slice end to end

`DiscoverFeaturedPresenter` is the clearest example. It manages its own coroutine scope, its own `ObservableLoadingCounter`, its own `UiMessageManager`, and its own auth observation:

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

When the user logs in, the Featured section triggers a refresh independently. The Catalog section does the same for its own data. Neither section knows the other exists. The host does not need to orchestrate auth-driven refreshes; each child responds to the signal directly.

The data decomposition mirrors the presenter split. The aggregate `DiscoverShowsInteractor` is retired. In its place are five focused interactors: `ObserveFeaturedShowsInteractor`, `ObserveTopRatedShowsInteractor`, `ObservePopularShowsInteractor`, `ObserveTrendingShowsInteractor`, and `ObserveUpcomingShowsInteractor`. Each one wraps a single repository call:

```kotlin
@Inject
public class ObserveFeaturedShowsInteractor(
    private val repository: FeaturedShowsRepository,
) : SubjectInteractor<Unit, List<ShowEntity>>() {

    override fun createObservable(params: Unit): Flow<List<ShowEntity>> =
        repository.observeFeaturedShows()
}
```

A focused interactor is trivial to test and trivial to trace when something goes wrong. The aggregate hid which repository triggered a recomposition; the per-category interactors make that obvious.

## How the host coordinates without coupling

There are no presenter-to-presenter dependencies. The host exposes child presenters as public properties. The children do not hold references to each other or to the host.

Fan-out in the host dispatch is explicit and deliberate:

```kotlin
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
```

The host knows that `RefreshData` should reset both the featured and catalog sections. The children expose `refresh()` and `clearMessage()` as stable APIs. Anything that concerns only a single section, like navigating to a show detail or clicking "more" on a catalog row, stays inside that child's own dispatch.

## Both platforms, the same children

On **Android**, `DiscoverScreen` passes each child presenter directly to its section composable:

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

Each section composable calls `collectAsState()` on its own presenter's `StateFlow`. The host state drives only screen-level concerns like the loading indicator and the error view.

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

Each iOS section view subscribes to its own presenter state using `@StateValue`:

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

The Kotlin-to-Swift name export is worth a quick note. Because KMP flattens namespaces in the Objective-C header, action types across children need distinct names. `FeaturedShowClicked` (in the featured package) and `CatalogShowClicked` (in the catalog package) are both exported unambiguously. Collisions would produce fragile `_`-suffixed names that break whenever module structure changes, so keeping action names distinct across children is part of the contract.

## What the split produced

One `DiscoverShowsPresenter` with 17 dependencies became a host with 5 plus four children, each with 5 to 7 dependencies scoped to their own concern. The 11-flow combine became four independent combines inside their respective children. The 19-action sealed interface was retired in favor of 3 host actions plus per-child action types. The aggregate `DiscoverShowsInteractor` combining six repositories became five single-repository interactors.

Unit tests for the featured section no longer require constructing interactors for trending, popular, or catalog behavior. Each child's test stands up only what that child needs.

## When to reach for this pattern

**Reach for child contexts when** a screen has independent, simultaneously-alive blocks that each manage state on their own. If the blocks have separate data sources, separate loading states, or separate actions that have no cross-section meaning, they are candidates for their own child presenter.

**Accept the trade-off.** More files, more generated graph interfaces, and a small amount of explicit fan-out in the host dispatch. For a screen like Discover, that trade-off is worth it. For a simple form screen with a single submit action, it is overkill.

The test for whether to split: could you write a focused unit test for one block without knowing anything about the other blocks? If not, the coupling is already there, and it belongs somewhere explicit.

## Pull requests

[TODO: Add PR link]

## Final thoughts

The monolith was not a design choice so much as accumulated inertia. Each feature added to Discover was one more dependency and one more branch in dispatch, and no single addition felt dramatic. The structural cost only became visible when the file crossed 400 lines and a test setup took longer than the test itself.

Decompose's `childContext` gave me the right primitive. The codegen made the per-child DI graph automatic. The result is a screen where each section can be reasoned about, tested, and extended without touching anything else. That is what good decomposition looks like.

## Resources

- [Component-based Approach: Implementing Screens with Decompose (Artur Artikov, ITNEXT)](https://itnext.io/component-based-approach-implementing-screens-with-the-decompose-library-2b47f3e40bf6)
- [Decompose on GitHub](https://github.com/arkivanov/Decompose)
- [Decompose documentation](https://arkivanov.github.io/Decompose/)
- [Decentralizing Navigation in KMP: Part 1](/posts/decentralizing_navigation_kmp/)
- [Decentralizing Navigation in KMP: Part 2](/posts/decentralizing_navigation_kmp_part2/)
- [Decentralizing Navigation in KMP: Part 3](/posts/decentralizing_navigation_kmp_part3/)

This is a post in the **Tv Maniac Journey** series.

Until we meet again, folks. Happy coding! ✌️
