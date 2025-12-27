# Copilot Instructions — Warrior Platform

## Project Overview
2D platformer built with Python Arcade. **All assets are procedurally generated** via code (PIL for sprites, Pyglet for sounds) — no external files. Features dynamic weather, 4 enemy types with unique AI, and SQLite score persistence.

**Objective**: Defeat all enemies before time runs out while preserving health. Collect hearts and grab the Super Sword to double damage.

## Architecture

| File | Purpose |
|------|---------|
| `game.py` | Main game: `GameWindow` class, physics, AI, HUD, state machine |
| `assets/sprites.py` | All sprite generation via PIL (`make_*_textures()`) |
| `db_view.py` | SQLite database inspection utility |

**State Machine**: `self.state` ∈ `{'title', 'playing', 'victory', 'game_over'}`

## Critical Patterns

### Sprite Generation (sprites.py)
```python
def make_enemy_textures():
    img = Image.new("RGBA", (w, h), (0, 0, 0, 0))
    d = ImageDraw.Draw(img)
    # ... draw pixels ...
    return arcade.Texture(name="unique_name", image=ImageOps.flip(img))  # ALWAYS flip!
```
⚠️ **All textures require `ImageOps.flip()`** — PIL and Arcade use opposite Y-coordinates.

### Enemy Spawn Pattern (game.py)
```python
def spawn_slime(self, x, y, min_x, max_x):
    enemy = arcade.Sprite()
    enemy.enemy_tex = make_slime_textures()  # Dict with walk/hurt/die animations
    enemy.type = "slime"                      # Used for AI branching in on_update()
    enemy.bound_left, enemy.bound_right = min_x, max_x  # Patrol boundaries
    enemy.name_text = arcade.Text(...)        # Pre-create for performance
```

### Performance Rules
- **Never create `arcade.Text` per frame** — cache in `_ensure_*_ui()` methods
- Use `arcade.SpriteList(use_spatial_hash=True)` for collision-heavy lists
- Enemy HP bars only render after first damage (`enemy.show_hp_bar = True`)
- Title screen: texts cached via `_ensure_title_ui()`
- End screens (victory/game over): texts cached via `_ensure_end_ui()`
- Upgrade banner: texts cached by content/size, only updates transparency on draw via `_ensure_banner_ui()`
- Enemy names: created at spawn and only repositioned per frame

## Constants (game.py lines 31-50)
```python
PLAYER_MOVE_SPEED = 4.0
PLAYER_JUMP_SPEED = 12.0
GRAVITY = 0.6
SLIME_SPEED = 1.4
GOBLIN_SPEED = 2.2
ORC_SPEED = 1.6
BAT_SPEED = 2.6
ATTACK_DURATION = 0.36
ATTACK_HIT_START = 0.12  # Window active between ~0.12s and ~0.28s
ATTACK_HIT_END = 0.28
PLAYER_MAX_HP = 5        # Float-based (supports 0.5 damage)
PLAYER_INVULN = 1.0      # Invulnerability duration after taking damage
```

## Player Mechanics
- **Base attack damage**: 1 (2 with Super Sword)
- **Attack hitbox**: Displaced in front of warrior (current direction), slightly above feet
- **Knockback**: Applied on contact damage with horizontal push
- **Invulnerability**: 1.0s after receiving damage

## Enemy AI (all in `on_update()`)
| Type | HP | Damage | Speed | Behavior |
|------|----|--------|-------|----------|
| Slime | 3 | 0.5 | 1.4 | Ping-pong patrol |
| Goblin | 3 | 1.0 | 2.2 | Ping-pong patrol (faster) |
| Orc | 4 | 1.5 | 1.6 | Ping-pong patrol (tanky) |
| Bat | 2 | 1.0 | 2.6 | Wave flight + diving attack |

### Bat Diving Attack Details
- **Activation**: Player airborne, horizontally close (<240px), bat above player
- **Behavior**: Pursues player during dive (horizontal + height), stops immediately if player lands
- **Duration**: Max ~1.2s dive, then ~2.5s cooldown
- **Collision**: If hits platform during dive, lands on top and ends dive
- **Initial cooldown**: Prevents immediate dive on spawn

