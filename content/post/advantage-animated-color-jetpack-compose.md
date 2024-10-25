+++
author = "huyhunhngc"
title = "Advanced animated color schemes in Jetpack Compose"
date = "2024-09-15"
description = "Complex compositions, such as a sequence of colors, not only the background but also changes to text, image tint, etc.,"
tags = [
    "Kotlin", "Jetpack Compose", "Animation"
]
+++

[![Static Badge](https://img.shields.io/badge/View%20the%20article%20on%20medium-1f2328?style=for-the-badge&logo=medium&logoColor=fff)](https://medium.com/@huyhunhngc/advanced-animated-color-schemes-in-jetpack-compose-c1b9232f4c0b)

### Taking inspiration from: Animate background color

The simplest example when you want to change the background with animated color.

![inspiration](https://developer.android.com/static/develop/ui/compose/images/animations/animated_forever.gif "Animating background color of composable")

```kotlin
val animatedColor by animateColorAsState(
    if (animateBackgroundColor) colorGreen else colorBlue,
    label = "color"
)
Column(
    modifier = Modifier.drawBehind {
        drawRect(animatedColor)
    }
) {
    // your composable here
}
```

### Usage

Generate the corresponding color scheme and provide it to the app theme using `CompositionLocal`.

```kotlin
val colorScheme = LocalDynamicAnimatedTheme.current
val surfaceContainer by animateColor(colorScheme.surfaceContainer)
val primaryContainer by animateColor(colorScheme.primaryContainer)
val secondaryColor by animateColor(colorScheme.secondary)
val primaryColor by animateColor(colorScheme.primary)
val tertiaryColor by animateColor(colorScheme.tertiary)
val tertiaryContainer by animateColor(colorScheme.tertiaryContainer)
```

Create `animateColor` composable

```kotlin
@Composable
fun animateColor(targetValue: Color): State<Color> {
    return animateColorAsState(
        targetValue = targetValue,
        label = "dynamic",
        animationSpec = tween(AnimatedThemeDuration)
    )
}
```

Let's see the results:
<iframe src="https://drive.google.com/file/d/1stQ0YLL31mUPoxt0ZgOwmM2b7Jgi3H06/preview" width="270" height="600" allow="autoplay"></iframe>
