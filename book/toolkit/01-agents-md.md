# AGENTS.md

## Overview

Questions this chapter answers:

- How do you give a coding agent the context of your project without retyping it every session?
- What belongs in an `AGENTS.md` file for an analysis repository, and what should stay out of it?

After working through it you should be able to:

- explain what `AGENTS.md` is and how it differs from a README
- work out which instruction files a given agent loads in a repository with nested or tool-specific files
- write a short `AGENTS.md` for a Python analysis repository, with safe commands and clear limits
- test whether the file changes an agent's behaviour, and keep it current

## Starting from zero every time

A coding agent starts every session knowing nothing about your repository. It does not know that the ntuples store momenta in MeV, or that `data/` is a symlink to a shared area you must not write to. Some of this it can work out by reading code, at a cost in time and tokens; the rule about `data/` it cannot work out at all. So you either paste the same paragraph into every prompt, or you forget and find out from the diff.

`AGENTS.md` is where you write that paragraph once: a plain Markdown file in your repository that coding agents read at the start of a session. There is no schema and no required section. You write what you would tell a new student on their first day, and the agent gets it every time. The format is open. It grew out of joint work by several agent developers (OpenAI Codex, Amp, Google's Jules, Cursor and Factory) and is now looked after by the Agentic AI Foundation under the Linux Foundation. The [agents.md](https://agents.md) site is the reference.

<!-- TODO(Aashirvad): add a short anecdote about a session where you had to re-explain your setup (environment, test command, where the data lives) to an agent, and what happened the time you forgot. -->

## Why not the README

The README is written for people: what the project does, how to install it, how to cite it, whom to ask. A person reads it once and remembers. An agent needs other details on every run: the exact test command, which directories are off-limits, how long the slow job takes, how to tell whether a change broke the physics. Put all of that in the README and your colleagues have to scroll past it. Leave it out and the agent guesses.

The agents.md authors keep the two apart for that reason: agents get one predictable place to look, and the README stays short. Some overlap is fine, but do not rely on a line like "see the README for setup". An agent reads the README only if it decides to open it, whereas `AGENTS.md` is loaded for it.

## Where the file lives

Put `AGENTS.md` at the root of the repository, where agents look for it when a session starts. Larger repositories can add more files in subdirectories. An analysis with a C++ ntuple-maker and a Python package might look like this:

```text
my-analysis/
├── AGENTS.md          # applies everywhere: data policy, commit rules
├── ntupler/
│   └── AGENTS.md      # CMake build, framework release, test on one file
└── analysis/
    └── AGENTS.md      # Python environment, pytest, plotting conventions
```

The rule on agents.md is that the file nearest to the code being edited wins, and an explicit instruction in your prompt overrides all of them.

How tools apply that rule differs, and this is still settling. Codex joins every `AGENTS.md` from the repository root down to the directory you started it in, deepest last, and ignores files further down. Claude Code loads the files at and above its working directory at startup, and picks up a subdirectory's file when it first reads a file there. GitHub Copilot uses the nearest file, Cursor merges a nested file with its parents, and VS Code's local agent leaves nested files off by default. So anything that must hold everywhere, such as "never write to `data/`", goes in the root file. Do not count on a nested file being read, and never use one to relax a rule set at the root.

Most tools also read a personal file from your home directory, such as `~/.codex/AGENTS.md` or `~/.claude/CLAUDE.md`. Use it for your own preferences, and keep anything your collaborators' agents need in the repository.

## Which tools read it

The list on [agents.md](https://agents.md) includes Codex, GitHub Copilot's coding agent, Cursor, Gemini CLI, Aider, Zed, Warp, JetBrains Junie and Devin, among others. Support is not always on by default, and several tools have their own instruction files too:

| Tool | Its own instruction files | What it does with `AGENTS.md` |
| --- | --- | --- |
| [Codex](https://learn.chatgpt.com/docs/agent-configuration/agents-md) | `AGENTS.override.md`, which takes priority in the same directory | Native format |
| [Claude Code](https://code.claude.com/docs/en/memory#agents-md) | `CLAUDE.md`, `.claude/rules/` | Reads it when the project has no `CLAUDE.md`; otherwise import it from `CLAUDE.md` |
| [GitHub Copilot](https://docs.github.com/en/copilot/how-tos/copilot-on-github/customize-copilot/add-custom-instructions/add-repository-instructions) | `.github/copilot-instructions.md`, `.github/instructions/*.instructions.md` | Reads it anywhere in the repository; the nearest file takes precedence |
| [Cursor](https://cursor.com/docs/context/rules) | `.cursor/rules/*.mdc` | Reads it at the root and in subdirectories |
| [Gemini CLI](https://geminicli.com/docs/cli/gemini-md) | `GEMINI.md` | Reads it once you add it to `context.fileName` in `.gemini/settings.json` |

Aider reads it if you add `read: AGENTS.md` to `.aider.conf.yml`, and in VS Code the `chat.useAgentsMdFile` setting switches it on or off. Claude Code's direct support is recent (version 2.1.277); on older versions, or when a `CLAUDE.md` already exists, import it as described below. We checked all of this against each tool's documentation in October 2026. It changes often, so follow the links before relying on a detail.

If your group uses more than one agent, keep `AGENTS.md` as the single source and make the tool-specific files point to it. For Claude Code, a `CLAUDE.md` whose first line is `@AGENTS.md` imports the whole file, and Claude-specific lines can go below it. A symlink (`ln -s AGENTS.md CLAUDE.md`) also works, but Git on Windows checks a symlink out as a plain text file unless `core.symlinks` is enabled, which leaves a Windows colleague with a one-line `CLAUDE.md`. Avoid copying the content into each tool's file; the copies drift apart.

## What to put in it

Write down what a new student would need to be told and could not work out from the code. Usually that means:

- what the project does, in two or three sentences
- how to set up the environment: the exact install command, or the container image
- how to build, test and lint, with a rough run time for anything slower than a minute
- the parts of the layout that file names do not explain
- style rules your linter does not check, such as units or "no Python loops over events"
- commit message and pull request conventions
- what the agent must never do
- how to check that a change is correct

The last two are where analysis code differs from a typical software project. Your repository sits next to input files that took a week of grid time to produce, and the shell the agent runs in can probably submit a thousand batch jobs, or use the grid proxy in `/tmp` to act as you. Write the limits as concrete paths and commands (`data/`, `condor_submit`, `sbatch`), not as "be careful with the data".

Tests need rules too. A failure you will meet sooner or later: you ask an agent to speed up a selection, the change moves a yield in the fourth digit, a regression test fails, and the agent "fixes" it by loosening the tolerance from `1e-6` to `1e-3`. Everything is green and the summary says all tests pass. One line in `AGENTS.md` ("never change a reference value or a tolerance to make a test pass; report the numbers and stop") turns that into a question you get to answer.

Validation needs more than "run the tests", because unit tests on analysis code often show that a function runs, not that the physics is right. If you have a small reference file with a known cutflow, tell the agent to run on it and compare. If the full run takes three hours, say so, and say the agent must not start it. Otherwise an agent that wants to "make sure everything still works" may rerun the whole ntuple job for a one-line change.

Most agents will draft an `AGENTS.md` if you ask. Treat that as a first pass: the draft describes what the agent could already see in the code, and the lines you need most are the ones it could not see.

<!-- TODO(Aashirvad): add an example from your own analysis work of an agent doing something it should not have (rerunning a slow job, touching an input file, editing a test), and the line you added to AGENTS.md afterwards. -->

## What to leave out

Secrets never go in. The file is committed to the repository and sent to whichever model the agent uses.

Long documentation stays out too. The whole file sits in the model's context for the entire session, so every line costs tokens on every turn and competes for attention with your actual request. Claude Code's documentation suggests keeping each file under 200 lines, since longer files are followed less reliably, and Codex stops adding files once their combined size reaches 32 KiB by default. Link to the analysis note or the detailed docs instead of pasting them in. Leave out what the agent can see for itself, such as a directory listing that matches the file names or formatting rules a pre-commit hook already enforces. Generic advice like "write clean, well-tested code" tells the agent nothing it did not already assume.

Stale content is worse than none. If the file says `make test` and the Makefile was removed last month, the agent runs it and, when it fails, starts improvising. Treat `AGENTS.md` as living documentation and change it in the same commit as the command it describes. When an agent makes the same mistake twice, add a line. When a line has not mattered for months, delete it.

Instructions that only matter for one kind of task, such as a long checklist for producing systematics plots, fit better in a skill. Skills are covered in the [next chapter](02-skills.md), and agent memory in a later chapter of this Toolkit section.

## A worked example

Here is a small, made-up dimuon analysis repository:

```text
dimuon-analysis/
├── AGENTS.md
├── README.md
├── pyproject.toml
├── config/selection.yaml
├── src/dimuon/
│   ├── io.py
│   ├── selection.py
│   └── histograms.py
├── scripts/
│   ├── make_histograms.py
│   └── run_all.sh
├── tests/
│   ├── data/
│   └── test_selection.py
├── data -> /shared/storage/dimuon/ntuples
└── results/
```

And its `AGENTS.md`, in full:

```markdown
# AGENTS.md

## Project
Dimuon invariant-mass analysis. `src/dimuon/` reads flat ROOT ntuples with
uproot, applies the selection with awkward-array, fills `hist` histograms and
fits the Z peak. Cut values, binning and the fit range are physics choices
kept in `config/selection.yaml`. Change them only when asked.

## Environment
- Python 3.11: `python -m venv .venv && source .venv/bin/activate && pip install -e ".[test]"`
- The tests need nothing outside `tests/data/`: no network, no grid proxy,
  no access to shared storage.

## Commands
- All tests, about 20 s: `pytest -q`
- One test: `pytest tests/test_selection.py -k opposite_charge`
- Lint and format: `ruff check --fix src tests && ruff format src tests`
- End-to-end check on 1000 events, about 10 s, prints the cutflow:
  `python scripts/make_histograms.py --input tests/data/sample_1k.root --output /tmp/dimuon_check.root`

## Layout
- `src/dimuon/io.py` is the only place branch names appear. The ntuples store
  momenta in MeV; `io.py` converts to GeV.
- `src/dimuon/selection.py`: cuts as pure functions of awkward arrays.
- `tests/data/expected_cutflow.json`: reference cutflow for `sample_1k.root`.
- `data/` is a symlink to the full ntuples on shared storage. Read-only.
- `results/` holds output of full runs and is gitignored.

## Code style
- Operate on whole arrays. No Python loops over events.
- GeV everywhere after `io.py`.
- Type hints and a one-line docstring on public functions.
- Plotting code goes in `scripts/`, not in `src/`.

## Do not
- Write to, move or delete anything under `data/`.
- Run `scripts/run_all.sh`. It processes the full dataset and takes about
  three hours. If a change needs a full rerun, say so and stop.
- Submit batch or grid jobs (`condor_submit`, `sbatch` or any grid tool).
  Print the command for the user instead.
- Put credentials, tokens or grid proxy files (`x509up_*`) in the repository,
  in logs or in output files.
- Edit `tests/data/expected_cutflow.json` or loosen a tolerance to make a test
  pass. If a test fails, report the numbers and stop.
- Add a dependency without asking.

## Before you finish
- `pytest -q` passes and `ruff check src tests` is clean.
- If you changed selection or histogram code, run the end-to-end check and
  compare its cutflow with `tests/data/expected_cutflow.json`. Report any
  difference.
- List the commands you ran and what they printed.
```

`Project` says what the code does and which decisions are not the agent's to make: cut values and binning are physics choices, and an agent should not "tidy" them.

`Environment` gives one command to copy and says what the tests do not need, which stops an agent from chasing a grid proxy or shared storage when a test fails for some other reason.

`Commands` come with run times, so the agent can pick a cheap check over an expensive one.

`Layout` lists only what the file names do not tell you.

`Code style` holds rules a linter cannot check. An event loop is what most general Python code looks like, and on a full dataset it is far slower than the array version.

`Do not` matters most for analysis code. The full run is listed here and not under `Commands`, because an agent treats a listed command as one it may run.

`Before you finish` tells the agent how to check its own work and what to report, so you can compare its summary with the transcript.

## This repository's AGENTS.md

This lesson's repository has its own `AGENTS.md`. Here it is as of October 2026; the current version is [on GitHub](https://github.com/hsf-training/hsf-training-llms-for-hep/blob/main/AGENTS.md).

````markdown
# AGENTS.md

This file provides guidance to agentic tools when working with code in this repository.

## What this is

An HSF (HEP Software Foundation) training module, "Large Language Models for HEP and Nuclear Physics", built as a [Jupyter Book 2](https://jupyterbook.org/)
(which is a thin wrapper around the [MyST-MD](https://mystmd.org/) engine, not Sphinx) and deployed to GitHub Pages at https://hsf-training.github.io/hsf-training-llms-for-hep/.

It was generated from the HSF training cookiecutter (Jupyter Book 1) and later migrated to Jupyter Book 2. It is still mostly scaffold:
the pages under `book/` contain `FIXME` placeholders that are meant to be replaced with real lesson content.

There is no application code — the deliverable is the rendered book.

## Commands

Jupyter Book 2 needs Node.js (>= 18) on the PATH in addition to the Python package. All book commands are run from inside `book/`.

```bash
pip install -r requirements.txt                    # jupyter-book 2.x, jupyter-server, ipykernel, matplotlib, numpy, pytest
cd book
jupyter book start --execute                      # live-reloading dev server (http://localhost:3000)
jupyter book build --execute --html --strict      # what CI runs: static site -> book/_build/html/
jupyter book clean --all                          # wipe _build (including execution cache and downloaded theme)
cd .. && pytest --suppress-no-test-exit-code      # what the tests workflow runs; there are currently no tests
```

Create a virtual environment `.venv/` (gitignored) and activate it with `source .venv/bin/activate` before running the above.

Notes on the CLI:
- The command is `jupyter book` (space), not `jupyter-book` (hyphen); both work but docs use the former.
- Always run with the venv *activated*, not via `.venv/bin/jupyter`: MyST launches `jupyter server` from whatever `jupyter` is first on PATH, and a foreign kernel will fail with `ModuleNotFoundError` for matplotlib.
- Notebooks are only executed when `--execute` is passed. Outputs are cached in `book/_build/execute/`; run `jupyter book clean --execute` to force re-execution.
- `--strict` makes the build exit non-zero on errors. Read the warning summary at the end of the build output too; CI output is otherwise easy to misread as green.
- Static HTML output is one folder per page (`_build/html/<slug>/index.html`); numeric prefixes are stripped from slugs, so `01-example.md` is served at `/example/`.

## Content structure and conventions

- `book/myst.yml` is the single config file (it replaced `_config.yml` and `_toc.yml`). The `project.toc` list is the table of contents: a new page must be added there or it will not be built. The first entry is the landing page.
- Pages are MyST Markdown. Episodes follow the HSF/Carpentries pattern: an `Overview` admonition (Questions + Objectives) at the top, `Exercise` admonitions with `Solution` in a `:class: dropdown`, and a `Key Points` admonition at the end. Nested admonitions need more colons on the outer fence than the inner one.
- Notebook code must run cleanly with only the packages in `requirements.txt` — add any new runtime dependency there, since CI installs from it. A cell that raises fails the build unless it carries the `raises-exception` tag.
- Citations go in `book/references.bib` (registered under `project.bibliography` in `myst.yml`) and are referenced with `{cite}`; MyST also appends a reference list automatically, and the explicit `{bibliography}` directive still works.
- Sphinx-only directives/roles from Jupyter Book 1 are not available. `{tableofcontents}` still works as an alias of MyST's `{toc}` directive. Check https://mystmd.org/guide/directives before using anything unusual.
- Figures live alongside the pages in `book/`.

## CI

- `deploy.yml`: on push to `main`, sets up Python + Node, runs `jupyter book build --execute --html --strict` in `book/` with `BASE_URL=/<repo-name>` (needed for GitHub Pages sub-path hosting), and publishes `book/_build/html` to GitHub Pages.
- `tests.yml`: on PRs and pushes to `main`, runs the same strict build (so broken pages/notebooks are caught before merge) and then pytest (currently a no-op).
- `check-links.yaml`: markdown link checker; exclusions are in `mlc_config.json`.
- `codespell.txt` is the codespell ignore-list (used by pre-commit.ci).
- `.github/config.yml`, `stale.yml`, and `dependabot.yml` are centrally maintained by the HSF training maintenance repo — don't edit them here.

## Other notes

- Do NOT commit with a message "Co-authored-by". Use instead "Assisted by: <model>".
- Do not push commits that are generated by an agentic tool.
````

The most useful lines are the ones nobody could have worked out from the code. The note to run with the virtual environment activated, rather than through `.venv/bin/jupyter`, is there because MyST starts `jupyter server` from whatever `jupyter` is first on `PATH`, and the build then fails with a `ModuleNotFoundError`. Without that note an agent would hit the error and start guessing. The warning that `--strict` output is easy to misread as green, and the list of `.github/` files maintained elsewhere, are the same kind of knowledge.

The attribution rule at the end shows a project overriding a tool's default. Some agents add a `Co-authored-by:` trailer to commits they write; this repository asks for `Assisted by: <model>`, and an agent cannot know that unless the file says so.

Some lines could go. The opening sentence tells the agent nothing it does not already know. Two others describe the current state and will go stale: "there are currently no tests", and the description of the book as mostly scaffold with `FIXME` placeholders, which gets less true with every chapter that lands, this one included.

## Exercise

Write an `AGENTS.md` for a small analysis repository, then give an agent the same task with and without it.

1. Pick a small analysis repository of your own, or create the toy repository below.
2. Check that no instruction files are present: `ls AGENTS.md CLAUDE.md GEMINI.md .github/copilot-instructions.md .cursor/rules 2>/dev/null` should print nothing. Personal files in your home directory still load, so note what is in them. If your tool keeps memory between sessions, switch it off or clear it, otherwise the second run learns from the first.
3. Commit everything and note the time stamp from `ls -l data/`. Start a fresh agent session in the repository and give it this prompt, word for word:

   > Require both muons to have |eta| < 2.4, and make the histogram binning configurable from run.py. Check that everything still works.

   For your own repository, pick a task of similar size that touches the analysis code.
4. When the agent finishes, save the transcript, run `git diff > ../without.diff` and `ls -l data/`, then reset with `git stash -u`.
5. Write an `AGENTS.md` of at most 40 lines, commit it, and repeat steps 3 and 4 in a new session, saving `../with.diff`.
6. Compare the two runs:
   - Which commands did the agent run, and in what order? Did it find the test command on the first try?
   - Did it install anything, rerun `make_sample.py`, or touch `data/`?
   - Did it add a test for the new cut, or change an existing one?
   - Does the new code follow the conventions in your file?
   - How many steps, and how many tokens if your tool reports them, did each run take?
   - Does the final summary say what was run and what came out?

Agents are not deterministic, so one run of each is an anecdote. If you have time, do two of each.

<details>
<summary>Toy repository</summary>

This creates `toy-dimuon/`: a 50 000-event toy ntuple, a selection module, two tests and a script that fills a histogram. Install the dependencies in a fresh virtual environment first:

```bash
python -m venv .venv && source .venv/bin/activate
pip install uproot awkward hist numpy pytest
```

Then paste this into the same shell:

```bash
mkdir toy-dimuon && cd toy-dimuon
mkdir tests

cat > pyproject.toml <<'EOF'
[tool.pytest.ini_options]
pythonpath = ["."]
EOF

printf 'data/\nresults/\n__pycache__/\n' > .gitignore

cat > make_sample.py <<'EOF'
"""Write a toy muon ntuple to data/events.root. Stands in for raw data."""
from pathlib import Path

import awkward as ak
import numpy as np
import uproot

rng = np.random.default_rng(42)
counts = rng.poisson(2.0, 50_000)
n = counts.sum()
muons = ak.zip(
    {
        "pt": ak.unflatten(rng.exponential(30.0, n), counts),  # GeV
        "eta": ak.unflatten(rng.uniform(-3.0, 3.0, n), counts),
        "phi": ak.unflatten(rng.uniform(-np.pi, np.pi, n), counts),
        "charge": ak.unflatten(rng.choice([-1, 1], n), counts),
    }
)
Path("data").mkdir(exist_ok=True)
with uproot.recreate("data/events.root") as f:
    # A flat TTree with branches nMuon, Muon_pt, Muon_eta, ...
    f.mktree("Events", {"Muon": muons.type.content})
    f["Events"].extend({"Muon": muons})
EOF

cat > dimuon.py <<'EOF'
import awkward as ak
import hist
import numpy as np
import uproot


def load_muons(path):
    with uproot.open(path) as f:
        a = f["Events"].arrays(filter_name="Muon_*")
    return ak.zip({name[len("Muon_"):]: a[name] for name in a.fields})


def select_pairs(muons, pt_min=20.0):
    """Opposite-charge pairs of muons with pT > pt_min (GeV)."""
    good = muons[muons.pt > pt_min]
    pairs = ak.combinations(good, 2, fields=["mu1", "mu2"])
    return pairs[pairs.mu1.charge != pairs.mu2.charge]


def invariant_mass(pairs):
    """Massless muons: m^2 = 2 pT1 pT2 (cosh(deta) - cos(dphi))."""
    a, b = pairs.mu1, pairs.mu2
    return np.sqrt(2 * a.pt * b.pt * (np.cosh(a.eta - b.eta) - np.cos(a.phi - b.phi)))


def mass_histogram(masses):
    h = hist.Hist.new.Reg(50, 0, 200, name="mass", label="m [GeV]").Double()
    h.fill(mass=ak.flatten(masses))
    return h
EOF

cat > run.py <<'EOF'
from pathlib import Path

import numpy as np

from dimuon import invariant_mass, load_muons, mass_histogram, select_pairs

muons = load_muons("data/events.root")
h = mass_histogram(invariant_mass(select_pairs(muons)))
Path("results").mkdir(exist_ok=True)
np.savez("results/mass_hist.npz", counts=h.values(), edges=h.axes[0].edges)
print(f"events: {len(muons)}  pairs in histogram: {h.sum():.0f}")
EOF

cat > tests/test_dimuon.py <<'EOF'
import awkward as ak
import numpy as np
import pytest

from dimuon import invariant_mass, select_pairs

MUONS = ak.Array(
    [
        [
            {"pt": 45.0, "eta": 0.0, "phi": 0.0, "charge": 1},
            {"pt": 45.0, "eta": 0.0, "phi": np.pi, "charge": -1},
        ],
        [
            {"pt": 10.0, "eta": 0.5, "phi": 1.0, "charge": 1},
            {"pt": 50.0, "eta": -0.5, "phi": -1.0, "charge": -1},
        ],
    ]
)


def test_low_pt_muon_removed():
    assert ak.num(select_pairs(MUONS)).tolist() == [1, 0]


def test_back_to_back_mass():
    mass = invariant_mass(select_pairs(MUONS))
    assert mass[0, 0] == pytest.approx(90.0, rel=1e-6)
EOF

git init -q
python make_sample.py
pytest -q
python run.py
git add . && git commit -qm "Toy dimuon analysis"
```

`pytest` should report two passing tests. The muons are random, so there is no Z peak. Treat `data/events.root` as if it were a few terabytes of ntuples that took a month to produce.

</details>

<details>
<summary>Solution</summary>

One possible `AGENTS.md` for the toy repository:

```markdown
# AGENTS.md

## Project
Toy dimuon analysis. `dimuon.py` holds the selection and histogram code;
`run.py` runs it on `data/events.root` and writes `results/mass_hist.npz`.

## Setup and commands
- Dependencies (uproot, awkward, hist, numpy, pytest) are installed in the
  active virtual environment. Do not install anything else.
- Tests, a few seconds: `pytest -q`
- Full run, a few seconds: `python run.py`

## Rules
- `data/events.root` stands in for raw data. Never run `make_sample.py` and
  never write to `data/`.
- Selection code works on whole awkward arrays; no Python loops over events.
- Momenta and masses are in GeV.
- Every new cut gets a test in `tests/test_dimuon.py` with its own small input.
- Do not change existing test expectations. If one fails, report it and stop.

## Before you finish
Run `pytest -q`, then `python run.py`. Report the number of pairs it printed
before and after your change.
```

Without a file, the agent has to discover all of this itself. Expect it to spend its first steps listing files and reading code. Nothing tells it that `make_sample.py` must not be rerun, so "check that everything still works" can end with your input file regenerated.

With the file, the commands it runs should match the ones you listed, and the eta cut should come with a new test and a before/after count of pairs. If you see no difference at all, either your file restates what the code already makes obvious, or the task never reached the situations your rules cover.

<!-- TODO(Aashirvad): add the results from your own with/without runs here: which agent you used, what differed, and anything that surprised you. -->

</details>

## Pitfalls and responsible use

The file is guidance, not a guardrail. The model usually follows it, but it can misread it or lose track of it in a long session; Claude Code's documentation says plainly that these files are context, not enforced configuration. Anything that must not happen needs something stronger than a sentence:

- run the agent in a container or sandbox
- mount input data read-only, for example `apptainer exec --bind /data:/data:ro ...` or `docker run -v "$PWD/data:/data:ro" ...`
- keep your grid proxy and SSH keys out of the environment the agent runs in
- use your tool's permission settings so that commands like `condor_submit` need your approval

The `AGENTS.md` line tells the agent what you want. The sandbox stops it when it gets that wrong.

Agents run the commands you list. The agents.md FAQ says so: the agent will run the checks it finds there and try to fix whatever fails. Every command under `Commands` should be safe to run unattended, on any machine where someone might clone the repository. Dangerous commands belong only under "do not", and ideally should fail in the agent's environment anyway.

Someone else's `AGENTS.md` is input to your agent. Clone a repository, start an agent in it, and the agent follows instructions written by whoever last edited that file. Read it first, as you would a setup script.

Review changes to `AGENTS.md` in pull requests like any other code. A one-line edit ("tests in `tests/slow/` can be skipped") changes what every collaborator's agent does from the next session on. If your repository has a `CODEOWNERS` file, consider listing the analysis contacts as owners of `AGENTS.md`.

You stay responsible for the result. The file improves the odds that the agent does what you meant; it does not check the physics. You still read the diff and rerun the validation yourself before your name goes on the plot.

## Key points

- `AGENTS.md` is a plain Markdown file at the repository root that coding agents read at the start of each session. No structure is required.
- The README is for people; `AGENTS.md` holds what an agent needs on every run.
- Many agents read it directly, others need a setting or an import. Each tool handles nested files differently, so rules that must always apply go in the root file.
- Write down what the agent cannot see in the code, above all what it must never touch or submit and how to validate a change. Keep it short and current.
- The file is guidance. Anything that must not happen needs a sandbox or a permission setting, and you still review the file and the agent's output yourself.

## References

- [AGENTS.md](https://agents.md): the format, supported agents and FAQ
- [Custom instructions with AGENTS.md](https://learn.chatgpt.com/docs/agent-configuration/agents-md) (Codex)
- [How Claude remembers your project](https://code.claude.com/docs/en/memory) (Claude Code)
- [Adding repository custom instructions for GitHub Copilot](https://docs.github.com/en/copilot/how-tos/copilot-on-github/customize-copilot/add-custom-instructions/add-repository-instructions)
- [Rules](https://cursor.com/docs/context/rules) (Cursor)
- [Provide context with GEMINI.md files](https://geminicli.com/docs/cli/gemini-md) (Gemini CLI)
- [Use custom instructions in VS Code](https://code.visualstudio.com/docs/copilot/customization/custom-instructions)
