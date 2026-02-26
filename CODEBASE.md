# How `githate` Works — Explained Like You're 10

Okay so imagine you have a bunch of friends on GitHub. Some of them follow you. But sometimes, secretly, one of them **unfollows** you. You'd never know. That's annoying. This tool fixes that. It watches your followers and tells you when someone bounces.

---

## The Big Picture

When you run `githate` in your terminal, here's what happens at a high level:

1. It logs into GitHub on your behalf (like signing in with your GitHub account)
2. It takes a photo of who's following you right now
3. Next time you run it, it takes another photo
4. It compares the two photos
5. It shows you who left — those are your **haters**

That's it. That's the whole app.

---

## Folder Structure

```
hater/
├── bin/
│   └── hater.js          ← The front door. Where everything starts.
├── lib/
│   ├── commands/         ← One file per thing you can do (scan, login, etc.)
│   ├── utils/            ← Helper tools (saving data, talking to GitHub, etc.)
│   └── ui/               ← Making things look pretty in the terminal
└── package.json          ← The app's ID card. Lists what it's called + what packages it uses.
```

Think of `bin/` as the **receptionist** — it greets you and sends you to the right room. Think of `lib/` as the **actual rooms** where work gets done.

---

## `bin/hater.js` — The Front Door

This is the very first file that runs when you type `githate` in your terminal.

It uses a package called **Commander** (more on that below) to read what you typed and figure out what to do.

- You typed `githate scan`? → Go run the scan command.
- You typed `githate login`? → Go run the login command.
- You typed just `githate` with nothing else? → Open the interactive REPL shell (like a mini chat app inside your terminal).

It also registers a **secret hidden command** called `__daemon` — this is used internally when the app wants to start a background process. You never type this yourself; the app calls itself with this flag.

---

## `lib/commands/` — The Actual Features

Each file here is one thing the app can do.

---

### `login.js` — Logging Into GitHub

This handles signing you in. It uses something called the **GitHub Device Flow**, which works like this:

1. The app asks GitHub: "Hey, I want to log in."
2. GitHub gives it a special code, like `ABCD-1234`.
3. The app copies that code to your clipboard automatically.
4. It tells you: "Go to github.com/login/device and paste this code."
5. You do that in your browser.
6. GitHub tells the app: "Okay, they confirmed it. Here's their access token."
7. The app saves that token to your computer so you don't have to log in again.

An **access token** is like a keycard — it proves to GitHub that you are who you say you are.

---

### `logout.js` — Logging Out

Simple. It just deletes your saved keycard (the token) from your computer. Next time you try to do something, it'll ask you to log in again.

---

### `scan.js` — The Main Thing

This is the heart of the app. When you run `githate scan`, here's what happens step by step:

1. **Get your GitHub login** — reads the saved token from your computer
2. **Ask GitHub** for your current followers list (using the GitHub API)
3. **Compare** it to the list saved from last time
4. **First time ever running it?** → Just save the list. Nothing to compare yet.
5. **Not the first time?** → Show who's new (with a green `+`) and who left (with a red `✕`)
6. **Save the unfollowers** to a permanent "hall of shame" list
7. **Update the saved list** so next scan has fresh data to compare against

Each new follower and unfollower is shown with a dramatic 500ms pause for effect. 😂

---

### `watch.js` — The Background Scanner

This is like `scan.js` but it **never stops**. It runs, waits a bit, runs again, waits, runs again... forever.

This is what the background **daemon** runs. You never call this directly. When you run `githate start`, the app secretly starts a background version of itself that runs `watch.js` on repeat.

When it detects a change (someone followed or unfollowed you), it fires a **desktop notification** — like the little pop-up alerts you get from Slack or Discord.

---

### `haters.js` — The Hall of Shame

Shows everyone who has ever unfollowed you. Reads from the permanent record that `scan.js` keeps building up. Groups repeat offenders (people who unfollowed you more than once — cowards) and shows when they last did it.

---

### `followers.js` and `following.js` — List Views

Simple commands that just show you:
- `followers.js` → everyone currently following you
- `following.js` → everyone you are currently following

Both talk to GitHub's API to get live data.

