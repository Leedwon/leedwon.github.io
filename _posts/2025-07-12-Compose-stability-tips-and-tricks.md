---
title: Compose stability tips and tricks
date: 2025-07-15
categories: [Android]
tags: [android, jetpack compose]
img_path: /assets/img/
---

Jetpack Compose uses stability to decide whether it can skip a composable during recomposition.
If your app uses too many unstable types, you might run into performance issues. That’s why it helps to understand what stability means in Compose, how to check it, and how to avoid common pitfalls.

## Stability in a nutshell 🥜 

The Android docs define a stable type as:

> A type is stable if it is immutable, or if it is possible for Compose to know whether its value has changed between recompositions. A type is unstable if Compose can't know whether its value has changed between recompositions.

Definitions alone can be a bit dry, so let’s look at some examples.

### Immutable data class with primitives

> A type is stable if it is immutable

```kotlin
data class Hero(val name: String, val power: String)
```
`Hero` is an immutable data class. Once a `Hero` is created, its values can't be changed. All properties are primitives (`String`), which are marked as immutable by default, that's why `Hero` is **stable** ✅.

### Mutable data class

```kotlin
data class UnstableHero(var name: String, var power: String)
```

Here, both properties are mutable. That means you can change the values after the object is created - but Compose won’t be notified when this happens, that's why `UnstableHero` is **unstable** ❌.

### Mutable but stable

> A type is stable if it is possible for Compose to know whether its value has changed between recompositions.

How does Compose *know*?

Compose provides an observable types such as `MutableState<T>`. Changes to mutable state schedule recomposition of any composable function that reads it. With that in mind, we can make a class that is both mutable and stable:

```kotlin
class MutableStableHero(
    name: String,
    power: String
) {
    var name by mutableStateOf(name)
    var power by mutableStateOf(power)
}
```

Even though the properties are mutable, `mutableStateOf` ensures that changes are tracked. Each update triggers Compose to recompose, that's why `MutableStableHero` is **stable** ✅ .

## How to check stability 🔍 

You now have a basic idea of what stability is. But what if you're unsure if a type is stable?

Luckily, there’s a way to find out.


### Compose compiler reports

Compose can generate reports showing stability of classes and composables.
These are disabled by default, but can be enabled in your module’s `build.gradle`:
```kotlin
  android { ... }

  composeCompiler {
    reportsDestination = layout.buildDirectory.dir("compose_compiler")
    metricsDestination = layout.buildDirectory.dir("compose_compiler")
  }
```

Run `./gradlew buildRelease` to generate a report. (For accurate results it is recommended to generate reports for release builds).

Check the generated folder:
`build/compose_compiler/`

