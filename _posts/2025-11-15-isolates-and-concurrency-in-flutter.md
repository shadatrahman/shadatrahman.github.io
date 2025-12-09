---
title: Isolates and Concurrency in Flutter
date: 2025-12-09 00:00:00 +0600
categories: [Flutter, Performance]
tags: [flutter, dart, isolates, concurrency, performance, async]
description: A quick guide to using isolates in Flutter to keep your UI responsive during heavy computations.
---

When your Flutter app processes large amounts of data or performs heavy computations, the UI can freeze. This happens because Dart runs on a single thread by default, so all your code executes on the main isolate that also handles UI rendering.

## What are Isolates?

Isolates let you run code in parallel. Each isolate has its own memory space and runs independently, allowing you to offload heavy work from the main UI thread.

## Using compute()

The easiest way to use isolates is with Flutter's `compute()` function. It spawns an isolate, runs your function, and returns the result.

Here's a simple example:

```dart
import 'package:flutter/foundation.dart';
import 'package:flutter/material.dart';

// This function runs in a separate isolate
int _heavyComputation(int number) {
  int result = 0;
  for (int i = 0; i < number; i++) {
    result += i * i; // Heavy computation
  }
  return result;
}

class MyWidget extends StatefulWidget {
  @override
  _MyWidgetState createState() => _MyWidgetState();
}

class _MyWidgetState extends State<MyWidget> {
  int? _result;
  bool _isProcessing = false;

  Future<void> _processData() async {
    setState(() => _isProcessing = true);
    
    // Heavy computation runs in isolate - UI stays responsive!
    final result = await compute(_heavyComputation, 10000000);
    
    setState(() {
      _result = result;
      _isProcessing = false;
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Isolates Example')),
      body: Column(
        children: [
          if (_isProcessing) LinearProgressIndicator(),
          ElevatedButton(
            onPressed: _isProcessing ? null : _processData,
            child: Text('Process Data'),
          ),
          if (_result != null)
            Text('Result: $_result'),
        ],
      ),
    );
  }
}
```

The `compute()` function runs `_heavyComputation` in a separate isolate. While that's running, your UI thread stays free to handle user interactions, scrolling, and animations.

## Important Notes

- The function you pass to `compute()` must be top-level or static. Instance methods won't work.
- Data gets copied between isolates since they don't share memory. Keep this in mind for large datasets.
- `compute()` is great for one-off tasks. For long-running work with progress updates, use `Isolate.spawn()` instead.

## When to Use Isolates

Use isolates for:
- Parsing large JSON files
- Image processing
- Complex calculations
- Any operation taking more than 16ms (one frame at 60fps)

Don't use isolates for:
- Simple operations (the overhead isn't worth it)
- UI updates (isolates can't access UI directly)
- Frequent small tasks (too much overhead)

That's the basics of using isolates in Flutter. They're straightforward and make a huge difference in keeping your app responsive during heavy computations.
