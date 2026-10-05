# Maze Race

A single self-contained HTML file that generates a maze and races seven pathfinding
algorithms through it side by side. No build step, no dependencies, no network — open
`index.html` in a browser and it works.

## Use it

Set a width and height, press **Generate maze**, then **Race**. Every selected algorithm
solves the *same* maze simultaneously, each in its own panel, advancing at the same number
of cells per frame — so what you watch is which one wastes the least effort.

| Control | What it does |
|---|---|
| Width / Height | Maze size, 3–200 cells per side |
| Loops | Removes dead ends to add cycles. At 0% the maze is *perfect* (see below) |
| Seed | Reproducible generation — same seed, same maze |
| Speed | Replay rate, 1 to ~1000 cells per frame |

Keyboard: <kbd>Space</kbd> play/pause, <kbd>R</kbd> new maze.

## Frontier Quest

`quest.html` is a game built on the same idea: every search algorithm is one rule for picking
the next cell from the frontier. Instead of watching, you play the rule yourself, one click
per expansion, and the algorithm happens. Also a single self-contained file.

| Mode | What you do |
|---|---|
| Trace levels | Be BFS (oldest cell), DFS (newest), Greedy (lowest h), A* (lowest f = g + h), UCS (lowest g). Three hearts; a wrong pick explains which cell the rule wanted and why |
| Bets | Predict the race (shorter path, bigger frontier, fewer cells explored), then watch it play out |
| Cut the Tree | Minimax with alpha-beta: flip cards left to right and prune only what is proven irrelevant |
| Final Exam | 10 timed questions drawn from a bank of 20 |
| Endless Frontier | 60-second time attack where the rule changes every board |
| Escape the Bots | Run the maze yourself against DFS, BFS and A*, all exploring at 10 cells a second |

Every trace map was chosen by search so its lesson holds however the player breaks ties:
Greedy always ends 4 steps longer than A*, DFS always returns a 16-step path where 8 exists,
and A* with 3 × h always misses the cheapest route. Bet rounds generate a fresh maze each
time and keep only mazes where the race proves the answer. Progress, stars and best scores
are saved in `localStorage`.

## The algorithms

| | Behaviour |
|---|---|
| Breadth-First Search | Floods evenly outward. Always optimal |
| A* (Manhattan) | Beelines toward the goal. Always optimal |
| Greedy Best-First | Pure heuristic — usually the least work, often a worse route |
| Dijkstra | Same frontier as BFS here, but pays for a priority queue |
| Bidirectional BFS | Grows from both ends, meets in the middle |
| Depth-First Search | Dives blindly down corridors |
| Wall Follower | Keeps its right hand on the wall. O(1) memory, never sees a map |

## Two things worth knowing

**At Loops 0% every algorithm finds the "shortest" route, and that means nothing.** The
generator carves a perfect maze — a spanning tree — and a tree has exactly one simple path
between any two cells. Any algorithm that finds *a* path has found *the* path. Raise the
Loops slider and the maze gains cycles; only then do Greedy, DFS and Wall Follower start
returning genuinely worse routes. On a 120×120 at 100% loops: BFS/A*/Dijkstra/Bidirectional
all return 343, while Greedy returns 473, Wall Follower 449 and DFS 611.

**Fewest cells explored is not the same as fastest.** The results table measures real CPU
time separately, averaged over repeated uninstrumented runs, because a single run is faster
than the browser clock can resolve. On an 81×81 maze A* explores far fewer cells than BFS
but takes **3× longer**, and Dijkstra takes **5× longer** — every expansion costs a heap
push/pop with log-n comparisons, while BFS just bumps an array index. A*'s smarter search
doesn't pay for its own overhead until the maze gets large or entering a cell gets expensive.

## How long does it actually take?

Microseconds. The search completes before the first frame is drawn — the animation is a
slow-motion replay, typically tens of thousands of times slower than reality. Measured in
Chrome, 20% loops:

| Maze | Cells | BFS | A* | Dijkstra | Greedy | DFS | Bidirectional |
|---|---|---|---|---|---|---|---|
| 21×21 | 441 | 6 µs | 9 µs | 8 µs | 8 µs | 6 µs | 9 µs |
| 41×41 | 1,681 | 24 µs | 50 µs | 43 µs | 25 µs | 11 µs | 28 µs |
| 81×81 | 6,561 | 68 µs | 228 µs | 335 µs | 87 µs | 82 µs | 88 µs |
| 150×150 | 22,500 | 506 µs | 1.02 ms | 1.11 ms | 167 µs | 152 µs | 409 µs |
| 200×200 | 40,000 | 930 µs | 2.25 ms | 2.04 ms | 318 µs | 162 µs | 895 µs |

Single-threaded JavaScript; compiled C would be several times quicker again.

## Implementation notes

- **Generation** is an iterative randomised depth-first carve over a seeded `mulberry32`
  PRNG, optionally braided afterwards by opening dead ends.
- **Representation** is one `Uint8Array` of wall bits plus a flat `Int32Array` adjacency
  table (4 slots per cell, `-1` for a wall), so the solvers never allocate during search.
- **Rendering** paints each newly expanded cell directly to the visible canvas and restores
  just that cell's wall segments, rather than clearing and redrawing the maze every frame.
- **Solvers** take an optional recorder; passing `null` gives a clean run for timing, so the
  instrumentation that drives the animation never pollutes the measurements.

Note that browsers suspend `requestAnimationFrame` when a tab is hidden, so a race pauses
mid-replay if you switch away and resumes when you come back. The timings are unaffected —
they're computed up front.
