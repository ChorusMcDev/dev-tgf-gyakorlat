# dev-tgf-gyakorlat — `klechino`

Developer Application — **Practical Phase (Phase 3)**.

Hey **klechino**! This is your personalized practical assignment. Stay on the `klechino` branch — do not switch back to `main`, everything happens right here.

---

## 0. Why did you get this specific task?

Based on your application and our voice chat, we put together a task that matches your interests and what you enjoy building:

- **PvP & Duels (`craftypvpduels`):** Since you mentioned you love making PvP plugins and your favorite project was a duel core with a kit editor and queue system, we want you to build a clean, focused miniature version of that: **`TgfDuels`**.
- **Paper API & Basic Java:** Pure Paper API in Java 21 — no NMS, no BungeeCord, and no overly complex frameworks required.
- **MySQL & SQLite:** You mentioned experience with both, so the plugin will store player stats and kit layouts using SQLite and MySQL.
- **Dupe Prevention & Server Performance (TPS):** On your form you gave solid answers on handling dupes and keeping TPS high. This task lets you put that into practice by keeping database calls off the main thread and making sure players can't dupe items through the kit editor or duel inventory restoration.
- **Git Workflow:** You'll use this branch to show your progress step by step through clean commits.

---

## 1. The Assignment: `TgfDuels` — 1v1 Duel Queue & Kit Editor Plugin

Build a **Paper** plugin where players can customize their personal kit layout in a GUI Kit Editor, join a 1v1 matchmaking queue, fight in an isolated duel without losing or duplicating their survival inventory, and track their Win/Loss/ELO stats in a database.

### Target Environment

- **Paper 1.21.4**, **Java 21**
- Build system: **Gradle** or **Maven** (your choice, but `./gradlew build` or `mvn package` must compile cleanly out of the box)
- External libraries are allowed (e.g. HikariCP), but **every dependency must be justified** in `DECISIONS.md`

### Features to Implement

**1. `/duel editor` — Kit Layout Editor (GUI)**
- Opens a GUI where the player can rearrange the default PvP kit items (e.g. sword, golden apples, potions, arrows, bow) to fit their preferred hotbar/inventory slot layout, then click a **Save** button.
- Players **cannot** bring their own items into the editor GUI, and **cannot** take editor items out into their real inventory.
- Saving persists the custom slot layout to the database (`sqlite` or `mysql`) asynchronously so it survives server restarts.
- Includes a **Reset to Default** button in the GUI.

**2. `/duel queue` & `/duel leave` — Matchmaking Queue**
- `/duel queue` puts the player into a 1v1 waiting queue (or removes them via `/duel leave`).
- A player cannot join the queue if they are already in the queue, already in a duel, currently have the Kit Editor open, or are dead/sleeping.
- When 2 players are in the queue, a match starts automatically:
  1. Both players' current survival state (**inventory, armor, offhand, health, hunger, XP, active potion effects, and original location**) is safely snapshotted.
  2. Their inventories/effects are cleared, they are healed/fed, and they are teleported to two configured arena spawn points (`arena.spawn1`, `arena.spawn2` from `config.yml`).
  3. Each player receives the default PvP kit arranged according to their saved **Kit Editor layout** (or the default layout if they never saved one).
  4. A 3-second countdown runs before combat is enabled (movement/damage before countdown finishes is blocked).

**3. Duel Lifecycle & Clean Restoration**
- The duel ends when one player dies (or would take fatal damage — you may cancel fatal damage or handle `PlayerDeathEvent` cleanly without dropping items on the ground), or when a player disconnects (`PlayerQuitEvent`).
- During an active duel:
  - Players **cannot drop items** on the ground that survive the duel, **cannot open storage containers** (chests, ender chests, barrels, shulkers), and **cannot run unrelated teleport/inventory commands**.
- When the duel ends:
  - Any arena item drops are cleaned up.
  - Both players have their duel kit wiped and their **exact pre-duel survival inventory, armor, offhand, health, hunger, XP, potion effects, and original location restored**.
  - Win, loss, and ELO changes (+25 / -20, minimum 0) are saved asynchronously to the database.

**4. `/duel stats [player]` & `/duel setspawn <1|2>`**
- `/duel stats [player]` shows Wins, Losses, Current Winstreak, and ELO from the database.
- `/duel setspawn 1` and `/duel setspawn 2` (admin permission `tgfduels.admin`) save the arena spawn locations to `config.yml`.

**5. Database Layer**
- Selectable in `config.yml`: `sqlite` or `mysql`. **The exact same plugin jar must work with both.**
- Tables are created automatically on startup (`CREATE TABLE IF NOT EXISTS ...`).
- All kit layouts and player stats persist across server restarts.

---

## 2. Core Grading Criteria — Where Your Points Come From

The command setup is the easy part. Your score is mainly decided by these three areas:

### A) Zero Item Duplication & State Corruption

PvP / Duel plugins are a common source of item dupes on SMP servers. You must have **working protection** against these 5 scenarios:

