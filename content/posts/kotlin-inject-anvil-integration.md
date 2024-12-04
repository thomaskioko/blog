---
title: "Integrate Kotlin-Inject-Anvil To Tv Maniac"
date: "2024-11-30"
draft: false
hideToc: true
tags: ["KMM",  "kotlin-inject", "kotlin-inject-anvil", "anvil",]
series: "Tv Maniac Journey"
---

# Intro
If you've used Anvil before, you know it takes away alot the boilerplate code and make DI seamless. If Anvil is new to you, it basically allows you to contribute dagger modules and compoment interfaces to your DI graph and merges all the contributions and add them to your component during compilation. Ralf Wonderatschek and Gabriel Peal gave an in-depth talk about this. [Dagger + Anvil: Learning to Love Dependency Injection.](https://www.droidcon.com/2022/06/28/dagger-anvil-learning-to-love-dependency-injection/). You should check it out.

 I have been using [kotlin-inject](https://github.com/evant/kotlin-inject) on my pet project for a while now and I have had a good time with it coming from using Dagger in other projects. One thing I missed was using Anvil. This was not availalbe until recently. [kotlin-inject-anvil](https://github.com/amzn/kotlin-inject-anvil?tab=readme-ov-file) joined the chat. 
 
 This article will focus on my expericence and journey integrating/migrating to kotlin-inject-anvil into the project. 

 ## Koltlin-Inject-Anvil Integration

 Before integrating [kotlin-inject-anvil](https://github.com/amzn/kotlin-inject-anvil?tab=readme-ov-file), one thing that bothered me was how to approach the integration/migration. I thought the process would be a pain as I already have multiple modules in my project. Do I rip the bandaid off and do it all at once? Is it possible to do it gradually? Spoiler alert: it is possible to do it gradually. This approach might not work for your project, depending on the size of the team. There are multiple ways of doing this, but this worked for me. This approach made it easier to determine if I broke the current implementation or introduced new errors.
 
 Here's a quick overview of how I approached the migration.

 - Add dependencies
 - Apply`@ContributesTo` annotation
 - Apply `@ContributesBinding` annotation
 - Add ksp kotlin-inject-anvil compiler dependencies.
 - Delete component interfaces. 
 - Replace `@Component` with `@MergeComponent` and create subcomponent.

 Let's take a quick look at how each of these steps is implemented. If you'd like to see the code, here's the [pull request](https://github.com/thomaskioko/tv-maniac/pull/363).

 
 ### Add kotin-inject-anvil Dependencies.
 This is pretty straightforward. We need to add the dependencies to our project.

 ``` yaml
 kotlinInject-anvil-compiler = { group = "software.amazon.lastmile.kotlin.inject.anvil", name = "compiler", version.ref = "kotlin-inject-anvil" }
 kotlinInject-anvil-runtime = { group = "software.amazon.lastmile.kotlin.inject.anvil", name = "runtime", version.ref = "kotlin-inject-anvil" }
 kotlinInject-anvil-runtime-optional = { group = "software.amazon.lastmile.kotlin.inject.anvil", name = "runtime-optional", version.ref = "kotlin-inject-anvil" }
 ```
 
 `kotlinInject-anvil-runtime-optional` is optional, and your project would work without it. I added it so I can get rid of my custom scope and use kotlin-inject-anvil's scopes to keep everything consistent.

 To make things easier, I created a bundle with kotlin-inject dependencies, and I use that instead.

``` yaml
 [bundles]
kotlinInject = [
  "kotlinInject-runtime",
  "kotlinInject-anvil-runtime",
  "kotlinInject-anvil-runtime-optional"
]
```

We can then add it to our module like so. `implementation(libs.bundles.kotlinInject)`


### Add `@ContributesTo` Annotation
We can now annotate our interface components with `@ContributesTo`. I also replaced my custom scope with kotlin-inject-anvil scope: `@ApplicationScope` -> `@SingleIn(AppScope::class)`. As I mentioned, this is optional, and it will work with your custom scopes. Here's how the component looks.

##### Before
``` kotlin
interface CastComponent {

  @Provides
  @ApplicationScope  
  fun provideCastDao(bind: DefaultCastDao): CastDao = bind

  @Provides
  @ApplicationScope
  fun provideCastRepository(bind: DefaultCastRepository): CastRepository = bind
}

```
##### After

``` kotlin
@ContributesTo(AppScope::class)
interface CastComponent {

  @Provides
  @SingleIn(AppScope::class)
  fun provideCastDao(bind: DefaultCastDao): CastDao = bind

  @Provides
  @SingleIn(AppScope::class)
  fun provideCastRepository(bind: DefaultCastRepository): CastRepository = bind
}
```

One small thing I did later was move the `@SingleIn` annotation to the class instead of having it in the binding functions.

### Add `@ContributesBinding` Annotation

The next thing we can do is annotate all classes that have interface implementations with `@ContributesBinding`. Once we've plugged everything in, Anvil will provide the bindings for us, and we can get rid of the component above with the manual binding.


##### Before

```kotlin
@Inject
class DefaultCastRepository(
  private val dao: CastDao,
) : CastRepository {
    ...
}
```

##### After
```kotlin
@Inject
@ContributesBinding(AppScope::class)
class DefaultCastRepository(
  private val dao: CastDao,
) : CastRepository {
    ...
}
```

### Add KSP Dependencies.
In order to check if the changes we've made work as intedned, we can add Kotlin inject Anvil compiler dependency which will generate the component classes.
`addKspDependencyForAllTargets(libs.kotlinInject.anvil.compiler)`. `addKspDependencyForAllTargets` is an extension function that created KSP configurations for each target. e.g `kspAndroid` `kspIosArm64`

We can buld our app and take a look at the generated code.

![Generated Code](https://github.com/user-attachments/assets/e6f836eb-8012-4e1f-9d93-73c2e96cf6bf)


Anvil will generate the bindings for us similarly to what we had above. This will be generated for all our classes annotated with `@ContributesBinding(AppScope::class)`.

``` kotlin
@Origin(value = DefaultCastRepository::class)
public interface ComThomaskiokoTvmaniacDataCastImplementationDefaultCastRepository {
  @Provides
  public
      fun provideDefaultCastRepositoryCastRepository(defaultCastRepository: DefaultCastRepository):
      CastRepository = defaultCastRepository
}

```

### Delete Manual Bindings. 

Now that our bindings and components are being generated for us, we can delete our component interfaces that have provider functions. 

In my previous implementation, each module was responsible for creating its own DI component. The shared module then added all these SuperType Components to the parent/final component for each platform component. This is a bit painful and can easily get out of hand as your project grows. 😮‍💨

![SharedComponent](https://github.com/user-attachments/assets/1941b434-dd31-49d6-a265-92f893bb2739)

Thanks to kotlin-inject-anvil, we can get rid of these as it's now generated for us once we add the merge annotation. 🥳

## Final Boss: `@MergeComponent` Annotation

### `@ContributesSubcomponent` Annotation

Since we can only have one component annotated with `@MergeComponent`, we need to annotate `ActivityComponent` to `@ContributesSubcomponent`, create a factory with will be implemented by our parent scope.

##### Before

``` kotlin
@SingleIn(ActivityScope::class)
@Component
abstract class ActivityComponent(
  @get:Provides val activity: ComponentActivity,
  @get:Provides val componentContext: ComponentContext = activity.defaultComponentContext(),
  @Component
  val applicationComponent: ApplicationComponent =
    ApplicationComponent.create(activity.application),
) : NavigatorComponent, TraktAuthAndroidComponent {
  abstract val traktAuthManager: TraktAuthManager
  abstract val rootPresenter: RootPresenter

  companion object
}
```


##### After

You should note that we converted our abstract class to an interface as only interfaces can be annotated with contributed `@ContributesSubcomponent`. For more details on usage of the annotation and behavior [see the documentation.](https://github.com/amzn/kotlin-inject-anvil/blob/main/runtime/src/commonMain/kotlin/software/amazon/lastmile/kotlin/inject/anvil/ContributesSubcomponent.kt)

``` kotlin
@ContributesSubcomponent(ActivityScope::class)
@SingleIn(ActivityScope::class)
interface ActivityComponent {
  @Provides
  fun provideComponentContext(
    activity: ComponentActivity
  ): ComponentContext = activity.defaultComponentContext()

  val traktAuthManager: TraktAuthManager
  val rootPresenter: RootPresenter

  @ContributesSubcomponent.Factory(AppScope::class)
  interface Factory {
    fun createComponent(
      activity: ComponentActivity
    ): ActivityComponent
  }
}
```


### `@MergeComponent` Annotation

In order to create our graph and our compoments to our graph, we need to replace `kotlin-injects` `@Component` with `kotlin-inject-anvil` `@MergeComponent` and get rid of the `SharedComponent`.

##### Before

``` kotlin
@Component
@SingleIn(AppScope::class)
abstract class ApplicationComponent(
  @get:Provides val application: Application,
) : SharedComponent() {
  abstract val initializers: AppInitializers

  companion object
}
```

##### After
Added annotation and removed the supertype form the application component and added `ActivityComponent.Factory`.

```kotlin
@MergeComponent(AppScope::class)
@SingleIn(AppScope::class)
abstract class ApplicationComponent(
  @get:Provides val application: Application,
) : ActivityComponent.Factory {
  abstract val initializers: AppInitializers
  abstract val activityComponentFactory: ActivityComponent.Factory

}
```

And now, if we look at the generated code, we can see Anvil adds all the generated components to our graph when we compile the app.

![Merged Component](https://github.com/user-attachments/assets/79136c55-9bd8-4e7e-aa8e-d6534a3db4c0)


If you forget to delete any provide functions, you will get the following error at compile time.

``` gradle
e: [ksp] Cannot provide: com.thomaskioko.tvmaniac.data.cast.api.CastDao
e: [ksp] as it is already provided
```

 This is expected and you can track down the duplicate provide method and delete it.


## Conclusion
With this in place we have now gotten rid of manual bindings, replacing that with `@ContributesTo` and `@ContributesBinding`. We also deleted our god component class and in turn getting rid a lot of boilerplate thanks to anvil.

[@Ralf](https://x.com/vRallev) and all the contributors have done an amazing job with this. The integration was really smooth. I'm looking forward to how these libraries evolve.

Until we meet again, folks. Happy coding! ✌️


### References
- [Dagger + Anvil: Learning to Love Dependency Injection.](https://www.droidcon.com/2022/06/28/dagger-anvil-learning-to-love-dependency-injection/).
- [KSP with Kotlin Multiplatform](https://kotlinlang.org/docs/ksp-multiplatform.html)
- [Kotlin Inject Anvil README](https://github.com/amzn/kotlin-inject-anvil)