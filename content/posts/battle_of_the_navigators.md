---
title: "Navigating the Waters of Kotlin Multiplatform: Exploring Navigation Solutions"
date: 2024-01-11
draft: false
hideToc: false
tags: ["KMP", "decompose", "navigation"]
series: ["Tv Maniac Journey"]
---


**Disclaimer**: This blog post does not aim to compare various navigation libraries; instead, it offers insights into my personal journey of arriving at a navigation solution. It's important to clarify that the intention is not to provide a comparative analysis of different libraries but rather to share my decision-making process.

### Exploring Navigation Solutions:

Now, let's delve into the journey. My initial focus was on streamlining Compose Screens and refining the presentation layer. I wanted to relocate the navigation logic from the screens to the presentation layer. This decision stemmed from my perspective that navigation is inherently more of a state management concern than a UI element. After conducting some brief research, I decided to centralize and share the navigation logic. Have the logic in a separate navigation module and have it handle routing to different screens. After some experimentation, I ended up sharing the navigation logic between iOS and Android. So, let's take a look at how we ended up here.


### Navigating the Sea Of Libraries

There are a couple of things to consider before deciding what library to use, as there's no "best" library. This could be something like how new the library is. What does the support look like? Is it actively maintained? Might it add complexity to your app? If you are in a team, how many people are familiar with it? How does it impact your architecture? I could go on, but it depends on what you are looking for. Since this is a pet project, I can comfortably try out different 3rd party solutions. 

There are a couple of third-party alternatives that you can choose from:

- [Voyager:](https://voyager.adriel.cafe/) A pragmatic approach to navigation 
- [Decompose:](https://arkivanov.github.io/Decompose/) An advanced approach to navigation that covers the full lifecycle and any potential dependency injection.
- [Appyx:](https://bumble-tech.github.io/appyx/) Model-driven navigation with gesture control.
- [PreCompose:](https://tlaster.github.io/PreCompose/) A navigation and view model inspired by Jetpack Lifecycle, ViewModel, LiveData, and Navigation


#### Voyager
[Voyager](https://voyager.adriel.cafe/) is a new navigation library built specifically for Compose multiplatform, which is designed to make navigation in Compose-based apps simpler and more intuitive.

##### Setup & General Use
My experience with [Voyager](https://voyager.adriel.cafe/) was seamless. It's easy to set up and easy to get things going. There's also `ScreenModel` class for multiplatform state management.

##### Dependency injection
Voyager integrates well with the different dependency injection frameworks. It currently supports [koin](https://insert-koin.io/), [kodein](https://github.com/kosi-libs/Kodein), hilt, and [Kotlin-inject](https://github.com/evant/kotlin-inject).

##### Summary
I did not go with Voyager because I wanted to keep the UI side native per platform (Android: Compose & iOS SwiftUI) and only share the ScreenModel. It's technically possible to create a custom ScreenModel/ViewModel where the Lifecycle is bound on the Voyager lifecycle and, on iOS, the UiViewController lifecycle. However, I was looking for something that works out of the box. This was an oversight on my end but it was a good experiment. If/when I create a Compose Multiplatform App, I will go with Voyager. 


#### Decompose

[Decompose](https://github.com/arkivanov/Decompose) is a Kotlin Multiplatform library for breaking down your code into tree-structured lifecycle-aware business logic components (aka BLoC), with routing functionality and pluggable UI

##### Setup & General Use
Decompose does a good job of decoupling from the OS libraries. I had a hard time getting things up and running compared to my experience with Voyager. Once you understand the core concepts, it's not that bad. The good thing is the project is well documented, and there are multiple project samples. [John O'Reilly's](https://johnoreilly.dev/) project, Confetti, also helped greatly with my migration.

##### Dependency injection
Decompose is flexible and should work with any dependency injection framework. It currently supports [koin](https://insert-koin.io/), [kodein](https://github.com/kosi-libs/Kodein), and [Kotlin-inject](https://github.com/evant/kotlin-inject), which I use in [my project](https://github.com/thomaskioko/tv-maniac).

##### Summary
[Decompose](https://github.com/arkivanov/Decompose) works for me because it allows me to keep the UI native UI so and share the Navigation and Presentation logic between platforms.

### Pre-Decompose

Let's take a trip down memory lane and see what things looked like before moving to decompose. Each platform was responsible for handling its navigation—i.e Jetpack Navigation on Android and Navigation API for iOS. The image below shows how the Android architecture looked like. This was highly inspired by [Lachlan McKee's](https://medium.com/@LachlanMcKee) article [Scalable Jetpack Compose Navigation](https://medium.com/bumble-tech/scalable-jetpack-compose-navigation-9c0659f7c912).

![Android Architecture](https://github.com/thomaskioko/tv-maniac/assets/841885/dcc2e36a-3802-41bb-819e-57b261dd8d27)

Each feature module is responsible for creating it's own navigation factory then the App module adds the set of factories to the NavGraph.

### Hello Decompose

Let's quickly look at how things look with decompose in place. I will not go through the setup process. There are multiple articles out there on how to do this but here's an overview of how the architecture looks like with Decompose. 

![TvManiac Architecture - Decompose](https://github.com/thomaskioko/tv-maniac/assets/841885/1c1e25bb-8ee0-42e5-adc4-b27d4a018dd5)

Thanks to the implementation of decompose, navigation logic joins the shared codebase (KMM). This means that each platform can have its own tailored UI implementation, such as Android with Jetpack Compose and iOS with Swift. By isolating the navigation logic from platform-specific UI concerns, developers can efficiently manage and maintain codebases across different platforms while ensuring optimal user experiences

Below is an example of how I share some logic to hide or show the bottomBar on each platform.

| Android Hide TabBar | iOS Hide TabBar |
|--------------------|--------------------|
| ![AndroidTabBar](https://github.com/thomaskioko/tv-maniac/assets/841885/dca5fa6d-2910-4806-b757-0225bc5271cf) | ![iOSTabBar](https://github.com/thomaskioko/tv-maniac/assets/841885/0e1a4768-657c-4a13-bc59-871217ccb400) |


### Until next time

There's no silver bullet for navigation yet. So use what works for you or your team. One significant takeaway from this experiment is the importance of considering an iOS perspective first. The adage 'If it works on iOS, it should translate smoothly to Android' underscores the value of prioritizing platform considerations in development decisions."

Thank you for taking time to join me along this journey. 

With that, my app has been Decomposed. ✌️


### Resources

- [Decompose Documentation.](https://arkivanov.github.io/Decompose/)
- [Shared Navigation on Kotlin Multiplatform with Decompose (KMP)](https://www.youtube.com/watch?v=g4XSWQ7QT8g)