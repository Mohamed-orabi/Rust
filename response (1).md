# Chapter 5 — Build a Dungeon Crawler: A Comprehensive Professor-Style Walkthrough

Welcome. In this chapter you are going to build the **foundational scaffolding** of a roguelike dungeon-crawler game in Rust, using the `bracket-lib` crate. By the end, you'll have:

1. A program split into **modules** (`map`, `player`, `map_builder`, `camera`).
2. A **tile-based map** stored efficiently in a `Vec<TileType>`.
3. A **player avatar** (`@`) that walks around, blocked by walls.
4. A **procedural dungeon generator** that carves rooms and corridors.
5. A **graphical rendering system** with **two layers** (map + entities) and a **camera** that follows the player.

I'll explain every concept top-to-bottom — the *what*, the *why*, and the *how* — and I will not skip the small notes. Let's begin.

---

## Section 1 — The Big Picture and Motivation

So far in the book, all your code lived in **one file**: `main.rs`. That's fine for "Hello World," but games quickly grow to thousands of lines. The chapter opens by pointing out three concrete pains that come from a single-file approach, and three corresponding benefits of splitting code into **modules**:

1. **Findability.** A file named `map.rs` is where map code lives — obvious. Compare that to "scroll to around line 500 of `main.rs`." When code grows, you need *spatial* navigation.
2. **Compilation speed.** Cargo (Rust's build tool) can compile multiple modules **concurrently**. One huge file is one compilation unit; many small modules can run on multiple CPU cores in parallel.
3. **Bug isolation.** When a module has a narrow, well-defined interface, bugs in it cannot easily corrupt unrelated parts of the program. This is loose coupling, and it matters more the larger your codebase gets.

Coming from C#, you can think of this exactly like splitting a giant `.cs` file into multiple files with `namespace MyGame.Map`, `namespace MyGame.Player`, etc. The mechanism is different (Rust uses `mod` instead of `namespace`), but the *intent* is identical: organize code so humans (and compilers) can reason about it.

---

## Section 2 — Crates and Modules: The Namespace Hierarchy

This is foundational Rust knowledge, so let's be precise.

### Crates

A **crate** is the largest unit of compilation in Rust. It is essentially "a package with its own `Cargo.toml`." Two examples from your project:

- **Your game** (`dungeoncrawl`) is a crate.
- **`bracket-lib`** is a crate (a dependency you list in `Cargo.toml`).

Crates can depend on other crates. `bracket-lib` itself depends on smaller crates like `bracket-terminal` and `bracket-random`. The diagram in the book shows this as a tree, with `Cargo.toml` at the top defining what crates your project uses.

> **C# analogy:** A crate is like a **NuGet package** or a **.csproj/.dll**. Your game is one assembly; `bracket-lib` is another.

### Modules

A **module** is a subdivision *inside* a crate. It can be either:

- a single file (e.g., `map.rs`), or
- a directory (you'll see this later in the book — "Multi-File Modules").

In this chapter, you'll use single-file modules.

### Crates and Modules as Namespaces

This is the key conceptual move: crates and modules form a **namespace hierarchy**. Examples:

- `bracket_lib::prelude` means "the `prelude` module inside the `bracket-lib` crate."
- `crate::map` means "the `map` module at the root of *the current crate*."
- `crate::map::region::chunk` shows that modules can nest arbitrarily deep.

The diagram in the chapter visualizes:

```
Crate: Your Game
├── Main Module (main.rs)         → fn main()
├── crate::map Module (map.rs)    → pub fn draw_map()
└── crate::player Module (player.rs) → pub fn update(), pub fn render()

Crate: Bracket-Lib (dependency)
├── (also has dependencies — bracket-terminal, bracket-random, etc.)
```

> **C# analogy:**
> - `crate::` ≈ the **root namespace** of your assembly.
> - `super::` ≈ going **up one namespace level**.
> - `bracket_lib::prelude::*` ≈ `using BracketLib.Prelude;` (importing all public types from a namespace).

### The "Re-exported" Concept

The diagram has a subtle but important note: bracket-terminal and bracket-random are marked **"Only accessible if re-exported."** This means: even though `bracket-lib` depends on `bracket-terminal`, *your* game cannot use `bracket-terminal` types directly unless `bracket-lib` chooses to make them visible to its users via `pub use`. This is exactly the design principle you'll use in your own `prelude` shortly.

---

## Section 3 — Making the Stub Map Module

Now we move from theory to action.

### Step 1: Create the project

```bash
cargo new dungeoncrawl
```

Then follow the "Hello, Bracket Terminal" steps from the earlier chapter to display "Hello, Bracket World." Ensure `Cargo.toml` has:

```toml
[dependencies]
bracket-lib = "~0.8.1"
```

The `~0.8.1` syntax is a semver range: "0.8.1 or any later patch (0.8.2, 0.8.3...) but not 0.9." This pins the *minor* version for stability.

### Step 2: Create `map.rs`

Add an empty file `src/map.rs`. By itself, this file is invisible to the compiler — Rust does *not* automatically pick up files in your `src/` directory the way some other languages do.

### Step 3: Tell `main.rs` the module exists

At the top of `main.rs`, add:

```rust
mod map;
```

This single line does two things:

1. Tells Rust: "Look for a file `src/map.rs` (or directory `src/map/mod.rs`) and **include it in the build**."
2. Creates the namespace prefix `map::`, so you can now refer to things inside that file as `map::something()`.

You can also "import" specific names using `use`, exactly like a C# `using static`:

```rust
use map::my_function;
my_function(); // Now callable without the prefix.
```

---

## Section 4 — Module Scoping and `pub`

Here is one of the most important rules in Rust:

> **Everything inside a module is private by default.**

If `map.rs` contains `fn helper()`, code in `main.rs` cannot call `map::helper()` — the compiler will flat-out refuse. To make something visible from outside, you mark it `pub`:

| Item type | Public form |
|---|---|
| Function | `pub fn my_function()` |
| Struct | `pub struct MyStruct` |
| Enum | `pub enum MyEnum` |
| Implemented method | `impl MyStruct { pub fn my_method(...) }` |

There's also **per-field visibility for structs**:

```rust
pub struct MyPublicStructure {
    pub public_int: i32,
    private_int: i32,   // No `pub` — invisible outside this module.
}
```

From inside the module, you have full access to `private_int`. From outside, you can only touch `public_int`.

> **Why this matters:** This is *encapsulation* enforced by the compiler. In C#, `private` is enforced too, but Rust's defaults are stricter — *everything* is private until you opt-in. This forces you to think about your public API.

### The Law of Demeter (sidebar)

The chapter highlights the **Law of Demeter** in a note:

> "Each module should have only limited knowledge about other units, and should only talk to its friends."

This is a principle from the book *The Pragmatic Programmer*. In plain English: don't reach through several objects to grab something deep inside them. Don't write `customer.GetAccount().GetOwner().GetAddress().GetCity()`. Instead, ask the customer for what you really need (`customer.GetCity()`), and let *it* figure out how.

The benefit is **loose coupling**: if you change how addresses are stored internally, you don't break every caller — you only update the customer's `GetCity()` method.

---

## Section 5 — Organizing Imports With a Prelude

After a while, writing `map::TileType`, `map::Map::new()`, `crate::map::Map`, and so on becomes tedious. Library authors solve this by making a **prelude**: a single module that re-exports everything users typically need.

You used `bracket-lib::prelude::*` already. Now you'll build your own. Add this to `main.rs`:

```rust
mod map;                  // ❶

mod prelude {             // ❷
    pub use bracket_lib::prelude::*;        // ❸
    pub const SCREEN_WIDTH: i32 = 80;       // ❹
    pub const SCREEN_HEIGHT: i32 = 50;
    pub use crate::map::*;                   // ❺
}

use prelude::*;           // ❻
```

Let's dissect every annotation:

**❶ `mod map;`** — pulls the `map.rs` file into the build, as discussed.

**❷ `mod prelude { ... }`** — here `mod` is used with curly braces to define a module **inline**, in the same file. This is fine because the prelude is small and only used to glue things together. Note: it does **not** need `pub` because it lives at the crate root and is visible throughout the crate.

**❸ `pub use bracket_lib::prelude::*;`** — this is **re-exporting**. The `pub use` syntax says: "Import these names into *this* module, **and** make them visible to anyone who imports *this* module." Without `pub`, the import would only be visible inside `prelude` itself. The result: anyone who writes `use prelude::*;` gets all of `bracket-lib`'s prelude for free.

**❹ Public constants** — `pub const` makes the constants visible. `SCREEN_WIDTH = 80` and `SCREEN_HEIGHT = 50` give a screen of 80×50 character cells. The `i32` type is a 32-bit signed integer; we use `i32` because most coordinate math in `bracket-lib` uses `i32`.

**❺ `pub use crate::map::*;`** — re-export everything public from your `map` module. Notice it uses `crate::map`, not just `map`. From *inside* the prelude module, plain `map::` would look for a `map` submodule inside `prelude`. `crate::` explicitly says "start at the crate root."

**❻ `use prelude::*;`** — at last, bring the prelude into `main.rs`'s scope, so you can use everything without qualification.

### The chapter's sidebar: Accessing Other Modules

Two important path keywords:

- `super::` → "the module immediately above me in the tree."
- `crate::` → "the root of my crate" (i.e., `main.rs` or `lib.rs`).

> **C# analogy:** `crate::` is like an absolute namespace path starting with `global::MyAssembly.`. `super::` is like writing `..` in a file path.

---

## Section 6 — Setting Up the Map Module

Open `map.rs`. Add:

```rust
use crate::prelude::*;
const NUM_TILES: usize = (SCREEN_WIDTH * SCREEN_HEIGHT) as usize;
```

Two important things happen here:

1. **`use crate::prelude::*;`** — pulls in everything from your prelude (which itself pulls in `bracket-lib`'s prelude, your constants, and re-exports). This is why preludes are so powerful: each module just imports the prelude and gets everything.

2. **`const NUM_TILES`** — a compile-time constant defined as `(80 * 50) = 4000`. The `as usize` is a **type cast**. Without it, the expression's type would be `i32` (because `SCREEN_WIDTH` is `i32`), and Rust would refuse to use it as a vector size, which must be `usize`.

> **Rule:** A `const` in Rust can only be assigned an expression that the compiler can evaluate **at compile time**. So `(SCREEN_WIDTH * SCREEN_HEIGHT)` works (both are constants), but `(some_function())` would not — unless `some_function` is a `const fn`.

### Sidebar: What Is a `usize`?

This is a fundamental Rust concept. Most numeric types are *fixed-size*: `i32` is always 32 bits, `i64` is always 64 bits. But `usize` (and its signed cousin `isize`) is **the natural pointer-size integer for the CPU**:

- On a 32-bit machine, `usize` is 32 bits.
- On a 64-bit machine, `usize` is 64 bits.

`usize` is used to **index collections** and represent memory sizes, because it can always hold a valid index into any array/vector that fits in memory.

> **C# analogy:** `usize` is closely analogous to C#'s `nuint` (unsigned native int) — same idea, sized to the platform.

---

## Section 7 — Storing the Dungeon Map

### Why a Vector?

Most 2D games use a **tile grid**. The chapter gives a clear intuition: each tile is a small piece of the world, and the same tile graphic is reused everywhere of that type. **Entities** (player, monsters, chests) are drawn *on top of* tiles.

The book includes an illustrated grid where:
- Wall tiles, floor tiles, etc. form the grid.
- A knight (`@`) and a chest are entities overlaid on top.

For an 80×50 map you have **4,000 tiles**. Storing them as a flat `Vec<TileType>` is efficient because:
- Memory is **contiguous** (cache-friendly).
- Indexing is O(1).
- The whole map can be cloned, serialized, or scanned linearly.

> **Why not `Vec<Vec<TileType>>`?** A vector of vectors creates many separate heap allocations and breaks cache locality. A single flat vector is much faster, at the small price of needing a manual 2D→1D index calculation (which we'll write shortly).

### Representing Tiles with an Enum

```rust
#[derive(Copy, Clone, PartialEq)]
pub enum TileType {
    Wall,
    Floor,
}
```

`enum` here is a **C-style enum** — a tag-only enumeration without associated data (Rust enums *can* hold data, but these don't).

The `#[derive(...)]` attribute auto-generates trait implementations:

- **`Clone`** → adds a `.clone()` method. Calling `mytile.clone()` makes a **deep copy** of the value. For a struct, this clones every field. Useful when you need a copy to mutate independently, or to satisfy the borrow checker.

- **`Copy`** → changes assignment semantics. Without `Copy`, `let b = a;` **moves** the value (so `a` is no longer usable). With `Copy`, `let b = a;` **copies** it (both `a` and `b` are valid afterward). Small POD-like types (primitives, small enums) should be `Copy` for ergonomics. Clippy will warn you if you take a `&TileType` when a `TileType` copy would be cheaper.

- **`PartialEq`** → enables `==` and `!=`. The "partial" reflects that some types (like `f32`) have `NaN` values that don't equal themselves; for our enum, equality is total but `PartialEq` is the standard derive.

`pub enum TileType` is public, and your prelude does `pub use crate::map::*`, so the type is available everywhere via the prelude.

The chapter notes: walls render as `#` and the player as `@` — classic ASCII roguelike conventions, dating back to the 1980 game *Rogue*.

### The `Map` Struct

```rust
pub struct Map {
    pub tiles: Vec<TileType>,
}
```

Both the struct and its `tiles` field are public — external code can read and modify the vector directly. That's a deliberate choice for simplicity here; in a larger codebase you might keep the field private and expose getters/setters.

### The Constructor

```rust
impl Map {
    pub fn new() -> Self {
        Self {
            tiles: vec![TileType::Floor; NUM_TILES],
        }
    }
}
```

A few things to unpack:

- `impl Map { ... }` opens an **inherent implementation block** — methods and associated functions for the `Map` type.
- `pub fn new() -> Self` — `Self` (capital S) is shorthand for the type the `impl` is for (here, `Map`).
- `vec![TileType::Floor; NUM_TILES]` — the `vec!` macro has a **repeat form**: `vec![value; count]` creates a `Vec` of length `count` where every element is a clone of `value`. This requires `TileType: Clone`, which is why we derived `Clone`.

Result: a 4,000-element vector filled entirely with `TileType::Floor`. An open empty plain. We'll carve walls later.

---

## Section 8 — Indexing the Map (Striding)

A `Vec` is **one-dimensional**, but our map is conceptually 2D. We need a function to convert `(x, y)` coordinates into a flat index. This is called **striding**.

### Row-First Encoding

The book uses **row-first** (also called row-major) encoding. Imagine a 5×3 grid:

```
Index layout:
 0  1  2  3  4     ← row 0 (y=0)
 5  6  7  8  9     ← row 1 (y=1)
10 11 12 13 14     ← row 2 (y=2)
```

Row 0 is stored first (indices 0–4), then row 1 (indices 5–9), then row 2 (10–14).

The formula to convert (x, y) → index:

```
index = (y * WIDTH) + x
```

And the reverse:

```
x = index % WIDTH      // modulus (remainder of integer division)
y = index / WIDTH      // integer division (rounds toward zero)
```

> **Integer division note (very important):** In Rust, dividing two integers **always rounds down toward zero**, throwing away the remainder. `7 / 2 == 3`, not `3.5`. This is exactly what we want for the `y = index / WIDTH` computation.

The `%` operator is **modulus**. `17 % 5 == 2`. Combined with integer division, `index = 17, WIDTH = 5` gives `x = 17 % 5 = 2` and `y = 17 / 5 = 3`. Verify visually: starting from 0, the 18th cell (index 17) is at column 2 of row 3. ✓

### Why Row-First?

The chapter mentions this in the rendering section, but it's worth highlighting now: row-first means consecutive memory locations correspond to consecutive cells in a row. When you iterate `for y { for x { ... } }`, you walk memory linearly — **maximally cache-friendly**. If you iterated `for x { for y { ... } }` with row-first layout, you'd jump around in memory and slow down significantly.

### Sidebar: Row-First, Column-First, or Morton Encoding

The book mentions three encodings:

- **Row-first (Y-first)** — rows stored together. What we use.
- **Column-first (X-first)** — columns stored together. Some game engines use this; it's the same idea rotated 90°.
- **Morton Encoding** (a.k.a. Z-order curve) — interleaves the bits of x and y. The result is that *nearby tiles in 2D tend to be nearby in memory*, which can improve cache behavior for algorithms that access neighborhoods (like blur filters or fluid simulations). It's more complex to compute, so the chapter wisely says: **"Unless your encoding is causing performance problems, you probably don't need Morton Encoding."**

### Implementing the Indexing Function

```rust
pub fn map_idx(x: i32, y: i32) -> usize {
    ((y * SCREEN_WIDTH) + x) as usize
}
```

Two details:

1. The function is declared **outside the `impl Map` block** but still inside the `map` module. This makes it a **free function**, not a method. You call it as `map_idx(x, y)`, not `map.map_idx(x, y)`. Good design: indexing is a pure math operation that doesn't depend on a particular `Map` instance.

2. **`as usize` cast** — multiplication happens in `i32`, then the result is cast to `usize`. Be aware: if `(y * SCREEN_WIDTH) + x` is **negative**, the cast produces a huge `usize` (because of two's complement) and indexing will panic. We avoid negatives by always pairing this with a bounds check (which we'll add soon).

---

## Section 9 — Rendering the Map

Now we draw the map to the terminal. Add this method inside `impl Map`:

```rust
pub fn render(&self, ctx: &mut BTerm) {
    for y in 0..SCREEN_HEIGHT {
        for x in 0..SCREEN_WIDTH {
            let idx = map_idx(x, y);
            match self.tiles[idx] {
                TileType::Floor => {
                    ctx.set(x, y, YELLOW, BLACK, to_cp437('.'));
                }
                TileType::Wall => {
                    ctx.set(x, y, GREEN, BLACK, to_cp437('#'));
                }
            }
        }
    }
}
```

Let's go through it piece by piece:

- **`&self`** — a *shared (immutable) reference* to the Map. The render function does not modify the map; it only reads.
- **`ctx: &mut BTerm`** — a *mutable reference* to the terminal context. Drawing changes the terminal's pixel buffer, so we need mutable access.
- **`for y in 0..SCREEN_HEIGHT`** — `0..SCREEN_HEIGHT` is a `Range` from 0 (inclusive) to `SCREEN_HEIGHT` (**exclusive**). So y goes 0, 1, 2, ..., 49. To make a range inclusive of the upper bound, you'd write `0..=SCREEN_HEIGHT`.
- **Y-first outer loop** — as discussed, this matches row-first storage and is cache-optimal.
- **`match self.tiles[idx]`** — pattern matching on the tile type. The `match` is exhaustive: if you added a third variant to `TileType` without updating this match, the compiler would refuse to compile. This is one of Rust's most valuable safety features.
- **`ctx.set(x, y, foreground, background, glyph)`** — draws a single character at (x, y).
- **`to_cp437('.')`** — `bracket-lib` uses **Code Page 437** (the classic IBM PC character set) for glyph indices. `to_cp437` converts a Unicode `char` to the appropriate CP437 byte. This is how roguelikes have historically rendered their `#`s and `@`s.

The result: floor tiles appear as yellow dots, wall tiles as green hashes.

---

## Section 10 — Wiring the Map into `main.rs`

Now we use the Map. Update `main.rs`:

```rust
struct State {
    map: Map,
}

impl State {
    fn new() -> Self {
        Self { map: Map::new() }
    }
}

impl GameState for State {
    fn tick(&mut self, ctx: &mut BTerm) {
        ctx.cls();           // Clear the screen.
        self.map.render(ctx);
    }
}

fn main() -> BError {
    let context = BTermBuilder::simple80x50()
        .with_title("Dungeon Crawler")
        .with_fps_cap(30.0)        // ❶
        .build()?;

    main_loop(context, State::new())
}
```

**❶ `.with_fps_cap(30.0)`** — the chapter highlights this one. It tells the engine to target 30 frames per second. Two benefits:

1. The OS knows it can let the CPU rest between frames (lower power usage, less heat).
2. The player won't move too fast — at 200+ FPS, holding an arrow key would teleport them across the screen.

The `BError` return type and `?` operator handle errors from the build call: `?` propagates errors up to the caller automatically.

Running the program now shows a screen full of yellow dots — your empty floor map.

---

## Section 11 — Adding the Adventurer (the Player)

The player needs:
1. A position (which tile they're on).
2. Logic to move them based on keyboard input.
3. A way to stop them from walking through walls or off the map.

### Extending the Map API

Before we build the Player, we add three helper methods to `Map`:

```rust
pub fn in_bounds(&self, point: Point) -> bool {
    point.x >= 0 && point.x < SCREEN_WIDTH
        && point.y >= 0 && point.y < SCREEN_HEIGHT
}
```

- **`Point`** is a struct from `bracket-lib` holding an `x: i32` and `y: i32`, plus useful operator overloads (you can do `point_a + point_b`).
- The function checks all four bounds with `&&` (logical AND). All four conditions must be true. This is **short-circuit evaluation**: if `point.x >= 0` is false, the rest isn't even evaluated.

```rust
pub fn can_enter_tile(&self, point: Point) -> bool {
    self.in_bounds(point)
        && self.tiles[map_idx(point.x, point.y)] == TileType::Floor
}
```

Critical design choice: this **calls `in_bounds` first**. If we didn't, and the point were outside the map, `map_idx` could produce an invalid index and `self.tiles[idx]` would panic. Short-circuit `&&` saves us — if `in_bounds` is false, the right side is never evaluated. Defensive programming at the language level.

`== TileType::Floor` works because we derived `PartialEq` on `TileType`.

```rust
pub fn try_idx(&self, point: Point) -> Option<usize> {
    if !self.in_bounds(point) {
        None
    } else {
        Some(map_idx(point.x, point.y))
    }
}
```

This returns `Option<usize>` — either `Some(index)` (valid) or `None` (out of bounds). `Option` is Rust's substitute for nullable values, but unlike `null` in C# (which can crash at runtime), `Option` forces you to handle both cases via pattern matching. **No `NullReferenceException` is possible.**

> **Why is this `try_idx` useful?** Later, when carving tunnels, we'll iterate coordinates that *might* be off-map. Using `try_idx` lets us silently skip out-of-bound positions with a clean `if let Some(idx) = ...` pattern instead of crashing.

### Creating the Player Module

Create `src/player.rs`. In `main.rs`, add it to your module list and prelude:

```rust
mod map;
mod player;          // ← new

mod prelude {
    pub use bracket_lib::prelude::*;
    pub const SCREEN_WIDTH: i32 = 80;
    pub const SCREEN_HEIGHT: i32 = 50;
    pub use crate::map::*;
    pub use crate::player::*;    // ← new re-export
}
```

### The Player Struct

In `player.rs`:

```rust
use crate::prelude::*;

pub struct Player {
    pub position: Point,
}

impl Player {
    pub fn new(position: Point) -> Self {
        Self { position }
    }
}
```

Notice we're storing position as a `Point`, not as two separate `i32`s. `Point` is provided by `bracket-lib` and supports arithmetic (`+`, `-`) and methods like `Point::zero()` and `Point::new(x, y)`. Using a domain type instead of raw integers reduces bugs (you can't accidentally pass `y` where `x` was expected).

`Self { position }` is **field init shorthand**: when the variable name matches the field name, you can omit `position: position`.

### Rendering the Player

```rust
pub fn render(&self, ctx: &mut BTerm) {
    ctx.set(
        self.position.x,
        self.position.y,
        WHITE,
        BLACK,
        to_cp437('@'),
    );
}
```

Identical mechanics to map rendering: place an `@` character at the player's position in white on black.

### Moving the Player

This is where input handling and game logic come together:

```rust
pub fn update(&mut self, ctx: &mut BTerm, map: &Map) {
    if let Some(key) = ctx.key {
        let delta = match key {
            VirtualKeyCode::Left  => Point::new(-1,  0),
            VirtualKeyCode::Right => Point::new( 1,  0),
            VirtualKeyCode::Up    => Point::new( 0, -1),
            VirtualKeyCode::Down  => Point::new( 0,  1),
            _ => Point::zero(),
        };

        let new_position = self.position + delta;
        if map.can_enter_tile(new_position) {
            self.position = new_position;
        }
    }
}
```

Detailed breakdown:

- **`&mut self`** — the function mutates the player's position, so we need a mutable reference.
- **`ctx: &mut BTerm`** — we read `ctx.key` (the most recently pressed key, if any).
- **`map: &Map`** — *shared* reference, because we only **read** from the map (calling `can_enter_tile`).

- **`if let Some(key) = ctx.key`** — `ctx.key` is `Option<VirtualKeyCode>`. If a key was pressed this frame, it's `Some(key)`; if not, `None`. The `if let` syntax pattern-matches: the body runs only when we got a `Some`, and `key` is bound to the inner value. This is much cleaner than a full `match` with a `None => {}` arm.

- **The `match key`** — pattern-matches on the key code. The four arrow keys map to direction deltas:
  - Left → `(-1, 0)` (decrease x)
  - Right → `(1, 0)` (increase x)
  - Up → `(0, -1)` (decrease y — remember y grows *downward* on a screen)
  - Down → `(0, 1)`

- **`_ => Point::zero()`** — the wildcard arm catches **any other key**. `Point::zero()` is `(0, 0)`, meaning no movement. This is required because `match` must be exhaustive — every possible `VirtualKeyCode` must be covered, and there are dozens of them.

- **`let new_position = self.position + delta;`** — uses `Point`'s overloaded `+` operator. Equivalent to `Point::new(self.position.x + delta.x, self.position.y + delta.y)` but cleaner.

- **`if map.can_enter_tile(new_position)`** — the guard. If the move is legal (in bounds and onto a floor), commit it. Otherwise, the player stays put. This single check handles both "don't walk off the edge" and "don't walk through walls" in one line — that's the power of composing small functions.

### Integrating Player into `State`

```rust
struct State {
    map: Map,
    player: Player,
}

impl State {
    fn new() -> Self {
        Self {
            map: Map::new(),
            player: Player::new(
                Point::new(SCREEN_WIDTH / 2, SCREEN_HEIGHT / 2)
            ),
        }
    }
}

impl GameState for State {
    fn tick(&mut self, ctx: &mut BTerm) {
        ctx.cls();
        self.player.update(ctx, &self.map);
        self.map.render(ctx);
        self.player.render(ctx);    // ← rendered AFTER map: on top.
    }
}
```

The starting position is the **center of the screen** (`SCREEN_WIDTH / 2`, `SCREEN_HEIGHT / 2` = 40, 25). With integer division, no rounding issues.

**Render order matters.** We draw the map first, then the player on top — otherwise the player's `@` would be hidden under floor `.`s. This is the painter's algorithm: paint from back to front.

Run the program: you'll see an `@` in the middle of a yellow-dotted floor, and you can move it with arrow keys. Since the whole map is `Floor`, you can move anywhere. You won't see walls until we generate them.

---

## Section 12 — Building a Dungeon (Procedural Generation)

Now we generate a real dungeon — a set of rooms connected by corridors, carved from a solid block of rock.

### Creating the Map Builder Module

Add `src/map_builder.rs`. Register it in `main.rs`:

```rust
mod map;
mod map_builder;          // ← new
mod player;

mod prelude {
    pub use bracket_lib::prelude::*;
    pub const SCREEN_WIDTH: i32 = 80;
    pub const SCREEN_HEIGHT: i32 = 50;
    pub use crate::map::*;
    pub use crate::player::*;
    pub use crate::map_builder::*;       // ← new
}
```

In `map_builder.rs`:

```rust
use crate::prelude::*;
const NUM_ROOMS: usize = 20;
```

20 rooms produces a nicely populated dungeon. Try changing it — fewer rooms means a sparser, easier dungeon; more rooms means denser overlap and longer corridors.

### The MapBuilder Struct

```rust
pub struct MapBuilder {
    pub map: Map,
    pub rooms: Vec<Rect>,
    pub player_start: Point,
}
```

- **`map: Map`** — the builder *owns* its own Map. It mutates this map during construction, then hands it over to the game.
- **`rooms: Vec<Rect>`** — list of rooms placed so far. `Rect` is another `bracket-lib` helper representing an axis-aligned rectangle. It supports collision tests (`.intersect(other)`), iteration over interior points, and finding its center.
- **`player_start: Point`** — where the player should appear.

### Step 1: Fill the Map With Walls

The previous map was all floor. The dungeon generation approach is the inverse: start with *all walls* and *carve out* floors.

```rust
fn fill(&mut self, tile: TileType) {
    self.map.tiles.iter_mut().for_each(|t| *t = tile);
}
```

This is functional-style Rust:

- **`self.map.tiles.iter_mut()`** → produces an iterator over `&mut TileType` (mutable references to each tile).
- **`.for_each(|t| *t = tile)`** → for each mutable reference `t`, dereference it (`*t`) and assign `tile` to the underlying location.
- **The asterisk (`*`)** — `t` is `&mut TileType`. Writing `t = tile` would try to change the reference itself (not allowed). Writing `*t = tile` writes through the reference to the value it points to.

Calling `fill(TileType::Wall)` solidifies the entire map into stone.

### Step 2: Carving Random Rooms

```rust
fn build_random_rooms(&mut self, rng: &mut RandomNumberGenerator) {     // ❶
    while self.rooms.len() < NUM_ROOMS {                                // ❷
        let room = Rect::with_size(                                     // ❸
            rng.range(1, SCREEN_WIDTH - 10),
            rng.range(1, SCREEN_HEIGHT - 10),
            rng.range(2, 10),
            rng.range(2, 10),
        );
        let mut overlap = false;                                         // ❹
        for r in self.rooms.iter() {                                     // ❺
            if r.intersect(&room) {
                overlap = true;
            }
        }
        if !overlap {                                                    // ❻
            room.for_each(|p| {
                if p.x > 0 && p.x < SCREEN_WIDTH
                    && p.y > 0 && p.y < SCREEN_HEIGHT
                {
                    let idx = map_idx(p.x, p.y);
                    self.map.tiles[idx] = TileType::Floor;
                }
            });
            self.rooms.push(room);
        }
    }
}
```

Let's walk the annotations:

**❶** The function takes a **mutable reference to a `RandomNumberGenerator`**. Using a single PRNG instance throughout map generation is important: if you seed it with a known value, the *exact same dungeon* is reproduced. This is invaluable for debugging — bug only happens in one specific layout? Seed the RNG and you can reproduce it reliably.

**❷ `while self.rooms.len() < NUM_ROOMS`** — keep going until we've placed 20 rooms. Notice this is a `while` loop, not a `for` loop: we may *attempt* many more iterations than 20 because rooms that overlap get rejected.

**❸** Generate a random room:
- `rng.range(1, SCREEN_WIDTH - 10)` returns a random `i32` in `[1, SCREEN_WIDTH - 10)`. So x can be 1..69.
- Same for y: 1..39.
- Width: 2..9.
- Height: 2..9.

The `-10` margin and `1` lower bound ensure rooms don't touch the very edge (we always want a border of walls).

**❹–❺** Test the candidate against every already-placed room. `r.intersect(&room)` returns `true` if the rectangles overlap. If any do, mark `overlap = true`. A small inefficiency: even after finding one overlap, we continue checking. Cleaner with a `break`, but functionally fine.

**❻** If there was no overlap, carve the room. `room.for_each(|p| ...)` runs the closure on every point inside the rectangle. The inner check verifies the point is within map bounds (defense in depth) before converting to an index and setting the tile to `Floor`. Then `self.rooms.push(room)` records the room so future rooms know to avoid it.

> **The asterisk note revisited:** Note that this `for_each` is `Rect::for_each` (from `bracket-lib`), which iterates over `Point` *by value*, not `Iterator::for_each` over mutable references. So no dereferencing needed here.

### Step 3: Carving Corridors

We want **dog-leg corridors**: an L-shape made of one horizontal segment and one vertical segment, joined at a single corner.

#### Vertical tunnel

```rust
fn apply_vertical_tunnel(&mut self, y1: i32, y2: i32, x: i32) {
    use std::cmp::{min, max};
    for y in min(y1, y2) ..= max(y1, y2) {
        if let Some(idx) = self.map.try_idx(Point::new(x, y)) {
            self.map.tiles[idx as usize] = TileType::Floor;
        }
    }
}
```

Two important details:

- **`use std::cmp::{min, max};` inside the function** — Rust allows `use` declarations inside function bodies. This restricts the import to that function only, keeping namespaces clean.
- **`min(y1, y2) ..= max(y1, y2)`** — the `..=` is the **inclusive range** operator. We need both endpoints carved. `min`/`max` are necessary because the caller could pass `y1 > y2` or vice versa, but Rust's range iterator only goes in increasing order. Without min/max, half the time the range would be empty and no corridor would appear.
- **`try_idx`** — uses the Option-returning version. If the point is off-map, we silently skip it.

#### Horizontal tunnel

Identical structure, but iterates `x`:

```rust
fn apply_horizontal_tunnel(&mut self, x1: i32, x2: i32, y: i32) {
    use std::cmp::{min, max};
    for x in min(x1, x2) ..= max(x1, x2) {
        if let Some(idx) = self.map.try_idx(Point::new(x, y)) {
            self.map.tiles[idx as usize] = TileType::Floor;
        }
    }
}
```

#### Connecting all rooms

```rust
fn build_corridors(&mut self, rng: &mut RandomNumberGenerator) {
    let mut rooms = self.rooms.clone();
    rooms.sort_by(|a, b| a.center().x.cmp(&b.center().x));        // ❶

    for (i, room) in rooms.iter().enumerate().skip(1) {           // ❷
        let prev = rooms[i - 1].center();                          // ❸
        let new = room.center();

        if rng.range(0, 2) == 1 {                                  // ❹
            self.apply_horizontal_tunnel(prev.x, new.x, prev.y);
            self.apply_vertical_tunnel(prev.y, new.y, new.x);
        } else {
            self.apply_vertical_tunnel(prev.y, new.y, prev.x);
            self.apply_horizontal_tunnel(prev.x, new.x, new.y);
        }
    }
}
```

This is the most interesting bit of generation logic, so let's go slowly.

**❶ Sort rooms by center x-coordinate.** `rooms.sort_by(...)` takes a closure that compares two elements. `a.center().x.cmp(&b.center().x)` returns an `Ordering` enum (`Less`, `Equal`, `Greater`). 

> **Why sort?** Without sorting, room order is random — room #1 might be in the top-right, room #2 in the bottom-left. Their corridor would snake diagonally across the whole map, potentially overlapping many other rooms and creating an ugly dungeon. Sorting by x means consecutive rooms in the list are *spatially close* on the x-axis, so corridors stay short and local.
>
> Also note: we **cloned** `self.rooms` (`let mut rooms = self.rooms.clone();`). We sort the *clone*, leaving the original room list intact. This avoids subtle bugs if the original order mattered elsewhere.

**❷ `.enumerate().skip(1)`** — chaining iterator adapters.
- `.enumerate()` wraps each item in a tuple `(index, item)`.
- `.skip(1)` discards the first item.

The result: we get `(1, room_1)`, `(2, room_2)`, ..., `(19, room_19)`. We need to skip room 0 because we look at `rooms[i - 1]` — for `i = 0`, that would be `rooms[-1]`, which is invalid. (Rust would actually panic with an integer underflow because `usize` can't be negative.)

**❸** Get the **previous room's center** and the **current room's center**. These are the two endpoints we want to connect.

**❹** Flip a coin: `rng.range(0, 2)` returns 0 or 1 (exclusive upper bound). 
- If 1: dig the horizontal piece first, then the vertical piece.
- If 0: dig the vertical piece first, then the horizontal.

Both produce an L-shape, just with the corner on opposite ends. This randomness adds visual variety.

### Sidebar: Tuples

The chapter has a sidebar on tuples, used in `.enumerate()` above.

```rust
let tuple = (a, b, c);         // Construct.
let first = tuple.0;            // Access by position.
let (x, y, z) = tuple;          // Destructure.
```

Tuples are anonymous **heterogeneous** records — values of different types grouped together. They're great for ad-hoc return values from a function, but their fields are positional, not named, so for long-lived data you should prefer a `struct`.

> **C# analogy:** Tuples in C# 7+ are very similar: `(int, string) t = (5, "hi"); var (n, s) = t;`.

### Step 4: The Builder Constructor

Now we tie it all together:

```rust
pub fn new(rng: &mut RandomNumberGenerator) -> Self {
    let mut mb = MapBuilder {
        map: Map::new(),
        rooms: Vec::new(),
        player_start: Point::zero(),
    };
    mb.fill(TileType::Wall);
    mb.build_random_rooms(rng);
    mb.build_corridors(rng);
    mb.player_start = mb.rooms[0].center();    // ❶
    mb
}
```

The sequence is:
1. Start with a default `MapBuilder` (Map of all Floor, no rooms, player at origin).
2. **Fill** with walls (now everything is solid stone).
3. **Carve rooms.**
4. **Carve corridors** between them.
5. **❶** Set the player's start position to the center of the **first** room. This guarantees they spawn on a floor tile, not inside a wall.
6. Return the populated `MapBuilder`.

> **Wait, why isn't `Map::new()` an empty wall map already?** Because `Map::new()` defaults to all `Floor`. Then `fill(Wall)` overwrites every tile. It's slightly wasteful, but very clear. We could redesign `Map::new()` to take a default tile, but the current design is simpler.

### Consume the MapBuilder API

In `main.rs`:

```rust
fn new() -> Self {
    let mut rng = RandomNumberGenerator::new();
    let map_builder = MapBuilder::new(&mut rng);
    Self {
        map: map_builder.map,
        player: Player::new(map_builder.player_start),
    }
}
```

We create an RNG, build the dungeon, and then **move** the resulting `map` and `player_start` into our `State`. After this function returns, `map_builder` is dropped (we used everything we needed from it).

Run the game: rooms connected by corridors, your `@` standing in the middle of room #0. You can walk around but can't pass through walls. The dungeon shape changes every run because the RNG is seeded from the system clock by default.

---

## Section 13 — Graphics, Camera, Action

Now we upgrade from ASCII to **bitmap graphics** — moving from a programmer-art roguelike look to actual tile graphics.

### Why Programmer Art?

The chapter makes a critical point about game development workflow:

> "ASCII is a great prototyping tool... Most games feature graphics, but this early in development isn't the right time to find an artist..."

The principle: don't invest heavily in polish until your **gameplay is proven fun**. Programmer art is rough placeholder graphics — good enough to evaluate feel, cheap enough to throw away. If you commission a beautiful sword sprite and then decide to remove swords from the game, you've wasted the artist's time and your money.

### How `bracket-lib` Renders Graphics

`bracket-lib` is fundamentally a **terminal emulator**. Every screen cell shows a glyph from a **font file**. Normally, a font file contains letters and symbols. But the trick is: `bracket-lib` will happily use *any* PNG image as a font file. If you replace the `#` character's bitmap with a wall texture, then drawing `#` on screen draws the wall texture.

So you can have *graphical tiles* without writing a new rendering system — just by editing a PNG with The Gimp (or any image editor).

### Setting Up the Resource

1. Create `resources/` directory in your project root.
2. Place `dungeonfont.png` inside (provided with the book's source code).

The chapter shows a table mapping ASCII characters to graphics:

| Glyph | Represents | Glyph | Represents |
|---|---|---|---|
| `#` | Dungeon Wall | `@` | The Player |
| `.` | Dungeon Floor | `>` | Down Stairs |
| `"` | Forest Wall | `E` | Ettin |
| `;` | Forest Floor | `O` | Ogre |
| `\|` | Amulet of Yala | `o` | Orc |
| `!` | Healing Potion | `g` | Goblin |
| `{` | Dungeon Map | `s` | Rusty Sword |
| `S` | Shiny Sword | `/` | Huge Sword |

The font file is a 16×16 grid of 32×32 pixel tiles — the standard CP437 layout with custom replacements.

> **Credit your artists** (sidebar): The chapter calls out the importance of attributing artists. Dungeon graphics from "Buch" (OpenGameArt), potions from Melissa Krautheim's Fantasy Magic Set, swords from Melle's Fantasy Sword Set, monsters from Dungeon Crawl Stone Soup (CC0 license, packaged by Chris Hamons). Even free assets deserve credit — it's basic professional ethics.

### Graphics Layers

So far we've drawn everything to one virtual canvas. With graphics, this causes a problem: drawing the player `@` *replaces* whatever was underneath, leaving a hard black square around the sprite instead of showing the floor through it.

The fix: **layered consoles**.

- Layer 0 (bottom): the map (walls and floors). Opaque.
- Layer 1 (top): entities (player). **Transparent background** — only the sprite pixels are drawn, so the floor below shows through.
- Layer 2 (future chapter): HUD/UI text.

### The Viewport Problem

Each 32×32 tile is much bigger than an ASCII glyph. If we tried to display an 80×50 grid of 32×32 tiles, the window would be 2560×1600 pixels — bigger than many laptop screens. The solution: render only a smaller **viewport** centered on the player.

Add to your prelude in `main.rs`:

```rust
pub const DISPLAY_WIDTH: i32 = SCREEN_WIDTH / 2;     // 40
pub const DISPLAY_HEIGHT: i32 = SCREEN_HEIGHT / 2;   // 25
```

The window will show 40×25 tiles. At 32 pixels each, that's 1280×800 — manageable.

### Initializing Layered Graphics

The terminal builder gets more elaborate:

```rust
let context = BTermBuilder::new()                                       // ❶
    .with_title("Dungeon Crawler")
    .with_fps_cap(30.0)
    .with_dimensions(DISPLAY_WIDTH, DISPLAY_HEIGHT)                     // ❷
    .with_tile_dimensions(32, 32)                                       // ❸
    .with_resource_path("resources/")                                   // ❹
    .with_font("dungeonfont.png", 32, 32)                               // ❺
    .with_simple_console(DISPLAY_WIDTH, DISPLAY_HEIGHT, "dungeonfont.png")  // ❻
    .with_simple_console_no_bg(DISPLAY_WIDTH, DISPLAY_HEIGHT, "dungeonfont.png")  // ❼
    .build()?;
```

**❶ `BTermBuilder::new()`** — instead of the convenience method `simple80x50()`, use the bare `new()` and configure everything explicitly. This is the **builder pattern**: each method returns `Self`, so you chain configuration calls and finish with `.build()`.

**❷ `with_dimensions(40, 25)`** — sets the logical size (in tiles) of consoles added afterward.

**❸ `with_tile_dimensions(32, 32)`** — each tile is 32×32 pixels in the window.

**❹ `with_resource_path("resources/")`** — tells the engine where to find image files.

**❺ `with_font("dungeonfont.png", 32, 32)`** — registers a font file. The 32×32 here is the size of each glyph *within* the font texture. Usually equal to tile_dimensions, but they could differ for special rendering tricks.

**❻ `with_simple_console(...)`** — adds the **first console layer** (index 0) — the map layer. Opaque background.

**❼ `with_simple_console_no_bg(...)`** — adds the **second console layer** (index 1) — the entity layer. Notice `no_bg`: background pixels are transparent. When you draw the player here, only the sprite shows; the floor underneath remains visible.

### The Camera

Create `src/camera.rs`:

```rust
use crate::prelude::*;

pub struct Camera {
    pub left_x: i32,
    pub right_x: i32,
    pub top_y: i32,
    pub bottom_y: i32,
}

impl Camera {
    pub fn new(player_position: Point) -> Self {
        Self {
            left_x:   player_position.x - DISPLAY_WIDTH  / 2,
            right_x:  player_position.x + DISPLAY_WIDTH  / 2,
            top_y:    player_position.y - DISPLAY_HEIGHT / 2,
            bottom_y: player_position.y + DISPLAY_HEIGHT / 2,
        }
    }

    pub fn on_player_move(&mut self, player_position: Point) {
        self.left_x   = player_position.x - DISPLAY_WIDTH  / 2;
        self.right_x  = player_position.x + DISPLAY_WIDTH  / 2;
        self.top_y    = player_position.y - DISPLAY_HEIGHT / 2;
        self.bottom_y = player_position.y + DISPLAY_HEIGHT / 2;
    }
}
```

The camera stores **the four edges of the visible window in world coordinates**. The player is at the center, so:

- left edge = player.x − half the display width
- right edge = player.x + half the display width
- (same for top/bottom)

Both `new` and `on_player_move` do exactly the same calculation. Why two methods? Conceptually: `new` is "create a fresh camera at player spawn," `on_player_move` is "update an existing camera when the player walks." It's slight code duplication for clarity. You could factor it into one `set_center` method if you preferred.

> **Edge case (not addressed in the chapter):** When the player is near the edge of the map (say, at x=2), `left_x` will be -18. The map will render only what's in bounds and leave the rest blank. This is acceptable behavior; some games clamp the camera so it never extends past the map edges. You could add that later if desired.

### Wiring the Camera

Add it to `main.rs`:

```rust
mod camera;

mod prelude {
    // ... existing ...
    pub use crate::camera::*;
}

struct State {
    map: Map,
    player: Player,
    camera: Camera,    // ← new
}

impl State {
    fn new() -> Self {
        let mut rng = RandomNumberGenerator::new();
        let map_builder = MapBuilder::new(&mut rng);
        Self {
            map: map_builder.map,
            player: Player::new(map_builder.player_start),
            camera: Camera::new(map_builder.player_start),  // ← new
        }
    }
}
```

The camera is initialized at the player's starting position, so the player appears at the center of the screen on the first frame.

### Updated Map Rendering with Camera

```rust
pub fn render(&self, ctx: &mut BTerm, camera: &Camera) {
    ctx.set_active_console(0);                              // ❶
    for y in camera.top_y .. camera.bottom_y {
        for x in camera.left_x .. camera.right_x {
            if self.in_bounds(Point::new(x, y)) {           // ❷
                let idx = map_idx(x, y);
                match self.tiles[idx] {
                    TileType::Floor => {
                        ctx.set(
                            x - camera.left_x,              // ❸
                            y - camera.top_y,
                            WHITE, BLACK,
                            to_cp437('.'),
                        );
                    }
                    TileType::Wall => {
                        ctx.set(
                            x - camera.left_x,
                            y - camera.top_y,
                            WHITE, BLACK,
                            to_cp437('#'),
                        );
                    }
                }
            }
        }
    }
}
```

Three new concepts:

**❶ `ctx.set_active_console(0)`** — directs subsequent draw calls to layer 0 (the map). Without this, drawing could go to whichever layer was last active.

**�2** Now we iterate **world coordinates** from `camera.top_y` to `camera.bottom_y`. These may be **negative** (when the camera is at the map's edge), so we call `in_bounds` to skip off-map positions.

**❸** Critical transformation: `x - camera.left_x` converts from **world** coordinates to **screen** coordinates. If `x = 50` and `camera.left_x = 30`, then the screen position is 20. The viewport is 40 columns wide (0..40), so this gives us valid screen coordinates only if x is between left_x and right_x — which the loop guarantees.

Floor/wall colors changed from yellow/green to white/white because the colors are now applied to **graphical tile sprites**, and white doesn't tint them.

### Updated Player Rendering and Movement

In `player.rs`:

```rust
pub fn update(&mut self, ctx: &mut BTerm, map: &Map, camera: &mut Camera) {
    if let Some(key) = ctx.key {
        let delta = match key {
            VirtualKeyCode::Left  => Point::new(-1,  0),
            VirtualKeyCode::Right => Point::new( 1,  0),
            VirtualKeyCode::Up    => Point::new( 0, -1),
            VirtualKeyCode::Down  => Point::new( 0,  1),
            _ => Point::zero(),
        };

        let new_position = self.position + delta;
        if map.can_enter_tile(new_position) {
            self.position = new_position;
            camera.on_player_move(new_position);    // ← new
        }
    }
}
```

Camera is now `&mut Camera` (mutable). When the player successfully moves, the camera follows. Note: the camera updates *only* on successful moves — if you walk into a wall, the camera stays put, which feels correct.

```rust
pub fn render(&self, ctx: &mut BTerm, camera: &Camera) {
    ctx.set_active_console(1);                       // ← layer 1, transparent
    ctx.set(
        self.position.x - camera.left_x,
        self.position.y - camera.top_y,
        WHITE, BLACK,
        to_cp437('@'),
    );
}
```

The player draws to **layer 1**, the transparent layer, so the floor tile beneath the `@` remains visible. The world→screen translation is the same as for the map.

### Clearing Layers and Calling Functions

The `tick` function must clear *each* layer separately:

```rust
fn tick(&mut self, ctx: &mut BTerm) {
    ctx.set_active_console(0);
    ctx.cls();
    ctx.set_active_console(1);
    ctx.cls();
    self.player.update(ctx, &self.map, &mut self.camera);
    self.map.render(ctx, &self.camera);
    self.player.render(ctx, &self.camera);
}
```

If you forgot to clear layer 1, the player's previous positions would smear across the screen like a snake's trail. If you forgot to clear layer 0, walls and floors would draw on top of themselves each frame — usually no visible difference, but wasteful.

**Argument flow:**
- `player.update` takes `&mut camera` (it may modify it).
- `map.render` takes `&camera` (read-only).
- `player.render` takes `&camera` (read-only).

This is Rust's ownership rules in action: you can have **either** one mutable reference **or** any number of immutable references at a time, but never both. Within `player.update`, no other code holds a reference to the camera; within `map.render` and `player.render`, the camera is shared (read-only) so multiple references would be fine — but here we only need one.

Run the program: a beautiful graphical dungeon scrolls smoothly as your knight (the new graphical player) walks around.

---

## Section 14 — Wrap-Up and the Bigger Picture

Let me weave everything together.

### What You Built

| Concept | File | Purpose |
|---|---|---|
| Modules | All `.rs` files | Organize code into navigable, compilable units |
| Prelude | `main.rs` | One-stop import for common items |
| `TileType` enum | `map.rs` | Tagged representation of map cells |
| `Map` struct | `map.rs` | Flat-vector storage with row-first striding |
| Map helpers | `map.rs` | `in_bounds`, `can_enter_tile`, `try_idx`, `render` |
| `Player` struct | `player.rs` | Position, input handling, rendering |
| `MapBuilder` | `map_builder.rs` | Procedural dungeon: rooms + corridors |
| `Camera` | `camera.rs` | Viewport into a world larger than the screen |
| Layered consoles | `main.rs` | Map below, entities above (with transparency) |

### Key Rust/Systems Ideas You've Practiced

1. **Module system as namespaces** — `mod`, `pub`, `use`, `pub use`, `crate::`, `super::`. The compiler enforces what's visible and what isn't. This is structural, not stylistic — it makes large codebases tractable.

2. **Enums with derived traits** — `#[derive(Copy, Clone, PartialEq)]` for cheap, comparable value types. Pattern matching with `match` provides exhaustive case handling that the compiler verifies.

3. **`Option<T>` instead of nulls** — `try_idx` returns `Option<usize>`. Callers must explicitly handle `None`, eliminating an entire class of runtime errors.

4. **Ownership and references** — `&self` for read-only access, `&mut self` for mutation. Functions accept `&Map` or `&mut Camera` according to whether they read or write. The compiler statically prevents data races.

5. **Iterators and closures** — `iter_mut().for_each(|t| *t = tile)`, `rooms.iter().enumerate().skip(1)`, `rooms.sort_by(|a, b| ...)`. Composable, lazy, and highly optimized.

6. **Row-first striding** — converting 2D coordinates to flat-array indices for cache-friendly storage and traversal.

7. **Procedural generation pipeline** — fill → carve rooms (with collision rejection) → connect with dog-leg corridors. The pattern is reusable: it's the basic algorithm in many roguelikes (Rogue, NetHack, Brogue all use variants).

8. **Layered rendering with a camera** — separating world coordinates from screen coordinates, and stacking transparent layers to support visual overlays.

### Why This Architecture Is Sound

Each module has a **single responsibility**:
- `map.rs` knows about tiles and indexing — nothing about input, players, or rendering decisions like "where is the camera?"
- `player.rs` knows about player position, input, and self-rendering — nothing about how the map is generated.
- `map_builder.rs` knows about dungeon generation — nothing about how the map will be rendered or moved through.
- `camera.rs` knows only about the viewport.
- `main.rs` (`State`) is the **conductor**: it owns everything and orchestrates the tick.

The Law of Demeter holds: the player asks the map "can I enter this tile?" rather than reading the map's internals. The camera doesn't know about the player struct, only about a `Point`. Each module is independently testable and refactorable.

### What's Coming Next

The chapter closes by pointing forward: in the next chapter you'll add **monsters** using an **Entity-Component-System (ECS)** architecture, which generalizes the "things in the world" pattern. The Player and any future Goblin/Orc/Ogre will share components (Position, Render) without inheritance, and **systems** will operate on them in bulk. You're now perfectly positioned for that — you have a map, a player, and the rendering scaffold; the ECS will absorb player.rs into a more flexible structure.

That's the whole chapter. You've built the bones of a roguelike — every step from "empty file" to "graphical dungeon with a moving player" is now under your fingers. Read back through any section that still feels loose, and try the suggested experiment of changing `NUM_ROOMS` to see how the dungeon density changes. Hands-on practice is what cements the concepts.
