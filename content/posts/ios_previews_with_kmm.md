---
title: "Enhancing iOS UI Development With Swift UI Packages"
date: "2024-09-07"
draft: false
hideToc: true
tags: ["iOS", "SwiftUI", "SwifUI Previews", "KMM"]
series: "Tv Maniac Journey"
---

# Enhancing iOS UI Development: Hello Swift UI Packages

Remember our adventure in [Going Modular — The Kotlin Multiplatform Way?](https://thomaskioko.me/posts/going_modular_the_kotlin_multiplatform_way/) Well, this is a continuation of that. (Sort of 😁). I say sort of because this article focuses on the Swift side of things. We will explores how creating UI components in a separate Swift package can significantly improve the development experience when working on the iOS App.

## The KMM Preview Headache 🤕

One of the biggest headaches while working with KMM on iOS is that Previews don't load in XCode. I don't know why XCode does not play nice with KMM. It either takes too long to load or fails. The only way to see what I am creating is by running the application. This is frustrating as it slows development time.

### Code

[Here is the PullRequest](https://github.com/thomaskioko/tv-maniac/pull/286) with the changes if you are interested.


## Enter Separate Swift Packages 📦
By relocating our UI components into a separate Swift package, we're not just organizing our code, but we're also unlocking a host of benefits that can significantly enhance our development experience:

1. **Functional SwiftUI Previews**
    - By isolating views from the KMM framework, we can load views faster, providing real-time UI development feedback and improving the developer experience.

2. **Improved Reusability**

    - Components can be easily reused across the application, leading to a more modular architecture and reducing code duplication.


3. **Streamlined Testing**

    - With this in place, Snapshot testing becomes a viable option for ensuring UI consistency. Coming soon. 🤓


4. **Optimized Project Structure**

    - A clear structure makes it easy to maintain and update our components.


### App structure:

I created two packages:

- **SwiftUIComponents:** This package contains common views used in various screens. Colors, Buttons, CardView, PosterImage etc
- **TVManiacUI:** This Package contains Screen UI Views. eg. CastList, TrailerList, ShowInfoView. etc

Below is how the structure of the App looks like.

```
TvManiac/
├── shared/
│   └── ... (Kotlin shared code)
├── androidApp/
│   └── ... (Android-specific code)
├── iosApp/
│   ├── iosApp.xcodeproj
│   ├── iosApp/
│   │   └── ... (iOS app-specific code)
│   └── Modules/
│       └── SwiftUIComponents/
│       │   ├── Package.swift
│       │   └── Sources/
│       │       └── SwiftUIComponents/
│       │           └──Components/
│       │                 ├── BottomNavigation.swift
│       │                 ├── HeaderContentView.swift
│       │                 └── ...
│       │           ├── CollapsibleView.swift
│       │           ├── TrailerItemView.swift
│       │           └── ...
│       └── TVManiacUI/
│           ├── Package.swift
│           └── Sources/
│               └── TVManiacUI/
│                   └──Cast/
│                       └── CastListView.swift
│                   └──Trailers/
│                       └── TrailersList.swift
│                   └── ...
```


## Implementing the Approach 🚧
Integrating a package in XCode is a straightforward process. These are the steps I followed: 

1. **Package Creation and Configuration**

- Create a new Swift package (`SwiftUIComponents`) using your preferred method (Xcode UI or Swift Package Manager CLI).
- Configure the `Package.swift` file to specify iOS deployment target and any necessary dependencies.


2. **Component Migration**

- Migrate SwiftUI views and components from your KMP iOS project to the new package.
- Refactor as needed to ensure components are self-contained and don't rely on KMP-specific code.


3. **Dependency Management**

- If your UI components require a specific package, use Swift Package Manager's dependency declaration to manage package relationships. In my case, I added `SDWebImageSwiftUI` and `YouTubePlayerKit` and removed these from the main app.


4. **Final Step**

- In your main iOS project, go to File > Add Packages.
- Select `SwiftUIComponents` package.
- (Optional)** Create a new scheme and select the framework. In this case, `SwiftUIComponents`. This allows us to switch between the Main app and the UI package.

With that in place, here's an example of how the project structure looks after implementing this approach. 🥳

##### UI Components Preview
![XcodePreview](https://github.com/user-attachments/assets/ddd1e486-40da-4bb0-b5e3-14cc7061b916)

##### Screen Components Preview
![ScreenComponentsPreview](https://github.com/user-attachments/assets/fa546cc4-e563-4a7b-b734-af1a39d6cc1d)

# Conclusion
I highly recommend trying this approach. It's made my life easier, my code cleaner, and the general structure better. Next, we might look at adding snapshot tests, adding a CI/CD job and some more cleanup. 

Big shoutout to my colleague Daniel. If you ever read this, thanks for being the best Second class mobile citizen. Until we meet again folks.

Happy coding! ✌️