1. **Kit Editor Smuggling:** A player tries to shift-click, number-key swap (`HOTBAR_SWAP`), offhand-swap (`F`), drag, drop (`Q`), or close the GUI while holding an item on their cursor (`event.getCursor()`) to smuggle kit items into their survival inventory — or smuggle survival items into the kit editor.
2. **Pre-Duel / Post-Duel Inventory Dupe:** A player dies in a duel, disconnects mid-duel, or the server shuts down (`onDisable`) during a duel. Result: the player never keeps the duel kit items in survival, never drops the duel kit for someone else to pick up, and never loses their original survival inventory.
3. **Same-Tick Double Queue / Quit Race:** Two players match in the exact same tick one of them runs `/duel leave`, opens `/duel editor`, or disconnects — or a player dies in the exact same tick their opponent disconnects. Only one match termination runs, stats are updated **once** (not twice), and inventories are restored **once** (not duplicated).
4. **Death Screen / Respawn Desync:** If you allow actual player death instead of cancelling fatal damage, restoring their survival inventory before they respawn will wipe their items or duplicate them on the ground. Handle fatal damage or respawn restoration cleanly.
5. **Arena Cross-Contamination:** If an arena is currently occupied, another pair in the queue must either wait until the arena is free or use separate arena instances/spawns so players outside the duel cannot pick up items or hit dueling players.

In `DECISIONS.md`, briefly explain how your code prevents each of these 5 scenarios.

### B) Never Block the Main Server Thread

- **No** database queries (`SELECT`, `INSERT`, `UPDATE`) may run on the main server thread.
- Conversely, **Bukkit API calls** (teleporting players, modifying inventories, opening GUIs, changing health/effects) **must only run on the main thread**.
- Properly bridge async DB calls and main-thread Bukkit operations (e.g. `CompletableFuture` or `BukkitScheduler`).

### C) SQL Injection Prevention

Every SQL query must use parameterized `PreparedStatement`s (`?` placeholders). Do not use string concatenation in SQL queries.

---

## 3. Git Requirements

- Work directly on this **`klechino` branch** and push your commits here.
- Make sure **all source files (`src/main/java/...` and `src/main/resources/...`) are committed and pushed.**
- **Minimum 8 meaningful commits** following your progress as you build the plugin.
  - A single `"done"` / `"final"` commit at the end containing the whole project is not accepted — we want to see your development workflow.
  - One commit = one logical unit of work.
- Commit messages in **English**, imperative mood:
  - `add sqlite and mysql storage provider`
  - `implement kit editor gui and click guards`
  - `fix inventory restore on player quit during duel`
- **`.gitignore` is required.** Do NOT commit: `build/`, `target/`, `*.jar`, `.idea/`, `.gradle/`, `run/`, `logs/`, `*.db`, or real credentials.

---

## 4. Required Repository Structure

```text
├── .gitignore
├── README.md              ← DO NOT overwrite or delete this file!
├── SETUP.md               ← written by you: how to build, configure, and test
├── DECISIONS.md           ← written by you: architectural decisions (see below)
├── build.gradle(.kts) or pom.xml
└── src/main/java/...      ← your Java source code
    └── resources/
        ├── plugin.yml
        └── config.yml
```

### `SETUP.md`
Short and practical: exact build command, where the output `.jar` is generated, how to configure `config.yml`, and step-by-step commands to set the arena spawns and test a duel.

### `DECISIONS.md`
This is the most important written part. It should cover:

1. **How did you solve each of the 5 dupe/corruption scenarios in Section 2/A?** (2–3 sentences per scenario).
2. **How do you make sure no DB calls run on the main thread and no Bukkit API calls run on async threads?**
3. **What did you leave out, and why?** If you ran out of time for something, write it down honestly. Documenting a known limitation is much better than hiding a bug.
4. **AI Usage.** If you used ChatGPT, Copilot, or any AI tool, mention what you used it for. **Using AI is not banned and does not lose points** — we use it too. However, **during the short voice defense on Discord, you must be able to explain every line of code you submitted**.

---

## 5. What We Are NOT Asking For

To save your time, do NOT build any of these:

- Bot AI / NPC citizens / pathfinding bots (player-vs-player 1v1 only)
- NMS / packets / reflection hacks
- BungeeCord / Velocity cross-server queues
- Multi-arena world cloning or schematic pasting (1 configured arena where the queue waits until it's free is completely fine!)
- Party duels, 2v2s, tournaments, or leaderboards GUI

A **clean, working, dupe-free** 1v1 duel & kit editor plugin is worth much more to us than a huge plugin with bugs.

---

## 6. Grading Breakdown

| Category | Weight |
| --- | --- |
| Dupe prevention, inventory restoration & state safety (2/A) | **35%** |
| Async DB I/O & Main-thread Bukkit safety (2/B) | **20%** |
| Code quality, structure, and readability | 15% |
| Database layer: SQLite + MySQL, `PreparedStatement` (2/C) | 10% |
| Git workflow & commit history | 10% |
| `DECISIONS.md` and `SETUP.md` quality | 10% |

**Immediate rejection:** copied plugin code without attribution · a single dump-commit at the end · inability to explain your own code on the Discord call.

---

## 7. Deadline & Submission

- **Deadline: 10 days from the creation of this branch.** You will get the exact date on Discord.
- **Submission:** push your work to this `klechino` branch, then ping us on Discord when you're ready.
- **Live Defense:** after submission, we'll do a ~15-minute voice call with screen sharing on Discord to walk through a few questions about your code and test a couple of scenarios live.
- If you need extra time, **let us know in advance**. Asking for an extension early is fine; disappearing silently is not.

---

## 8. If You Get Stuck

Ask on Discord! Asking good questions on Discord **does not cost you points** — that's how normal teamwork works with us too. We are happy to help with:

- Git commands / branch questions
- Paper API or Gradle / Maven setup issues
- Clarifying anything in this task description

What we won't solve for you: the state management and dupe-prevention logic itself — that's the core of the task.

Good luck and have fun coding!
