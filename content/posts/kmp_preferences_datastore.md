---
title: "Setting up DataStore on KMP"
date: "2022-12-15"
draft: false
hideToc: false
tags: ["KMP", "datastore"]
series: ["Tv Maniac Journey"]
---

In this article, well, take a look at how to DataStore and how to use it in a Kotlin Multiplatform project.

Recently Google [announced](https://android-developers.googleblog.com/2022/10/announcing-experimental-preview-of-jetpack-multiplatform-libraries.html) a couple of Jetpack Multiplatform libraries. One is DataStore, a storage solution that allows you to store key-value pairs. This has been rewritten & is now available on Kotlin Multiplatform, targeting Android & iOS. 

> Something to take note of is that these libraries are experimental and should not be used on production apps

### What you will learn:

1.  Setting up Datastore.
2.  Dependency Injection.
3.  Creating a repository
4.  Writing test.

### TvManiac Project

The changes to this are included in a PR I’m working on. If you are curious, you can check it out on [GitHub](https://github.com/c0de-wizard/tv-maniac/pull/48/commits/5e0dec9ed729bc1c6da23196c92475b8984edf76).


### Let’s get started

For our project, we need to add the preference core dependency.

```
dependencies {
    implementation("androidx.datastore:datastore-preferences-core:#dataStoreVersion")
}
```


Now we need to create a function in `commonMain`to help us create an instance of our DataStore object. We can use `expect/actual` pattern and have each platform have its own implementation, but I will create a function and set it up when setting up injection. It's a very simple function that takes in two parameters: Coroutine scope and the filePath.

```
fun createDataStore(
    coroutineScope: CoroutineScope,
    producePath: () -> String
): DataStore<Preferences> = PreferenceDataStoreFactory.createWithPath(
    corruptionHandler = null,
    migrations = emptyList(),
    scope = coroutineScope,
    produceFile = { producePath().toPath() },
)


internal const val dataStoreFileName = "tvmainac.preferences_pb"
```

And that’s it. 🙂 We can now move over to the injection of the DataStore. As I mentioned in my previous article, we have a “hybrid injection setup” where we use Hilt on the Android side and Koin on iOS; however, I plan on migrating to Koin in the future. 😅

### Dependency Injection:

#### Android:

This should be self-explanatory:

```
@Provides
@Singleton
fun provideDataStore(
    @ApplicationContext context: Context,
    @DefaultCoroutineScope defaultScope: CoroutineScope
): DataStore<Preferences> = createDataStore(
    coroutineScope = defaultScope,
    producePath = { context.filesDir.resolve(dataStoreFileName).absolutePath }
)
```

#### IOS:

For iOS, it’s almost similar. We need to create a function that creates the dataStoretore object and then add that to koin module

```
fun dataStore(scope: CoroutineScope): DataStore<Preferences> = createDataStore(
    coroutineScope = scope,
producePath = {
    val documentDirectory: NSURL? = NSFileManager.defaultManager.URLForDirectory(
        directory = NSDocumentDirectory,
        inDomain = NSUserDomainMask,
        appropriateForURL = null,
        create = false,
        error = null,
    )
    requireNotNull(documentDirectory).path + "/$dataStoreFileName"
  }
)
actual fun settingsModule(): Module = module {
    single { dataStore(get()) }
}
```

Boom. We can now create a repository and use it to get the theme and set the theme. The implementation looks like so.

```
class SettingsRepositoryImpl(
    private val dataStore: DataStore<Preferences>,
    private val coroutineScope: CoroutineScope
) : SettingsRepository {


override fun saveTheme(theme: String) {
        coroutineScope.launch {
            dataStore.edit { settings ->
                settings[KEY_THEME] = theme
            }
        }
    }


override fun observeTheme(): Flow<Theme> = dataStore.data.map { theme ->
        when (theme[KEY_THEME]) {
            "light" -> Theme.LIGHT
            "dark" -> Theme.DARK
            else -> Theme.SYSTEM
        }
    }


companion object {
        val KEY_THEME = stringPreferencesKey("app_theme")
   
```

### Testing

The test is fairly simple. but I had a few gotcha moments.

#### Creating the DataStore & repository objects.

```
private var preferencesScope: CoroutineScope = CoroutineScope(testCoroutineDispatcher + Job())   
private val dataStore: DataStore<Preferences> = PreferenceDataStoreFactory.createWithPath(
        corruptionHandler = null,
        migrations = emptyList(),
        scope = preferencesScope,
        produceFile = { "test.preferences_pb".toPath() },
    )
private val repository = SettingsRepositoryImpl(dataStore, testCoroutineScope)
```

Something to note: We are using different scopes for the DataStore and the repository. If we don’t, the tests will fail with the following:

`You should either maintain your DataStore as a singleton or confirm that there is no two DataStore's active on the same file (by confirming that the scope is cancelled).`

#### Clear/remove the key after running a test.

The other thing we need to do is clear the saved item after running a test, or it will read the saved value causing the test to fail. We do that by using `@AfterTest` annotation. We also need to cancel the context associated with the data store.

```
@AfterTest
fun clearDataStore() = runBlockingTest {
        dataStore.edit {
            it.remove(KEY_THEME)
        }
        preferencesScope.cancel()
    }

Since DataStore uses coroutines, we will use [Turbine](https://github.com/cashapp/turbine) for tests. This isn’t complicated. Here, we test that the theme gets updated.

@Test
fun when_theme_is_changed_correct_value_is_set() = runBlockingTest {
repository.observeTheme().test {
        repository.saveTheme("dark")
        awaitItem() shouldBe Theme.SYSTEM //Default theme
        awaitItem() shouldBe Theme.DARK
    }
}
```

One thing that I will improve in the future is deleting the generated test file. Right now, we ignore all `preferences_pb` files created during testing.

### Final Thoughts

We’ve covered steps for using DataStore in a Kotlin Multiplatform project. I must say, it was super simple to get this up and running. You can use this in your ViewModel. For my project, I’m using a [FlowRedux StateMachine](https://github.com/freeletics/FlowRedux), which I will discuss in the next post.

### Resources

-   Check out the [DiceRoller](https://github.com/android/kotlin-multiplatform-samples/tree/main/DiceRoller) sample app.