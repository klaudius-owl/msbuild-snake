# 🐍 MSBuild Snake

**Every game tick is a full `dotnet msbuild` invocation.** The build IS the engine.

MSBuild reads game state from files, computes snake movement, collision detection, food placement, and scoring — all in XML + shell commands inside `<Exec>` tasks. The browser is a dumb renderer that shows whatever JSON MSBuild spits out.

No C#. No game loop in JS. One `.proj` file that runs your Snake game at ~1 FPS because each frame cold-starts the entire MSBuild evaluation engine. *Gloriously unhinged.*

## Architecture

```
Browser (dumb renderer)
   │
   │  HTTP GET /tick?dir=right
   ▼
bridge.py (Python, 20 lines)
   │
   │  subprocess: dotnet msbuild -t:Tick -p:Dir=right
   ▼
Game.proj (MSBuild XML)
   │
   ├── Read state from .state/ files (snake, dir, food, score, walls, alive, ticks, grow)
   ├── Compute new head position (shell arithmetic in Exec)
   ├── Check collisions: wall boundary, self-intersection, obstacle walls
   ├── Check food: spawn new food via PRNG if eaten
   ├── Update snake body: shift segments, handle growth
   ├── Write new state back to .state/ files
   └── Output JSON to stdout
   │
   ▲
bridge.py captures stdout JSON
   │
   ▼
Browser renders the frame, waits, sends next tick
```

Each frame costs ~200ms (MSBuild startup + XML parse + shell exec + file I/O). The browser polls at a configurable interval for "real-time" gameplay.

## Install & Play

```bash
# Prerequisites: .NET SDK 8.0+ (https://dotnet.microsoft.com/download)
git clone https://github.com/klaudius-owl/msbuild-snake.git
cd msbuild-snake

# Build everything, init game state, start the bridge server & open browser
dotnet msbuild -t:Play
```

That's it. `Play` does everything:
1. Generates `build/snake.html` (the dumb renderer)
2. Generates `build/bridge.py` (the HTTP→MSBuild bridge)
3. Initializes `.state/` files (starting position, food, etc.)
4. Starts the bridge server on port 8765
5. Opens the browser

### Manual Setup

```bash
# Step 1: Build the HTML renderer + bridge server
dotnet msbuild -t:Build

# Step 2: Start a new game (resets state)
dotnet msbuild -t:New

# Step 3: Start the bridge server
python3 build/bridge.py &

# Step 4: Open build/snake.html in your browser
```

### Playing Without a Browser

You can play entirely from the terminal:

```bash
# Start a new game
dotnet msbuild -t:New

# Each command is one game tick:
dotnet msbuild -t:Tick -p:Dir=right
dotnet msbuild -t:Tick -p:Dir=right
dotnet msbuild -t:Tick -p:Dir=down
dotnet msbuild -t:Tick              # continues in current direction

# Check current state
dotnet msbuild -t:Status
```

Each `Tick` outputs JSON:
```json
{"alive":true,"score":10,"snake":"5,3|4,3|3,3|2,3","food":"12,7","walls":"","dir":"right","ticks":4,"ate":0}
```

## Difficulty Levels

```bash
dotnet msbuild -t:Play -p:Difficulty=easy     # No walls, relaxed
dotnet msbuild -t:Play -p:Difficulty=normal   # No walls (default)
dotnet msbuild -t:Play -p:Difficulty=hard     # Center cross walls
dotnet msbuild -t:Play -p:Difficulty=insane   # Cross + corner walls
```

Wall layouts are defined as MSBuild `<ItemGroup>` entries:

```xml
<ItemGroup Condition="'$(Difficulty)'=='hard'">
  <Wall Include="h1"><X>10</X><Y>8</Y></Wall>
  <Wall Include="h2"><X>10</X><Y>9</Y></Wall>
  <!-- ... -->
</ItemGroup>
```

## Customize

