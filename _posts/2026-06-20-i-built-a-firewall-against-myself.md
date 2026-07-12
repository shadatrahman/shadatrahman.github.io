---
title: "I Built a Firewall Against Myself"
date: 2026-06-20 00:00:00 +0600
categories: [Development, AI]
tags: [productivity, python, networking, macos, pf, pac, browser-extension, ai, focus]
description: claude-gate blackholes claude.ai in /etc/hosts, drops the traffic at the packet filter, and makes my shell refuse to run the CLI. Not to block a company. To block me. Default state is off — and turning it on has to be a decision.
---

The tool is called **claude-gate**, and every single person who hears the name assumes it's an API gateway. A proxy. Some load-balancing thing that sits in front of the Anthropic API and does routing.

It is the exact opposite.

It is a **firewall between me and Claude**, and I am the only thing it is defending against.

Its default state — the state it lands in the moment you install it, the state it returns to whenever anything goes wrong — is **blocked**. `claude.ai` resolves to `127.0.0.1`. The packet filter drops the traffic. And if you type `claude` in a terminal, your shell tells you no:

```
✖ Claude Proxy is OFF
Enable it from the tray app to access Claude.
```

Turning it *on* is the deliberate act. Not turning it off.

This is a post about why I did that, and about the surprisingly deep stack of network plumbing it takes to block a website from *yourself* in a way you can't casually undo.

---

## The Problem Is Not Willpower

Let me be precise about the failure mode, because "I got distracted" is not it.

The problem is that reaching for an AI agent had become **frictionless**, and frictionless is not the same as free. Every time I hit a mildly annoying problem — a config I half-remember, a regex I could write in ninety seconds, a bug I hadn't actually read the stack trace for yet — my hands would open a Claude session before my brain had finished forming the question.

That's not a productivity gain. That's a reflex. And it has two costs that took me a while to see clearly.

**The first is that I stopped thinking first.** Not always. But often enough that I noticed my own reasoning getting flabby. The ninety seconds I used to spend staring at a problem — the ones where you actually *understand* it — were getting outsourced to a model that would give me a plausible answer to a question I hadn't finished asking.

**The second is money.** Tokens cost real money, and a reflex is expensive in a way a decision isn't.

The fix for a reflex is not more willpower. **The fix for a reflex is friction.** You cannot out-discipline a habit that executes faster than deliberation. You have to put something physical in the path, so that "open Claude" stops being a twitch and starts being a *choice you make on purpose*.

So I built the friction.

---

## Three Locks, Because One Is a Suggestion

Here's what I learned quickly: blocking a website from yourself is trivial. Blocking a website from yourself *in a way that survives your own cleverness at 2am* requires more than one lock. Because the person trying to get around it is **you**, and you know exactly where you put the key.

So there are three layers, and they cover different escape routes.

### Lock 1: The `/etc/hosts` sinkhole

The bluntest instrument. Every Anthropic domain gets pointed at localhost:

```bash
127.0.0.1 claude.ai # CLAUDE-PROXY-BLOCK
127.0.0.1 www.claude.ai # CLAUDE-PROXY-BLOCK
127.0.0.1 api.claude.ai # CLAUDE-PROXY-BLOCK
127.0.0.1 anthropic.com # CLAUDE-PROXY-BLOCK
127.0.0.1 www.anthropic.com # CLAUDE-PROXY-BLOCK
127.0.0.1 api.anthropic.com # CLAUDE-PROXY-BLOCK
127.0.0.1 console.anthropic.com # CLAUDE-PROXY-BLOCK
127.0.0.1 statsig.anthropic.com # CLAUDE-PROXY-BLOCK
127.0.0.1 sentry.anthropic.com # CLAUDE-PROXY-BLOCK
```

Every line carries the marker comment `# CLAUDE-PROXY-BLOCK`. That's not decoration — it's what makes the whole thing **idempotent and reversible**. Enabling is one `sed`:

```bash
sed -i '' '/# CLAUDE-PROXY-BLOCK/d' /etc/hosts
```

Strip every marked line, leave everything else in `/etc/hosts` untouched. No parsing, no state file, no risk of eating a line that some other tool put there. If you're going to write into a file you don't own, **tag your lines and only ever delete your own tags.** I've seen far too many tools that "manage" `/etc/hosts` by rewriting the whole thing, and every one of them eventually eats something.

