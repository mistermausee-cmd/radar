# Working on this repository

Loaded automatically. These are the rules for this repo specifically; they sit on top
of whatever general working agreement is in force.

---

## Read this first, in this order

1. **`docs/dev/ONBOARDING.md`** — the bootstrap that actually works in a cold sandbox, the
   full gate with its measured outputs, and where his captures live. Every command in it
   was run; the outputs are what it printed. Start there rather than assembling the
   environment from this file.
2. **`docs/dev/ARCHITECTURE.md`** — the system map. Packages, the packet path, the MITM,
   the overlay, the frontend, and a **Traps** section listing the seventeen mistakes that
   have actually cost time here. Read the Traps section even if you read nothing else.
3. **`CHANGELOG.md`** — what has already been measured, decided and refuted. Most "new"
   questions have an answer in here with a number attached. Reading it first is how you
   avoid re-deriving something, or worse, contradicting a measurement.
4. **`docs/project/TODO.md`** — only what is still open, with what is known about each. Its
   top carries a handoff block naming what the next session needs from him, in order.
5. `docs/technical/<subsystem>.md` — the deep dive for the part you are touching, and only
   that one.

`docs/README.md` indexes the rest. `docs/project/DOC_AUDIT.md` is worth one read: it is the
account of how these documents drifted and what now catches it.

---

## What this project is

A fork of `Nouuu/Albion-Online-OpenRadar`. Go backend, vanilla-JS frontend, one
self-contained Windows exe plus a second exe for the overlay. It reads Albion Online's
Photon traffic on UDP 5056 and draws a radar in the browser.

**The fork's difference from upstream is the MITM.** `internal/mitm` sits in the packet
path via WinDivert, substitutes the Diffie-Hellman exchange, derives the AES session
key and decrypts the KeySync event to get the 8-byte XorCode that unlocks other
players' live positions. Upstream is passive-only and its README still says "zero
injection"; that is true of upstream, not of this fork with MITM enabled.

The person you are working for is not a programmer. He supplies pcap captures, session
logs, screenshots and the game client. Every piece of engineering is yours: read the
capture, find the mechanism, measure it, fix it, prove the fix, build the archive.

**He cannot see the terminal.** Surface command output, file contents and errors in the
reply.

---

## The one rule everything else follows from

**An unknown is measured, asserted, or the feature that depends on it does not run.
It is never filled with a plausible value.**

A wire field, a formula or a layout you guessed and made look like a constant is the
worst artifact you can produce here, because it reads as knowledge. It has happened
in this repo and the cost is on the record: the item id table was indexed
`wireId - 1`, every displayed piece of gear was the wrong item, and it looked
plausible for weeks.

Corollaries that come up constantly:

- **A missing Photon parameter is not zero** unless you measured that it is. Two live
  examples that go opposite ways: an absent `param[3]` in `HealthUpdate` **is** zero,
  proved by `Respawn` arriving 5.94-5.98 s later on three of three occurrences; an
  absent `param[3]` in the `GetCharacterEquipment` response is **unknown** and is not
  read at all, because nothing established what it means.
- **A success return is not evidence.** Name the independent observable that confirms
  the effect happened. A call that returns zero having done nothing is the failure
  mode to make loud. The overlay's own shaping is the model: `SetWindowRgn` returning
  non-zero is not the evidence, `GetWindowRgnBox` read back out of the system is.
- **A comparison against our own artifact is not a verification.** 2.8.2 concluded the
  ability icons were correct by comparing his screenshot against the shipped sprite files —
  which only ever proved the renderer draws the file it asks for. Measuring the same files
  against **the CDN they were copied from** found that **63 of the 198 abilities in his
  captures, 31.8%, were showing a different spell's picture**. Anything claiming an asset is
  right has to be checked against the source it came from, not against the copy.
- **A check that only runs on a surface he does not look at is not a check.**
  `SpellTableFreshness` was fed only from `PlayerListRenderer`, and the player card does not
  render in the strip's own window — so on the one screen he actually watches it had zero
  samples and could never fire. Anything added to the card gets asked whether the strip
  window needs it too.
