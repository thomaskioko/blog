---
title: "Internationalization (I18n) in Kotlin Multiplatform: Part 2"
date: "2025-06-09"
draft: false
hideToc: true
tags: ["KMM", "i18n", "Internationalization"]
series: "Tv Maniac Journey"
---

# Intro

In the previous article [Internationalization (I18n) in Kotlin Multiplatform](https://thomaskioko.me/posts/localization/), we explored how to modularize the `:i18n` module using [Moko Resources](https://github.com/icerockdev/moko-resources) for handling string resources across platforms.

In this follow-up article, we'll dive deeper into:
- Implement dynamic language switching without app restarts
- Create a testable localization architecture
- Handle platform-specific locale implementations

The Settings Screen demonstrates our new approach to localization, enabling dynamic language switching without app restarts on both Android and iOS. For this demo, we support English, French, and German languages. The language change is persisted even after changing screens or killing the app.

> **Note**: Migrating localization from UI to the presentation layer will be done in a follow-up task.

| Android | iOS |
|---------|-----|
| ![Android Localization Demo](https://github.com/user-attachments/assets/14402695-6f92-427c-8e07-36eb6dea84b4) | ![iOS Localization Demo](https://github.com/user-attachments/assets/5467a160-39c1-4454-b24e-f4ec75f80346) |


Let's explore how we can create a more maintainable and robust localization solution.



## The Great Migration: From UI to Presenter 🚀

**Wait a minute sir!** Localization is part of the UI layer and is not business logic. I hear you, and you are not wrong (maybe 😏). If you ask me, "**It depends.**" Our approach of managing it in the presentation layer is deliberate and addresses specific requirements:

##### Why Not in the UI Layer?

1. **Dynamic Language Switching**
   - Immediate language changes without app restarts
   - Centralized language management across the app

2. **Cross-Platform Consistency**
   - Single source of truth for language selection
   - Platform-specific implementations are abstracted


So, should I handle localization in the UI or Presentation layer, it really depends on your needs. With that said, let's take a look at how everything looks.

## Architecture Overview

Here's how our updated localization architecture looks:

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│                 │     │                 │     │                 │
│    UI Layer     │────►│    Presenter    │────►│      i18n       │
│    (Platform)   │     │    (Common)     │     │      (API)      │
│                 │     │                 │     │                 │
└─────────────────┘     └─────────────────┘     └─────────────────┘
                                │
                                │
                                ▼
                          ┌─────────────┐
                          │             │
                          │  DataStore  │
                          │    (APi)    │
                          │             │
                          └─────────────┘
```

The architecture follows a clean separation of concerns:
- **UI Layer**: Displays content from the Presenter
- **Presenter**: Handles localization logic and depends on both Locale API and DataStore
- **i18n**: Provides string resources
- **DataStore**: Persists user language preferences


## Implementation Details

### 1. The Locale API

We created a `LocaleProvider` interface to abstract locale-related operations:

```kotlin
interface LocaleProvider {
    val currentLocale: Flow<String>
    suspend fun setLocale(languageCode: String)
    fun getSupportedLocales(): Flow<List<String>>
}
```

This interface is implemented for each platform (Android, iOS) with platform-specific logic.

### 2. Persisting Language Preferences

We use DataStore to save and retrieve the user's language preference:

```kotlin
interface DatastoreRepository {
    suspend fun saveLanguage(languageCode: String)
    fun observeLanguage(): Flow<String>
}
```

### 3. Device Locale vs. App-Preferred Locale

The `DefaultLocaleProvider` does a couple of things

- Get the local from dataStore.
- Get the user's prefered device langauages.

```kotlin
class DefaultLocaleProvider(
    private val platformLocaleProvider: PlatformLocaleProvider,
    private val datastoreRepository: DatastoreRepository,
) : LocaleProvider {
    override val currentLocale: Flow<String> {
        return datastoreRepository.observeLanguage()
    }

    override fun getPreferredLocales(): Flow<List<String>> {
        return platformLocaleProvider.getPreferredLocales()
    }    
}
```

### 4. Platform-Specific Implementations

Each platform has its own implementation of the `PlatformLocaleProvider`. The Android implementation handles locale changes through the `Context`, while the iOS implementation uses `NSUserDefaults.` Instead of getting all the locales, we get the list of prefered languages the user has added on their devices.  I have set it up like this because I need to add proper translations for multiple languges.

##### iOS Platform Implementation
``` kotlin
public actual class PlatformLocaleProvider {
    public actual fun getPreferredLocales(): Flow<List<String>> {
        val availableIdentifiers = NSLocale.preferredLanguages
            .filterIsInstance<String>()
            .mapNotNull { identifier ->
                identifier.split('-', '_').firstOrNull()?.lowercase()
            }
            .distinct()

        return flowOf(availableIdentifiers)
    }
}
```

##### Android Platform Implementation
``` kotlin
public actual class PlatformLocaleProvider(
    private val context: Context,
) {
    public actual fun getPreferredLocales(): Flow<List<String>> {
        val userLocales = userLocales()
        val defaultLocale = listOf(Locale.getDefault().language)

        return flowOf(if (userLocales.isNotEmpty()) userLocales.map { it.language }.sorted() else defaultLocale)
    }

    private fun userLocales(): List<Locale> {
        val locales = context.resources.configuration.locales
        return (0 until locales.size()).mapNotNull { index ->
            val javaLocale = locales.get(index)
            val language = javaLocale.language
            val country = javaLocale.country.toCountryOrNull()
            if (country != null) {
                Locale(language, country)
            } else {
                Locale(language)
            }
        }
    }
}
```

### 5. Connecting to Moko Resources

We created a `MokoLocaleInitializer` to make Moko Resources aware of locale changes:

```kotlin
class MokoLocaleInitializer(
    private val localeProvider: LocaleProvider,
    private val dispatchers: AppCoroutineDispatchers,
) : AppInitializer {
    override fun init() {
        GlobalScope.launch(dispatchers.main) {
            localeProvider.currentLocale.collect { locale ->
                StringDesc.localeType = StringDesc.LocaleType.Custom(locale)
            }
        }
    }
}
```

### 6. The Presenter Layer

The presenter is responsible for:
- Loading users' supported languages.
- Providing localized strings to the UI.
- Handling language change requests and update datastore with the new language.

Here's a simplified version of our `SettingsPresenter`:

```kotlin
class SettingsPresenter(
    private val datastoreRepository: DatastoreRepository,
    private val localeProvider: LocaleProvider,
    private val localizer: Localizer,
) {
    val state: StateFlow<SettingsState> = combine(
        _state,
        datastoreRepository.observeLanguage(),
        localeProvider.getSupportedLocales(),
    ) { currentState, selectedLanguage, supportedLocales ->
        currentState.copy(
            supportedLanguages = supportedLocales,
            selectedLanguage = selectedLanguage,
            languageLabel = localizer.getString(StringResourceKey.LabelSettingsLanguage),
            languageMessageLabel = localizer.getString(StringResourceKey.LabelSettingsLanguageMessage),
            ...
        )
    }.stateIn(
        scope = coroutineScope,
        started = SharingStarted.WhileSubscribed(),
        initialValue = _state.value,
    )

    fun dispatch(action: SettingsActions) {
        when (action) {
            is LanguageSelected -> {
                coroutineScope.launch {
                    localeProvider.setLocale(action.languageCode)
                }
            }
        }
    }    
}
```

## Testing Localization

We can then tests that changing the locale returns the correct string.:

```kotlin
class LocalizedStringTest {

    @Test
    fun should_return_english_string_for_default_locale() = runTest {
        localeProvider.setLocale("en")
        StringDesc.localeType = StringDesc.LocaleType.Custom("en")
        val result = localizer.getString(ButtonErrorRetry)
        result shouldBe "Retry"
    }

    @Test
    fun should_return_french_string_for_fr_locale() = runTest {
        localeProvider.setLocale("fr-FR")
        StringDesc.localeType = StringDesc.LocaleType.Custom("fr-FR")

        val result = localizer.getString(ButtonErrorRetry)
        result shouldBe "Réessayer"
    }

    @Test
    fun should_return_german_string_for_de_locale() = runTest {
        StringDesc.localeType = StringDesc.LocaleType.Custom("de-DE")

        val result = localizer.getString(ButtonErrorRetry)
        result shouldBe "Wiederholen"
    }
    // More tests...
}
```


## Conclusion

By moving localization to the presenter layer, we address several key concerns:

1. **Dynamic Language Switching**: Users can change languages without restarting the app.
2. **Consistent Experience**: Language changes are applied consistently across the app
3. **Improved Testability**: We can easily test localization logic in isolation

This approach leverages the strengths of KMP by sharing localization logic across platforms while allowing for platform-specific implementations where needed.

In the next article, we'll focus on:
- Migrating localization from UI to presentation layer
- Adding multiple language translations
- Implementing RTL layout support
- ...

Until then, happy coding! ✌️

### Resources
- [Per-app language preferences](https://developer.android.com/guide/topics/resources/app-languages)
- [Structure your app for localization](https://developer.apple.com/localization/)