There's a self-check in the README that I find pleasingly blunt:

```bash
grep CLAUDE-PROXY-BLOCK /etc/hosts | wc -l    # 9 when OFF, 0 when ON
```

### Lock 2: The packet filter

The hosts file only catches DNS. It does nothing about traffic sent to a raw IP.

So on macOS there's a `pf` anchor, written fresh each time the gate closes:

```bash
IFACE=$(route -n get default 2>/dev/null | awk '/interface:/{print $2}')
IFACE="${IFACE:-en0}"
echo "block drop out quick on $IFACE proto tcp from any to $PROXY_IP port $PROXY_PORT" \
    > /etc/pf.anchors/claude-proxy
pfctl -a claude-proxy -f /etc/pf.anchors/claude-proxy 2>/dev/null
pfctl -e 2>/dev/null || true
```

Note it discovers the default interface at runtime rather than hardcoding `en0`, because I use this on wifi *and* ethernet *and* tethered, and a rule bound to the wrong interface is a rule that does nothing while looking like it does something.

(On Linux this becomes an `iptables`/`nft` REJECT rule against the same address. Same idea, different noun.)

### Lock 3: The PAC file, and the discard port

This is my favourite, because it's the one with a genuinely nice trick in it.

A **PAC** (Proxy Auto-Config) file is a JavaScript function the OS calls to decide how to route each request. Both macOS's network stack and the browser consult it.

When the gate is open, matching hosts route through a real proxy:

{% raw %}
```python
def pac_enabled(host: str, port: int, patterns: list[str]) -> str:
    cond = _shexpr_match_clauses(patterns)
    return f"""function FindProxyForURL(url, host) {{
    if ({cond}) {{
        return "PROXY {host}:{port}";
    }}
    return "DIRECT";
}}
"""
```
{% endraw %}

And when it's closed:

{% raw %}
```python
def pac_blocked(patterns: list[str]) -> str:
    cond = _shexpr_match_clauses(patterns)
    return f"""function FindProxyForURL(url, host) {{
    if ({cond}) {{
        return "PROXY 127.0.0.1:9";       # ← port 9
    }}
    return "DIRECT";
}}
"""
```
{% endraw %}

**`127.0.0.1:9`.** Port 9 is `DISCARD` — an ancient, reserved service whose entire specified behaviour is to throw away everything sent to it. Usually nothing is even listening.

So the browser dutifully tries to reach `claude.ai` *through a proxy that is a black hole*. The connection doesn't get refused with a nice error. It doesn't fall back to direct. It just goes nowhere and dies quietly.

There is something extremely satisfying about routing your own bad habit into the discard service.

---

## The Shell Guard: Shadowing Your Own Binary

The three network locks stop the browser. They also stop the CLI — but they stop it *slowly*, with a confusing timeout and a network error, and a confusing error is an invitation to start debugging instead of getting the message.

So the CLI gets its own, much more direct treatment: the installer appends a shell **function** named `claude` to `.zshrc` and `.bashrc`, which shadows the real binary.

```bash
# >>> claude-proxy >>>
claude() {
    if [ -f "$HOME/.claude-proxy-enabled" ]; then
        . "$HOME/.claude-proxy-enabled"      # sources http_proxy/https_proxy exports
        command claude "$@"                  # `command` = skip the function, run the real binary
    else
        printf '\033[31m\xe2\x9c\x96 Claude Proxy is OFF\033[0m\n' >&2
        echo "Enable it from the tray app to access Claude." >&2
        return 1
    fi
}
# <<< claude-proxy <<<
```

(Lightly trimmed — the real one also supports pointing at a specific binary.)

Two details worth stealing.

**`command claude "$@"`.** Inside a shell function named `claude`, calling `claude` would call *the function*, forever, until your stack gives out. `command` bypasses function lookup and runs the actual binary on `PATH`. If you've ever written a shell wrapper around a tool of the same name, you have met this bug, and you have met it in a hurry.

**The `# >>> claude-proxy >>>` markers.** Same trick as the hosts file. Install and uninstall are both idempotent, because they operate on a marked region rather than on "the end of the file" or a line number. Running the installer twice does not give you two functions.