- **Two of the three JS checks will pass a file the browser refuses to load.** Measured at
  2.8.3 on a duplicate top-level declaration: `vitest` reported 76 tests passing on it,
  `tsc --noEmit` said nothing, `eslint` called it a parse error, and the Playwright suite
  failed with the browser's own `SyntaxError`. **Run `npm run lint` before believing
  `vitest`.**
- **Silent failure is a defect.** A decode that yields a finite garbage number instead
  of an error, a state only an event can leave, a counter that is summed and never
  printed, a `-tz` flag whose parameter the function ignores — all of these have been
  real here.

---

## Verification is part of the deliverable

"It should build" is not a result. Paste the command and its real output.

**The full bootstrap and the full gate are in `docs/dev/ONBOARDING.md`**, with the version
of every tool and the outputs each command printed at 2.8.1. Do not reconstruct it from
memory; three things in it are not guessable:

- **Go 1.26 must be downloaded.** The sandbox ships 1.25.1 and `go.mod` pins 1.26.
- **golangci-lint must be exactly v2.12.2**, the version CI pins. 2.13.2 reports a
  different set of findings, which is how 2.8.1 shipped with a red gate: its verification
  block recorded twelve `canonicalheader` findings from 2.13.2 while the pinned 2.12.2 was
  reporting three `misspell` ones nobody had seen. **A linter version is part of a result
  — record it beside the count.** And run `golangci-lint cache clean` before any lint
  result you are going to quote: it caches per package, so a green run right after editing
  one package is green for the other packages for the wrong reason.
- **`tools/uipreview` must be built to `/projects/sandbox/bin/uipreview`** before any
  `uitest` script will start. All nineteen hard-code that path.
- **Clone with `--filter=blob:none`.** 9 seconds against 53, and `.git` 55 MB against
  3.6 GB, because most of the pack is committed build archives. Nothing breaks — measured,
  including `make check-docs`, which is the only thing here that reads history.

Two gate steps are newer than most of the documentation and both found real defects the
first time they ran:

```bash
make check-docs      # documents whose subject code changed after the document did
make check-windows   # the 2178 lines behind //go:build windows, which CI on Linux cannot see
```

`/tmp` is cleared between shell invocations — write scratch files under
`/projects/sandbox/`. `xxd` is not installed; use `od -A x -t x1z`.

### Prove the fix by reverting it

Every fix gets a test that fails when the fix is undone, and the report says which
test and how many assertions. Put the defect back, run the test, paste the failure,
put the fix back. A test that passes both ways is not pinning anything.

### Every claim carries its number

"Faster", "more reliable", "usually" are opinions. Give the figure, how it was
obtained, and on what input — and name the capture when the input is traffic.

**Name the host when the figure is a time.** `bench:hotpath` reports 0.189 ms/frame on one
machine and 0.363 on another, on identical code. A timing number without its host cannot be
compared to anything, and one was quoted as a threshold for four releases.

---

## Recording changes

**`CHANGELOG.md` is not optional.** Any change that reaches a build gets an entry, and
the format plus the four questions an entry has to answer are in that file's header.
Read it before writing one.

The division of the four project documents:

| File | Holds |
|---|---|
| `CHANGELOG.md` | what changed, per version, with the measurement and what pins it |
| `docs/project/TODO.md` | **only what is still open** |
| `docs/dev/ARCHITECTURE.md` | how the system works now |
| `docs/technical/<x>.md` | how one subsystem works, and what was measured to establish it |

When something closes, it moves from TODO.md to CHANGELOG.md — it is not left in both.
When a measurement refutes an earlier claim of ours, that gets its own entry naming the
claim and the number that killed it; the old entry is not silently edited. The record
of having been wrong is the most useful part of the file.

**And the technical document gets read.** This is the step that was missing and it is why
`HARVEST_EVENTS.md` described event codes that had been regenerated out from under it four
months earlier. A `docs/technical/` file that argues toward a decision becomes wrong the
moment the decision ships — `OVERLAY_TRANSPARENCY.md` described the two-window build as one
of five future options for two releases after it shipped. `make check-docs` names the
documents whose subject moved; clearing a row means reading the document and committing
whatever it needed.

The GitHub release notes are a different artifact, generated by `git-cliff` from
conventional commit subjects (`cliff.toml`). Do not treat it as a substitute.

