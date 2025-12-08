---
title: "Building a Design System in SwiftUI: An Android Developer's Perspective"
date: "2025-12-06"
draft: false
hideToc: true
tags: ["swift", "swiftui", "design-system", "kmp", "ios"]
series: "Tv Maniac Journey"
---

# Intro

While building [Tv Maniac](https://github.com/thomaskioko/tv-maniac), a Kotlin Multiplatform project, I needed consistent styling across Android and iOS. On Android, Jetpack Compose gives us `MaterialTheme` out of the box. On iOS? We have to build our own. This article walks through the approach I took to create a design system in SwiftUI that mirrors the ergonomics of Compose theming.

If you'd like to see the code, [here's the pull request](https://github.com/thomaskioko/tv-maniac/pull/687).

## The Problem

In Jetpack Compose, accessing design tokens is straightforward:

```kotlin
Text(
    text = "Hello",
    color = MaterialTheme.colorScheme.primary,
    style = MaterialTheme.typography.bodyMedium
)
```

`MaterialTheme` provides global access to colors, typography, and shapes through Compose's `CompositionLocal` system. Any composable in the tree can access these values without explicitly passing them down.

SwiftUI doesn't have a built-in equivalent. You can use `@Environment(\.colorScheme)` for light/dark mode detection, but there's no `MaterialTheme.colorScheme.primary`. This means developers often resort to hardcoded values scattered across views, or passing theme objects through initializers. Neither approach scales well.

What I wanted: centralized design tokens accessible from anywhere in the view hierarchy, just like Compose.

## The Solution

The approach involves three parts working together:

1. **Protocol** - Defines what a theme provides (colors, typography, spacing, shapes)
2. **Environment** - Makes the theme available throughout the view tree
3. **Property Wrapper** - Provides clean access syntax in views

Let's look at each piece.

### The Theme Protocol

First, define what a theme contains:

```swift
public protocol TvManiacTheme {
    var colors: TvManiacColorScheme { get }
    var typography: TvManiacTypographyScheme { get }
    var spacing: TvManiacSpacingScheme { get }
    var shapes: TvManiacShapeScheme { get }
}
```

Then create concrete implementations for light and dark themes:

```swift
public struct LightTheme: TvManiacTheme {
    public let colors = TvManiacColorScheme.light
    public let typography = TvManiacTypographyScheme.default
    public let spacing = TvManiacSpacingScheme.default
    public let shapes = TvManiacShapeScheme.default
}
```

### The @Theme Property Wrapper

SwiftUI's `Environment` system is the equivalent of Compose's `CompositionLocal`. We create a custom environment key and wrap it in a property wrapper for ergonomic access:

```swift
public struct TvManiacThemeKey: EnvironmentKey {
    public static let defaultValue: TvManiacTheme = LightTheme()
}

@propertyWrapper
public struct Theme: DynamicProperty {
    @Environment(\.tvManiacTheme) private var theme: TvManiacTheme
    public var wrappedValue: TvManiacTheme { theme }
    public init() {}
}
```

The `DynamicProperty` conformance ensures SwiftUI updates views when the theme changes (e.g., switching between light and dark mode).

## Design Tokens

I organized tokens into four groups, following Material 3 conventions:

**Colors** use semantic naming (`primary`, `onPrimary`, `surface`, `onSurface`, etc.) rather than literal names. This makes it easy to swap entire color schemes without changing view code:

```swift
public struct TvManiacColorScheme {
    public let primary: Color
    public let onPrimary: Color
    public let surface: Color
    public let onSurface: Color
    public let error: Color
    public let onError: Color
    // ... additional tokens
}
```

**Typography** mirrors Material 3's type scale with 15 styles from `displayLarge` down to `labelSmall`. Each uses a custom font with appropriate size and weight.

**Spacing** follows a consistent scale (0, 2, 4, 8, 12, 16, 24, 32, 48, 64) for predictable layouts.

**Shapes** define corner radii for small, medium, large, and extra-large components.

## Usage

Injecting the theme happens once at the app level:

```swift
ContentView()
    .environment(\.tvManiacTheme, colorScheme == .dark ? DarkTheme() : LightTheme())
```

Then any view can access tokens with the `@Theme` wrapper:

**Before (hardcoded values):**
```swift
Text(show.title)
    .font(.system(size: 16, weight: .semibold))
    .foregroundColor(Color(hex: "1F2123"))
```

**After (themed values):**
```swift
@Theme private var theme

Text(show.title)
    .font(theme.typography.titleMedium)
    .foregroundColor(theme.colors.onSurface)
```

The themed version adapts automatically to light/dark mode and keeps styling consistent across the app.

## Adoption Strategy

You don't have to migrate everything at once. I approached this incrementally:

1. **Create the design system** - Build the theme protocol, tokens, and property wrapper in a separate module
2. **Migrate screen by screen** - Pick one view, replace hardcoded values with theme tokens, verify it works
3. **Add deprecation warnings** - Mark old color/font extensions as `@available(*, deprecated, message: "Use theme.colors instead")` to guide future development

For teams wanting stricter enforcement, [SwiftLint custom rules](https://realm.github.io/SwiftLint/custom_rules.html) can catch hardcoded values:

```yaml
custom_rules:
  no_hardcoded_colors:
    regex: 'Color\(hex:'
    message: "Use theme.colors instead of hardcoded hex values"
    severity: warning
```

This flags any `Color(hex: "...")` usage during builds, nudging developers toward the design system.

## Key Takeaways

- **SwiftUI Environment = Compose CompositionLocal**: Both provide implicit data flow through the view hierarchy
- **Property wrappers make access ergonomic**: `@Theme` is as clean as `MaterialTheme` access in Compose
- **Explicit tokens beat hardcoded values**: Changing a color once updates it everywhere
- **Protocol-based design enables flexibility**: Easy to add new themes or override tokens for specific screens

Building a design system takes initial effort, but pays off quickly. Every new view I create now uses consistent styling without hunting for the "right" hex code or font size. The iOS app looks cohesive with its Android counterpart, which matters for a KMP project where users might switch between platforms.

With this in place, we can now add more dynamic themes into the app. Another thing I might actually consider is having all the tokens come from KMM. It's just an idea but time will tell. 

Until we meet again, folks. Happy coding! ✌️