```bash
# Big grid
dotnet msbuild -t:Play -p:GridSize=30

# Custom colors
dotnet msbuild -t:Play -p:ColorSnake=#ff6b6b -p:ColorBg=#1e1b4b

# Faster polling (browser-side, in ms)
dotnet msbuild -t:Play -p:Speed=150
```

### All Properties

| Property | Default | Description |
|----------|---------|-------------|
| `GridSize` | `20` | Grid dimensions (NxN) |
| `CellSize` | `20` | Pixel size per cell |
| `Speed` | `200` | Browser poll interval in ms |
| `Difficulty` | `normal` | `easy`/`normal`/`hard`/`insane` |
| `BridgePort` | `8765` | Bridge server port |
| `ColorBg` | `#0a0a0a` | Background color |
| `ColorSnake` | `#4ade80` | Snake body color |
| `ColorSnakeHead` | `#22c55e` | Snake head color |
| `ColorFood` | `#ef4444` | Food color |
| `ColorWall` | `#64748b` | Wall color |
| `ColorText` | `#e2e8f0` | Text color |
| `ColorAccent` | `#facc15` | Accent/score color |

## Targets

| Target | Description |
|--------|-------------|
| `-t:Play` | Build + init + start bridge + open browser |
| `-t:Build` | Generate HTML renderer + bridge.py |
| `-t:New` | Reset game state (start fresh) |
| `-t:Tick` | Run one game tick (`-p:Dir=up/down/left/right`) |
| `-t:Status` | Show current game state |
| `-t:Help` | Show available targets and properties |
| `-t:Clean` | Remove build output and state |

## How MSBuild Runs the Game

The `Tick` target is where the magic happens. Every invocation:

1. **Reads state** — `ReadLinesFromFile` loads `.state/snake`, `.state/dir`, `.state/food`, `.state/score`, etc.
2. **Processes input** — Validates direction change (prevents 180° reversal)
3. **Computes physics** — Shell arithmetic inside `<Exec>` calculates new head position, shifts body segments
4. **Checks collisions** — Shell script checks wall boundaries, self-intersection, and obstacle walls
5. **Handles food** — If head == food position, increment score, spawn new food via PRNG (`$((ticks * 7 + 3) % (grid * grid))` mapped to coordinates)
6. **Writes state** — `WriteLinesToFile` updates all `.state/` files
7. **Outputs JSON** — `echo` the game state as JSON to stdout, captured by the bridge

### Key MSBuild Tricks

- **`%24` for `$`** — MSBuild tries to evaluate `$((x+1))` as a property. Escape as `%24((x+1))`.
- **`%25` for `%`** — Modulo operator conflicts with MSBuild item metadata `%(...)`. Escape as `%25`.
- **`&gt;` for `>`** — XML requires entity escaping for shell redirects.
- **`$([System.Math]::Floor(...))`** — MSBuild's `Divide()` returns doubles; use `Math::Floor` for integer division.
- **Individual state files** — Can't use a single file because `WriteLinesToFile` splits on `;` (semicolons are item separators in MSBuild).
- **PRNG without random()** — Food placement uses deterministic hash: `(ticks * 7 + 3) % (grid * grid)`, with collision retry by incrementing.

## Controls

- **Arrow keys** or **WASD** to move
- **Touch/swipe** on mobile
- **On-screen direction buttons** on touch devices

The browser just sends your input as the `dir` parameter on the next tick. MSBuild decides whether to accept it (no 180° reversals).

## Requirements

- [.NET SDK](https://dotnet.microsoft.com/download) 8.0+
- Python 3 (for the bridge server)
- A modern web browser
- Bash/sh (the `Exec` tasks use shell commands)

## Performance

Each tick takes ~200ms on a modern machine. This is the cold-start cost of:
- Parsing the MSBuild XML schema
- Evaluating all properties and conditions
- Running the `Exec` shell commands
- File I/O for state persistence

The browser HUD shows builds-per-second and milliseconds-per-build. It's slow. That's the point.

## License

MIT
