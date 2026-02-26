# Technical Reference — `githate`

Full technical breakdown of architecture, data flow, module contracts, and implementation details.

---

## Package Identity

```json
{
  "name": "githate",
  "version": "1.3.3",
  "type": "module",
  "bin": { "githate": "bin/hater.js" }
}
```

- Pure **ESM** (`"type": "module"`) — all files use `import`/`export`, no `require()`
- Exposed as `githate` binary via the `bin` field in `package.json`
- No build step — Node.js runs the source files directly

---

## Directory Map

```
hater/
├── bin/
│   └── hater.js                   # CLI entry point — Commander setup + default REPL
├── lib/
│   ├── commands/
│   │   ├── clear-cache.js         # Wipe conf store after confirmation
│   │   ├── daemon-control.js      # start/stop/status wrappers over daemon.js
│   │   ├── followers.js           # List authenticated user's followers (paginated)
│   │   ├── following.js           # List users the authenticated user follows (paginated)
│   │   ├── haters.js              # Read + display historical unfollowers from store
│   │   ├── login.js               # GitHub OAuth device flow → store token
│   │   ├── logout.js              # Delete token + username from store
│   │   ├── repl.js                # readline REPL shell — default interactive mode
│   │   ├── scan.js                # One-shot follower diff + persist changes
│   │   └── watch.js               # Infinite polling loop — used by daemon process
│   ├── ui/
│   │   ├── art.js                 # figlet ASCII art banners
│   │   ├── display.js             # Terminal output helpers (chalk, boxen, clack)
│   │   └── repl-text.js           # Static text blocks for REPL (welcome, help, why)
│   └── utils/
│       ├── auth.js                # Octokit singleton + device flow auth
│       ├── checker.js             # Follower diff computation (pure function)
│       ├── daemon.js              # Spawn/kill/probe background Node process
│       ├── notify.js              # OS desktop notifications via node-notifier
│       └── store.js               # Persistent config via conf (token, followers, haters)
└── package.json
```

---

## Entry Point: `bin/hater.js`

### Responsibilities
1. Bootstrap: loads `.env` via `dotenv/config`, reads `package.json` for version
2. Commander program setup with `.name()`, `.version()`, `.description()`
3. Command registration — every command registered twice (with and without `/` prefix alias)
4. Default behavior: if `process.argv.length < 3`, bypass Commander entirely and call `startRepl()`
5. Hidden `__daemon` command: `hater.js __daemon --interval <n>` → runs `watch.js`

### Command → Module Map

| CLI command | Module |
|---|---|
| `login` | `lib/commands/login.js` |
| `logout` | `lib/commands/logout.js` |
| `scan` | `lib/commands/scan.js` |
| `haters` | `lib/commands/haters.js` |
| `followers` | `lib/commands/followers.js` |
| `following` | `lib/commands/following.js` |
| `follow <username>` | `lib/commands/follow.js` *(missing)* |
| `unfollow <username>` | `lib/commands/unfollow.js` *(missing)* |
| `start [interval]` | `lib/commands/daemon-control.js → start()` |
| `stop` | `lib/commands/daemon-control.js → stop()` |
| `status` | `lib/commands/daemon-control.js → status()` |
| `clearcache` | `lib/commands/clear-cache.js` |
| `__daemon` | `lib/commands/watch.js` *(hidden)* |

### Default REPL Invocation
```js
if (process.argv.length < 3) {
  const { startRepl } = await import('./lib/commands/repl.js');
  await startRepl(true); // true = show welcome screen
  process.exit(0);
}
```

---

## `lib/utils/store.js`

### Storage Backend
Uses the `conf` npm package with project name `"githate-cli"`. `conf` stores data in a platform-appropriate config directory:
- **macOS**: `~/Library/Preferences/githate-cli/`
- **Linux**: `~/.config/githate-cli/`
- **Windows**: `%APPDATA%/githate-cli/`

### Schema

| Key | Type | Default | Description |
|---|---|---|---|
| `token` | `string` | — | GitHub OAuth access token |
| `username` | `string` | — | Authenticated GitHub username |
| `followers` | `string[]` | `[]` | Last-known follower login array |
| `lastCheck` | `string` | — | ISO 8601 timestamp of last scan |
| `haters` | `{login: string, detectedAt: string}[]` | `[]` | Cumulative unfollower log |

