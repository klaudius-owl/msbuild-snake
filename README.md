# 🐍 MSBuild Snake

A real-time Snake game **generated entirely by MSBuild**. The build IS the game. MSBuild properties control every aspect of gameplay — grid size, speed, colors, difficulty, wall layouts.

No C#. No pre-built assets. Just one `.proj` file that outputs a playable HTML game.

## Install & Play

```bash
# Prerequisites: .NET SDK 8.0+ (https://dotnet.microsoft.com/download)
git clone https://github.com/klaudius-owl/msbuild-snake.git
cd msbuild-snake

# Build & open in browser
dotnet msbuild -t:Play

# Or just build (output: build/snake.html)
dotnet msbuild -t:Build
```

## Difficulty Levels

```bash
dotnet msbuild -t:Play -p:Difficulty=easy     # 180ms/tick, no walls
dotnet msbuild -t:Play -p:Difficulty=normal    # 120ms/tick, no walls (default)
dotnet msbuild -t:Play -p:Difficulty=hard      # 80ms/tick, center cross walls
dotnet msbuild -t:Play -p:Difficulty=insane    # 45ms/tick, cross + corner walls
```

## Customize Everything

Every game parameter is an MSBuild property you can override at build time:

```bash
# Big grid, fast, wrapping edges
dotnet msbuild -t:Play -p:GridSize=30 -p:Speed=60 -p:WrapAround=true

# Custom colors
dotnet msbuild -t:Play -p:ColorSnake=#ff6b6b -p:ColorBg=#1e1b4b -p:ColorFood=#fbbf24

# Long snake, grows fast
dotnet msbuild -t:Play -p:InitialLength=10 -p:GrowthRate=3
```

### All Properties

| Property | Default | Description |
|----------|---------|-------------|
| `GridSize` | `20` | Grid dimensions (NxN) |
| `CellSize` | `20` | Pixel size per cell |
| `Speed` | `120` | Milliseconds per tick (lower = faster) |
| `Difficulty` | `normal` | `easy` / `normal` / `hard` / `insane` |
| `WrapAround` | `false` | Snake wraps screen edges |
| `ShowGrid` | `true` | Show grid lines |
| `InitialLength` | `3` | Starting snake length |
| `GrowthRate` | `1` | Segments gained per food |
| `ColorBg` | `#0a0a0a` | Background color |
| `ColorSnake` | `#4ade80` | Snake body color |
| `ColorSnakeHead` | `#22c55e` | Snake head color |
| `ColorFood` | `#ef4444` | Food color |
| `ColorWall` | `#64748b` | Wall color |
| `ColorText` | `#e2e8f0` | Text color |
| `ColorAccent` | `#facc15` | Accent/score color |

## How It Works

MSBuild generates a self-contained HTML file with embedded JavaScript. The game logic (rendering, input, collision, scoring) is JS, but **everything is parameterized through MSBuild properties**:

- **Grid/speed/colors** → MSBuild properties interpolated into the generated JS/CSS
- **Wall layouts** → MSBuild `<ItemGroup>` with `<Wall>` items specifying X,Y coordinates
- **Difficulty presets** → MSBuild `<PropertyGroup>` conditions that override speed and add walls
- **Output** → `Exec` task writes the HTML via shell heredoc (avoids `WriteLinesToFile` semicolon splitting)

The wall system is extensible — add your own `<Wall>` items to create custom levels:

```xml
<ItemGroup>
  <Wall Include="mywall1"><X>3</X><Y>3</Y></Wall>
  <Wall Include="mywall2"><X>3</X><Y>4</Y></Wall>
  <!-- ... -->
</ItemGroup>
```

### MSBuild Features Used

- Property interpolation into generated code
- `<ItemGroup>` batching for wall coordinate generation (`@(Wall->'[%(X),%(Y)]', ',')`)
- Conditional `<PropertyGroup>` for difficulty presets
- `Exec` task with heredoc for file generation
- `$([MSBuild]::Multiply())` for canvas size calculation
- `MakeDir`, `RemoveDir` for build output management
- ANSI escape sequences for colored terminal output

## Controls

- **Arrow keys** or **WASD** to move
- **Touch/swipe** on mobile
- **On-screen buttons** on touch devices

## Targets

| Target | Description |
|--------|-------------|
| `-t:Build` | Generate the HTML game |
| `-t:Play` | Build & open in browser |
| `-t:Clean` | Remove build output |
| `-t:Help` | Show available properties |

## Requirements

- [.NET SDK](https://dotnet.microsoft.com/download) 8.0+
- A modern web browser
- Shell with heredoc support (bash, zsh, sh — i.e., not Windows CMD; use WSL or PowerShell)

## License

MIT
