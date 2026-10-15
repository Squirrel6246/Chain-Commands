# Chain Commands

Run several commands from one line, with conditions, piping, waiting and loops. Type `/chain`, separate your commands with operators, and they run in order.

```
/chain gamemode creative && time set day && weather clear
```

No command blocks needed for quick setups, and no datapack needed for simple logic.

## Features

- **Chaining with success/failure control**: `&&` (next only if it worked), `||` (next only if it failed), `;;` (always)
- **Stop on failure**: failed `&&` steps skip the rest, `strict` mode halts the whole chain on the first failure, and `stop` ends it on demand
- **If / elif / else**: `if <command> then ... else ... end`, plus `unless` and a `compare` test for numbers and text
- **Piping**: send one command's output into the next with `|`, or drop it **anywhere** with `{out}`, `{result}` and `{num}`
- **Reach back**: `{out2}` and `{result3}` grab the output of earlier steps
- **Wait**: `wait 20` or `wait 1s` pauses the chain, then carries on
- **Repeat**: `repeat 5 say hi`
- **Dry run**: `/chain dry ...` shows how your line was split without running anything
- **Quote- and bracket-aware**: operators inside "quotes", `[ ]` and `{ }` (NBT/JSON) are left alone
- **Permission-safe**: every command runs as you, each one permission-checked on its own

## Quick examples

```
/chain scoreboard players get @s kills | say I have {result} kills
```
```
/chain if execute if entity @s[tag=vip] then say welcome back else say hello end
```
```
/chain scoreboard players get @s kills ;; if compare {result} >= 10 then say pro elif compare {result} >= 5 then say good else say rookie end
```
```
/chain give @s diamond || stop your inventory is full ;; say enjoy!
```
```
/chain say 3 ;; wait 1s ;; say 2 ;; wait 1s ;; say 1 ;; wait 1s ;; say go!
```

## Cheat sheet

| Syntax | Meaning |
|---|---|
| `a && b` | run `b` if `a` succeeded |
| `a \|\| b` | run `b` if `a` failed |
| `a ;; b` | run `b` regardless |
| `a \| b` | like `&&`, plus pipe `a`'s output into `b` |
| `strict ...` | stop the chain at the first failure |
| `if C then A elif C then B else D end` | conditionals (`unless` also works) |
| `compare a == b` | `== != < <= > >= contains` |
| `{out}` `{result}` `{num}` | previous output text / numeric result / first number |
| `{out2}` `{result2}` `{num2}` | the same, from the 2nd step |
| `{out:json}` | output escaped for JSON text |
| `wait 20` / `wait 1s` | pause (max 1 hour) |
| `repeat 5 <command>` | run up to 5 times (max 100) |
| `echo <text>` | message only you see |
| `stop [message]` | end the chain |

To use an operator as plain text, put it in quotes or write `\&&`, `\||`, `\;;` or `\|`. Type `/chain` on its own for an in-game reminder.

## Supported versions

| Minecraft | Loader | Requires |
|---|---|---|
| 26.2 | Fabric (0.19.3+) | Fabric API, Java 25 |
| 26.3 | Fabric (0.19.3+) | Fabric API, Java 25 |

There is a separate download for each Minecraft version. Pick the file that matches your game version.

## Installation

1. Install [Fabric Loader](https://fabricmc.net/use/) for your Minecraft version.
2. Install [Fabric API](https://modrinth.com/mod/fabric-api).
3. Put the Chain Commands jar for your version in your `mods` folder.

**Singleplayer:** just install it. **Servers:** install it on the server. Players joining don't need it on their client.

## Limits and notes

- Up to 128 steps, 1000 commands, 100 repeats and a 1-hour wait per chain.
- A command counts as failed if it errors, doesn't exist, or never runs (like an `execute if` whose condition is false).
- Output that is piped onward is hidden, unless that command failed.
- Each nested `if` needs its own `end`. On one line, `else`, `elif` and `end` are keywords; put message text containing those words on its own `;;` step.
- Tab completion doesn't work inside the chained part.
- A chain that is waiting is forgotten if the server restarts.

## License

MIT. Bug reports and suggestions are welcome in the issue tracker.
