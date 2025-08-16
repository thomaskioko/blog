---
title: "Simplifying KMP Test Infrastructure with kotlin-inject-anvil"
date: "2025-08-02"
draft: false
hideToc: true
tags: ["KMM",  "kotlin-inject", "kotlin-inject-anvil", "dependency injection",]
series: "Tv Maniac Journey"
---


Testing in projects can quickly become complex, especially when managing dependencies across different platforms. In this post, I'll share my journey of refactoring TvManiac's test infrastructure from manually creating fake presenter factories to using proper dependency injection with [kotlin-inject-anvil](https://github.com/amzn/kotlin-inject-anvil) to create a test component that wires the fixtures for us.

## The Problem: Manual Test Doubles

Previously, our tests looked like this. This was a big object and I had to change it every time I added a new feature/depednecy to the project.

```kotlin
class DefaultRootComponentTest {
    @Test
    fun `initial state should be Home`() = runTest {
        val testComponent = TestComponent::class.create()
        val rootComponent = DefaultRootPresenter(
            componentContext = componentContext,
            rootNavigator = FakeDefaultRootNavigator(),
            discoverFactory = FakeDiscoverPresenterFactory(),
            homeFactory = FakeHomePresenterFactory(
                traktAuthRepository = FakeTraktAuthRepository()
            ),
            // ... more fake factories
        )

        // Test implementation
    }
}
```

`RootComponentTest` required manually creating fake presenter factories, leading to:
- Boilerplate code duplication across tests
- Maintenance overhead when adding new dependencies
- No compile-time verification of dependency completeness

## The Solution: Test Components with DI

The goal was simple: leverage kotlin-inject-anvil to create test components that automatically provide all required dependencies, just like our production code.

> Note: This pattern is not used across the entire project. I am using this in the rootComponent as this requires quite some dependencies and the graph is a bit complex. Other unit tests use the fakes directly and there's no need for the test component. This solution is for complex objects. In this case, the App's root component.

### Step 1: Creating the Test Scope

First, I defined a dedicated scope for test dependencies:

```kotlin
interface TestScope
```

### Step 2: Test Modules for Dependencies

To provide test implementations, I created `TestDataModule` that adds all the required depednecies. We use fakes in the case and not the actual implementation. With the use of dependency inversion, we can easily create fake implementations.

```kotlin

@ContributesTo(TestScope::class)
interface TestDataModule {
    @Provides
    @SingleIn(TestScope::class)
    fun provideDatastoreRepository(): DatastoreRepository = FakeDatastoreRepository()

    @Provides
    @SingleIn(TestScope::class)
    fun provideTraktAuthManager(): TraktAuthManager = FakeTraktAuthRepository()
}
```

### Step 3: Platform-Specific Test Components

I then created platform-specific test components instead of trying to share them in `commonMain`. Since we are using DI, anvil will generate the speficif component for each plaform. We need to extend the platform component in our test since this will have all the bindigns with everything stiched together. So we need to do structure our tests to follow this structure.

> Note: That generated type is named after our component with the Merged suffix: `TestJvmComponentMerged`.


```kotlin

@Component
@SingleIn(TestScope::class)
@MergeComponent(TestScope::class)
abstract class TestJvmComponent : TestJvmComponentMerged {
    abstract val datastoreRepository: DatastoreRepository
    abstract val traktAuthManager: TraktAuthManager
    abstract val rootPresenterFactory: DefaultRootPresenter.Factory
    abstract val homePresenterFactory: DefaultHomePresenter.Factory
}
```

The iOS component follows the same pattern:

```kotlin

@Component
@SingleIn(TestScope::class)
@MergeComponent(TestScope::class)
abstract class TestIosComponent : TestIosComponentMerged {
    abstract val datastoreRepository: DatastoreRepository
    abstract val traktAuthManager: TraktAuthManager
    abstract val rootPresenterFactory: DefaultRootPresenter.Factory
    abstract val homePresenterFactory: DefaultHomePresenter.Factory
}
```


### Step 4: Refactoring Tests

The real magic happens in the test refactoring. I transformed tests to use an abstract class pattern that separates shared test logic from platform-specific DI setup. The platform spefict test simply initialize the objects and all the tests reside in `commonMain`. Each platform then initialized it's component. 
- `Jvm` -> `TestJvmComponent`
- `iOS` -> `TestIosComponent`

#### `commonMain`

```kotlin
abstract class DefaultRootComponentTest {
    abstract val rootPresenterFactory: DefaultRootPresenter.Factory
    abstract val datastoreRepository: DatastoreRepository

    @Test
    fun `initial state should be Home`() = runTest {
        val rootComponent = rootPresenterFactory.create(componentContext)

        turbineScope {
            rootComponent.stack.test {
                val config = awaitItem().active.configuration
                assertThat(config).isInstanceOf(Config.Home::class)
            }
        }
    }
}
```

Platform-specific implementations then provide the dependencies:

#### `jvmTest`

```kotlin

internal class DefaultRootComponentJvmTest : DefaultRootComponentTest() {
    private val testComponent: TestJvmComponent = TestJvmComponent::class.create()

    override val rootPresenterFactory: DefaultRootPresenter.Factory
        get() = testComponent.rootPresenterFactory

    override val datastoreRepository: DatastoreRepository
        get() = testComponent.datastoreRepository
}
```

#### `iOSTest`

```kotlin

internal class DefaultRootComponentJvmTest : DefaultRootComponentTest() {
    private val testComponent: TestIosComponent = TestIosComponent.create()

    override val rootPresenterFactory: DefaultRootPresenter.Factory
        get() = testComponent.rootPresenterFactory

    override val datastoreRepository: DatastoreRepository
        get() = testComponent.datastoreRepository
}
```

This allows us to easily scale our tests. Say we add support for desktop app, we only need to add the `DesktopComponent` and leverage the existing tests.

## Key Learnings

### 1. Dependnency Inversion & Tests

The project uses Dependency Inversion pattern and this makes testing effective. This allows us to decouple dependencies and test them in isolation. We also use Fakes in the projects and no mocks. With the use of dependency inversion, we can avoid using mocks in the project. Below are some good articles that talk more about this. 

- [Fakes Are Great, But Mocks I Hate](https://www.billjings.net/posts/title/fakes-are-great-but-mocks-i-hate/?up=technical)
- [Replacing Mocks](https://ryanharter.com/blog/2020/06/replacing-mocks/)
- [Test Doubles](https://abseil.io/resources/swe-book/html/ch13.html)
- [Use test doubles in Android](https://developer.android.com/training/testing/fundamentals/test-doubles#types)


### 2. Platform-Specific Test Components

Initially, I tried placing `TestComponent` in `commonMain`, but this doesn't work with kotlin-inject's code generation. Each platform needs its own component to properly generate the dependency graph.

### 3. Useful for Complex Objects

As much as this seems like overkill for this project, this demonstrates how this can be done in bigger projects. This also allows us to easily add new dependencies without touching the tests


```


## What's Next?

This test infrastructure sets the foundation for more robust testing patterns. Future improvements could include:
- Adding integration test components with real implementations
- Exploring shared test components for cross-platform integration tests

The complete implementation is available in the [TvManiac repository](https://github.com/thomaskioko/tv-maniac).
---