And notice the flag file `~/.claude-proxy-enabled` isn't just a boolean — it's a *sourced shell script* that exports the proxy environment variables. Its **existence** is the switch, and its **contents** are the configuration. One file, two jobs, no parsing.

---

## Sudo, But Only Barely

Here's the uncomfortable engineering problem at the center of this thing.

Flipping the gate means editing `/etc/hosts` and reloading `pf`. Both need root. But the whole *point* is that flipping the gate has to be a single click on a tray icon — if it demands a password every time, I'll disable the tool inside a week.

Which leaves you standing in front of a genuinely bad menu:

1. **Run the tray app as root.** A GUI app, running a network stack, as root, forever. Absolutely not.
2. **Prompt for a password on every toggle.** Correct, and I will uninstall it by Thursday.
3. **`NOPASSWD: ALL`.** Passwordless root for everything, so a focus tool can edit a text file. This is how you find out what "attack surface" means.

None of those. The answer is passwordless sudo for **exactly two scripts and nothing else**:

```python
def install_sudoers(user: str, scripts: Iterable[Path]) -> Path:
    lines = [f"{user} ALL=(ALL) NOPASSWD: {p}" for p in scripts]
    body = "\n".join(lines) + "\n"
    target = Path("/etc/sudoers.d/claude-proxy")

    with tempfile.NamedTemporaryFile("w", delete=False, suffix=".sudoers") as tmp:
        tmp.write(body)
        tmp_path = tmp.name

    chk = subprocess.run(["sudo", "visudo", "-cf", tmp_path], capture_output=True, text=True)
    if chk.returncode != 0:
        os.unlink(tmp_path)
        raise RuntimeError(f"visudo validation failed: {chk.stderr}")

    subprocess.run(["sudo", "install", "-m", "0440", "-o", "root", "-g", "wheel",
                    tmp_path, str(target)], check=True)
    os.unlink(tmp_path)
    return target
```

`scripts` is exactly `enable.sh` and `disable.sh`. That's the entire grant.

But the line I want you to look at is this one:

```python
chk = subprocess.run(["sudo", "visudo", "-cf", tmp_path], ...)
```

**Never write to `/etc/sudoers.d/` without validating first.** A syntax error in a sudoers file doesn't fail gracefully — it can lock you out of `sudo` **entirely**, on the whole machine, and the tool you'd normally use to fix that is `sudo`. It is one of the few genuinely bricking mistakes you can make on a Unix box from userspace.

So: write to a temp file, run `visudo -cf` against it, and only if it validates do you `install` it into place with mode `0440` owned by `root:wheel`. If validation fails, delete the temp file and raise. The bad rule never touches `/etc`.

I want to be honest that this is still a real trade. I have granted a background tray app passwordless root on two shell scripts, in exchange for a focus tool I'll actually keep using. That's a *considered* trade, not a free one, and the narrowness of the grant plus the `visudo` gate is what makes me willing to make it.

---

## Two Enforcers, One Truth

The OS-level locks don't cover the browser extension, and the extension can't reach into `pf`. They're separate enforcement domains that have to agree, at all times, or the whole thing has a hole in it.

So there's a tiny local HTTP server — `ThreadingHTTPServer` on `127.0.0.1:9876` — exposing one endpoint:

```python
payload = {
    "enabled":        m.is_enabled(),
    "state":          m.state(),          # "on" | "off" | "paused"
    "pauseRemaining": m.pause_remaining(),
    "realIP":         fut_real.result(timeout=10),
    "proxyIP":        fut_proxy.result(timeout=10),
    "latencyMs":      fut_lat.result(timeout=10),
    "bytesBlocked":   fut_bytes.result(timeout=10),
    ...
}
```