### Exported API

```js
getStoredToken()         → string | undefined
setStoredToken(t)        → void
deleteStoredToken()      → void

getStoredUsername()      → string | undefined
setStoredUsername(u)     → void
deleteStoredUsername()   → void

getStoredFollowers()     → string[]
setStoredFollowers(arr)  → void

getLastCheck()           → string | undefined
setLastCheck(iso)        → void

getStoredHaters()        → {login, detectedAt}[]
addHaters(logins[])      → void   // appends with current new Date().toISOString()

clearStore()             → void   // conf.clear() — wipes everything
```

---

## `lib/utils/auth.js`

### Octokit Singleton

```js
let octokitInstance = null;

export async function getOctokit() {
  if (!octokitInstance) {
    const token = getStoredToken();
    octokitInstance = new Octokit({ auth: token }); // unauthenticated if no token
  }
  return octokitInstance;
}
```

Singleton is module-scoped. Creating a new Octokit is cheap but avoids redundant instantiation across multiple command calls in the REPL.

### Device Flow Authentication

```js
export async function authenticateWithDeviceFlow()
```

1. Instantiates `createOAuthDeviceAuth` from `@octokit/auth-oauth-device` with:
   - `clientType: "oauth-app"`
   - `clientId`: `process.env.GITHUB_CLIENT_ID ?? "Ov23liyj3Hj0HTe76fD9"`
   - `scopes: ["read:user", "user:follow"]`
   - `onVerification`: callback that copies the device code to clipboard via `clipboardy` and prints the GitHub verification URL
2. Calls `auth({ type: "oauth" })` — this blocks until the user authorizes in browser
3. Returns `token` from the result

### Token Verification

```js
export async function verifyToken(token)
```

Creates a one-off `new Octokit({ auth: token })` and calls `octokit.rest.users.getAuthenticated()`. Returns `{ login }` on success.

---

## `lib/utils/checker.js`

### Contract

```js
export async function getFollowerState(octokit, targetUser)
```

**Input**: authenticated Octokit instance, GitHub username string

**Process**:
1. Paginates `octokit.paginate(octokit.rest.users.listFollowersForUser, { username: targetUser, per_page: 100 })`
2. Maps to array of login strings
3. Reads `getStoredFollowers()` from store
4. Computes:
   - `newFollowers` = `currentFollowerLogins.filter(l => !storedFollowers.includes(l))`
   - `unfollowers` = `storedFollowers.filter(l => !currentFollowerLogins.includes(l))`
   - `isFirstRun` = `storedFollowers.length === 0`

**Returns**:
```js
{
  currentFollowerLogins: string[],
  storedFollowers: string[],
  newFollowers: string[],
  unfollowers: string[],
  lastCheck: string | undefined,   // from store
  isFirstRun: boolean
}
```

**Note**: Pure in terms of side effects — does not write to store. Callers (`scan.js`, `watch.js`) are responsible for persisting results.

---

## `lib/commands/scan.js`

### Execution Flow

```
getOctokit()
  → resolve targetUser (store.getStoredUsername ?? octokit.users.getAuthenticated ?? prompt)
  → getFollowerState(octokit, targetUser)
  → if isFirstRun:
      setStoredFollowers(currentFollowerLogins)
      print baseline count
    else:
      for each newFollower: print green "+", 500ms pause
      for each unfollower: print red "✕", 500ms pause, addHaters([login])
      setStoredFollowers(currentFollowerLogins)
      setLastCheck(new Date().toISOString())
```

### Error Handling
- `status 403` → rate limit message
- `status 404` → user not found message
- Both exit gracefully without crashing

### Known Bug
`scan.js` renders unfollower output using `boxen` but does not import it:
```js
// Missing: import boxen from 'boxen';
// ...
console.log(boxen(...)); // ReferenceError at runtime
```

---

## `lib/commands/watch.js`

### Execution Flow

```js
async function checkLoop(octokit, targetUser, intervalMinutes) {
  while (true) {
    // same diff logic as scan.js
    // on changes: sendNotification() for new followers and unfollowers
    // update store
    await new Promise(r => setTimeout(r, intervalMinutes * 60 * 1000));
  }
}
```

