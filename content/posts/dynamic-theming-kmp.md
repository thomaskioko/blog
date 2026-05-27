------
title: "Dynamic Theming: Building on a Design System Foundation"
date: "2025-12-10"
draft: false
hideToc: true
tags: ["kotlin", "kmp", "swiftui", "compose", "design-system", "theming"]
series: "Tv Maniac Journey"
---

In my [previous article](/posts/swiftui-design-system-from-android-perspective/), I covered building a design system in SwiftUI that mirrors Jetpack Compose's `MaterialTheme`. I mentioned that with this foundation in place, adding dynamic themes would be straightforward. Well, I recently implemented exactly that, and I wanted to share how the design system made this possible with minimal platform-specific code.

This foundation simplifies the implementation of dynamic themes with minimal platform specific code.

| Android | iOS |
| -- | -- |
| ![DynamicThemingAndroid](https://github.com/user-attachments/assets/fc8b1973-663e-4880-9c53-db08e7699e59) |![DynamicThemingiOS](https://github.com/user-attachments/assets/e562b46e-01a9-4b45-a7f0-d70d76d744db) |

## Motivation for dynamic themes

I implemented dynamic themes to provide users with personality options beyond standard light and dark modes. Themes can set the mood for an entertainment application, ranging from subtle aesthetics to terminal styles. I wanted to avoid duplicating theme management logic on each platform.

## Shared logic approach

Theme selection resides in the shared KMP layer as business logic. The shared code manages:
- Available themes list
- Current theme state
- Persistence via DataStore
- Theme change emissions

Platform UI layers receive the selected theme and apply corresponding colors without managing state or persistence.

## Shared theme definitions

I created an `AppTheme` enum in Kotlin as the single source of truth. Adding a new theme requires one update to this enum. It includes metadata such as `isDark` to coordinate system chrome behavior (status bar, navigation bar) across platforms.

## Theme state distribution

The root presenter emits theme state as a `StateFlow`. Android collects this in the Activity while iOS observes it through a Swift friendly wrapper. Neither platform manages persistence; they react to changes from shared code.

## Platform integration details

On Android, I map `AppTheme` to Compose's `ColorScheme`. The Activity observes the state and passes it to `TvManiacTheme`, which applies the colors.

On iOS, I bridge `AppTheme` to `DeviceAppTheme` from the design system. Theme state changes trigger updates to the SwiftUI environment. Child views using `@Theme` automatically resolve the new colors.

The design system uses semantic tokens like `theme.colors.primary` and `theme.colors.surface`. Views reference these tokens, and the theme determines their actual color values.

## Design system advantages

A centralized design system ensures that when the theme changes:
1. Shared code emits the new theme.
2. Platforms update their theme environment.
3. Semantic token references automatically resolve to new colors.
4. Views re-render without modification.

Views remain unaware of specific color values, depending instead on semantic tokens.

## Theme selector component

I implemented a `ThemeSelectorView` that renders swatches in a grid. It is generic over the `ThemeItem` type, allowing it to adapt automatically when new enum cases are added.

## Implementation insights

Shared state prevents Android and iOS from drifting out of sync in theme selection or persistence. The design system investment was a prerequisite that simplified the implementation phase.

Defining color values in shared code is a potential future improvement. Currently, each platform defines hex values for its color schemes. Shared definitions would work for simple cases but may complicate native color system integration.

## Design system migration strategy

Building a design system requires significant upfront effort, including auditing scattered colors and defining semantic tokens. Without this infrastructure, dynamic theming requires updating hardcoded references across every file, increasing regression risks.

I recommend the following migration path:
1. **Perform an audit.** Use lint rules to identify hardcoded colors and duplicates.
2. **Establish the complete system.** Define the token structure (primary, secondary, surface, background) and naming conventions before integration.
3. **Migrate incrementally.** Update high traffic screens first and transition others during routine changes.
4. **Enforce tokens.** Deprecate hardcoded color utilities to encourage migration without blocking releases.

## Final Thougts

Foundational infrastructure simplifies future development. Adding themes now takes minutes because platforms use semantic tokens. Defining a color scheme updates the entire application without requiring file searches or manual edits.

Implementation details are available in the [pull request](https://github.com/thomaskioko/tv-maniac/pull/688).

Until we meet again, folks. Happy coding! ✌️

## Resources

- [Design System Article](/posts/swiftui-design-system-from-android-perspective/)