---

## Conventions

- **Comments carry the reason, never the line below them.** The house style is to
  record *why* a value is what it is and what the alternative cost — measured numbers
  belong in the comment next to the constant. Match it. And when the constant moves, the
  number in the comment moves with it: `overlay_api.go` narrated a 187 px row and a 40 px
  cell for a release after they became 271 and 44.
- **Commit subjects:** `<area>: <what changed, in plain words>`, lowercase, no
  trailing period, e.g. `players: read the health the spawn packet has been carrying
  all along`. Bodies carry the measurement.
- **Read the neighbouring files first.** A correct patch in the wrong idiom is a
  rejected patch.
- **Every document declares its subjects.** A new `docs/` file opens with
  `<!-- docstamp: verified <version> | subjects: <paths> -->`, or `subjects: none` written
  out if it documents a decision rather than code. `make check-docs` fails on a document
  without one.
- **Frontend tests are `_<name>.test.js`, co-located.** The underscore is mandatory:
  `embed_prod.go` embeds `web/scripts` without `all:`, which is what keeps test files
  out of the production binary. CI enforces it.
- **Pure logic goes in its own module so it can be tested** without a DOM or a live
  server — `WireHealth.js`, `PlayerAfk.js`, `FrameThrottle.js`, `ExactItemPower.js`
  are the pattern: a small file with the measurement in the doc comment and a test
  file beside it.
- **Overlay logic goes in `internal/ui/overlayshape`, not `cmd/overlay`.** All of
  `cmd/overlay` is `//go:build windows` and has **no test files**; the only gate that
  compiles it is `make check-windows`. `overlayshape` is portable, carries 34 Go tests, and
  is why the geometry defects of v58 and v59 were fixable without a Windows machine.
- **Never hand-edit the Photon code tables.** `web/scripts/utils/EventCodes.js`,
  `OperationCodes.js` and the Go mirrors are generated from the client's own enums.
  Codes above ~473 renumber on game patches; use the generated symbol, never a
  literal. And when they are regenerated, the documents that quote numbers from them are
  what goes stale — that is exactly what happened to `HARVEST_EVENTS.md`.
- **Real game data backs every test that touches the database layer.** Load it via
  `web/scripts/__fixtures__/realDatabases.js`. A mock that lies in sync with a wrong
  assertion hides exactly the class of bug this project keeps hitting.

---

## Releasing

1. Bump `cmd/radar/versioninfo.json` **and `cmd/overlay/versioninfo.json`** — four
   `Major`/`Minor`/`Patch` fields plus the two version strings, in each.
2. Write the `CHANGELOG.md` entry. Name the golangci-lint version beside its finding count.
3. Move anything that closed out of `docs/project/TODO.md`.
4. **Run `make check-docs` and clear every row it names.** A release that leaves a document
   describing code it no longer describes is how four months of drift accumulated.
5. Rewrite `packaging/OpenRadar-MITM/README.txt` for the new build. It is **in
   Russian**, addressed to him, and it leads with the reproduction rather than the
   fix. Numbers, not adjectives.
6. `make windivert-bundle VERSION=<x.y.z>` — produces `dist/OpenRadar-MITM/` and
   rewrites the root `OpenRadar-MITM.zip`, which **is** committed. That is ~60 MB added to
   the pack permanently, every release, because a zip does not delta-compress. The pack is
   3.6 GB and ~4.9 GB of raw blob is exactly this, four of the five paths no longer even in
   the tree. The clone cost is already solved — clone blobless, see `ONBOARDING.md` — but
   the growth is not. `docs/project/TODO.md` → *Repository hygiene* has the measurement and
   the one decision left: publish the bundle as a GitHub Release asset instead.
7. Check the artifact, do not assume it:
   - the new version appears **twice in UTF-16** (the version resource) and **once in
     ASCII** (the ldflags string);
   - the previous versions appear **zero** times;
   - beware false positives — `2.7.0` shows up seven times in ASCII as
     `github.com/clipperhouse/uax29/v2 v2.7.0`, a dependency that shares the number;
   - `level="requireAdministrator"` is present in the radar's manifest (the overlay's is
     deliberately `asInvoker`);
   - grep the exe for a file name from anything newly added, to confirm it embedded;
   - build a Linux binary with the same ldflags and run `-version` to see the string.
