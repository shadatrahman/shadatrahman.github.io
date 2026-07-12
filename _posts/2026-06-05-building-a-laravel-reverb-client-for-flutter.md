---
title: "The Bugs That Only Exist on a Real Server — Building a Laravel Reverb Client for Flutter"
date: 2026-06-05 00:00:00 +0600
categories: [Flutter, Development]
tags: [flutter, dart, websockets, laravel, reverb, pusher, real-time, open-source, package]
description: I published pusher_reverb_flutter because the official Pusher SDK didn't fit Laravel Reverb. Then real servers taught me five reconnect bugs, two kinds of ping, and one protocol that lies in its own documentation.
---

Every WebSocket client works perfectly until someone's WiFi drops.

I've been maintaining [`pusher_reverb_flutter`](https://pub.dev/packages/pusher_reverb_flutter) — a pure-Dart client for [Laravel Reverb](https://reverb.laravel.com/) — since October. It has all the things you'd expect: public channels, private channels, presence channels, AES-256-CBC encrypted channels, a stream-based API, 90%+ test coverage.

And for the first several releases, every single one of those tests passed while the package quietly *did not work* on a real server.

Not in an obvious way. It connected. It subscribed. It received events. It looked *great*. And then your phone would go through a tunnel, come back, and the app would sit there in perfect, confident silence forever — connected to a server that had no idea who it was.

This is a post about that gap. About the specific, nasty, hard-won difference between "my WebSocket client passes its tests" and "my WebSocket client survives contact with reality."

---

## Why Write Another Pusher Client?

Laravel Reverb is Laravel's first-party WebSocket server. It speaks the **Pusher protocol**, which is great news, because that means every existing Pusher client should Just Work with it.

Should.

The official `pusher_channels_flutter` package is a Flutter wrapper around the *native* Pusher SDKs — a Java/Kotlin one on Android, a Swift one on iOS. That's a reasonable design, and it's also where the friction starts. Reverb deployments in the wild need two things the official client makes painful:

1. **Custom WebSocket paths.** Reverb frequently sits behind a reverse proxy at something like `/ws` or `/app/reverb`, not at the root. The native SDKs assume Pusher's own cloud topology.
2. **Dynamic authentication headers.** Private and presence channels authenticate against *your* Laravel backend, which means *your* auth scheme — usually a bearer token that rotates. You need to compute the auth headers fresh, per subscription, at the moment of subscribing. Baking a static header map in at construction time doesn't cut it.

Fighting a native SDK wrapper for both of those is miserable. The Pusher protocol, meanwhile, is just JSON over a WebSocket. It's genuinely simple.

So I wrote a native Dart implementation. No platform channels, no native SDKs, no `MethodChannel` marshalling — just `web_socket_channel`, some JSON, and the `encrypt` package for encrypted channels. It runs on Android, iOS, macOS, Windows, and Linux for free, because it's just Dart.

The authorizer is the whole reason the package exists, and it's a single function type:

```dart
final client = ReverbClient(
  host: 'my-app.com',
  port: 443,
  appKey: 'my-reverb-key',
  useTLS: true,
  customPath: '/ws',                              // ← reverse proxy, no problem

  // Called fresh on every private/presence subscription, with the LIVE socket ID
  authorizer: (channelName, socketId) async {
    final token = await authStore.currentAccessToken();   // ← rotated? fine.
    return {'Authorization': 'Bearer $token'};
  },
);
```

That's it. That's the pitch. Everything after this point is me learning that the simple part was the easy part.

---

## The Protocol, in Ninety Seconds

You need this to follow the bugs, so:

You open a WebSocket. The server immediately sends you a **`pusher:connection_established`** message containing your **socket ID** — a unique identifier for *this specific connection*. Remember that. It's about to ruin my week.

To subscribe, you send `pusher:subscribe` with a channel name. For a **private** or **presence** channel, you must first POST that socket ID plus the channel name to your Laravel backend, which signs it and hands back an auth token. Then you subscribe *with* the token.

The server confirms with `pusher_internal:subscription_succeeded`. Now you receive events on that channel.

Simple. Four messages. What could go wrong?

---

## Bug #1: The Crash That Taught Me To Distrust `data`

