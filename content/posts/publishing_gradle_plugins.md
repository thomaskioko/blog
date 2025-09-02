---
title: "Publishing Gradle Plugins to Maven Central"
date: "2025-08-30"
draft: false
hideToc: true
tags: ["Gradle", "Maven Central", "Publishing", "Build Tools", "Kotlin"]
series: "Tv Maniac Journey"
---

# Intro

If you've been following my TvManiac journey, you know I'm a fan of keeping things modular and reusable. Recently, I hit a point where my Gradle build logic grew into a collection of 10 specialized plugins. They were working great as a local `includeBuild`, but I started thinking — what if I could publish these plugins and use them like any other dependency?

This article walks through my journey of transforming local Gradle plugins into published Maven Central artifacts. We'll cover setting up the publishing infrastructure, testing locally with `mavenLocal()`, and implementing a clean versioning strategy.

## The Old

My TvManiac project had evolved to use custom Gradle plugins for everything — from configuring Kotlin Multiplatform modules to setting up Spotless formatting. Here's what the structure looked like:

```
tv-maniac/
├── tooling/
│   └── tvmaniac-gradle-plugins/
│       ├── build.gradle.kts
│       └── src/main/kotlin/.../
│           ├── AppPlugin.kt
│           ├── AndroidPlugin.kt
│           ├── KotlinMultiplatformPlugin.kt
│           └── ... (7 more plugins)
└── settings.gradle.kts
```

The main project used these plugins via `includeBuild("tooling")` in settings.gradle.kts. It worked, but every project sync meant rebuilding the plugins. Plus, I couldn't share these tools with other projects without copying code around. Not ideal.

## Setting Up the Maven Publish Plugin

The first step was configuring the Gradle Maven Publish Plugin. After some research, I settled on the [Vanniktech plugin](https://vanniktech.github.io/gradle-maven-publish-plugin/) — it handles the heavy lifting of publishing to Maven Central with minimal configuration.

> **Note:** This is a high level walkthrough and I will not cover signing and setting up an account on Maven central. You can check out this guide for more details on this. [Publish JVM Library to Maven Central with Gradle (2025 Guide)](https://www.youtube.com/watch?v=nd2ULXyBaV8&ab_channel=Gradle)

### Plugin Configuration

Here's how I configured the publishing in `plugins/build.gradle.kts`:

```kotlin
plugins {
    id("java-gradle-plugin")
    alias(libs.plugins.kotlin.jvm)
    alias(libs.plugins.publish)
}

// Standard Gradle plugin configuration
gradlePlugin {
    plugins {
        create("appPlugin") {
            id = "com.thomaskioko.tvmaniac.gradle.app"
            implementationClass = "com.thomaskioko.tvmaniac.gradle.plugin.AppPlugin"
        }
        // ... Other plugins
    }
}

// Maven Publishing configuration
mavenPublishing {
    publishToMavenCentral()
    
    pom {
        name.set(providers.gradleProperty("POM_NAME").get())
        description.set(providers.gradleProperty("POM_DESCRIPTION").get())
        inceptionYear.set(providers.gradleProperty("INCEPTION_YEAR").get())
        url.set(providers.gradleProperty("POM_URL").get())
        
        licenses {
            license {
                name.set(providers.gradleProperty("POM_LICENSE_NAME").get())
                url.set(providers.gradleProperty("POM_LICENSE_URL").get())
            }
        }
        
        developers {
            developer {
                id.set(providers.gradleProperty("POM_DEVELOPER_ID").get())
                name.set(providers.gradleProperty("POM_DEVELOPER_NAME").get())
                url.set(providers.gradleProperty("POM_DEVELOPER_URL").get())
            }
        }
        
        scm {
            url.set(providers.gradleProperty("POM_SCM_URL").get())
            connection.set(providers.gradleProperty("POM_SCM_CONNECTION").get())
            developerConnection.set(providers.gradleProperty("POM_SCM_DEV_CONNECTION").get())
        }
    }
}
```

I have all the metadata coming from `gradle.properties`, keeping the build script clean and the configuration centralized. You could also add them here, which is also okay. 

## Migration Strategy

Before trying things out, let's talk strategy. You don't have to migrate everything at once. Here's the incremental approach I used:

### Phase 1: Extract and Stabilize
First, I moved the plugins to a separate project while keeping `includeBuild`. This let me:
- Ensure plugins work in isolation
- Stabilize the API without breaking the main build
- Set up proper tests for the plugins

### Phase 2: Parallel Testing
I ran both configurations in parallel:

```kotlin
// In settings.gradle.kts
pluginManagement {
    repositories {
        mavenLocal() // For testing published version
        // ... other repos
    }
    // Temporarily keep this for quick switching
    // includeBuild("tooling") 
}
```

This allowed quick A/B testing between local and published versions.

## Testing Locally with mavenLocal()

Before going live on Maven Central, I needed to test everything locally. This is where `mavenLocal()` becomes your best friend.

### Publishing to Maven Local

First, I published the plugins to my local Maven repository:

```bash
./gradlew publishToMavenLocal
```

This command builds the plugins and installs them to `~/.m2/repository`. Now they're available locally just like any other Maven dependency.

### Consuming from Maven Local

To test the published plugins, I updated my main project's `settings.gradle.kts`:

```kotlin
pluginManagement {
    repositories {
        mavenLocal() // Check local Maven first
        mavenCentral()
        google()
        gradlePluginPortal()
    }
}
```

And updated the version catalog (`gradle/libs.versions.toml`):

```toml
[versions]
tvmaniac-plugins = "1.0.0"

[plugins]
tvmaniac-application = { id = "io.github.thomaskioko.gradle.plugins.app", version.ref = "app-plugins" }
tvmaniac-android = { id = "io.github.thomaskioko.gradle.plugins.android", version.ref = "app-plugins" }
tvmaniac-kmp = { id = "io.github.thomaskioko.gradle.plugins.multiplatform", version.ref = "app-plugins" }
# ... more plugins
```

The moment of truth — removing `includeBuild("tooling")` and running a build. When it worked, I knew we were onto something good.

Now that everything works, you can publish your plugins to Maven Central. After setting up your credentials, run ./gradlew publishToMavenCentral --no-configuration-cache. Note: this task currently does not support the configuration cache. Once complete, you’ll see your artifact in your Maven account, ready to be published and made publicly available.
![Published Plugin](https://github.com/user-attachments/assets/5323ff48-6e4d-4554-b700-8c50aa273796)

One area for future improvement is to automate the publishing process and integrate it into continuous integration (CI), ensuring releases are consistent and effortless.


## Wrapping Up

Publishing Gradle plugins might seem daunting at first, but with the right tools and approach, it's surprisingly straightforward. The [Vanniktech plugin](https://vanniktech.github.io/gradle-maven-publish-plugin/) handles most of the complexity, leaving you to focus on writing great plugins.

You can find the complete source code for the plugins on [GitHub](https://github.com/thomaskioko/app-gradle-plugins).

Until next time, happy coding! ✌️

## Resources

- [Publish JVM Library to Maven Central with Gradle (2025 Guide)](https://www.youtube.com/watch?v=nd2ULXyBaV8&ab_channel=Gradle)
- [TvManiac Gradle Plugins](https://github.com/thomaskioko/app-gradle-plugins)
- [Gradle Maven Publish Plugin](https://vanniktech.github.io/gradle-maven-publish-plugin/)
- [Maven Central Portal](https://central.sonatype.com)