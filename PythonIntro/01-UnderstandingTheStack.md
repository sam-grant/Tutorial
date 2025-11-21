# Understanding the Mu2e Python Stack

## Overview

The Mu2e Python "stack" refers to the collection of tools and libraries available for Python-based analysis. It consists of:

1. **Standard scientific Python packages** (numpy, matplotlib, pandas, scipy, etc.)
2. **High-energy physics packages** (uproot, awkward, vector)
3. **Mu2e-specific utilities** (pyutils)

All of these are bundled together in the Mu2e Python environment for your convenience.

## The Mu2e Python Environment

### What is it?

The Mu2e Python environment is a pre-configured conda/mamba environment containing all the packages you need for analysis. Think of it as a "batteries included" Python installation where everything is already set up and version-controlled.

### Where is it?

The environment lives in two places:

1. **On CVMFS** (for Mu2e gpvms and EAF): `/cvmfs/mu2e.opensciencegrid.org/env/ana/current/`
2. **Local installations**: Can be set up anywhere using conda/mamba

### How do I activate it?

The activation method depends on where you're working:

#### On Mu2e gpvms (traditional approach)
```bash
mu2einit  # or: source /cvmfs/mu2e.opensciencegrid.org/setupmu2e-art.sh
pyenv ana  # Activate the Python environment
```

The `pyenv` command is a Mu2e-specific helper that:
- Activates the conda environment
- Sets up necessary environment variables
- Handles token authentication for data access

#### On EAF (Elastic Analysis Facility)
```bash
# One-time setup: create a symlink
ln -s /cvmfs/mu2e.opensciencegrid.org/env/ana/current ~/.conda/envs/mu2e_env

# Then activate
mamba activate mu2e_env
```

#### Local/custom installations
```bash
conda activate ana_v2.4.0  # or whatever your environment is named
```

### What's included?

As of version 2.4.0, the environment includes:

**Data handling:**
- `uproot` - Read ROOT files without requiring ROOT
- `awkward` - Efficiently handle jagged/nested data structures
- `pandas` - Tabular data analysis

**Scientific computing:**
- `numpy` - Numerical arrays and operations
- `scipy` - Scientific algorithms
- `scikit-learn` - Machine learning
- `pytorch`, `tensorflow` - Deep learning

**Visualization:**
- `matplotlib` - Publication-quality plots
- `plotly`, `dash` - Interactive visualizations
- `hist` - Histogram utilities

**HEP-specific:**
- `vector` - Lorentz vectors and coordinate systems
- `particle` - Particle data and properties

**Mu2e-specific:**
- `pyutils` - Mu2e analysis utilities (the focus of this tutorial)

## The pyutils Package

`pyutils` is the Mu2e-developed suite of tools that makes common analysis tasks easier. It provides:

### Core modules

| Module | Purpose | Key Features |
|--------|---------|--------------|
| `pyprocess` | Data processing | File lists, SAM datasets, parallelization |
| `pyread` | File reading | Local and remote file access |
| `pyimport` | Data import | TTree branch importing to awkward arrays |
| `pyselect` | Selection cuts | Pre-defined physics selections |
| `pycut` | Cut management | Track cut flow, toggle cuts, save states |
| `pyplot` | Plotting | Publication-quality histograms and graphs |
| `pyprint` | Data inspection | Human-readable event display |
| `pyvector` | Vector operations | 3D vector calculations |
| `pymcutil` | MC utilities | Truth matching and MC information |

### Design philosophy

`pyutils` is designed to:

1. **Minimize boilerplate** - Common tasks should be simple
2. **Use standard tools** - Built on uproot, awkward, matplotlib
3. **Be discoverable** - All documentation accessible via `help()`
4. **Support workflows** - From quick checks to production analysis

### When to use what

```
Need to...                          Use...
────────────────────────────────────────────────────────────
Load a single file                  → pyprocess.Processor
Load multiple files/SAM dataset     → pyprocess.Processor
Apply standard track cuts           → pyselect.Select
Build complex cut flows             → pycut.CutManager
Make a quick histogram              → pyplot.Plot.plot_1D()
Understand data structure           → pyprint.Print
Calculate track momentum magnitude  → pyvector.Vector
```

## How Everything Fits Together

Here's the typical workflow:

```
1. Activate environment
   ↓
2. Import packages (pyutils, awkward, etc.)
   ↓
3. Load data (pyprocess → pyread → pyimport)
   ↓
4. Inspect structure (pyprint)
   ↓
5. Apply selections (pyselect, pycut)
   ↓
6. Calculate quantities (pyvector, numpy)
   ↓
7. Visualize results (pyplot, matplotlib)
   ↓
8. Export/save (pandas, numpy)
```

Each step uses the appropriate tool from the stack. You'll see this workflow in action in the next section.

## Key Concepts

### Awkward Arrays

Most Mu2e data is stored in "jagged" or "nested" structures - for example, each event may have a different number of tracks, and each track may have a different number of hits. Regular numpy arrays can't handle this.

`awkward` arrays are designed for exactly this use case. They:
- Allow variable-length subarrays
- Support nested structures (like structs in C++)
- Provide numpy-like operations
- Are very fast

Example:
```python
# Each event has a different number of tracks
tracks_per_event = [[track1, track2], [track1], [track1, track2, track3]]
# This is natural in awkward, impossible in numpy
```

### EventNtuple

EventNtuple is Mu2e's standard ROOT ntuple format for analysis. It contains:
- Reconstructed tracks (`trk` branch)
- Track segments (`trksegs` branch)
- Calorimeter clusters (`clu` branch)
- CRV coincidences (`crvcoincs` branch)
- MC truth information (when available)
- And much more

`pyutils` is specifically designed to work with EventNtuple data.

### Branches vs Arrays

In ROOT terminology:
- A **TTree** contains **branches**
- Each **branch** contains data for one variable across all events

When you import with `pyutils`:
- Branches → awkward arrays
- You work with arrays in Python, not ROOT objects

## Common Patterns

### Pattern 1: Import and inspect
```python
from pyutils.pyprocess import Processor
from pyutils.pyprint import Print

proc = Processor()
data = proc.process_data(file_name="myfile.root", branches=["trk"])

printer = Print()
printer.print_n_events(data, n_events=1)
```

### Pattern 2: Select and plot
```python
from pyutils.pyselect import Select
from pyutils.pyplot import Plot
import awkward as ak

selector = Select()
electron_mask = selector.is_electron(data)
electrons = data[electron_mask]

plotter = Plot()
momenta = ak.flatten(electrons.trk.mom.mag)
plotter.plot_1D(momenta, nbins=100, xmin=0, xmax=110,
                xlabel="Momentum [MeV/c]", title="Electron momentum")
```

### Pattern 3: Cut flow analysis
```python
from pyutils.pycut import CutManager

cuts = CutManager()
cuts.add_cut("electron", "Select electrons", selector.is_electron(data))
cuts.add_cut("high_mom", "p > 100 MeV/c", ak.any(data.trk.mom.mag > 100, axis=-1))

cut_flow = cuts.create_cut_flow(data)
print(cuts.format_cut_flow(cut_flow))
```

## Summary

You now understand:
- What the Mu2e Python environment is and how to activate it
- What `pyutils` provides and when to use each module
- How awkward arrays handle jagged data
- Common analysis patterns

In the next section, we'll put this knowledge into practice with a complete analysis example.

## Navigation

- Next: [Your First Analysis: A Complete Walkthrough](02-FirstAnalysis.md)
- [Back to Main](README.md)
