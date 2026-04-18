# CLAUDE.md

Guidance for AI assistants (Claude Code, etc.) working in this repository.

## Project overview

`rank_photos` is a small command-line tool that helps a photographer sort a
folder of JPEGs by quality. It uses the [Elo rating system](http://en.wikipedia.org/wiki/Elo_rating_system):
the user is shown two photos at a time in a Matplotlib window, picks the
better one, and the program updates each photo's Elo score. After N rounds,
photos are ranked highest to lowest. State is cached so ranking can be
resumed and so newly added photos in the directory are picked up on the next
run.

## Repository layout

- `rank_photos.py` — the entire application (~495 lines, executable via
  `#!/usr/bin/env python` shebang). All classes and the CLI live here.
- `README.rst` — user-facing docs (install, CLI usage, example output).
- `screenshot.jpg` — image embedded by `README.rst`.

There is no `tests/`, no `requirements.txt`, no `setup.py`, no CI config,
and no linter config. The repo has been quiet since 2018; treat it as
mature and stable.

## Run & develop

Install dependencies:

```bash
pip install exifread numpy matplotlib
```

Run on a directory of JPEGs:

```bash
./rank_photos.py -r 3 path/to/photos/
```

CLI arguments (parsed by `argparse` in `main()`):

- `photo_dir` (positional): directory to scan for `*.jpg`.
- `-r/--n-rounds` (int, default `3`): number of complete shuffle-and-pair
  passes through the photo set.
- `-f/--figsize` (two ints, default `20 12`): Matplotlib figure size in
  inches.

The Matplotlib window is interactive — clicking a "Select" button or
pressing `LEFT`/`RIGHT` chooses the better photo. **It needs a real
display**; it cannot run headless or in CI.

## Architecture

All in `rank_photos.py`:

- `Photo` (`rank_photos.py:29`) — wraps one JPEG. Loads it via
  `matplotlib.image.imread`, downsamples (stride pick) to ~800px on the
  long edge for snappy redraws, and applies EXIF orientation with
  `numpy.rot90`. Holds Elo `score` (default `1400.0`), `wins`, `matches`.
- `Display` (`rank_photos.py:161`) — Matplotlib figure with two image
  panels and two "Select" button axes. Blocks on `plt.show()` until the
  user clicks a button or presses an arrow key; the choice is stored in
  `self._choice` (`Photo.LEFT` or `Photo.RIGHT`).
- `EloTable` (`rank_photos.py:258`) — dict of `filename -> Photo`.
  `rank_photos(n_iterations, figsize)` shuffles the keys each round and
  walks pairs, instantiating a `Display` per matchup. `__score_result`
  (`rank_photos.py:336`) does the Elo math with `K = 32`.
- `main()` (`rank_photos.py:366`) — `argparse`, `os.chdir(photo_dir)`,
  loads `ranking_table.json` if present, globs for `*.jpg`, runs the
  ranking loop, then writes `ranking_table.json` and `ranked.txt` back
  into `photo_dir`.

## Elo algorithm

Standard pairwise Elo update (`rank_photos.py:336`):

```
E_a = 1 / (1 + 10 ** ((R_b - R_a) / 400))
R_a' = R_a + K * (actual_a - E_a)
```

Constants: starting rating `1400.0`, K-factor `32.0`. K is settable via
`EloTable(max_increase=...)` but the CLI does not expose it.

## Persistence

Both files live inside `photo_dir` (because `main()` does
`os.chdir(args.photo_dir)`):

- `ranking_table.json` — full state. Schema:
  ```json
  {
    "photos": [
      {"filename": "...", "score": 1432.7, "matches": 2, "wins": 2}
    ]
  }
  ```
  Sorted high score first.
- `ranked.txt` — human-readable table with columns
  `Rank  Score  Matches  Win %  Filename`.

On startup, any `*.jpg` not already in `ranking_table.json` is added with
default Elo `1400.0`.

## Python 2 notice

**This is a Python 2 codebase. Do not migrate to Python 3 without an
explicit user request.** Specific Py2-only constructs in the source:

- `print` statements without parentheses (`rank_photos.py:485`,
  `rank_photos.py:490`).
- `xrange(...)` (`rank_photos.py:308`, `rank_photos.py:314`).
- Implicit integer division: `n_photos / 2` (`rank_photos.py:312`).
- `np.random.shuffle(self._photos.keys())` (`rank_photos.py:306`,
  `rank_photos.py:310`) relies on `dict.keys()` returning a list.

Run the script with `python2`, not `python3`.

## Conventions

- 4-space indentation.
- Two blank lines between top-level definitions; two blank lines between
  methods inside a class as well.
- Spaces around `=` in default arguments (e.g.
  `def __init__(self, filename, score = 1400.0, ...)`). Match the
  existing style rather than PEP 8 here.
- Leading underscore for private methods (`_read_and_downsample`).
- Double underscore for name-mangled methods (`__score_result`).
- Section dividers inside long functions look like
  `#---------------- ... ` comments — keep them if you split a function.

## What's missing (don't add silently)

- No tests, no CI, no `requirements.txt`, no `setup.py`, no formatter.
  If a task seems to imply adding any of these, confirm scope with the
  user first.

## Known quirks and latent bugs (leave alone unless asked)

These are real but intentional-looking artifacts. Do **not** "helpfully"
clean them up as part of unrelated work — they may be load-bearing or
simply not the user's current focus:

- `loosing_photo` is misspelled in `__score_result` (`rank_photos.py:336`,
  parameter and local references).
- `EloTable.rank_photos` calls `np.random.shuffle` on `dict.keys()`
  (`rank_photos.py:306`, `rank_photos.py:310`) — Py2-only behavior.
- `Display._choice` is read from outside the class
  (`rank_photos.py:328`) — a deliberate simplification.
- Bare `except:` swallows all errors when reading the EXIF orientation
  tag (`rank_photos.py:135`).
- Only `*.jpg` is globbed; `*.JPG`, `*.jpeg`, PNG, HEIC, etc. are
  ignored (`rank_photos.py:435`).
