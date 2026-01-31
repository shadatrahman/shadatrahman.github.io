---
title: Migrating to Jetpack Compose — A Flutter Developer's Guide
date: 2025-01-31 00:00:00 +0600
categories: [Android, Flutter, Development]
tags: [jetpack-compose, flutter, android, kotlin, mobile-development, declarative-ui, migration]
description: Coming from Flutter? Jetpack Compose feels like déjà vu with a Kotlin accent. Here's how to translate your Flutter mental model to Compose—widgets, state, navigation, and the gotchas that got me.
---

So you're a Flutter developer. You love declarative UI, hot reload, and building beautiful apps that run everywhere. But now you've got an Android-only project, or your team is going native, or you're just curious what the other side looks like. Maybe you're thinking: "How different can it really be? Declarative is declarative, right?"

I'll be honest—I had the same thought. After years of `Widget build(BuildContext context)`, I sat down with Jetpack Compose and thought, "This is just Flutter with Kotlin and a different accent." And you know what? I wasn't entirely wrong. But I also wasn't entirely right. There are enough differences to trip you up, and enough similarities to make the transition feel strangely familiar. Like meeting a cousin who grew up in another country.

This guide is for you—the Flutter developer who's ready to add Jetpack Compose to their toolkit. I'll map your existing knowledge to Compose, point out the gotchas, and help you avoid the mistakes I made. No judgment, no "you should have learned Kotlin first." Just a practical walkthrough from someone who's been in both worlds.

## The Big Picture: Same Family, Different Dialect

Both Flutter and Jetpack Compose are **declarative UI frameworks**. You describe *what* the UI should look like given the current state, and the framework figures out *how* to get there. No more imperative "findViewById and update this label." Just "here's the state, here's the UI." If you're comfortable with that paradigm in Flutter, you're 90% of the way there.

The main shift? **Composables instead of Widgets.** In Flutter, everything is a Widget. In Compose, everything is a `@Composable` function. Same idea—build a tree of UI components—different syntax. You'll get used to it faster than you think.

## Mental Model Mapping: Flutter → Compose

### Widgets Become Composables

**Flutter:**
```dart
class GreetingWidget extends StatelessWidget {
  final String name;
  
  const GreetingWidget({super.key, required this.name});

  @override
  Widget build(BuildContext context) {
    return Text('Hello, $name!');
  }
}
```

**Compose:**
```kotlin
@Composable
fun Greeting(
  name: String,
  modifier: Modifier = Modifier
) {
  Text(text = "Hello, $name!", modifier = modifier)
}
```

Notice the pattern: instead of a class with a `build` method, you have a function annotated with `@Composable`. Parameters become function parameters. And here's the first big difference: **modifiers**.

In Flutter, layout and styling often come from wrapper widgets: `Padding(padding: EdgeInsets.all(16), child: Text(...))`. In Compose, you chain **modifiers** on the component: `Text(..., modifier = Modifier.padding(16.dp))`. It's more like CSS or SwiftUI—you decorate the component rather than wrapping it. Took me a week to stop instinctively reaching for `Padding` wrappers. Old habits die hard.

### State: setState vs remember

This is where Flutter devs usually get tripped up. In Flutter, you use `StatefulWidget` and `setState()`. In Compose, state is handled with `remember` and `mutableStateOf`.

**Flutter:**
```dart
class CounterScreen extends StatefulWidget {
  @override
  _CounterScreenState createState() => _CounterScreenState();
}

class _CounterScreenState extends State<CounterScreen> {
  int _counter = 0;

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        Text('$_counter'),
        ElevatedButton(
          onPressed: () => setState(() => _counter++),
          child: Text('Increment'),
        ),
      ],
    );
  }
}
```

**Compose:**
```kotlin
@Composable
fun CounterScreen() {
  var counter by remember { mutableStateOf(0) }
  
  Column {
    Text(text = counter.toString())
    Button(onClick = { counter++ }) {
      Text("Increment")
    }
  }
}
```

