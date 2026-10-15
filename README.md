# Chain Commands (Fabric, Minecraft 26.2 and 26.3)

Run several commands from one line, with conditions, piping, waiting and loops.

    /chain gamemode creative && time set day && weather clear
    /chain scoreboard players get @s kills | say I have {result} kills
    /chain if execute if entity @s[tag=vip] then say welcome back else say hello end
    /chain say 3 ;; wait 1s ;; say 2 ;; wait 1s ;; say 1 ;; wait 1s ;; say go!

## Contents

1. [Quick start](#1-quick-start)
2. [How a line is read](#2-how-a-line-is-read)
3. [Everything at a glance](#3-everything-at-a-glance)
4. [Options: `strict` and `dry`](#4-options-strict-and-dry)
5. [Operators: `&&` `||` `;;` `|`](#5-operators)
6. [What counts as success or failure](#6-what-counts-as-success-or-failure)
7. [Conditions: `if` `unless` `elif` `else` `end`](#7-conditions)
8. [`compare`](#8-compare)
9. [Built-in commands: `echo` `stop` `wait` `repeat`](#9-built-in-commands)
10. [Passing values between commands (placeholders and piping)](#10-passing-values-between-commands)
11. [Quoting and escaping](#11-quoting-and-escaping)
12. [Limits](#12-limits)
13. [Recipes](#13-recipes)
14. [Error messages](#14-error-messages)
15. [Good to know](#15-good-to-know)
16. [Building the mod](#building-1-what-you-need)

---

## 1. Quick start

1. Install the mod (see "Installing the jar" at the bottom). It is only needed on the server, or in singleplayer.
2. In chat or a command block, type: `/chain say hello && say world`
3. Type `/chain` on its own for a short in-game reminder of the syntax.
4. Not sure how a line will be read? Put `dry` in front: `/chain dry say a && say b`. Nothing runs; it just shows the plan.

`/chain` is a normal command. It works in chat, command blocks, functions, and inside other commands
(`/execute as @a run chain say hi && say there`).

---

## 2. How a line is read

Everything after `/chain` is split into **steps** at the operators (`&&`, `||`, `;;`, `|`). Each step is
either a normal Minecraft command or one of this mod's built-in words (`if`, `wait`, `echo`, ...).
Steps run **strictly left to right**. There is no operator precedence and no grouping with brackets.

    /chain  tp @s 0 100 0  &&  say arrived  ||  say could not teleport
            ^step 1           ^step 2          ^step 3

- A leading `/` on any step is optional: `/chain /say hi && /say there` works.
- Empty steps are ignored (`a && && b` is `a && b`).
- Steps are numbered 1, 2, 3, ... as you typed them. These numbers are what `{out2}` and `{result3}` refer to.
  `/chain dry ...` shows the numbering.
- Every step runs as **you** (the person or command block that typed `/chain`), so Minecraft's normal
  permission checks apply to each command individually. `/chain` can't do anything you couldn't do by hand.

---

## 3. Everything at a glance

**Options** (only at the very start of the line)

| Word | What it does |
|---|---|
| `strict` | Stop the whole chain at the first command that fails |
| `dry` | Show how the line was split, run nothing |

**Operators** (between steps)

    a && b    run b only if a succeeded
    a || b    run b only if a failed
    a ;; b    run b no matter what
    a | b     like &&, and also pipe a's output into b

**Flow control**

| Word | What it does |
|---|---|
| `if <test> [then <command>]` | start a conditional block |
| `unless <test> [then <command>]` | same, but the test must fail |
| `elif <test> [then <command>]` | another test, only tried if nothing ran yet |
| `else [<command>]` | runs if no earlier branch ran |
| `end` | closes the block |
| `compare <a> <op> <b>` | a test for numbers and text: `==  !=  <  <=  >  >=  contains` |

**Built-in commands**

| Command | What it does |
|---|---|
| `echo <text>` | show a message to yourself only |
| `stop [message]` | end the chain, optionally saying why |
| `wait <time>` | pause the chain (`20`, `20t`, `1s`, `0.5s`) |
| `repeat <n> <command>` | run a command up to `n` times |

**Placeholders** (usable inside later commands)

| Placeholder | Value |
|---|---|
| `{out}` | text the previous command printed |
| `{result}` | number the previous command returned |
| `{num}` | first number found in the previous command's text |
| `{out2}` `{result2}` `{num2}` | the same, from step 2 (any step number) |
| `{out:json}` | `{out}` escaped for use inside JSON text |

---

## 4. Options: `strict` and `dry`

Options are plain words at the very start of the line, before the first command. You can use both,
in either order: `/chain strict dry ...`. They are not recognised anywhere else.

### `strict`

Without `strict`, a failed command only affects the operator right after it (see section 5).
With `strict`, **the first failed command ends the whole chain**, even across `;;`:

    /chain strict fill ~ ~ ~ ~5 ~5 ~5 stone ;; setblock ~ ~6 ~ torch ;; say built!

If the `fill` fails, the torch and the message never happen, and you see `Chain stopped: step 1 failed.`

Exceptions: if the very next operator is `||`, the failure is considered handled and the chain continues
(`strict a || b` still runs `b` when `a` fails). A failing `if` condition is just a test and never
triggers `strict`; a failing `compare` used as a step does.

### `dry`

Prints the plan and runs nothing. The numbers are the same ones you use in `{out2}`:

    /chain dry say a && if compare 1 < 2 then say b else say c end

    1. [start] say a
    2. [&&] if compare 1 < 2 then say b
        2+ else say c
        2+ end

Lines marked `2+` are the `else`/`elif`/`end` words that were split off the line above them.
`dry` also reports a line that is too long or empty.

---

## 5. Operators

    a && b    run b only if a succeeded
    a || b    run b only if a failed
    a ;; b    run b no matter what
    a | b     like &&, and also pipe a's output into b

Rules:

- A step that gets **skipped** does not change the "did the last command succeed?" memory. So after a
  failure, every following `&&` step is skipped (the chain "stops"), until a `||` or `;;` step.
- Because of that, `a && b || c` reads as "if a and b worked, done; otherwise run c": a simple if/else
  without any keywords.
- Evaluated left to right: `ok || x && y` runs `y` (`ok` succeeded, `x` was skipped, `y` still sees a success).

| Line | When `a` works | When `a` fails |
|---|---|---|
| `a && b` | b runs | b skipped |
| `a || b` | b skipped | b runs |
| `a ;; b` | b runs | b runs |
| `a && b || c` | b runs (c runs only if b fails) | c runs |
| `a && b && c` | b, c run | b, c skipped |

`|` (pipe) behaves like `&&` for running, and additionally hands a's output to b. See section 10.

---

## 6. What counts as success or failure

A command **succeeds** if it ran and reported success. It **fails** if:

- it reported an error (for example "No entity was found"), or
- it doesn't exist or its arguments are invalid, or
- it never ran, for example an `execute if ...` whose condition is false.

If a command produces several results at once (like `execute as @a run ...`), it counts as a success only
if at least one part succeeded and none failed.

The built-in commands: `echo`, `stop`, `wait` always succeed. `compare` succeeds if the comparison is true.
`repeat` succeeds if every run succeeded (or if `n` is 0).

A command that sends no feedback to you (like `/say`) still succeeds normally. It just leaves `{out}` empty.

---

## 7. Conditions

    if <test> then <command>
    if <test> then <command> else <command> end
    if <test> then <command> elif <test> then <command> else <command> end
    unless <test> then <command> else <command> end

The **test** is any command (it passes if the command succeeds) or a `compare`. Examples:

    if execute if entity @s[tag=vip] then say hi vip
    if execute if score @s kills matches 10.. then say pro
    if compare {result} >= 10 then say big

How it works:

- `unless` is `if` with the test reversed.
- `elif` is only tried when no earlier branch has run. `else` runs when no earlier branch has run.
- The test runs **silently**: you don't see its messages. Its output is still saved, so `{out}` works
  afterwards. If a branch command ran, `{out}` is that command's output instead.
- Without `then`, the steps that follow belong to the branch, until the next `elif`/`else`/`end`:

      /chain if execute if entity @a[tag=ready] ;; say starting ;; time set day ;; else ;; say waiting ;; end

- **Always close a block with `end`** when something follows it. An `if` with no `end` simply runs to the
  end of the chain: everything after it is part of the branch.
- **Nesting:** each `if` needs its own `end`. `if A then if B then X end end`. Use `elif` for chains of
  alternatives; `else if` starts a *new nested* block that needs its own `end`.
- An operator in front of an `if` decides whether the **whole block** is considered at all:
  `fail && if ... else ... end` skips the entire block, including the `else`.
- After `end`, the "last command succeeded" memory resets to success.
- `wait` is allowed inside a branch. It is not allowed as a test.

On a single line, the words `elif`, `else` and `end` (and a nested `if`/`unless` right after `then` or
`else`) are keywords. They only count in a step that **starts with** `if`, `unless`, `elif` or `else`.
If your message needs one of those words, put it on its own `;;` step:

    /chain if execute if entity @s ;; say that is the end ;; end

The words `if`, `unless`, `elif`, `else`, `end`, `echo`, `compare`, `stop`, `wait` and `repeat` are reserved
when they are the **first word of a step**.

---

## 8. `compare`

    compare <a> <operator> <b>

Spaces are required around the operator. `a` and `b` can be several words and can contain placeholders.

| Operator | Works on | True when |
|---|---|---|
| `==` | numbers or text | equal |
| `!=` | numbers or text | not equal |
| `<` | numbers only | a is less than b |
| `<=` | numbers only | a is less than or equal to b |
| `>` | numbers only | a is greater than b |
| `>=` | numbers only | a is greater than or equal to b |
| `contains` | text | a contains b |

- If both sides look like numbers they are compared as numbers. A Minecraft type suffix is fine:
  `20.0f == 20` is true, and so is `5b < 7`.
- Otherwise `==`, `!=` and `contains` compare as text (case-sensitive).
- `<`, `<=`, `>`, `>=` on non-numbers stop the chain with an error message instead of guessing.
- `compare` prints nothing. It just succeeds (true) or fails (false), so it works as an `if` test and
  also with `&&`/`||`:

      /chain scoreboard players get @s kills ;; compare {result} >= 10 && say pro || say rookie

Not available: `startswith`, `endswith`, regex matching, ranges, and combining tests with and/or/not in
one condition. For ranges use `execute if score ... matches 5..10`. For "and", nest ifs; for "or", use
`elif`; for "not", use `unless`.

---

## 9. Built-in commands

### `echo <text>`

Shows `<text>` to you only (nobody else sees it). Placeholders work. Always succeeds.

    /chain scoreboard players get @s kills ;; echo you have {result} kills

### `stop [message]`

Ends the chain immediately; later steps never run. With a message, it is shown to you.
Pair it with `||` for a conditional abort:

    /chain give @s diamond || stop no room in your inventory ;; say enjoy!

A `stop` inside a branch that isn't running does nothing.

### `wait <time>`

Pauses this chain, then continues with the next step. The game and other chains keep running normally.

| You write | Meaning |
|---|---|
| `wait 20` or `wait 20t` | 20 ticks (1 second) |
| `wait 1s` | 1 second (20 ticks) |
| `wait 0.5s` | half a second (10 ticks) |
| `wait 0` | no pause |

- The maximum is 72000 ticks (1 hour); bigger values are cut down to that.
- The time can be a placeholder: `wait {result}`.
- `wait` works as its own step or inside an `if` branch. It can't be used inside `repeat`
  or as an `if` test.
- A waiting chain is forgotten if the server stops or restarts.
- Anything unreadable (`wait soon`) stops the chain with `Usage: wait <ticks> or wait <seconds>s`.

### `repeat <n> <command>`

Runs `<command>` up to `n` times, one after another.

    /chain repeat 5 summon minecraft:lightning_bolt ~ ~ ~

- `n` is limited to 0-100 (bigger is cut down to 100). `n` can be a placeholder: `repeat {result} say hi`.
- It **stops at the first failure**.
- Placeholders in the command are worked out again on every round, so `{out}` is the previous round's output.
- Each round counts towards the 1000 commands per chain limit.
- The command can't be `wait`, `repeat`, or an `if`/`elif`/`else`/`end`. To repeat with pauses,
  write the steps out or use `wait` between several `repeat` steps.

---

## 10. Passing values between commands

Every command that runs has its **output** (the text it showed you, joined into one line) and its
**result** (the number it returned, such as the score `scoreboard players get` read) remembered.
You can drop these into any later step with placeholders.

| Placeholder | Value |
|---|---|
| `{out}` | the previous command's output text, for example `Steve has 5 [kills]` |
| `{result}` | the previous command's numeric result, for example `5` (0 if there is none) |
| `{num}` | the first number found in the output text, for example `20.0` from `...entity data: 20.0f`; uses `{result}` if the text has no number |
| `{out2}` `{result2}` `{num2}` | the same values, taken from step number 2 (use any step number up to 9999) |
| `{out:json}` `{out2:json}` | the text with `"` and `\` escaped, for safe use inside JSON strings |

Details:

- "Previous" means the **last command that actually ran** (skipped steps don't count).
- A numbered placeholder for a step that hasn't run or was skipped is empty (`{out5}` becomes nothing).
- Anything that isn't one of the above is left alone, so `{Tags:["x"]}`, `{}` and other NBT/JSON are safe.
- Placeholders work in normal commands, `echo`, `compare`, `stop`, `wait`, `repeat`, and `if` tests.
  They are never substituted into the keyword itself.
- Output from `echo` is its text, from `compare` is `true` or `false`, from `wait` is empty.
- Commands that message other players instead of you (`/say`, `/tellraw @a`) leave `{out}` empty.
- The result of `repeat` is its last round, and the result of an `if` is the branch command that ran
  (or the test, if no branch command ran).

### Piping with `|`

`a | b` runs `b` only if `a` succeeded (like `&&`), and gives `b` the output of `a`:

- If `b` contains a placeholder, the output goes exactly there.
- If `b` has **no** placeholder, the output is added to the **end** of `b`.

      /chain scoreboard players get @s kills | say kills so far:
      /chain data get entity @s Health | say health is {num}
      /chain data get entity @s CustomName | tellraw @a {"text":"{out:json}"}

### Who sees what

| Situation | Do you see the command's own messages? |
|---|---|
| A normal command | Yes |
| A command followed by `|` | No, unless it failed (so you can see why the pipe broke) |
| An `if`/`unless`/`elif` test | Never |
| `compare` | Never (it only succeeds or fails) |
| `echo` | Yes (not when followed by `|`) |

---

## 11. Quoting and escaping

Operators are recognised only **outside** of:

- `"double quotes"`
- square brackets `[ ... ]` and curly braces `{ ... }` (so NBT, JSON and selectors are safe)

    /chain say "fish && chips" && say done              (the first && is kept as text)
    /chain tellraw @a {"text":"a || b"} ;; say ok       (the || inside the JSON is kept)
    /chain data modify storage x y set value [I; 1,2,3] ;; say ok

To write an operator as plain text outside quotes or brackets, put a backslash in front of it:

    \&&    \||    \;;    \|

    /chain say red \&& blue          says "red && blue"
    /chain say left \| right         says "left | right"

Notes:

- Single quotes `'` are **not** special (apostrophes in chat messages are fine).
- If your quotes or brackets don't balance (for example `say 5" tall && say two`), the mod falls back to
  reading the line more simply, so a stray `[` or `"` won't swallow the rest of your chain.
- A lone `|` splits the line. If a chat message has a `|`, quote it or write `\|`.

---

## 12. Limits

| Limit | Value |
|---|---|
| Steps in one chain | 128 (the `else`/`elif`/`end` words of an if-line each count as a step) |
| Commands actually run per chain | 1000 |
| `repeat` count | 100 |
| `wait` | 1 hour (72000 ticks) |
| Chains waiting at the same time (whole server) | 500 |
| Chains that can start in one tick (whole server) | 256 (stops runaway loops) |

When a limit is hit, the chain stops and tells you which one.

---

## 13. Recipes

**Set up a world quickly**

    /chain gamemode creative && time set day && weather clear

**Fall back if something fails**

    /chain tp @s 100 70 100 || say that spot isn't available

**Only continue if every step worked**

    /chain give @s diamond_pickaxe && give @s torch 64 && say kit ready

**Abort cleanly, whatever happens afterwards**

    /chain strict clone ~ ~ ~ ~9 ~9 ~9 ~20 ~ ~ ;; fill ~ ~ ~ ~9 ~9 ~9 air ;; say moved

**If / else on a score**

    /chain scoreboard players get @s kills ;; if compare {result} >= 10 then say pro elif compare {result} >= 5 then say good else say rookie end

**Use a Minecraft test as the condition**

    /chain if execute if entity @e[type=zombie,distance=..10] then say run! else say all clear end

**Countdown**

    /chain say 3 ;; wait 1s ;; say 2 ;; wait 1s ;; say 1 ;; wait 1s ;; say go!

**Show a value from earlier**

    /chain scoreboard players get @s kills ;; scoreboard players get @s deaths ;; say kills {result1}, deaths {result2}

**Pipe a value into a command**

    /chain data get entity @s Health | say my health is {num}

**Repeat**

    /chain repeat 5 summon minecraft:lightning_bolt ~ ~ ~

**Run for every player**

    /execute as @a run chain effect give @s speed 30 && say speed boost!

**Preview first**

    /chain dry give @s diamond || stop full ;; say done

---

## 14. Error messages

| Message | Meaning / fix |
|---|---|
| `Nothing to run. Try /chain with no arguments for help.` | The line was empty |
| `Too many steps (max 128).` | Split it into several chains |
| `Too many chains started this tick; is something looping?` | A function or command block keeps starting chains |
| `Chain stopped: step N failed.` | `strict` mode; step N is the one that failed |
| `'else' without a matching 'if'.` (also `elif`, `end`) | A keyword with no open `if` before it |
| `Missing condition after 'if'.` | You wrote `if` with nothing to test |
| `'wait' can't be used as a condition.` | Put `wait` in its own step instead |
| `Usage: compare <a> <== != < <= > >= contains> <b>` | No valid operator found, or spaces missing around it |
| `'<' needs two numbers, got 'x' and 'y'` | `<`, `<=`, `>`, `>=` only work on numbers |
| `Usage: wait <ticks> or wait <seconds>s` | Unreadable time |
| `Too many chains are waiting right now.` | 500 waiting chains is the cap |
| `Usage: repeat <count> <command>` | Count missing or not a number, or no command |
| `'repeat' can't contain 'wait'.` (also `if` etc.) | See the `repeat` section |
| `Command limit reached (1000).` | The chain ran too many commands |
| `Chain crashed: ...` | A bug. Please report it with the line you typed |

Messages from the commands themselves (such as "Unknown command") appear as usual. Failures are shown
in red.

---

## 15. Good to know

- **Timing:** `/chain` queues the chain to start right after the current command finishes, in the same
  tick. This is what lets each command's result be checked before the next one starts.
- **What `/chain` itself returns:** the number of steps it read. It can't report on the finished chain,
  because a chain with `wait` may still be running. This matters if you use `execute store`.
- **Tab completion** doesn't work inside the chained part (Minecraft sees it as one block of text).
- **Command blocks and functions:** work the same way. Commands run as the command block or entity
  that ran `/chain`, and feedback may be silenced the same way it normally is.
- **Restarts:** chains that are waiting are forgotten when the server stops.
- **Strict vs. operators:** `strict` is "stop on first failure"; `&&` is "skip the next step on failure".
  Use whichever fits. They can be combined.

---

## Building: 1. What you need

| Tool | Version | Notes |
|------|---------|-------|
| JDK  | **25** | Minecraft 26.x requires Java 25. Check with `java -version`. |
| Internet | - | First build downloads Gradle, Loom, Minecraft and Fabric API. |

You do **not** need to install Gradle. The included wrapper (`gradlew` / `gradlew.bat`,
Gradle 9.7.1) downloads the right version itself.

---

## Building: 2. How the two versions work

Minecraft 26.2 and 26.3 are separate game versions, and Fabric ships a separate Fabric API
build for each. The mod's own code is shared; only the version numbers change.

    chain-commands/
    |- versions/
    |  |- 26.2.properties   <- minecraft_version + fabric_api_version for 26.2
    |  `- 26.3.properties   <- same for 26.3
    |- gradle.properties    <- shared settings (loader, loom, mod version, default target)
    |- build.gradle         <- reads versions/<target>.properties
    `- src/main/...         <- the mod code (shared by both)

You choose the target with `-Pmc=<version>`. That tells `build.gradle` to load
`versions/<version>.properties`, compile against that Minecraft + Fabric API, and name the jar
`chain-commands-<mod_version>+<mc>.jar`. The generated `fabric.mod.json` also pins the jar to
that Minecraft version (`~26.2` means 26.2.x only), so a 26.2 jar will refuse to load on 26.3
instead of crashing.

If you leave off `-Pmc`, it uses `default_mc` from `gradle.properties` (currently 26.3).

---

## Building: 3. Building

### Build one version

    # Linux / macOS
    ./gradlew build -Pmc=26.2
    ./gradlew build -Pmc=26.3

    # Windows
    gradlew.bat build -Pmc=26.2
    gradlew.bat build -Pmc=26.3

The jar to use appears in `build/libs/`:

    chain-commands-1.0.0+26.2.jar        <- install this
    chain-commands-1.0.0+26.2-sources.jar  (source code, don't install)

Run `./gradlew clean` between versions if you want `build/libs` to hold only one version.

### Build both at once

    ./build-all.sh        # Linux / macOS
    build-all.bat         # Windows

This loops over every file in `versions/`, builds each, and copies the jars into `dist/`:

    dist/chain-commands-1.0.0+26.2.jar
    dist/chain-commands-1.0.0+26.3.jar

(On Linux/macOS, if `gradlew` says "permission denied": `chmod +x gradlew build-all.sh`.)

---

## Building: 4. Testing

Loom adds run tasks, and they also respect `-Pmc`:

    ./gradlew runClient -Pmc=26.2       # launch a dev client on 26.2
    ./gradlew runServer -Pmc=26.3       # launch a dev server on 26.3 (accept run/eula.txt first)

In-game, try `/chain say one && say two && say three`.

## Building: 5. Installing the jar

For each Minecraft version you want to play:
1. Install Fabric Loader (0.19.3 or newer) for that Minecraft version.
2. Put the matching Fabric API jar in `mods/`.
3. Put the matching `chain-commands-...+<mc>.jar` in `mods/`.

The mod is needed on the server (or in singleplayer). Clients joining a server don't need it.

---

## Building: 6. Updating version numbers

Fabric API gets frequent updates, so the numbers in `versions/*.properties` will go stale.
Look up current ones at https://fabricmc.net/develop (pick your Minecraft version) and edit:

- `versions/26.2.properties` -> `fabric_api_version`
- `versions/26.3.properties` -> `fabric_api_version`
- `gradle.properties` -> `loader_version`, `loom_version` (shared by both)

Current values: Fabric API 0.161.0+26.2 and 0.162.0+26.3, Loader 0.19.5, Loom 1.18-SNAPSHOT.

## Building: 7. Adding another version later (e.g. 26.4)

1. Create `versions/26.4.properties`:

       minecraft_version=26.4
       fabric_api_version=<from fabricmc.net/develop>

2. Build with `./gradlew build -Pmc=26.4`. `build-all.sh` picks it up automatically.

## Building: 8. If the code differs between versions

Minecraft's internals sometimes change between releases. If something compiles on 26.2 but not
26.3 (or the reverse), put the version-specific class in its own folder:

    src/mc26.2/java/dev/chaincmd/...    <- only compiled when -Pmc=26.2
    src/mc26.3/java/dev/chaincmd/...    <- only compiled when -Pmc=26.3

Give both versions a class with the same name and methods, and call it from the shared code.
`build.gradle` adds the matching folder to the build automatically.

---

## Building: 9. Troubleshooting

- **"Unsupported class file major version" / Java errors**: you're not running JDK 25.
  Set `JAVA_HOME` to a JDK 25 install.
- **Plugin `net.fabricmc.fabric-loom` not found**: `settings.gradle` must keep the Fabric Maven
  repository (it does). Loom is a `-SNAPSHOT`, so you need internet access to Fabric's Maven.
- **Cannot resolve fabric-api**: the version in `versions/<mc>.properties` doesn't exist.
  Check the exact string (including the `+26.x` suffix) at fabricmc.net/develop.
- **"Unknown Minecraft target"**: the `-Pmc=` value needs a matching `versions/<value>.properties`.
- **Compile error in `ChainRun.java`**: this is the one Minecraft method the mod
  relies on that I couldn't check against the game's source. If it was renamed, fix the call in
  `ChainCommands.run(...)`; the error message names the replacement.
- **Gradle can't be downloaded**: the wrapper fetches from services.gradle.org; check your
  network/proxy.