```
c4250b5  fix: resolve null-safety crash in WebSocket message handling
```

The first thing a real server does is send you something you didn't expect.

Every Pusher message has an `event` and a `data`. So of course I wrote `jsonDecode(data as String)`. And it worked, until a message arrived where `data` was `null`. Or an already-decoded `Map`. Or an empty string. Crash — inside a stream handler, which in Dart means it doesn't politely return an error, it takes the whole connection down with it.

The lesson generalizes to every WebSocket client you will ever write: **the wire is hostile.** Every field is `dynamic` and every field is lying to you. Now every handler in the package looks like this:

```dart
if (data == null || data is! String) {
  onError?.call(ConnectionException(
    'Invalid connection data: expected String, got ${data?.runtimeType}',
  ));
  return;                       // ← report it, don't die of it
}
```

Not clever. Just defensive. A malformed message should surface as a typed error on your `onError` callback, never as a dead socket.

---

## Bug #2: The Idle Disconnect, and the Two Kinds of Ping

```
7fabb04  feat: implement server ping response to maintain WebSocket connection
b6ab9d2  feat: Add configurable WebSocket pingInterval for idle disconnect fix
```

Symptom: leave the app idle for a couple of minutes and the connection dies. No error. No close frame. Just... over.

This one took me two separate fixes across two months, because there are **two completely different pings** in play and I only knew about one.

**Ping the first — the application-level one.** The Pusher protocol has its own heartbeat *inside* the message stream. Reverb (with `ping_interval` configured) sends you a JSON message that says `{"event": "pusher:ping"}`, and if you don't send back `{"event": "pusher:pong"}`, it assumes you're a zombie and hangs up.

```dart
if (event == 'pusher:ping') {
  _sendMessage(jsonEncode({'event': 'pusher:pong', 'data': data}));
  return;
}
```

Three lines. Fixed it. Except it *didn't*, not entirely, and that's when I learned about—

**Ping the second — the protocol-level one.** Underneath the JSON messages, the WebSocket protocol *itself* has ping/pong control frames. These are invisible to your message handler; they live at the transport layer. And the things between your phone and your server — NAT gateways, load balancers, mobile carrier middleboxes — don't read your JSON. They see a TCP connection with no bytes on it and they reap it.

You can't fix that from inside the message stream, because the whole problem is that there *are* no messages.

`web_socket_channel` exposes this, and it's a one-liner once you know it exists:

```dart
IOWebSocketChannel.connect(uri, pingInterval: pingInterval)
```

Now the socket emits real WebSocket ping frames at the transport level, keeping the middleboxes convinced there's a live human on the other end.

The two are so easy to confuse that I left a comment in the source purely so future-me wouldn't:

```dart
/// Note: This is separate from the application-level `pusher:ping`/`pusher:pong`
final Duration? pingInterval;
```

**If your WebSocket dies when idle, you probably need both.** One keeps the *server* happy. The other keeps the *network* happy. They are not substitutes.

---

## Bug #3: The Protocol Lied To Me

```
da8d848  fix: apply subscription_succeeded so channel reaches subscribed state
```

