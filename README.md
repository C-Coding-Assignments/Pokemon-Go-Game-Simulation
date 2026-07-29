# Collection System Simulation

[![Build and Test](https://github.com/garrettbovo/Pokemon-Go-System-Simulation/actions/workflows/build.yml/badge.svg)](https://github.com/garrettbovo/Pokemon-Go-System-Simulation/actions/workflows/build.yml)
![C11](https://img.shields.io/badge/C-11-A8B9CC?logo=c&logoColor=black)
![C++17](https://img.shields.io/badge/C%2B%2B-17-00599C?logo=cplusplus&logoColor=white)
![Warnings](https://img.shields.io/badge/-Wall%20-Wextra-clean-brightgreen)

A modular **C engine** that emulates object-oriented polymorphism without a class in sight, then
gets wrapped in modern C++ across an `extern "C"` boundary with RAII-managed lifetimes.

The simulation models a Pokémon Go–style collection workflow — encounter, capture, and inventory
management over a **493-record dataset** — but the engineering is the point: function-pointer
dispatch tables, a hand-rolled trie with custom character indexing, a doubly linked list with
multi-key sort, and fully manual memory management.

**[→ Run it in your browser on OnlineGDB](https://www.onlinegdb.com/fKNqSTMIxO)** (no account needed)

---

## Quickstart

Requires GCC (or Clang) with C11 and C++17 support.

```bash
git clone https://github.com/garrettbovo/Pokemon-Go-System-Simulation.git
cd Pokemon-Go-System-Simulation/pokemon-cpp
make
./app
```

The build compiles clean under `-Wall -Wextra` with no warnings. `poke.txt` must be in the
working directory — it holds the 493-record dataset loaded at startup.

### A real session

```
What's your name, trainer? > Garrett

Welcome, Garrett, to the Programming I Safari Zone!
You'll have 30 chances to catch Pokémon, make them count!
Which region would you like to visit?

Enter Kanto, Johto, Hoenn, or Sinnoh > Kanto

Traveling to Kanto
==================== MENU ====================
• HUNT      - Go hunting for Pokémon!
• POKÉMON   - See the Pokémon you've caught.
• SORT      - Sort Pokémon you've caught.
• STATS     - See your catch statistics.
• INVENTORY - See your current inventory.
• NAME      - View a Pokémon's Pokédex entry (e.g., BULBASAUR).
• EXIT      - End your adventure.
==============================================
Selection > HUNT

A wild Horsea has appeared!

+-----------------+-------------+
|     Item        |  Inventory  |
+-----------------+-------------+
| 1. Poké Ball    |          10 |
| 2. Great Ball   |          10 |
| 3. Ultra Ball   |          10 |
+-----------------+-------------+
Choose ball (1, 2, or 3) > 3

Threw an Ultra Ball!
Congratulations! You caught Horsea!
```

On exit, the collection is written to `pokemons.txt`:

```
=======================================================================================
                                   Pokémon List
=======================================================================================
| Num   | Name                 | Type         | Region       | Catch % | Atk IV | Def IV | Sta IV |
=======================================================================================
| 116   | Horsea               | Water        | Kanto        | 50      | 10     | 15     | 04   % |
| 13    | Weedle               | Bug          | Kanto        | 50      | 7      | 4      | 10   % |
=======================================================================================
```

---

## Architecture

### Polymorphism in C, via function pointers

C has no vtables. This engine builds its own: three manager structs are initialized at startup,
each holding function pointers to its subsystem's operations. Downstream code never calls an
implementation directly — it dispatches through the interface.

```
main()
  │
  ├── initializeListManager()   → add, sort, reverse, delete, swap
  ├── initializeMenuManager()   → menu, hunt, stats, inventory, display, writeToFile
  └── initializeTrieManager()   → getNode, getCharIndex, insert, search, freeTrie
```

This is the same decoupling a C++ vtable or a COM interface pointer provides, done explicitly —
and it makes the implementations swappable without touching call sites.

### Trie with custom character indexing

Names are indexed into a trie at load time for **O(k) lookup**, where k is name length.

The interesting part is the character index. A naive `c - 'a'` mapping breaks on real data, and
this dataset is full of it — `Farfetch'd`, `Mr. Mime`, `Ho-Oh`, `Porygon-Z`. The indexing
function maps uppercase, lowercase, apostrophes, hyphens, and periods to distinct trie slots, so
those names resolve through the same code path as any other, with no special casing in search.

Verified against the live build:

```
Selection > NAME
Enter Pokémon name > FARFETCH'D

=========================================
 Pokémon Information
=========================================
 Name       : Farfetch'd
 Type       : Normal
 Dex Entry  : "Farfetch'd is always seen with a stalk from a plant of some sort..."
=========================================
```

### Doubly linked list collection

Caught Pokémon live in a heap-allocated doubly linked list. Each node carries a full `Pokemon`
struct plus a pointer to a separately allocated `PokemonStatus` holding catch/seen counts and
individual values — keeping per-encounter state decoupled from static dataset records.

Supports insertion, deletion, multi-key sort (by name, type, or dex number), and in-place
reversal, all dispatched through `ListManager`.

### Hybrid C/C++ boundary

The C engine is exposed to C++ through `extern "C"` linkage in `PokemonWrapper.hpp`. The wrapper
class owns setup and teardown through **RAII**, so the C engine's manual lifecycle is managed by
C++ scope rules rather than by remembering to call cleanup.

### Manual memory management

All allocation goes through `malloc`/`free` with explicit cleanup paths — the trie is freed by
recursive descent, the linked list by forward traversal, both on exit.

---

## Project Structure

```
Pokemon-Go-System-Simulation/
│
├── pokemon-cpp/
│   ├── main.cpp                # C++ entry point and program control
│   ├── PokemonWrapper.cpp      # RAII wrapper over the C engine
│   ├── PokemonWrapper.hpp      # extern "C" boundary declarations
│   │
│   ├── pokemon.c               # Core engine: capture, trie, list, dispatch tables
│   ├── pokemon.h               # Structs, manager definitions, prototypes
│   │
│   ├── poke.txt                # 493-record dataset, loaded at runtime
│   ├── Makefile                # Builds ./app
│   └── .gitignore
│
├── .github/workflows/build.yml # CI: build + smoke test
└── README.md
```

---

## Gameplay Flow

1. **Load** — 493 records parsed from `poke.txt` into a static array; names indexed into the trie
2. **Region select** — one region chosen at start (Kanto, Johto, Hoenn, or Sinnoh); encounters draw from it
3. **Hunt** — a Pokémon is randomly selected; ball choice and per-Pokémon catch rate drive the outcome
4. **Capture** — on success, Attack/Defense/Stamina IVs are randomly assigned and the entry joins the collection
5. **Manage** — sort by name, type, or number; reverse; inspect stats and inventory
6. **Lookup** — `NAME` queries the trie for any Pokédex entry
7. **Export** — on exit, the collection is written to `pokemons.txt` as a formatted table

You start with 10 of each ball — 30 attempts total. The run ends when you exit or run out.

---

## Testing

CI builds the project and runs a smoke test on every push, exercising the full startup path:
dataset load, trie construction, and an apostrophe-containing lookup (`FARFETCH'D`) that would
fail under naive character indexing.

```bash
cd pokemon-cpp && make
printf "Trainer\nKanto\nNAME\nFARFETCH'D\nEXIT\n" | ./app
```

---

## Engineering Notes

- **C11** — function-pointer dispatch tables, manual memory management, pointer-based traversal, defensive input validation
- **C++17** — `extern "C"` interop, RAII resource management, wrapper class design
- **Data structures** — trie with custom character indexing, doubly linked list with multi-key sort
- **Build** — Makefile, GCC/Clang, clean under `-Wall -Wextra`

---

[linkedin.com/in/garrett--ellis/](https://www.linkedin.com/in/garrett--ellis/)
