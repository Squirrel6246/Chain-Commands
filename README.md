# Chain Commands

Run several commands from a single line. Type `/chain`, separate your commands with `&&`, and they run one after another.

```
/chain gamemode creative && time set day && weather clear
```

No more pasting commands one at a time, and no command blocks needed for quick setups.

## Features

- **One-line chaining**: `/chain <command> && <command> && <command>`
- **Quote-aware**: an `&&` inside double quotes is left alone, so text stays intact:
  `/chain say "fish && chips" && say done`
- **Escapable**: write `\&&` for a literal `&&` outside quotes
- **Permission-safe**: every command runs as you, and each one is permission-checked on its own. `/chain` can't run anything you couldn't run by hand
- **Forgiving**: if one command fails, the rest still run
- **Optional leading slashes**: `/chain /say hi && /say there` works too
- **Lightweight**: one command, no config files, no extra dependencies beyond Fabric API

## Examples

| You type | What happens |
|---|---|
| `/chain time set day && weather clear` | Sets day and clears the weather |
| `/chain gamemode creative @a && effect give @a night_vision infinite` | Sets everyone to creative, then gives night vision |
| `/chain say "Starting in 5 && counting"` | Sends one message (the `&&` is inside quotes) |
| `/chain say a \&& b` | Sends `a && b` |

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

## Notes and limitations

- A chain can hold up to 64 commands.
- Commands run in order, right after the `/chain` command itself finishes.
- Tab completion doesn't work inside the chained part, because Minecraft treats it as one block of text.
- If a command fails, you'll still see its normal error message, and the next command runs.

## Source and license

Released under the MIT license. Bug reports and suggestions are welcome in the issue tracker.