A user opened [issue #7](https://github.com/shadatrahman/pusher_reverb_flutter/issues): calling `whisper()` — the client-to-client event method, used for things like typing indicators — always threw `StateError: Channel must be subscribed`.

But it *was* subscribed. Events were flowing. You could watch them arrive.

Here's what was happening. I was reading the channel name for the `subscription_succeeded` confirmation out of the nested `data` object:

```dart
// ❌ Reads `channel` from inside data
final channelName = jsonDecode(data)['channel'];
```

The actual wire message puts `channel` at the **top level**:

```json
{
  "event": "pusher_internal:subscription_succeeded",
  "channel": "presence-chat",          ← up here
  "data": "{}"                         ← not in here
}
```

So `channelName` was always `null`, so the confirmation was never applied to any channel, so every channel in the package sat in `ChannelState.subscribing` **forever**. Events still worked, because event dispatch keyed off a different code path. Only `whisper()` — which correctly checks that you're subscribed before sending — ever noticed.

A bug that hides for three releases because only one method is honest enough to check its own preconditions.

```dart
// ✅ Top level, with a fallback for alternate payloads
String? channelName = decodedMessage['channel'] as String?;
channelName ??= subscriptionData['channel'] as String?;
```

**Lesson: `print()` the raw frames.** I lost hours to this because I was reading the protocol documentation instead of reading the bytes my actual server was actually sending. The docs describe the protocol. The server *is* the protocol.

---

## The Main Event: Five Reconnect Bugs Wearing a Trenchcoat

```
e4d926c  fix: auto-resubscribe channels after reconnect
```

That innocent little commit message is hiding a bloodbath. Version 0.0.8's changelog has **five** bug entries, and they're all the same bug wearing different hats:

> **A reconnect is not a resume. It's a new connection that has to pretend to be the old one.**

Reconnection itself was already handled — exponential backoff, `2^attempts` seconds, capped:

```dart
final delay = Duration(
  seconds: (pow(2, _reconnectAttempts) as int).clamp(1, _maxReconnectDelay),
);
```

That part worked fine. The socket came back. And that's exactly what made the rest so insidious: **the connection state said `connected`, and no events ever arrived again.** Silent, total, confident failure.

### 5a. The server has amnesia. The client doesn't.

When you reconnect, you get a **brand new WebSocket connection**. The server has never heard of you. It has *no idea* you were subscribed to seven channels ninety seconds ago.

My client, meanwhile, still had all seven `Channel` objects sitting happily in its `_channels` map. So when the app called `subscribeToChannel('orders')` again, my code found the cached object and cheerfully handed it back — *without ever sending a `pusher:subscribe` to the server.*

Client: "You're subscribed!" Server: "Who are you?" Both entirely sincere.

The fix is to resubscribe everything the moment the new connection is established:

```dart
if (event == 'pusher:connection_established') {
  socketId = connectionData['socket_id'] as String?;
  _setConnectionState(ConnectionState.connected);

  for (final channel in _channels.values.toList()) {
    channel.resetSubscriptionState();
    channel.subscribe();
  }
}
```

### 5b. The stale socket ID

This is my favourite, because it's a one-word bug.

```dart
class PrivateChannel extends Channel {
  final String socketId;    // ← `final`. Baked in at construction.
}
```

Remember the socket ID — the thing that identifies *this specific connection*, which I told you to remember? Private and presence channels authenticate by sending it to your Laravel backend for signing.

But it was `final`. Set once, when the channel object was created, on the *original* connection.

So after a reconnect: new connection, **new socket ID**. And my channel dutifully authenticated using the *old* one. Laravel signs `socket_id:channel_name` — the signature comes back valid-looking but bound to a connection that no longer exists. The server rejects the subscription. Auth fails.

Silently. Because of bug 5e, which we'll get to.

```dart
// ✅ Refresh the socket ID before every resubscribe
if (channel is PrivateChannel && socketId != null) {
  channel.socketId = socketId!;
}
```

One keyword — `final` — cost me every private channel in the package on every reconnect.

### 5c and 5d: The two bugs that fight each other

These two are worth the whole post, because their fixes pull in **opposite directions** and I had to satisfy both at once.

The package fires an `onConnected` callback when the connection is established, so apps can do setup. Naturally, apps subscribe and unsubscribe to channels inside it. Which creates a race with the resubscribe loop that runs right after.

**Bug 5c — the ghost subscription.** If the app calls `unsubscribeFromChannel('orders')` inside `onConnected`, but my resubscribe loop is iterating a list snapshotted *before* the callback ran, the loop resubscribes a channel the app just threw away. Now the server is pushing events down a channel no client object is listening to. A ghost. → *The fix wants me to snapshot **after** `onConnected`.*

**Bug 5d — the double subscribe.** If the app calls `subscribeToChannel('alerts')` for the first time inside `onConnected`, then `subscribeToChannel` sends `pusher:subscribe`... and then my loop, iterating a list snapshotted *after* the callback, sees the brand new channel and subscribes it **again**. Two `pusher:subscribe` frames for one channel. → *The fix wants me to snapshot **before** `onConnected`.*

Snapshot after, you get ghosts. Snapshot before, you get doubles. Both are right and they can't both be right.

The resolution: **snapshot before, then verify identity as you go.**

```dart
// Snapshot BEFORE onConnected — new channels added by the callback
// subscribe themselves, and must not be double-subscribed below.
final channelsToResubscribe = _channels.values.toList();

onConnected?.call(socketId);

for (final channel in channelsToResubscribe) {
  // ...but skip anything the callback removed or replaced.
  if (_channels[channel.name] != channel) continue;   // ← identity, not name

  channel.resetSubscriptionState();
  if (channel is PrivateChannel && socketId != null) {
    channel.socketId = socketId!;          // ← 5b
  }
  channel.subscribe().catchError(          // ← 5e
    (e) => onError?.call(ConnectionException(
      'Failed to resubscribe to ${channel.name}: $e',
    )),
  );
}
```

The snapshot is taken **before** the callback, so newly-added channels aren't in the list and can't be double-subscribed. And the `_channels[channel.name] != channel` check is an **identity** comparison, not a name comparison — so a channel that was removed (or removed *and re-added as a different object*) during the callback is skipped.

Fifteen lines. Five bugs. Every one of them only reachable by disconnecting a real device from a real server at exactly the wrong moment.

### 5e: The silence that hid everything else

And here's why every bug above was so hard to find.

`channel.subscribe()` returns a `Future`. When auth failed — because of the stale socket ID, say — that Future rejected. And nobody was awaiting it.

In Dart, an unhandled rejected Future doesn't crash your app and doesn't stop your loop. It just... goes somewhere. Maybe it prints to a console nobody's reading.

So the actual user-visible symptom of bugs 5a through 5d was **nothing at all.** No error. No exception. No state change. The connection said `connected`, the channel said `subscribing`, and events simply never came. I was debugging blind because my code was *swallowing the very error that would have told me what was wrong.*

That `.catchError(...)` routing failures to `onError` isn't the smallest fix in the list. It's the one that made the other four findable.

---

## What I'd Tell You About WebSocket Clients

**1. A reconnect is a new identity, not a resumed session.** Anything you cached that was scoped to the old connection — socket IDs, subscription state, sequence numbers, auth tokens — is now a lie. Go find every `final` field that was set at construction time and ask whether it survives a reconnect. Mine didn't.

**2. Test the disconnect, not the connect.** Every one of my tests connected, subscribed, asserted, and passed. Zero of them killed the socket mid-session and asserted that everything came back. That's the test that mattered and I didn't have it. Your happy path is not where your users live; they live on a train going into a tunnel.

**3. Unhandled Futures are how bugs hide.** An error that goes nowhere is worse than a crash. A crash tells you something. Route every async failure to a channel a human will actually see — and do it *first*, before you go hunting, or you'll be debugging with your eyes closed like I was.

**4. Read the wire, not the docs.** `subscription_succeeded` puts the channel name at the top level. The docs implied otherwise. Real servers are the specification; documentation is a rumour about the specification.

**5. "Idle" is not one problem.** Application-level heartbeats keep the *server* from reaping you. Transport-level ping frames keep the *network* from reaping you. You will probably need both, and no amount of staring at your message handler will reveal the second one.

**6. Ship it anyway.** `pusher_reverb_flutter` had a five-bug reconnect hole in it for months while people were using it in production. That's not a great feeling. But the bugs got found *because* it was published — issue #7 is why `whisper()` works today. A package that never ships has zero bugs and zero users, and those two numbers are related.

---

## Where It Is Now

v0.0.8 is on [pub.dev](https://pub.dev/packages/pusher_reverb_flutter), MIT licensed, pure Dart, and it now survives a reconnect with private and presence channels intact — which, embarrassingly, is a sentence I could not honestly have written a month ago.

If you're wiring Flutter to Laravel Reverb and the official SDK is fighting you over custom paths or rotating auth tokens, give it a look. And if you find a bug, [open an issue](https://github.com/shadatrahman/pusher_reverb_flutter). Genuinely — every good fix in this post came from someone who did.

The tests will keep passing either way. That's rather the problem with tests.

---

**Tags:** #Flutter #Dart #WebSockets #Laravel #Reverb #Pusher #RealTime #OpenSource