---

### `daemon-control.js` — The Remote Control for the Background Process

This handles the `start`, `stop`, and `status` commands for the background process.

- `start` → tells the daemon to start running in the background
- `stop` → kills it
- `status` → checks if it's alive and tells you the PID (a number your computer uses to identify a running process)

---

### `repl.js` — The Interactive Shell

When you just type `githate` with no arguments, instead of printing help and quitting, the app opens an **interactive shell** — like a mini command line inside the command line.

You see a prompt like this:
```
githate >
```

And you can type commands like `/scan`, `/login`, `/haters`, etc. It keeps running until you type `/quit`.

Fun extra: if you type `hi`, `hey`, `sup`, or similar greetings, it responds back with slang. Just for vibes.

It uses Node's built-in `readline` module — think of that as the part that listens to what you type in the terminal.

There's one tricky bit: some commands (like the confirm dialog in `/clearcache`) use a package called `@clack/prompts` which temporarily takes over the terminal input. So before running those commands, the REPL pauses the readline listener, lets clack do its thing, then hands control back. If something goes wrong and readline closes unexpectedly, the REPL restarts itself automatically.

---

### `clear-cache.js` — Nuclear Option

Wipes everything: your saved token, your username, your follower list, your haters list, everything. Asks for confirmation first so you don't do it by accident.

---

## `lib/utils/` — The Engine Room

These aren't commands you call directly. They're tools that the commands use to get things done.

---

### `store.js` — The Memory

The app needs to remember things between runs (your token, your followers, your haters). It uses a package called **`conf`** which just saves stuff to a JSON file on your computer.

Think of `store.js` as a tiny notebook that the app can read from and write to at any time. Every other part of the app that needs to save or load something goes through this.

What's stored:
| Thing | What it is |
|---|---|
| `token` | Your GitHub keycard |
| `username` | Your GitHub username |
| `followers` | The snapshot of who was following you last time |
| `lastCheck` | When the last scan happened |
| `haters` | The permanent list of people who unfollowed you |

---

### `auth.js` — The GitHub Handshake

This file is responsible for creating and maintaining the **Octokit** object. Octokit is the official JavaScript library for talking to the GitHub API — think of it as a fancy phone that only calls GitHub.

`auth.js` creates this phone once and reuses it everywhere (this is called a **singleton** — fancy word for "only make it once"). If you have a token saved, the phone is authenticated (GitHub knows who you are). If not, it's unauthenticated (you can still make some calls but GitHub will rate-limit you faster).

It also contains the **device flow login logic** (described in `login.js` above) and the OAuth client ID for the GitHub app.

---

### `checker.js` — The Diff Calculator

This is a pure utility function that does the actual comparison work.

It takes: your current followers (from GitHub) and your stored followers (from the notebook).
It returns: who's new, who left, and whether this is the first time ever running.

No side effects. It doesn't save anything. It just does the math and hands back the results. `scan.js` and `watch.js` both use this.

---

### `daemon.js` — The Process Manager

This manages the **background process** (the daemon).

When you run `githate start`:
1. It spawns a new, completely separate Node.js process
2. That process runs `node bin/hater.js __daemon --interval 120`
3. It's **detached** — meaning it keeps running even after you close your terminal
4. It has `stdio: 'ignore'` — meaning it doesn't print anything, it runs silently
5. The PID (process ID number) is saved to the store so we can kill it later

When you run `githate stop`:
1. It reads the saved PID
2. It sends a `SIGTERM` signal to that process — like tapping it on the shoulder and saying "please stop"
3. It clears the saved PID

When you run `githate status`:
1. It reads the saved PID
2. It tries `kill(pid, 0)` — this doesn't actually kill anything, it's just a trick to check if the process is still alive
3. It reports back: running, stopped, or stale (PID saved but process is gone)

---

### `notify.js` — The Pop-Up Alert

When the background daemon finds a change, it calls this. It fires a desktop notification using **`node-notifier`** — the same kind of notification you'd get from any other app on your Mac/Windows/Linux.

---

## `lib/ui/` — Making It Look Good

---

### `art.js` — The Big ASCII Art Banners

