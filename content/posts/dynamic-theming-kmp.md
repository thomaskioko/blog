---
title: "Dynamic Theming: Building on a Design System Foundation"
date: "2025-12-10"
draft: false
hideToc: true
tags: ["kotlin", "kmp", "swiftui", "compose", "design-system", "theming"]
series: "Tv Maniac Journey"
---

### Intro

In my [previous article](/posts/swiftui-design-system-from-android-perspective/), I covered building a design system in SwiftUI that mirrors Jetpack Compose's `MaterialTheme`. I mentioned that with this foundation in place, adding dynamic themes would be straightforward. Well, I recently implemented exactly that, and I wanted to share how the design system made this possible with minimal platform-specific code.

| Android | iOS |
| -- | -- |
| ![DynamicThemingAndroid](https://github.com/user-attachments/assets/fc8b1973-663e-4880-9c53-db08e7699e59) |![DynamicThemingiOS](https://github.com/user-attachments/assets/e562b46e-01a9-4b45-a7f0-d70d76d744db) |


### Why Dynamic Theming?

Beyond the standard light and dark modes, I wanted to give users more personality options. For an entertainment app like Tv Maniac, themes can set the mood. Some users prefer something subtle. Others want a terminal aesthetic with green text on black. Having options makes the app feel more personal. The theming style is inspired by [Pocket Casts Android](https://github.com/Automattic/pocket-casts-android) and [Pocket Casts iOS](https://github.com/Automattic/pocket-casts-ios) apps.

The challenge was implementing this without duplicating logic on each platform. I didn't want to manage theme state separately on Android and iOS.

### The Approach

The key realization was that theme selection is business logic, not UI logic. The user picks a theme, we persist that choice, and both platforms react to changes. This fits naturally into the shared KMP layer.

The shared code owns:

- The list of available themes
- Current theme state
- Theme persistence (via DataStore)
- Emitting theme changes to observers

Each platform's UI layer has one responsibility: take the selected theme and apply the corresponding colors. That's it.

### Shared Theme Definition

I created an `AppTheme` enum in shared Kotlin code. This enum is the single source of truth for both platforms. When I want to add a new theme, I add it here once. No need to update platform-specific enums that might drift out of sync.

The enum includes metadata like `isDark` which tells platforms whether to use light or dark system chrome (status bar, navigation bar). It also has display order for consistent UI ordering across platforms.

### State Flow

Theme state flows from the root presenter as a `StateFlow`. Android collects it in the Activity. iOS observes it through a Swift-friendly wrapper that bridges Kotlin flows to Swift. Neither platform decides what themes exist or manages persistence. They just react to changes from shared code.

This follows the same pattern I use for other shared state in the app. The presenter layer handles business logic. Platform UI layers consume state and render accordingly.

### Platform Integration

On Android, I map `AppTheme` to Compose's `ColorScheme`. The Activity observes theme state and passes it to `TvManiacTheme`. That composable applies the correct color scheme based on the selected theme.

On iOS, I bridge `AppTheme` to `DeviceAppTheme` (the Swift enum from the design system). When theme state changes, the root view updates the SwiftUI environment. Every child view using `@Theme` automatically picks up the new colors.

The design system I built previously made this trivial. Without it, I'd be updating every view that uses colors or fonts. With semantic tokens like `theme.colors.primary` and `theme.colors.surface`, the views don't need to change. They reference tokens. The theme decides what those tokens mean.

> **Note:** The token naming is up to you and you can/should name them to what works for your team. One thing to factor in when coming up with names is making sure there is minimal confusion on both platforms. Teams should be confused by the use of different naming styles.

### What Made This Easy

The payoff from the design system became clear during this implementation. When the theme changes from Light to Terminal:

1. Shared code emits the new theme
2. Platform observes the change and updates its theme environment
3. Every `theme.colors.primary` reference automatically resolves to Terminal's green
4. Views re-render.

This is why semantic naming matters. Views don't know what "primary" looks like. They just know they need "primary". The theme provides the actual color value. Swapping entire color schemes requires zero view modifications.

### The Theme Selector UI

I built a generic `ThemeSelectorView` component that renders theme swatches in a grid. It's generic over any type conforming to `ThemeItem`, so it doesn't know about specific themes. It just renders whatever you pass it.

Adding a new theme to the selector means adding the enum case. The UI adapts automatically.

### Lessons Learned

- **Shared state simplifies everything.** Having one source of truth for theme selection eliminated the potential for Android and iOS showing different themes or persisting them differently.

- **The design system is prerequisite infrastructure.** Attempting dynamic theming before centralizing tokens would have been painful. The design system investment paid off immediately.


### What I'd Do Differently

If starting fresh, I'd consider defining color values in shared code too. Right now, each platform defines its own hex values for Terminal green or Autumn orange. These could live in a shared module.

The tradeoff is complexity. Shared color definitions work for simple cases but get awkward when platforms need native color system integration. For Tv Maniac's scope, platform-defined colors are fine. Maybe a future experiment.

### Life Without a Design System

If your codebase doesn't have a design system, I won't sugarcoat it: building one requires significant upfront investment. You'll audit existing colors and fonts scattered across files. You'll have debates about token naming. You'll touch dozens of views during migration. It's not fun work.

But consider the alternative. Without centralized tokens, dynamic theming means updating every hardcoded color reference. Miss one and you get a white button on a white background in dark mode. Good luck finding it.

Here's a practical migration path:

**Start with an audit.** Create custom lint rules to flag hardcoded colors. You'll likely find duplicates, slight variations, and colors that serve no semantic purpose. This is your baseline—and the lint rules become your enforcement mechanism later.

**Build the complete design system first.** Define your full token structure upfront: primary, secondary, surface, background, error, text colors, and their variants. Get the naming conventions right. Have the debates about semantic meaning now, not later. A half-built system creates confusion about what exists and what doesn't. You are just creating the design system in this phase, no integration yet.

**Migrate incrementally.** With the full system in place, migrate screen by screen. Start with high traffic screens. Touch others opportunistically when you're already making changes. The system is complete; only the adoption is gradual. You can even do this behind a feature flag if you have a crazy codebase.

**Enforce gradually.** Once the system exists, mark hardcoded color utilities as deprecated. Developers see warnings, not build failures. Business doesn't stop. The deprecation warnings create pressure to migrate without blocking releases. For a while, your codebase will have both hardcoded values and tokens. That's fine. this is safer than doing a whole big bang re-write.


### Final Thoughts

The steep part is real. So is the payoff. With Tv Maniac, adding a new theme now takes minutes. I define the color scheme once per platform, and every screen automatically picks up the new values. No hunting through files. No missed spots. Change the token definition, and the entire app updates. That's only possible because both platforms use tokens everywhere.

That's the benefit of investing in foundational infrastructure. A bit of friction in the begining, but future you will forever be greatful. You can focus on building features with the confidence of having the same look on both platforms.


If you'd like to see the implementation, [here's the pull request](https://github.com/thomaskioko/tv-maniac/pull/688).


Until we meet again, folks. Happy coding! ✌️

### Resources

- [Design System Article](/posts/swiftui-design-system-from-android-perspective/)