You’ll find three reports:
- `modulename-classes.txt` -> Classes stability (the one we'll use here)
- `modulename-composables.txt` -> Info on restartable/skippable composables
- `modulename.composables.csv` -> CSV version that can be used in scripts

Here's a report for our earlier hero classes:
```
stable class Hero {
  stable val name: String
  stable val power: String
  <runtime stability> = Stable
}
unstable class UnstableHero {
  stable var name: String
  stable var power: String
  <runtime stability> = Unstable
}
stable class MutableStableHero {
  stable var name$delegate: MutableState<String>
  stable var power$delegate: MutableState<String>
  <runtime stability> = Stable
}
```

From this report, you can:
- Confirm whether a class is stable
- See which properties are stable or unstable
- Trace what's causing instability

## Common pitfalls & how to solve them 🕳️

Most of the time, Compose will infer stability for you.
But sometimes, things go sideways. Let’s go over a few tricky parts and see how to keep your code more stable.

### Collections

Let’s define an `Army` class with a list of heroes:

```kotlin
data class Army(
    val heroes: List<Hero>
)

val army = Army(
    heroes = listOf(
        Hero(name = "Hercules", power = "Strength"),
        Hero(name = "Achilles", power = "Invulnerability (except for heel)")
    )
)
```

This looks stable - `Army` is immutable, and `List<Hero>` looks fine at first glance.
Let’s check the report:

```
unstable class Army {
  unstable val heroes: List<Hero>
  <runtime stability> = Unstable
}
```

Whoops. Army is marked as unstable.

So what’s going on?

Compose doesn’t trust standard collections like `List`, `Map`, or `Set` - even when used immutably. That's because there’s no guarantee they won't be mutated elsewhere.

Here’s an example that shows the risk:
```kotlin
val sharedHeroes = mutableListOf(
    Hero(name = "Hercules", power = "Strength"),
    Hero(name = "Achilles", power = "Invulnerability  (except for heel)")
)

val army = Army(heroes = sharedHeroes)
```

Even though Army expects an immutable `List<Hero>`, we passed it a `MutableList`.
Later in a composable, we might do this:

```kotlin
(army.heroes as MutableList).add(Hero(name = "Perseus", power = "Petrification"))
```

Now the list has changed - but Compose has no idea. No recomposition will happen, and the UI won’t update.
This is exactly why Compose plays it safe and marks standard collections as unstable.

To fix this, we can use [kotlinx.immutable.collections](https://github.com/Kotlin/kotlinx.collections.immutable) which guarantees that collections are truly immutable:
```kotlin
data class StableArmy(
    val heroes: ImmutableList<Hero>
)
```
Now Compose knows the list is safe, and the report confirms it:
```
stable class StableArmy {
  stable val heroes: ImmutableList<Hero>
  <runtime stability> = Stable
}
```

### Java classes

Let's add birthday to our `Hero`
```kotlin
data class Hero(
    val name: String,
    val power: String,
    val birthday: Instant
)
```

We used `Instant` because Java docs says:

> This class is immutable and thread-safe.

It seems that hero should be stable, right?
But here’s what the report says:
```
unstable class Hero {
  stable val name: String
  stable val power: String
  unstable val birthday: Instant
  <runtime stability> = Unstable
}
```

The problem isn't with `Instant` itself, it's that Compose compiler can't verify its immutablility - Kotlin compiler plugins can only process Kotlin code.

We can fix this by using `@Stable` annotation. This tells the Compose compiler that **we guarantee** this class is stable. Once we do that, it becomes our responsibility to ensure that the class is either immutable or properly informs the Compose runtime about changes.

```kotlin
@Stable
data class Hero(
    val name: String,
    val power: String,
    val birthday: Instant
)
```
Let's check the report:
```
stable class Hero {
  stable val name: String
  stable val power: String
  unstable val birthday: Instant
}
```
Now the class is stable - even though `Instant` is still marked as unstable.

This works, but if you use `Instant` in lots of classes, repeating `@Stable` everywhere gets tedious.

Can we do better?

We want the Compose compiler to always treat `Instant` as `@Stable`. But it’s a third-party class that we don’t own, so we can’t just annotate `Instant` directly.

We could create a stable wrapper:
```kotlin
@Stable
data class StableInstant(val value: Instant)
```
But always using a wrapper can get annoying as well. So - can we actually do better?

### Stability configuration file

Turns out, we can. Compose supports a **stability configuration file**.

This is a plain text file with one class per row. It tells the Compose compiler that matching classes should be treated as stable.

For example, a simple configuration file could contain:
```
// Making Instant stable
java.time.Instant
```

You can also define more flexible rules using wildcards. Both single and double wildcards are supported:
```
// Making Instant stable 
java.time.Instant
// Consider any class in java.time stable 
java.time.*
```

To enable this configuration, add the following to your Compose compiler settings in `build.gradle`:
```kotlin
composeCompiler {
  stabilityConfigurationFiles = listOf(
    project.rootProject.layout.projectDirectory.file("compose_compiler_config.conf")
  )
}
```

Once `java.time.Instant` is included in config file, you don’t have to worry about it anymore.
`Instant` will now be considered stable:

```
stable class Hero {
  stable val name: String
  stable val power: String
  stable val birthday: Instant
  <runtime stability> = Stable
}
```

## Summary ✏️

- Primitive types (`String`, `Int`, `Float`, `Bolean`,etc.) are **immutable by default**.
- Immutable classes are **always stable**.
- Use `mutableStateOf` if you need a class to be **both mutable and stable**.
- Standard collections (`List`, `Map`, `Set`) are **not** stable by default.
- Use `kotlinx.immutable.collections` instead.
- To make Compose treat class as stable:
  - Annotate it with `@Stable` or
  - Add it to the stability config file.
  - ⚠️ When you do this, **you’re responsible** for ensuring the class is either immutable or properly informs the Compose runtime about changes.
- Use compiler reports to debug stability issues quickly.

**Stay stable!**

## Resources 📚 

- [https://developer.android.com/develop/ui/compose/performance/stability](https://developer.android.com/develop/ui/compose/performance/stability)
- [https://developer.android.com/develop/ui/compose/performance/stability](https://developer.android.com/develop/ui/compose/performance/stability)