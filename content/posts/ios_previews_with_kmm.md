------
title: "Enhancing iOS UI Previews with Swift Packages"
date: "2024-09-07"
draft: false
hideToc: true
tags: ["iOS", "SwiftUI", "SwifUI Previews", "KMM"]
series: "Tv Maniac Journey"
---

Remember our adventure in [Going Modular — The Kotlin Multiplatform Way?](https://thomaskioko.me/posts/going_modular_the_kotlin_multiplatform_way/) Well, this is a continuation of that. (Sort of 😁). I say sort of because this article focuses on the Swift side of things. We will explore how creating UI components in a separate Swift package can significantly improve the development experience when working on the iOS App.

I am now focusing on the Swift implementation and how creating UI components in separate Swift packages improves the iOS development experience.

The complete source code is available in the [pull request](https://github.com/thomaskioko/tv-maniac/pull/286).

## iOS preview limitations in KMP

SwiftUI Previews often fail to load or take excessive time when integrated directly with the KMM framework in Xcode. This forces developers to run the full application to verify UI changes, which slows development.

## Utilize separate Swift packages

Relocating UI components to dedicated Swift packages provides several advantages:

- **Functional SwiftUI Previews**: Isolating views from the KMM framework enables faster loading and real time feedback.
- **Improved Reusability**: Components become modular and easily reusable across the application.
- **Simplified Testing**: Modular components facilitate implementation of snapshot testing.
- **Optimized Structure**: A clear hierarchy improves component maintenance and updates.

## Application structure implementation

I created two primary packages to organize the UI:

- **`SwiftUIComponents`**: Contains shared views including buttons, cards, and poster images.
- **`TVManiacUI`**: Contains screen specific views such as cast lists and show information components.

The project structure follows this pattern:

```
TvManiac/
├── shared/
├── iosApp/
│   └── Modules/
│       └── SwiftUIComponents/
│       │   ├── Package.swift
│       │   └── Sources/
│       └── TVManiacUI/
│           ├── Package.swift
│           └── Sources/
```

## Implementation process

Integrating these packages into Xcode involves several steps.

### 1. Configure packages

Create the `SwiftUIComponents` package and configure the `Package.swift` file to specify deployment targets and dependencies.

### 2. Migrate components

Transition SwiftUI views from the main KMP iOS project to the new packages. Ensure components are self contained and do not depend on KMM specific code.

### 3. Manage dependencies

Declare package relationships within `Package.swift`. I moved dependencies like `SDWebImageSwiftUI` and `YouTubePlayerKit` to these UI packages.

### 4. Integrate into the main project

Add the packages to the main iOS project using Xcode's Add Packages feature. Creating a new scheme for the UI framework allows switching between the main application and the UI package for rapid development.

## Resulting previews

The implementation enables reliable previews for both shared components and full screens.

**Shared Components Preview:**
![XcodePreview](https://github.com/user-attachments/assets/ddd1e486-40da-4bb0-b5e3-14cc7061b916)

**Screen Components Preview:**
![ScreenComponentsPreview](https://github.com/user-attachments/assets/fa546cc4-e563-4a7b-b734-af1a39d6cc1d)

## Final considerations

Using separate Swift packages for UI components simplifies the development workflow and improves code quality. This approach addresses preview performance issues while establishing a modular architecture for the iOS application. Future improvements include automated CI/CD jobs and snapshot testing.

Until we meet again, folks. Happy coding! ✌️