(The IP, latency and byte-count probes are fanned out to a thread pool and joined with timeouts, so one slow network check can't hang the status endpoint.)

The MV3 extension polls that every 15 seconds and swaps its own `chrome.proxy.settings` PAC to match:

```javascript
async function pollStatus() {
  try {
    const r = await fetch(cfg.statusUrl, { cache: "no-store" });
    const data = await r.json();
    if (data.enabled !== lastEnabled) {
      lastEnabled = data.enabled;
      await applyProxy(data.enabled);
    }
  } catch (e) {
    if (lastEnabled !== false) {
      lastEnabled = false;
      await applyProxy(false);        // ← fail CLOSED
    }
  }
}
```

Look at the `catch`.

If the status server is unreachable — tray app crashed, not running, I killed it in a fit of pique — the extension does **not** assume everything's fine and open up. It assumes **blocked**.

**Fail closed.** The failure mode of the safety mechanism has to be *safety*. A lock that springs open when it gets confused is a door.

---

## The Pause Button, or: Designing for the Exception

Here's the part I got wrong at first, and it's the most interesting design lesson in the whole thing.

The original version had two states: on and off. And it was **too rigid**, in a specific and predictable way. Sometimes I genuinely need to look something up in the Claude web UI on a personal account for two minutes. Under a strict binary, my only option is to fully disable the gate — and once it's off, it stays off, because turning it back on is a thing I have to *remember* to do, and I never do.

**A safety mechanism that can only be removed permanently will be removed permanently.**

So there's a third state: **paused**. A timed, self-expiring exception.

```python
def pause(self, duration_sec: int | None = None, resume_to: str = "on") -> tuple[bool, str]:
    resume_at = 0 if not duration_sec else int(time.time()) + int(duration_sec)
    self._write_flag(_PAUSE_FLAG_BODY)
    self.sentinel.write_text(f"resume_at={resume_at}\nresume_to={resume_to}\n")
    ...
```

Two fields in a sentinel file, and both matter.

`resume_at` is a wall-clock deadline. There's no cron job, no OS timer, no daemon holding a countdown — the tray app's existing 15-second refresh loop simply checks whether the deadline has passed:

```python
def auto_resume_if_due(self) -> bool:
    rem = self.pause_remaining()
    if rem is None or rem == -1 or rem > 0:
        return False
    self.resume()
    return True
```

That's also called on startup, so an expired pause self-heals even if the app was closed the whole time. A timestamp in a text file plus an existing poll loop beats a scheduler you have to keep alive.

`resume_to` is the one I'm actually proud of. It records the state you were in **before** the pause, so resuming puts you *back where you were* rather than defaulting to some fixed value. Pause from "on," resume to "on." The pause is genuinely a parenthesis, not a reset.

The escape hatch **expires on its own**. That's the whole trick. I don't have to remember to re-arm it, and my future self — who is tired, and does not care about my goals — never gets a turn.

---

## What I'd Tell You

**1. Friction beats willpower.** You cannot out-discipline a reflex, because the reflex fires first. Put something physical in the path so the action requires a decision. The tray toggle takes two seconds — that's not a lot of friction. It turns out two seconds is enough, because two seconds is longer than a reflex.

**2. Make the safe state the default.** Not "block after I misbehave." Blocked at install, blocked at boot, blocked when the status server dies, blocked when the extension can't reach it. Getting access is the thing that requires an act. This is the same **fail-closed** instinct that should govern your tenant isolation and your auth — it's just pointed at your own attention.

**3. An escape hatch must expire on its own.** A permanent off switch will get flipped permanently. A *timed* exception gives you the flexibility you actually need without handing your future, tired self a way to quietly delete the whole system.

**4. Tag every line you write into someone else's file.** `# CLAUDE-PROXY-BLOCK` in `/etc/hosts`. `# >>> claude-proxy >>>` in `.zshrc`. Install and uninstall become one `sed` each, idempotent, and you will never eat a line that wasn't yours.

**5. `visudo -cf` before you touch `/etc/sudoers.d/`.** Validate on a temp file. A malformed sudoers entry can lock you out of `sudo` on the entire machine, and the tool for fixing that is `sudo`.

**6. `command` in a shell function that shadows a binary.** Otherwise you have written an infinite loop with a very confusing name.

---

## Does It Work?

Yes, and not in the way I expected.

I assumed the win would be *using Claude less*. It isn't, particularly — I still use it constantly, and it is still the most useful tool on my machine. Nothing about that changed.

What changed is that I now have to **decide** to use it. The two seconds it takes to click the tray icon is enough of a gap for the actual question to finish forming in my head. And a startling fraction of the time, in those two seconds, I realise I already know the answer, or that I haven't read the error message yet, or that what I actually needed was to think about the problem for ninety seconds like I used to.

The tool didn't make me use Claude less. It made me use it **on purpose**.

That turned out to be the thing I was missing.

---

**Tags:** #Productivity #Python #Networking #macOS #Focus #AI