Called immediately once, then loops with `setTimeout`. Runs in the daemon's detached process — no terminal output (stdout is ignored by the spawn configuration).

Desktop notifications are sent via `notify.js` for:
- New followers: `"New Follower! 🎉"` title
- Unfollowers: `"Someone unfollowed you 😤"` title

---

## `lib/utils/daemon.js`

### Process Spawning

```js
export async function startDaemon(intervalMinutes = 120) {
  const existing = store.get('daemonPid');
  if (existing) { /* check if alive, warn if so */ }

  const child = spawn(
    process.execPath,              // path to current node binary
    ['bin/hater.js', '__daemon', '--interval', String(intervalMinutes)],
    {
      detached: true,
      stdio: 'ignore',
      env: { ...process.env, GITHATE_DAEMON: 'true' }
    }
  );
  child.unref();                   // allow parent process to exit independently
  store.set('daemonPid', child.pid);
}
```

`child.unref()` is critical — it tells Node not to wait for the child process before exiting. Without this, `githate start` would block forever.

### Process Probing

```js
export function getDaemonStatus() {
  const pid = store.get('daemonPid');
  if (!pid) return { running: false };
  try {
    process.kill(pid, 0);  // signal 0 = probe only, does not kill
    return { running: true, pid };
  } catch {
    return { running: false, stale: true }; // PID saved but process is dead
  }
}
```

`kill(pid, 0)` is a POSIX convention: sending signal 0 doesn't kill the process but throws if the process doesn't exist or you don't have permission to signal it.

### Stopping

```js
export function stopDaemon() {
  const pid = store.get('daemonPid');
  process.kill(pid, 'SIGTERM');
  store.delete('daemonPid');
}
```

---

## `lib/commands/repl.js`

### Architecture

Uses Node's built-in `readline.createInterface`:
```js
const rl = readline.createInterface({
  input: process.stdin,
  output: process.stdout,
  prompt: 'githate > '
});
```

### Input Handling Logic

```
Line received
  │
  ├── Empty? → rl.prompt() (no-op)
  │
  ├── Greeting (hi/hey/sup/etc.)? → print slang response
  │
  ├── Starts with "/"? → handle command
  │       ├── /help → displayREPLHelp()
  │       ├── /why  → displayWhy()
  │       ├── /repo → print GitHub link
  │       ├── /haters, /clearcache, /clear, /quit, /exit → inline handlers
  │       └── everything else → dispatchCommand(cmd, args)
  │
  └── Bare text (no "/")? → hint: "prefix with /"
```

### stdin Conflict Handling

`@clack/prompts` takes over `process.stdin` when running interactive prompts. This conflicts with `readline`. Before calling any clack-based command:

```js
rl.pause();
await dispatchCommand(...);
rl.resume();
rl.prompt();
```

If clack closes stdin unexpectedly, the `close` event fires on `rl`. The REPL detects this and restarts itself:

```js
rl.on('close', () => {
  if (!isQuitting) {
    startRepl(false); // false = skip welcome screen on restart
  }
});
```

### `dispatchCommand(cmd, args)`

Switch statement covering: `scan`, `login`, `logout`, `followers`, `following`, `follow`, `unfollow`, `start`, `stop`, `status`. Each case dynamically imports and runs the corresponding module. The daemon start/stop commands hardcode `interval = 120`.

---

## `lib/ui/art.js`

```js
import figlet from 'figlet';
import gradient from 'gradient-string';

export function getBanner(text = 'GITHATE') {
  const ascii = figlet.textSync(text, { font: 'ANSI Shadow' });
  return gradient.retro(ascii);
}

export function getHaterReveal(text) {
  const ascii = figlet.textSync(text, { font: 'DOS Rebel' });
  return gradient.passion(ascii);
}
```

`figlet.textSync` is synchronous — it generates the ASCII art string inline. `gradient-string` wraps the entire multi-line string and applies a color gradient character-by-character.

---

## `lib/ui/display.js`

### ANSI Hyperlink

```js
export function link(text, url) {
  return `\x1b]8;;${url}\x1b\\${text}\x1b]8;;\x1b\\`;
}
```

This is the OSC 8 hyperlink escape sequence — supported in iTerm2, GNOME Terminal, Windows Terminal, and others. Renders as a clickable link in supported terminals.