The key insight: `remember` is like "keep this value across recompositions." When `counter` changes, the Composable recomposes (like Flutter's `build` running again), and the new value is displayed. The `by` delegate handles the get/set so you can write `counter++` instead of `counter = counter + 1`. Small thing, but it makes the code read nicely.

**Pro tip:** If you've used `useState` in React, Compose's `remember` + `mutableStateOf` will feel very familiar. Flutter's `StatefulWidget` is more verbose; Compose keeps everything in one function.

### Layout: Row and Column (You Know These)

Row and Column exist in both. The API is slightly different—Compose uses `horizontalArrangement` and `verticalArrangement` instead of `mainAxisAlignment`—but the idea is the same.

**Flutter:**
```dart
Row(
  mainAxisAlignment: MainAxisAlignment.center,
  children: [
    Icon(Icons.star),
    Text('Favorite'),
  ],
)
```

**Compose:**
```kotlin
Row(
  horizontalArrangement = Arrangement.Center,
  verticalAlignment = Alignment.CenterVertically
) {
  Icon(Icons.Default.Star, contentDescription = null)
  Text("Favorite")
}
```

One gotcha: in Compose, the main axis uses `Arrangement`, the cross axis uses `Alignment`. In Flutter it's `MainAxisAlignment` and `CrossAxisAlignment`. Different names, same concepts. You'll internalize it after a few screens.

### Lists: ListView.builder → LazyColumn

**Flutter:**
```dart
ListView.builder(
  itemCount: items.length,
  itemBuilder: (context, index) {
    return ListTile(
      title: Text(items[index].name),
    );
  },
)
```

**Compose:**
```kotlin
LazyColumn {
  items(items) { item ->
    ListItem(
      headlineContent = { Text(item.name) }
    )
  }
}
```

`LazyColumn` and `LazyRow` are the equivalents of `ListView.builder`—they only compose what's visible. No `ListView` vs `ListView.builder` distinction; in Compose, list composables are lazy by default. One less decision to make.

### Scaffold: Your Old Friend

Scaffold exists in Compose too. App bar, bottom nav, snackbars—same idea.

**Flutter:**
```dart
Scaffold(
  appBar: AppBar(title: Text('My App')),
  body: Center(child: Text('Hello')),
  floatingActionButton: FloatingActionButton(
    onPressed: () {},
    child: Icon(Icons.add),
  ),
)
```

**Compose:**
```kotlin
Scaffold(
  topBar = { TopAppBar(title = { Text("My App") }) },
  floatingActionButton = {
    FloatingActionButton(onClick = { }) {
      Icon(Icons.Default.Add, contentDescription = "Add")
    }
  }
) { paddingValues ->
  Box(
    modifier = Modifier
      .fillMaxSize()
      .padding(paddingValues),
    contentAlignment = Alignment.Center
  ) {
    Text("Hello")
  }
}
```

The main difference: Scaffold's body in Compose receives `paddingValues` as a lambda parameter—the insets for the app bar, FAB, etc. You're responsible for applying that padding. In Flutter, Scaffold often handles it for you. Small API difference, same result.

## Navigation: A Different Beast

Flutter's `Navigator.push` and `Navigator.pop` are straightforward. Compose has a dedicated Navigation library that's more structured—you define a graph of routes and navigate by route names or arguments.

**Flutter:**
```dart
Navigator.push(
  context,
  MaterialPageRoute(builder: (context) => DetailScreen(item: item)),
);
```

**Compose (Navigation Compose):**
```kotlin
val navController = rememberNavController()

NavHost(navController = navController, startDestination = "home") {
  composable("home") { HomeScreen(onItemClick = { item ->
    navController.navigate("detail/$item.id")
  }) }
  composable("detail/{itemId}") { backStackEntry ->
    val itemId = backStackEntry.arguments?.getString("itemId")
    DetailScreen(itemId = itemId, onBack = { navController.popBackStack() })
  }
}
```

It's more boilerplate upfront, but you get type-safe arguments and a clear navigation graph. If you've used go_router or similar in Flutter, the mental model is close. If you've only used the basic Navigator, give yourself a day to adjust.

## Theming: MaterialTheme vs ThemeData

**Flutter:**
```dart
MaterialApp(
  theme: ThemeData(
    colorScheme: ColorScheme.fromSeed(seedColor: Colors.blue),
    useMaterial3: true,
  ),
  home: HomePage(),
)
```

**Compose:**
```kotlin
MaterialTheme(
  colorScheme = lightColorScheme(),
  typography = Typography()
) {
  // Your app content
}
```

You access theme values with `MaterialTheme.colorScheme`, `MaterialTheme.typography`, etc.—similar to `Theme.of(context)` in Flutter. Dark mode is usually handled by wrapping with a different `ColorScheme` based on `isSystemInDarkTheme()`.

## Kotlin Quick Wins (If You're New to Kotlin)

Coming from Dart, Kotlin will feel familiar—null safety, data classes, lambdas. A few things that tripped me up:

1. **`val` vs `var`:** `val` is final, `var` is mutable. Same as `final` vs non-final in Dart.

2. **`?.` and `!!`:** Null safety. `item?.name` is like `item?.name` in Dart. `!!` means "I promise this isn't null" (use sparingly).

3. **`by` delegate:** In `var counter by remember { mutableStateOf(0) }`, `by` delegates the getter/setter so you can use `counter` directly instead of `counter.value`.

4. **Trailing lambdas:** `Column { ... }` is shorthand for `Column(content = { ... })`. Kotlin allows the last lambda to move outside the parentheses. You'll see this everywhere in Compose.

5. **`.dp` and `.sp`:** In Flutter you use `EdgeInsets.all(16)` or `fontSize: 16`. In Compose, `16.dp` and `16.sp` are extension properties. Make sure you have `import androidx.compose.ui.unit.dp` (and `.sp`).

## Common Gotchas (Learn From My Mistakes)

### 1. Recomposition Can Run *A Lot*

In Flutter, `build` runs when `setState` is called or when the parent rebuilds. In Compose, recomposition can happen when *any* state read during composition changes. If you read `viewModel.items` in your Composable, changing `items` will recompose. That's by design—but it means you need to be careful about what you put in composition. Don't do heavy work or launch coroutines without `LaunchedEffect` or `rememberCoroutineScope`. I learned this the hard way when my Composable was fetching data on every recomposition. Oops.

### 2. Keys Matter (Just Like Flutter)

When you have a list of composables, use `key()` to help Compose track identity—especially in `LazyColumn` with dynamic lists. Same idea as Flutter's `ValueKey` or `ObjectKey` in list items.

```kotlin
LazyColumn {
  items(items, key = { it.id }) { item ->
    ItemRow(item = item)
  }
}
```

### 3. Modifier Order Matters

Modifiers are applied in order. `Modifier.padding(16.dp).clickable { }` applies padding first, then makes the padded area clickable. `Modifier.clickable { }.padding(16.dp)` makes the whole thing clickable first, then adds padding. Same as Flutter's widget order—inner vs outer.

### 4. No Hot Reload (But There Is Live Edit)

Flutter's hot reload is magical. Compose doesn't have it in the same form—but Android Studio's **Live Edit** (for Kotlin 1.9+) can update Composables without a full rebuild. It's not as instant as Flutter, but it helps. Enable it and thank me later.

### 5. Composables Can't Return Nothing

Every Composable must call other Composables or emit UI. You can't have a Composable that sometimes returns `null` like you might with a conditional Widget in Flutter. Use `if` to conditionally compose:

```kotlin
@Composable
fun ConditionalContent(show: Boolean) {
  if (show) {
    Text("Visible")
  }
  // Don't return null—just don't call anything if you have nothing to show
}
```

## Migration Strategy: Start Small

If you're adding Compose to an existing Android app (Views/XML), the recommended approach is incremental:

1. **Build new screens in Compose.** Don't rewrite the whole app. Add new features as Compose screens.

2. **Use ComposeView in existing screens.** You can embed Compose inside a View hierarchy. One screen at a time.

3. **Extract reusable components.** As you build, create a library of shared Composables—buttons, cards, inputs. Same idea as Flutter widgets in a `/widgets` folder.

4. **Replace simple screens first.** Welcome screens, settings, confirmation dialogs—low risk, high learning value.

## Resources That Helped Me

- **Official: Flutter for Compose developers** — Flutter's docs have a [guide for Compose devs](https://docs.flutter.dev/flutter-for/compose-devs). Reading it *backwards* (as a Flutter dev going to Compose) is surprisingly useful. The concepts map cleanly.
- **Android Docs: Migrating to Compose** — The [migration strategy](https://developer.android.com/develop/ui/compose/migrate/strategy) and [interoperability APIs](https://developer.android.com/develop/ui/compose/migrate/interoperability-apis) are gold for incremental adoption.
- **Compose Pathway** — Google's structured learning path on Android Developers. Solid if you want a full curriculum.

## Wrapping Up

Migrating from Flutter to Jetpack Compose is less "learning a new framework" and more "learning a new dialect of a language you already speak." The declarative mindset, the composition model, the state-driven UI—it all translates. The differences are mostly syntactic and a few conceptual twists (modifiers, recomposition scope, Kotlin null safety).

Give yourself a week to build something small—a todo app, a settings screen, a detail page. You'll fumble with modifier order and Kotlin syntax at first. That's normal. By the second week, you'll be cruising. And honestly? Having both Flutter and Compose in your toolbox feels pretty good. Cross-platform when you need it, native when you want it.

If you're doing both—or thinking about it—drop a comment. I'm curious how many of us are living in both worlds now.

---

**Ready to try Compose?** Start with a new screen in your existing app, or spin up a fresh Compose project in Android Studio. The default template gives you a working counter—swap it for something you'd build in Flutter and see how it feels.

_#JetpackCompose #FlutterDev #AndroidDev #Kotlin #DeclarativeUI #MobileDevelopment_