Uses **`figlet`** to generate those huge text banners made of ASCII characters. Then colors them with **`gradient-string`**.

Two styles:
- `getBanner()` — "ANSI Shadow" font with a retro gradient (used at startup)
- `getHaterReveal()` — "DOS Rebel" font with a passion gradient (used for the HATERS screen)

---

### `display.js` — All The Output Helpers

Every time the app prints something to the terminal, it goes through one of these functions:

| Function | What it does |
|---|---|
| `displayIntro()` | Prints the big ASCII art banner |
| `displayError()` | Red box around an error message (uses `boxen`) |
| `displaySuccess()` | Green checkmark + message |
| `displayInfo()` | Blue diamond + dimmed message |
| `displayWarning()` | Yellow exclamation + message |
| `displayHater()` | Red `✕` + clickable username link + "unfollowed you" |
| `createSpinner()` | A loading spinner (uses `@clack/prompts`) |
| `link()` | Makes text in the terminal clickable (opens a URL) |

---

### `repl-text.js` — The REPL's Text Content

Three blocks of static text for the REPL:

- `displayWhy()` — The origin story. The author noticed their follower count dropped at 1am and couldn't figure out who left. So they built this.
- `displayREPLHelp()` — The `/help` screen. Lists all commands.
- `displayREPLWelcome()` — The welcome screen when you first open the REPL. Clears the terminal, shows the banner, shows a 5-step getting started guide.

---

## The Dependencies (packages) — What Each One Does

These are all listed in `package.json`. Each one is a pre-built tool someone else wrote that this app uses.

| Package | What it does in plain English |
|---|---|
| `commander` | Reads what you typed (`githate scan`) and routes it to the right function |
| `@octokit/rest` | The official GitHub API client — how the app talks to GitHub |
| `@octokit/auth-oauth-device` | Handles the "here's a code, go paste it in your browser" login flow |
| `conf` | Saves stuff to a JSON file on your computer so it's remembered next time |
| `chalk` | Colors text in the terminal (red, green, blue, etc.) |
| `picocolors` | Same as chalk but smaller/lighter — used in a few places |
| `boxen` | Draws a box border around text in the terminal |
| `figlet` | Turns normal text into giant ASCII art |
| `gradient-string` | Colors text with a gradient (multiple colors fading into each other) |
| `@clack/prompts` | Fancy interactive prompts — spinners, confirm dialogs, etc. |
| `clipboardy` | Copies text to your clipboard (used to auto-copy the OAuth device code) |
| `node-notifier` | Sends pop-up desktop notifications |
| `dotenv` | Reads a `.env` file and loads its values as environment variables |
| `pnpm` | The package manager used to install everything (listed as a dependency to lock its version) |

---

## The Flow, One More Time — From Start to Finish

```
You type: githate

   bin/hater.js wakes up
        │
        ├── Did you type a command like "scan"?
        │       └── Yes → find the right file in lib/commands/ and run it
        │
        └── Did you type nothing?
                └── Open the REPL (the interactive chat-like shell)

Inside a command (e.g. scan):
   → auth.js gives you the Octokit "phone"
   → checker.js asks GitHub for your followers and compares to the saved list
   → store.js reads the old list and saves the new one
   → display.js prints the results prettily

Background daemon (githate start):
   → daemon.js spawns a copy of itself as a hidden background process
   → That process runs watch.js forever
   → watch.js calls checker.js on repeat
   → When something changes: notify.js fires a desktop notification
   → store.js is updated so the main CLI sees the changes too
```

The **store** (`store.js` / `conf`) is the glue between everything. The daemon and the CLI are two separate processes. They don't talk to each other directly. They both just read and write the same file on disk, and that's how they share information.

---

## Known Bug Worth Noting

In `scan.js`, the code uses `boxen` to render unfollowers — but never imports it at the top of the file. So if someone unfollows you, it'll crash with a `ReferenceError: boxen is not defined`. Just a missing `import boxen from 'boxen'` at the top of the file.

---

That's the whole thing. A GitHub follower tracker, built with Node.js, that can run as a one-shot command, an interactive shell, or a silent background daemon.
