---
title: "Going Modular — The Kotlin Multiplatform Way"
date: "2022-01-19"
draft: false
hideToc: false
tags: ["KMP", "modularization", "architecture"]
series: ["Tv Maniac Journey"]
---


![](https://cdn-images-1.medium.com/max/800/0*NExDJVvg0K4-Pifg.jpeg)

I recently decided to dive into the Kotlin Multiplatform world. So far, so good. This article talks about my journey modularising a project I've been working on.

There are other ways of going about this, but this worked for me.

### TvManiac

I'm using [TMDB](https://developers.themoviedb.org/3) to fetch Tv Show information. Here's how it looks on both platforms. I have much work done on Android since that's my territory. I'm updating the iOS side of things as I learn. You can find the source code for the project on [Github](https://github.com/c0de-wizard/tv-maniac).

[**GitHub - c0de-wizard/tv-maniac: Tv-Maniac is a Multiplatform app (Android & iOS) for viewing TV…**  
_TvManiac is a Multiplatform app (Android & iOS) for viewing TV Shows information from TMDB. The aim of this project is…_github.com](https://github.com/c0de-wizard/tv-maniac "https://github.com/c0de-wizard/tv-maniac")[](https://github.com/c0de-wizard/tv-maniac)

![](https://cdn-images-1.medium.com/max/800/1*oP1Yfe_hxENFrbdC5Zxtvw.png)

### The Shared Module

Before discussing modularisation, let me share why I decided to do this. The shared module contains common logic for both platforms. (iOS & Android).

To be precise, we are sharing the Network and Cache code. A couple of great introduction articles are at the end of the post. Below is the shared module initially looked at.

![](https://cdn-images-1.medium.com/max/800/1*nlQDDg9wkQuNb46aC3TMxA.png)

So why modularise the shared module?

Well, I was initially using packages to structure code. It worked, but it was a pain navigating through packages. Since I plan on updating this project, I need to do things correctly. And that, folks, is why we are here.

One other thing, any time I change a class in the shared module, the whole module is built again.

### Going Modular

The image below shows what the shared module looks like after a couple of experiments here and there. But first, let's take a look at the project structure.

![](https://cdn-images-1.medium.com/max/800/1*UPOZpV5MDdU9mVTw2g3-IA.png)

Shared module structure

### Project Structure

We have a couple of modules but can group them into four main directories/modules.

-   `base:` This module contains iOS `ApplicationComponent` classes and configuration to export the ios framework. This is the module where KMMBridge is configured.
-   `core:database`: DB implementation. We are using [SQLDelight](https://cashapp.github.io/sqldelight/) for this project.
-   `core:datastore`: DB implementation. We are using [SQLDelight](https://cashapp.github.io/sqldelight/) for this project.
-   `core:networkutil`: Contains all "common" classes used by all modules. These can be util classes like CoroutineScope/Dispatchers, util classes, e.t.c.
-   `core:api`: Module with [Ktor](https://ktor.io/) implementation for each respective platform. TMBD and Trakt.
-   `data:` It contains a repository, data sources, and model classes
-   `domain:` It contains presentation implementation. In this case, we use [FlowRedux](https://github.com/freeletics/FlowRedux) to create a shared `state-machine` for both clients.

### Kmm precompiled script

Since most of the module's `**_build.gradle_**` files are the same, we can use a precompiled script to save a lot of repetition. We create a file `**_kmm-domain-plugin.gradle.kts_**` in the `**buildSrc**` directory.

![](https://cdn-images-1.medium.com/max/800/1*MPtqJUe0UBtcAoepfFG49A.png)

kmp precompiled-script

We can then add the plugin to the module and get rid of a lot of code. Now, we need to apply the script and add the module dependencies. As you can see, one advantage of this is having lean modules and don't need to have unnecessary dependencies.

![](https://cdn-images-1.medium.com/max/800/1*nCm_h-y99tYEU9pwt-qjMQ.png)

API build.gradle.kts

### Exposing dependencies on iOS

Now that we have our modules, we need to add them to the shared module since this is the entry point for iOS. By adding the modules to commonMain, Kmm will generate an `Obj-C Framework`.

To expose the dependencies, we do two things

1.  Add the modules as API in `commonMain` source set

![](https://cdn-images-1.medium.com/max/800/1*fhvO_IDDDvQJvgQQLTq38A.png)

2. Export the modules in the Framework configuration.

![](https://cdn-images-1.medium.com/max/800/1*CeoaEUtmbSRTYxY9JtfZ_Q.png)

You'll notice we are only adding API modules to the framework configuration. This is because we only need to expose the interfaces.

One last thing, and we can wrap things up. Let's look at Dependency injection.

### Dependency injection

I initially had what I wanted to call a mixed injection where Android uses [Dagger Hilt](https://developer.android.com/training/dependency-injection/hilt-android), and the Shared module uses [Koin](https://insert-koin.io/) to initialize the graph. I migrated the implementation to [kotlin-inject](https://github.com/evant/kotlin-inject/blob/main/README.md) by Eva Tatarka.

[**kotlin-inject/README.md at main · evant/kotlin-inject**  
_A compile-time dependency injection library for kotlin. settings.gradle pluginManagement { repositories {…_github.com](https://github.com/evant/kotlin-inject/blob/main/README.md "https://github.com/evant/kotlin-inject/blob/main/README.md")[](https://github.com/evant/kotlin-inject/blob/main/README.md)

Most modules have a `**di**` the package that contains module dependencies. As an example, this is what genre looks like: I might have yet to mention that I'm creating a Swift package and adding that to the iOS app. [John O'Reilly](https://twitter.com/joreilly) has a fantastic article on how to go about it. Read more [here](https://johnoreilly.dev/posts/kotlinmultiplatform-swift-package/). (Follow him if you want to get lost in the KMM universe. He does cover a lot of topics and shares valuable resources.)

### Summary

I hope this was useful as I walked you through how I approached this challenge. I know I could have gone deeper into things. Mostly because I'm sure, there are other ways of doing this. Better/more straightforward ways. If you have some insights, please feel free to reach out. Learning is an endless loop, and I'm up for it.

What I love about Kotlin Multiplatform is:

-   Most of the core business logic is shared. This allows me to spare some time and dive into the Apple world.
-   UI development is Native. So I can still keep up on what's happening on Android and learn a bit of Swift while at it.

Until we meet again. Adios.

### Resources

-   [Understand the KMM project structure.](https://kotlinlang.org/docs/kmm-understand-project-structure.html)
-   [Kotlin Multiplatform In Production — Kevin Galligan](https://youtu.be/hrRqX7NYg3Q)
-   [Using Swift Packages in a Kotlin Multiplatform project — John O'Reilly](https://johnoreilly.dev/posts/kotlinmultiplatform-swift-package/)
-   [Touchlab — KaMPKit](https://github.com/touchlab/KaMPKit)
-   [Multiple Kotlin Frameworks in an Application — Kevin Schildhorn](https://touchlab.co/multiple-kotlin-frameworks-in-application/)