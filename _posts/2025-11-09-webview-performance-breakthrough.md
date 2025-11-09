---
title: "Fixing a 4-5 Second WebView Reload Problem"
date: 2025-11-09 00:00:00 +0600
categories: [Flutter, Performance, Development]
tags: [flutter, webview, performance, optimization, architecture, mobile-development, navigation]
description: How we eliminated repeated 4-5 second webview reloads by keeping it alive instead of destroying it on every back navigation—a simple architectural change that transformed the user experience.
---

## The Problem We Had to Solve

We had a performance issue in our app. Every time users tapped on a queue item, they had to wait 4-5 seconds for the webview to load. 

The team gathered to figure it out. The problem was clear: our app was making users wait repeatedly for the same content to load, and it was affecting the user experience.

This wasn't something we could ignore.

---

## Understanding the Problem

Here's what was happening in the app:

1. User browses the queue list
2. User taps an item → Webview initializes from scratch (4-5 seconds)
3. User checks the content, hits back
4. **Webview gets destroyed** (to save memory, we thought)
5. User taps another item → Webview initializes from scratch AGAIN (another 4-5 seconds)
6. This repeats for every item they view

**What we observed:**
- **4-5 seconds** of loading every single time someone tapped a queue item
- The webview being completely destroyed whenever users went back to the list
- Every navigation to an item meant starting from zero
- Users were abandoning the flow after a few taps

Users were experiencing the same loading delay repeatedly. And it was creating a poor user experience.

### Why This Was Happening

The navigation pattern looked reasonable:

```dart
// User taps an item from the queue list
Navigator.push(
  context,
  MaterialPageRoute(
    builder: (context) => QueueItemView(item: selectedItem),
  ),
);

// The QueueItemView creates a webview
class QueueItemView extends StatelessWidget {
  final QueueItem item;
  
  @override
  Widget build(BuildContext context) {
    // ❌ Fresh webview every time this screen is built
    return WebView(
      initialUrl: item.url,
      onWebViewCreated: (controller) {
        // Starting everything from scratch
        // JavaScript context, DOM, APIs, assets... the works
      },
    );
  }
}
```

Here's what made it brutal:

**When user taps an item:**
- Navigator pushes a new route
- WebView initializes from zero
- JavaScript context boots up
- DOM renders
- API calls fire
- 4-5 seconds pass
- Finally, content appears

**When user hits back:**
- Navigator pops the route
- **WebView is completely destroyed**
- Everything is thrown away

**When user taps ANOTHER item:**
- Start the whole process over from step 1

It's like if you had to dismantle your entire home theater system, pack it away, and then unpack and set it up again every single time you wanted to watch a different movie. Technically it works, but why would you ever do that?

---

## The Solution: Keep It Alive

My boss suggested something simpler:

"Why don't we just keep the webview alive?"

**The idea:** Keep the webview running in the background. Don't destroy it—just hide it. Then use a message channel to tell it where to navigate. Let the frontend handle its own routing.

It was a simple insight. Instead of repeatedly destroying and recreating the webview, we could just maintain it and control its visibility.

### The Approach

Here's what changed:

**Old approach:**
```
Queue List → User taps item → Create & initialize webview (4-5s) → Show content
           ← User hits back ← Destroy webview ← 
Queue List → User taps another item → Create & initialize webview AGAIN (4-5s) → Show content
```

**New approach:**
```
Queue List (webview alive in background, hidden)
    ↓ User taps item
Show Webview + Send navigation message → Content appears quickly
    ↓ User hits back
Hide Webview (but keep it alive) → Back to queue list
    ↓ User taps another item
Show Webview + Send navigation message → Content appears quickly
```

The key difference: instead of destroying and recreating the webview, we keep it alive and toggle its visibility. When navigation is needed, we send a message rather than rebuilding everything.

---

## The Implementation

Here's how we implemented this solution.

### The Core Idea: Stack Instead of Navigator

The big change was moving from `Navigator.push/pop` (which destroys widgets) to a `Stack` with visibility toggling (which just hides/shows widgets).

### The Simplified Flutter Code

Here's the essence of what we changed. Instead of this old approach:

```dart
// ❌ OLD WAY: Navigator destroys the webview on pop
Navigator.push(
  context,
  MaterialPageRoute(
    builder: (context) => WebViewScreen(url: item.url),
  ),
);
```

We switched to this:

```dart
// ✅ NEW WAY: Stack keeps webview alive, just toggles visibility
bool _showWebView = false;

Widget build(BuildContext context) {
  return Stack(
    children: [
      QueueList(onItemTap: (item) {
        setState(() => _showWebView = true);
        // Tell webview to navigate to new content
      }),
      
      if (_showWebView)
        WebViewWidget(...), // This doesn't get destroyed!
    ],
  );
}
```