### Enemy UI
- **Name label**: Always visible above enemy with themed color:
  - Slime: green (80, 200, 120)
  - Goblin: green (60, 170, 90)
  - Orc: red (200, 70, 70)
  - Bat: purple (150, 100, 200)
- **HP bar**: Only appears after first damage (`enemy.show_hp_bar = True`)

## Items and Power-ups
| Item | Effect | Source |
|------|--------|--------|
| Heart | Heals 1.0 HP | 30% drop chance on enemy kill |
| Super Sword Chest | Doubles attack damage (1→2) | Top platform, one-time pickup |

## Scoring & Win/Lose Conditions
- **+100 points** per enemy defeated (after death animation)
- **Victory**: All enemies eliminated
- **Defeat**: HP reaches 0 OR timer reaches 0 with enemies remaining
- **Timer**: Starts at 05:00

## HUD Elements
- **Hearts** (top-left): Player HP, supports half-hearts
- **Score** (top-left): Current score
- **Timer** (top-right): Countdown from 05:00
- **Upgrade banner**: Displays ~3s when Super Sword collected

## Weather System
**Types**: `day_sunny`, `day_cloudy`, `day_rain`, `night_clear`, `night_cloudy`, `night_rain`
- Randomized at `setup()`
- Clouds: density/alpha vary by weather type
- Rain: drops with wind + occasional lightning flash

## Database
SQLite `warrior_platform.db`:
- `players(id, name)`
- `scores(player_name, score, created_at)`
- Top 5 displayed on end screens
- Fallback: `scores.txt` (appends `name;score`)

## Resolution & Scaling
- Fullscreen by default, uses actual window/monitor size
- World anchored to horizontal center
- HUD uses current dimensions for positioning
- No letterboxing — adapts to 1080p, 2K, 4K and different aspect ratios

## Running & Building
```bash
python game.py                    # Run (fullscreen)
pip install arcade pillow         # Dependencies
```

### Windows .exe (auto-py-to-exe)
**Hidden imports** (required):
- `arcade.gl`
- `arcade.gl.backends`
- `arcade.gl.backends.opengl`
- `arcade.gl.backends.pyglet`
- `assets.sprites`

**Additional args**: `--collect-submodules arcade.gl.backends --collect-submodules pyglet`

**Common errors**: If "Backend Provider 'opengl' not found", ensure hidden imports are set and rebuild with clean `build/` and `dist/` folders.

## Adding Features

**New Enemy**:
1. `assets/sprites.py`: Create `make_<enemy>_textures()` returning `{walk_right, walk_left, hurt_*, die_*}` dict
2. `game.py`: Add `spawn_<enemy>(x, y, min_x, max_x)` with `enemy.type = "<enemy>"`
3. `game.py` `on_update()`: Add AI branch in enemy loop

**New Pickup**: Add to `pickup_list`, handle collision in `on_update()` pickup section (~line 820)

**New Weather**: Add to `self.weather` choices in `setup()`, handle sky/effects in weather blocks

## Code References
| Feature | Location |
|---------|----------|
| Constants | `game.py:31-50` |
| Sky/weather generation | `game.py:151-232`, `game.py:860-922` |
| Chest & upgrade effect | `game.py:294-306`, `game.py:830-857` |
| Enemy spawns & attributes | `game.py:320-420` |
| Heart drop logic | `game.py:665-671` |
| Enemy AI & animation | `game.py:657-740` |
| End game & leaderboard | `game.py:872-984` |
| SFX generation | `game.py:992-1160` |
| Sprites & textures | `assets/sprites.py` |
| DB & scores | `game.py:101-110`, `game.py:1049-1150` |

## Conventions
- Brazilian Portuguese for comments and UI strings
- Damage values are floats (half-hearts = `0.5`)
- All enemies give +100 points and have 30% heart drop chance