8. Full gate green, including `make check-docs` and `make check-windows`.
9. Commit by theme, push to the current branch, and give him the branch or PR link.

The bundle README and the CHANGELOG entry say what was **not** done and why — never as
a promise to do it later.

---

## Offline tools

There is a lot here and none of it is discoverable by reading the source tree.

| Tool | Answers |
|---|---|
| `go run ./cmd/pcapanalyze <pcap>` | event and operation histograms with symbol names |
| `... -dump <event> -dump-count N` | every parameter of the first N occurrences of an event |
| `... -dump-op <operation>` | the same for operation requests **and** responses |
| `... -timeline <codes>` | one line per occurrence: `t=<s> code=<c> entity=<param0>` — this is the tool for "how long was this entity silent" |
| `go run ./cmd/movedump <pcap>` | move-event coordinate decode diagnostics |
| `go run ./tools/fragdiag -in <pcap>` | replays the MITM's fragment-hold state machine against real wire timings: how many datagrams are held, how many miss the deadline |
| `go run ./tools/flowdiag -in <pcap>` | per-connection flow view; `-mode spawns` needs `-xor <code>,...` |
| `go run ./tools/xorrecover -in <pcap>` | recovers a XorCode from a capture by brute force over position blobs; this is how the keysync test fixtures were made |
| `go run ./tools/offset-validate` | cross-validates the mob database typeId offset against a capture |
| `go run ./tools/photon-dump` | extracts per-scenario pcap fragments and WS-level JSON fixtures |
| `go run ./tools/anonymize-pcap` | scrubs MACs, IPs, timestamps and optionally a name; run before committing any fixture |
| `go run ./tools/docstamp` | names every document whose subject code changed after the document did; `--check` for CI, `--all` to see the in-step rows too |
| `npx tsx tools/spell-id-probe.ts --spawns <dump>` | re-verifies the spell numbering after a data update, with a `--without-channeling` control |
| `npx tsx tools/cooldown-window-probe.ts` | the probe that rejected `CastStart` as the cooldown trigger |
| `npx tsx tools/cooldown-fit.ts` | fits the cooldown-reduction transform from a purpose-made capture. The transform was settled as the identity in 2.6.0; this is the tool that would re-open it |
| `go run ./tools/uipreview -feed` | serves the real frontend with a synthetic entity feed — no libpcap, no game. This is what the Playwright suite drives, and it must be built to `/projects/sandbox/bin/uipreview` for the suite to start at all |

---

## Sandbox and GitHub

Repo: `mistermausee-cmd/albionradar`. His captures and logs arrive in a **separate**
repo, `mistermausee-cmd/anliiska`, branch `all_data`, under `logs/`:
`captures/*.pcap`, `console/*.log`, `sessions/*.jsonl` (server events, code and
param count only — no parameter values), `debug/*.jsonl` (frontend logs),
`errors/*.log`. Fetch it with `git -C /projects/sandbox/anliiska fetch origin
all_data && git reset --hard origin/all_data`.

`git` and `gh` are pre-authenticated. Use `gh api` REST endpoints; the `gh pr` and
`gh issue` subcommands are GraphQL-backed and fail in this environment.

**The repo's PR convention is a chain**: each branch targets the branch before it rather
than `main`. Check `gh api "repos/mistermausee-cmd/albionradar/pulls?state=all&per_page=10"`
before opening one. There are **no GitHub releases**, which is why the bundle zip is
committed.

**Confirm the branch before touching anything** — `git branch --show-current` — and stop if
it is not the one he named. The branch is chosen when the session is created; a URL pasted
into a message checks nothing out. Do not re-clone or switch to make the tree match a link:
his uncommitted state and the open PR live on the branch that is actually there.

Note what the chain cost until 2.8.4. `ci.yml` triggered on `pull_request: branches: [main]`,
so a PR was only ever checked when it closed the chain — 14 pull requests over the project's
history, 2 into `main`, and **one CI run in total**. The entire MITM was merged without CI
having looked at it. The filter is gone; every PR is checked now.
