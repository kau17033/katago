# CLAUDE.md

Guidance for AI assistants working in this repository.

## What this project is

KataGo is a Go (baduk/weiqi) engine: MCTS search with a neural network, plus the
full self-play reinforcement-learning training loop that produced its networks.
Two halves:

- **`cpp/`** — the C++14 engine (search, GTP/analysis interfaces, self-play data
  generation), built with CMake, with pluggable GPU backends.
- **`python/`** — PyTorch training, data shuffling, model export, and research
  tooling.

They meet at the trained-model format and at the self-play data files the engine
writes and the trainer consumes.

Docs: `README.md`, `Compiling.md`, `SelfplayTraining.md`, `TrainingHistory.md`,
`docs/`.

## Layout

```
cpp/
  main.cpp, main.h        Subcommand dispatch (katago <subcommand>)
  command/               One file per subcommand: gtp, analysis, benchmark,
                         match, selfplay, gatekeeper, contribute, genbook,
                         evalsgf, gputest, tune, misc
  search/                MCTS: search.cpp, asyncbot, subtree value bias,
                         eval cache, timing, patterns
  neuralnet/             Backends + model description
                         cudabackend, trtbackend (TensorRT), openclbackend,
                         eigenbackend, metalbackend; desc.*, nninputs.*,
                         nneval.* (batching), openclkernels, activations
  game/                  board.cpp, boardhistory.cpp, rules.cpp, graphhash
  program/               play.cpp (self-play/match orchestration), setup.cpp,
                         playutils, selfplaymanager, gtpconfig
  dataio/                sgf.*, trainingwrite.*, numpywrite.*, loadmodel,
                         poswriter, homedata
  book/                  Opening book generation
  distributed/           Client for katagotraining.org distributed training
  core/                  Foundational utilities (rand, threading, config,
                         logging, timing, hashing)
  external/              Vendored dependencies
  configs/               Example .cfg files (gtp_example.cfg, selfplay, match, …)
  tests/                 C++ test sources + models/, data/, gtp/, analysis/,
                         and results/ (expected output baselines)
  run*.sh                Test driver scripts (see Testing)
python/
  train.py               The training loop
  shuffle.py             Shuffles/prepares self-play data for training
  export_model_pytorch.py  Export a checkpoint to the engine's model format
  katago/                The training package (model definitions, data loading)
  selfplay/              Scripts driving the full self-play loop
  migrate_*.py           One-off checkpoint migrations for architecture changes
  tests/                 pytest unit tests for pure-Python logic
  configs/               Training configs
```

## Building

From `cpp/`, pick a backend at configure time:

```bash
cd cpp
cmake . -DUSE_BACKEND=OPENCL          # portable GPU
cmake . -DUSE_BACKEND=CUDA            # CUDA 11+ and matching cuDNN
cmake . -DUSE_BACKEND=TENSORRT        # TensorRT 8.5+
cmake . -DUSE_BACKEND=EIGEN -DUSE_AVX2=1    # CPU-only
make -j$(nproc)
```

Requires CMake ≥ 3.18.2, a C++14 compiler, zlib and libzip. Useful options:

- `-DUSE_AVX2=1` — AVX2/FMA for the Eigen backend (faster, not portable to old CPUs).
- `-DUSE_TCMALLOC=1` — strongly recommended for self-play; glibc malloc
  fragments badly with many threads and parallel games and will eventually
  exhaust memory.
- `-DBUILD_DISTRIBUTED=1` — contribute to public training. Needs OpenSSL, a real
  git clone (not an unzipped source), and a supported version: **`master` will
  not work** — use the latest release tag or the tip of `stable`. Don't bypass
  the version/safety checks.
- `-DNO_GIT_REVISION=1` — skip embedding the git hash.

CI builds OPENCL/Release on Linux (make) and macOS (Ninja).

See `Compiling.md` for Windows/MinGW and per-platform detail.

## Testing

The engine's own test suites are subcommands, wrapped by scripts in `cpp/`:

```bash
cd cpp
./katago runtests                  # what CI runs
./runoutputtests.sh                # → tests/results/runOutputTests.txt
./runsearchtests.sh                # downloads reference nets, writes tests/results/*
./runsearchtestsfp16.sh
./runsearchtestslimited.sh
./runcmdtests.sh
./rungpuerrortest.sh
```

`tests/results/` holds **checked-in expected output**. The scripts `tee` fresh
output over those files, so the real check is `git diff` afterwards — a
non-empty diff in `tests/results/` means behaviour changed. Only update those
baselines when the change is intentional, and say so.

`runsearchtests.sh` downloads several large networks from
katagotraining.org/GitHub releases into `cpp/models/`, so the first run needs
network access and disk space.

Python tests:

```bash
cd python && pytest                 # config in python/pytest.ini
pytest                              # from the repo root; root pytest.ini
                                    # restricts collection to python/tests
```

The root `pytest.ini` exists specifically to stop collection from crawling into
vendored trees under `tmp/` and `cpp/tmp/`; keep its `testpaths` and
`norecursedirs` intact.

## Conventions

- **`.clang-format` is present** — match it for C++ changes.
- **Search and eval changes are strength hypotheses.** Behaviour differences show
  up as diffs in `tests/results/`; there is no unit-test-style safety net for
  playing strength, so changes are validated by playing games/matches
  (`katago match`) rather than assertions.
- **Every neural-net backend must stay in sync.** `neuralnet/` has independent
  CUDA, TensorRT, OpenCL, Eigen, and Metal implementations of the same model;
  an architecture change means touching all of them plus `desc.*` and the
  Python exporter.
- **Model format changes are cross-cutting**: `python/export_model_pytorch.py`,
  `cpp/neuralnet/desc.*`, and the version checks must agree, and old nets should
  keep loading.
- **Self-play data format changes** affect `cpp/dataio/trainingwrite.*` and the
  Python data loading in `python/katago/`; both sides need updating together.
- Configuration is `.cfg` files parsed by `cpp/core/config_parser`; new engine
  options belong there with an entry in the relevant `cpp/configs/*_example.cfg`.
- `python/migrate_*.py` is the established pattern for making existing training
  checkpoints loadable after a model-architecture change — add one rather than
  breaking old checkpoints.

## Notes

- Subcommands are the engine's whole interface: `gtp`, `analysis`, `benchmark`,
  `match`, `selfplay`, `gatekeeper`, `contribute`, `genbook`, `evalsgf`,
  `gputest`, `tunecl`, `runtests`. New functionality usually means a new file in
  `cpp/command/` registered in `main.cpp`.
- Pre-trained nets come from https://katagotraining.org/; the engine does not
  bundle one.
- `SelfplayTraining.md` documents the end-to-end loop (selfplay → shuffle →
  train → export → gatekeeper); read it before changing anything in that chain.