### Spinner

```js
export function createSpinner() {
  return clack.spinner(); // from @clack/prompts
}
// Usage:
const s = createSpinner();
s.start('Loading...');
// ...
s.stop('Done');
```

### `displayError`

Uses `boxen` with `{ borderColor: 'red', borderStyle: 'round', padding: 1 }` for a bordered error box.

---

## Dependency Reference

| Package | Version | Role |
|---|---|---|
| `commander` | `^14.0.3` | CLI argument parsing and subcommand routing |
| `@octokit/rest` | `^20.0.0` | GitHub REST API client with pagination support |
| `@octokit/auth-oauth-device` | `^8.0.3` | GitHub device flow OAuth implementation |
| `conf` | `^15.1.0` | Persistent JSON config storage (platform-aware path) |
| `chalk` | `^5.6.2` | Terminal string styling (colors, bold, dim) |
| `picocolors` | `^1.1.1` | Lightweight color utility used in a few display helpers |
| `boxen` | `^8.0.1` | Draws a box border in the terminal (used in `displayError`) |
| `figlet` | `^1.10.0` | ASCII art text generation from font files |
| `gradient-string` | `^3.0.0` | Multi-color gradient applied to strings |
| `@clack/prompts` | `^1.0.0` | Interactive terminal prompts (spinner, confirm) |
| `clipboardy` | `^5.3.0` | Cross-platform clipboard read/write |
| `node-notifier` | `^10.0.1` | OS desktop notifications (macOS, Windows, Linux) |
| `dotenv` | `^17.3.1` | Loads `.env` file into `process.env` |
| `pnpm` | `^10.29.3` | Package manager version pin |

---

## Inter-Process Communication (Daemon ↔ CLI)

The daemon and CLI are separate OS processes. They share state exclusively through the `conf` store on disk — there is no IPC socket, no shared memory, no event emitter.

```
CLI process                    Daemon process
(githate status)               (node bin/hater.js __daemon)
     │                                   │
     └── reads conf file ←────────────── writes conf file
         (daemonPid,                     (followers, lastCheck,
          followers, haters)              haters, daemonPid)
```

Implications:
- The CLI always sees the latest data — no cache invalidation needed
- The daemon doesn't need to stay alive for data to persist
- No need for a server — `conf` handles file locking via `atomically` (its internal dep)

---

## Authentication Scopes

The OAuth app requests two scopes:
- `read:user` — read user profile and follower lists
- `user:follow` — follow/unfollow users (needed for planned `follow`/`unfollow` commands)

The GitHub OAuth app client ID is `Ov23liyj3Hj0HTe76fD9`. Overridable at runtime via `GITHUB_CLIENT_ID` environment variable.

---

## Known Issues

| Issue | Location | Impact |
|---|---|---|
| `boxen` used but not imported | `scan.js` | `ReferenceError` crash when unfollowers are detected |
| `follow.js` / `unfollow.js` missing | `bin/hater.js` registers these commands | `Error: Cannot find module` if these commands are used |
| `displayHelp()` in `display.js` is defined but never called | `display.js` | Dead code |
| `TIPS` array and `getRandomTip` defined but never exported or called | `display.js` | Dead code |

---

## Data Flow Summary

```
User Input
    │
    ▼
bin/hater.js (Commander)
    │
    ├──[scan]──► scan.js
    │               ├──► auth.js → getOctokit()
    │               ├──► checker.js → getFollowerState()
    │               │       └──► octokit.paginate(listFollowersForUser)
    │               │       └──► store.getStoredFollowers()
    │               └──► store.setStoredFollowers() + addHaters()
    │               └──► display.js (output)
    │
    ├──[login]─► login.js
    │               └──► auth.js → authenticateWithDeviceFlow()
    │               └──► store.setStoredToken() + setStoredUsername()
    │
    ├──[start]─► daemon-control.js → daemon.js
    │               └──► spawn(node bin/hater.js __daemon --interval N)
    │                       └──► watch.js (infinite loop)
    │                               ├──► checker.js
    │                               ├──► store.setStoredFollowers() + addHaters()
    │                               └──► notify.js → OS notification
    │
    └──[default]► repl.js (readline loop)
                    └──► dispatchCommand() → any command above
```