**How this works:**

1. **On screen load:** Create the webview once (takes 4-5 seconds, but only happens once)
2. **User taps item:** Set `_showWebView = true` and tell the webview what to show
3. **Webview appears quickly** because it's already loaded, just hidden
4. **User hits back:** Set `_showWebView = false` (webview hides but stays alive)
5. **User taps another item:** Back to step 2—fast because webview is still alive

The key point: `if (_showWebView)` removes it from the widget tree, but Flutter's state management keeps it alive. When you show it again, it's still there with all its state intact.

### How We Communicated Between Flutter and WebView

Instead of loading new URLs (which triggers full reloads), we used **message channels** to tell the webview what to show. Think of it like sending a text message to the webview saying "hey, navigate to item #2" instead of rebuilding the entire webview.

The frontend team built a router that listens for these messages and updates the content internally—no page reload needed. Flutter sends a message, the webview updates instantly.

**That's it.** No complex architecture. No caching layers. No pre-loading systems. Just: keep it alive, hide it when not needed, and talk to it via messages.

---

## The Results

The frontend team implemented the router, I handled the Flutter orchestration, and together we improved the experience significantly.

### Performance Metrics

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| **First item tap** | 4,500ms | 4,500ms* | Same (initial load) |
| **Second+ item taps** | 4,500ms each | ~200ms | **95% faster** |
| **WebView initializations per session** | Every tap | Once | Significant reduction |
| **Back/forward navigation** | Full reload | Fast | Much improved |
| **Memory footprint** | Stable | Stable** | Consistent |

*The first tap still takes 4-5 seconds because the webview needs to initialize. It only happens once per session.

**Memory stayed about the same. For our case, keeping one webview alive is more efficient than constantly creating and destroying them.

### User Experience Impact

**Before:**
```
Queue List → User taps item #1 → 4-5 sec loading → Content shows
           ← User goes back ←
Queue List → User taps item #2 → 4-5 sec loading AGAIN → Content shows
           ← User goes back ←  
Queue List → User taps item #3 → User gives up
```

**After:**
```
Queue List → User taps item #1 → 4-5 sec loading (one time) → Content shows
           ← User goes back ← Fast
Queue List → User taps item #2 → Fast → Content shows
           ← User goes back ← Fast
Queue List → User taps item #3 → Fast → Users keep browsing
```

The app experience improved significantly for the majority of interactions—browsing through multiple queue items.

---

## What I Learned (And You Can Too)

### 1. Not All Navigation Needs Navigator

When you learn Flutter, `Navigator.push` is the standard way to navigate between screens. It works well for most cases. However, `Navigator.pop` destroys widgets.

For simple widgets this isn't an issue. For expensive ones like webviews, this creates unnecessary overhead.

**Lesson:** When a widget is expensive to create, consider using a `Stack` with show/hide instead of Navigator push/pop.

### 2. Understanding Widget Lifecycle Matters

Every time you do `Navigator.pop()`, Flutter disposes of that entire widget and everything it built. Next time you navigate there, it builds everything again from scratch.

For a webview, that means:
- Destroying the JavaScript engine
- Clearing all the loaded content
- Discarding network requests
- Starting from zero

**Lesson:** Pay attention to when widgets get created and destroyed. Sometimes keeping them alive is more efficient.

### 3. Reduce Frequency Before Optimizing Speed

We didn't make the webview load faster. We didn't optimize the API calls. We didn't compress anything.

We simply stopped doing the expensive operation repeatedly.

**Lesson:** Before spending time making something fast, consider whether you can do it less often.

### 4. One Slow Experience > Ten Slow Experiences

Our fix doesn't eliminate the first 4-5 second wait. The webview still needs to initialize once. But that's okay because:
- Users only feel the pain once
- Every interaction after that is instant
- People browse multiple items, so the pain pays dividends

**Lesson:** It's okay to have some slowness if it means the rest of the experience is smooth.

---

## Final Thoughts

Key takeaways from this experience:

**Avoid unnecessary work.** We didn't optimize anything. We stopped doing the expensive operation repeatedly.

**Consider the full user journey.** We focused on "one 4-5 second load" and missed that users were experiencing this 3-4 times per session. The repetition was the real problem.

**Simple solutions can be effective.** We considered building a complex caching system. The actual solution was straightforward: a `Stack`, a boolean, and visibility toggling.

---

When dealing with performance issues, ask: am I trying to make something faster, or can I do it less often?

Sometimes the solution is straightforward: maintain state instead of recreating it.


---

**Tags:** #Flutter #WebView #Performance #Architecture #Optimization #MobileDevelopment

