# Your First Analysis: A Complete Walkthrough

## Introduction

In this tutorial, we'll perform a complete (albeit simple) analysis using the Mu2e Python stack. We'll:

1. Load EventNtuple data
2. Inspect the data structure
3. Apply selection cuts
4. Calculate derived quantities
5. Create publication-quality plots
6. Generate a cut flow table

This mirrors what you'd do in a real analysis, just on a smaller scale.

## Physics Goal

We'll analyze reconstructed electron tracks to:
- Understand the momentum distribution
- Examine track quality
- Study the spatial distribution at the tracker entrance
- Compare different particle types

## Setup

First, activate your environment and start Python or Jupyter:

```bash
# Activate the Mu2e Python environment
conda activate ana_v2.4.0  # Or: pyenv ana (on gpvms)

# Start Jupyter (optional)
jupyter notebook
```

## Step 1: Import Packages

```python
# External packages
import awkward as ak
import numpy as np

# pyutils modules
from pyutils.pyprocess import Processor
from pyutils.pyprint import Print
from pyutils.pyselect import Select
from pyutils.pycut import CutManager
from pyutils.pyplot import Plot
from pyutils.pyvector import Vector
```

**What's happening?**
- `awkward` and `numpy` provide array operations
- Each `pyutils` module handles a specific task in our analysis pipeline

## Step 2: Load the Data

```python
# Create a processor
proc = Processor(verbosity=1)

# Specify which branches to load
branches = [
    "trk",        # Track information
    "trksegs",    # Track segments
    "trkqual"     # Track quality MVA
]

# Load a file
# Replace with your own file path
file_path = "/path/to/your/eventntuple.root"
data = proc.process_data(
    file_name=file_path,
    branches=branches
)

print(f"Loaded {len(data)} events")
```

**What's happening?**
- `Processor` handles all the complexity of opening ROOT files and importing branches
- You specify which branches you need (don't load everything - it's slow!)
- The result is an awkward array containing your data
- `verbosity=1` gives helpful progress messages

**Common issue:** If the file doesn't exist, you'll get an error. Make sure the path is correct!

### Alternative: Working with SAM datasets

If you have a SAM dataset definition:

```python
# Instead of file_name, use defname
data = proc.process_data(
    defname="your-sam-definition",
    branches=branches,
    max_workers=4  # Parallel processing
)
```

## Step 3: Inspect the Data

Before doing any analysis, always inspect your data to understand its structure.

```python
# Create a printer
printer = Print(verbose=False)

# Look at the first event
printer.print_n_events(data, n_events=1)
```

**Output example:**
```
---> Printing 1 event(s)...

-------------------------------------------------------------------------------------
trk.status: [1]
trk.nhits: [69]
trk.nactive: [69]
trk.mom.fCoordinates.fX: [-2.47]
trk.mom.fCoordinates.fY: [102.53]
trk.mom.fCoordinates.fZ: [0.89]
trk.mom.mag: [102.57]
trkqual.result: [0.94]
trksegs.mom.fCoordinates.fX: [[-2.47]]
trksegs.mom.fCoordinates.fY: [[102.53]]
trksegs.mom.fCoordinates.fZ: [[0.89]]
...
-------------------------------------------------------------------------------------
```

**What's happening?**
- Each line shows a branch and its values for the first event
- Values in `[brackets]` indicate arrays (this event has 1 track)
- Double brackets `[[values]]` indicate nested arrays (track → segments)
- The field names follow the ROOT structure (e.g., `mom.fCoordinates.fX`)

**Understanding the structure:**
```python
# Check the array structure
print(data.type)
```

This shows you the full nested structure of your data.

## Step 4: Extract Basic Quantities

Let's extract some quantities we'll use later:

```python
# Get track momentum magnitude
# Note: trk is already a var * var structure (events * tracks)
track_mom = data.trk.mom.mag

# Get track quality
track_qual = data.trkqual.result

# Get number of active planes
n_active = data.trk.nactive

# Print some statistics
print(f"Total events: {len(data)}")
print(f"Events with at least one track: {ak.sum(ak.num(data.trk) > 0)}")
print(f"Total tracks: {ak.sum(ak.num(data.trk))}")
```

**What's happening?**
- We access nested fields using dot notation: `data.trk.mom.mag`
- `ak.num()` counts the number of items in each subarray
- These are still jagged arrays (different number of tracks per event)

## Step 5: Apply Selection Cuts

Now let's apply some physics cuts using `pyselect`:

```python
# Create selector
selector = Select()

# Define particle type selections
is_electron = selector.is_electron(data)
is_positron = selector.is_positron(data)
is_mu_minus = selector.is_mu_minus(data)

# Select downstream tracks (most common for conversion electrons)
is_downstream = selector.is_downstream(data, branch_name='trksegs')

# Print statistics
print(f"Electron events: {ak.sum(is_electron)}")
print(f"Positron events: {ak.sum(is_positron)}")
print(f"Mu- events: {ak.sum(is_mu_minus)}")
print(f"Downstream events: {ak.sum(is_downstream)}")
```

**What's happening?**
- Each selection returns a boolean mask (True/False for each event)
- These masks can be combined and applied to filter your data
- `pyselect` provides standard Mu2e selections

### Understanding masks

```python
# A mask is just a boolean array
print(f"Mask shape: {is_electron.type}")
print(f"First 10 events: {is_electron[:10]}")

# Apply a mask to get selected events
electron_data = data[is_electron]
print(f"Selected {len(electron_data)} electron events")
```

## Step 6: Build a Cut Flow with CutManager

For more complex analyses, use `CutManager` to track your cuts:

```python
# Create cut manager
cuts = CutManager(verbosity=1)

# Add cuts in order
# Note: we need event-level masks (not track-level)
has_track = ak.num(data.trk) > 0
cuts.add_cut(
    name="has_track",
    description="Event has at least one track",
    mask=has_track,
    active=True
)

cuts.add_cut(
    name="electron",
    description="Electron candidate",
    mask=is_electron,
    active=True
)

cuts.add_cut(
    name="downstream",
    description="Downstream track",
    mask=is_downstream,
    active=True
)

# Track quality cut (check if ANY track passes)
good_quality = ak.any(data.trkqual.result > 0.8, axis=-1)
cuts.add_cut(
    name="quality",
    description="Track quality > 0.8",
    mask=good_quality,
    active=True
)

# Momentum cut
good_momentum = ak.any(data.trk.mom.mag > 100, axis=-1)
cuts.add_cut(
    name="momentum",
    description="p > 100 MeV/c",
    mask=good_momentum,
    active=True
)

# Generate and display cut flow
cut_flow = cuts.create_cut_flow(data)
df = cuts.format_cut_flow(cut_flow)
print(df)
```

**Output example:**
```
        Cut Name              Description  N Events  N Pass  N Fail  Efficiency  Cumulative Eff
0      has_track  Event has at least ...     10000    9523     477      0.9523          0.9523
1       electron       Electron candidate      9523    7234    2289      0.7597          0.7234
2     downstream         Downstream track      7234    6891     343      0.9526          0.6891
3        quality    Track quality > 0.8      6891    6234     657      0.9047          0.6234
4       momentum        p > 100 MeV/c      6234    5891     343      0.9450          0.5891
```

**What's happening?**
- Each cut is applied in sequence
- You see both individual and cumulative efficiencies
- The DataFrame can be saved or plotted

### Managing cuts

```python
# Save the current state
cuts.save_state("nominal")

# Try disabling a cut
cuts.toggle_cut({"quality": False})
alt_flow = cuts.create_cut_flow(data)
print("Without quality cut:")
print(cuts.format_cut_flow(alt_flow))

# Restore original
cuts.restore_state("nominal")

# Get combined mask for all active cuts
final_mask = cuts.combine_cuts()
selected_data = data[final_mask]
print(f"After all cuts: {len(selected_data)} events")
```

## Step 7: Calculate Derived Quantities

Let's calculate some physics quantities using the selected data:

```python
# Work with selected events only
selected_data = data[final_mask]

# Initialize vector calculator
vec = Vector()

# Get 3D momentum vectors
mom_vec = vec.get_vector(selected_data.trksegs, 'mom')
print(f"Momentum vectors shape: {mom_vec.type}")

# Get position at tracker entrance
pos_vec = vec.get_vector(selected_data.trksegs, 'pos')

# Calculate transverse momentum
# First get x and y components
mom_x = selected_data.trksegs.mom.fCoordinates.fX
mom_y = selected_data.trksegs.mom.fCoordinates.fY
mom_t = np.sqrt(mom_x**2 + mom_y**2)

# Get radial position
pos_x = selected_data.trksegs.pos.fCoordinates.fX
pos_y = selected_data.trksegs.pos.fCoordinates.fY
pos_r = np.sqrt(pos_x**2 + pos_y**2)

print(f"Calculated {ak.sum(ak.num(mom_t))} transverse momenta")
```

**What's happening?**
- `pyvector` helps extract 3D vectors
- We can use numpy math operations on awkward arrays
- Results maintain the jagged structure

## Step 8: Create Plots

Now for the fun part - visualizing our results!

### Simple 1D histogram

```python
# Create plotter
plotter = Plot()

# Flatten the jagged momentum array (events * tracks → all tracks)
mom_flat = ak.flatten(selected_data.trk.mom.mag)

# Plot momentum distribution
plotter.plot_1D(
    array=mom_flat,
    nbins=100,
    xmin=95,
    xmax=110,
    xlabel="Track momentum [MeV/c]",
    ylabel="Tracks / 0.15 MeV/c",
    title="Electron momentum distribution",
    col='blue',
    stat_box=True,
    out_path="electron_momentum.png"
)
```

**What's happening?**
- `ak.flatten()` converts the jagged array to a flat array
- `plot_1D()` creates a publication-quality histogram
- The stat box shows mean, RMS, entries, etc.
- `out_path` saves the figure

### Overlaying multiple distributions

```python
# Compare different particle types
electron_mask = selector.is_electron(data)
positron_mask = selector.is_positron(data)

electron_mom = ak.flatten(data[electron_mask].trk.mom.mag)
positron_mom = ak.flatten(data[positron_mask].trk.mom.mag)

# Create overlay
hists_dict = {
    "e-": electron_mom,
    "e+": positron_mom
}

plotter.plot_1D_overlay(
    hists_dict=hists_dict,
    nbins=100,
    xmin=95,
    xmax=110,
    xlabel="Track momentum [MeV/c]",
    ylabel="Tracks / 0.15 MeV/c",
    title="Momentum comparison",
    norm_by_area=True,  # Normalize to same area
    leg_pos='upper left',
    out_path="momentum_comparison.png"
)
```

### 2D histogram

```python
# Plot momentum vs track quality
mom_flat = ak.flatten(selected_data.trk.mom.mag)
qual_flat = ak.flatten(selected_data.trkqual.result)

plotter.plot_2D(
    x=mom_flat,
    y=qual_flat,
    nbins_x=50,
    xmin=95,
    xmax=110,
    nbins_y=50,
    ymin=0,
    ymax=1,
    xlabel="Track momentum [MeV/c]",
    ylabel="Track quality",
    title="Momentum vs Quality",
    cmap='viridis',
    out_path="momentum_vs_quality.png"
)
```

### Multiple panels

```python
import matplotlib.pyplot as plt

# Create figure with subplots
fig, axes = plt.subplots(2, 2, figsize=(12, 10))

# Panel 1: Momentum
plotter.plot_1D(
    array=ak.flatten(selected_data.trk.mom.mag),
    nbins=50, xmin=95, xmax=110,
    xlabel="p [MeV/c]",
    ax=axes[0, 0],
    show=False
)

# Panel 2: Track quality
plotter.plot_1D(
    array=ak.flatten(selected_data.trkqual.result),
    nbins=50, xmin=0, xmax=1,
    xlabel="Track quality",
    ax=axes[0, 1],
    show=False
)

# Panel 3: Number of active planes
plotter.plot_1D(
    array=ak.flatten(selected_data.trk.nactive),
    nbins=50, xmin=0, xmax=100,
    xlabel="N active planes",
    ax=axes[1, 0],
    show=False
)

# Panel 4: Transverse momentum
plotter.plot_1D(
    array=ak.flatten(mom_t),
    nbins=50, xmin=0, xmax=110,
    xlabel="p_T [MeV/c]",
    ax=axes[1, 1],
    show=False
)

plt.tight_layout()
plt.savefig("analysis_summary.png", dpi=300)
plt.show()
```

## Step 9: Export Results

Save your results for later use:

```python
import pandas as pd

# Create a summary table
summary = {
    'Cut': [cut['name'] for cut in cut_flow],
    'N_Pass': [cut['n_pass'] for cut in cut_flow],
    'Efficiency': [cut['efficiency'] for cut in cut_flow]
}
df_summary = pd.DataFrame(summary)

# Save to CSV
df_summary.to_csv("cut_flow_summary.csv", index=False)
print("Saved cut flow to cut_flow_summary.csv")

# Save selected event numbers for further study
event_numbers = selected_data.evtinfo.event  # if you loaded evtinfo
# np.save("selected_events.npy", ak.to_numpy(ak.flatten(event_numbers)))
```

## Complete Example Script

Here's everything together in a single script:

```python
#!/usr/bin/env python
"""
My First Mu2e Analysis
A simple analysis of electron tracks in EventNtuple data
"""

import awkward as ak
import numpy as np
import matplotlib.pyplot as plt

from pyutils.pyprocess import Processor
from pyutils.pyselect import Select
from pyutils.pycut import CutManager
from pyutils.pyplot import Plot

# Configuration
INPUT_FILE = "/path/to/your/file.root"
BRANCHES = ["trk", "trksegs", "trkqual"]

def main():
    print("Starting analysis...")

    # Load data
    print("\n1. Loading data...")
    proc = Processor(verbosity=1)
    data = proc.process_data(file_name=INPUT_FILE, branches=BRANCHES)
    print(f"   Loaded {len(data)} events")

    # Define cuts
    print("\n2. Defining cuts...")
    selector = Select()
    cuts = CutManager()

    cuts.add_cut("has_track", "Has track", ak.num(data.trk) > 0)
    cuts.add_cut("electron", "Electron", selector.is_electron(data))
    cuts.add_cut("downstream", "Downstream", selector.is_downstream(data))
    cuts.add_cut("quality", "Quality > 0.8",
                 ak.any(data.trkqual.result > 0.8, axis=-1))
    cuts.add_cut("momentum", "p > 100 MeV/c",
                 ak.any(data.trk.mom.mag > 100, axis=-1))

    # Generate cut flow
    print("\n3. Cut flow:")
    cut_flow = cuts.create_cut_flow(data)
    print(cuts.format_cut_flow(cut_flow))

    # Apply cuts
    print("\n4. Applying cuts...")
    final_mask = cuts.combine_cuts()
    selected = data[final_mask]
    print(f"   Selected {len(selected)} events")

    # Create plots
    print("\n5. Creating plots...")
    plotter = Plot()

    mom = ak.flatten(selected.trk.mom.mag)
    plotter.plot_1D(
        array=mom,
        nbins=100, xmin=95, xmax=110,
        xlabel="Track momentum [MeV/c]",
        title="Electron momentum",
        out_path="momentum.png"
    )

    print("\n✅ Analysis complete!")
    print("   Output: momentum.png")

if __name__ == "__main__":
    main()
```

Save this as `my_first_analysis.py` and run:
```bash
python my_first_analysis.py
```

## Common Issues and Solutions

### Issue: "Branch not found"
**Problem:** You're trying to access a branch that doesn't exist
**Solution:** Check available branches with ROOT or load the file in ROOT browser first

### Issue: "Cannot apply mask"
**Problem:** Mask shape doesn't match data shape
**Solution:** Make sure your mask is at the event level, not track level
```python
# Wrong: track-level mask
mask = data.trk.mom.mag > 100  # This is jagged!

# Right: event-level mask
mask = ak.any(data.trk.mom.mag > 100, axis=-1)  # Boolean per event
```

### Issue: "Empty array" when flattening
**Problem:** No events passed your cuts
**Solution:** Check cut flow to see where events are lost
```python
print(cuts.format_cut_flow(cut_flow))
```

### Issue: Plots look wrong
**Problem:** Forgot to flatten jagged arrays
**Solution:** Always flatten before plotting
```python
# Wrong
plotter.plot_1D(data.trk.mom.mag)  # This is jagged!

# Right
plotter.plot_1D(ak.flatten(data.trk.mom.mag))
```

## Summary

You've now completed your first Mu2e analysis! You learned how to:

- ✅ Load EventNtuple data with `Processor`
- ✅ Inspect data structure with `Print`
- ✅ Apply physics cuts with `Select`
- ✅ Track cut flow with `CutManager`
- ✅ Calculate derived quantities
- ✅ Create publication-quality plots with `Plot`
- ✅ Export results

This workflow scales from quick checks to production analyses. The main differences are:
- More cuts and more complex selection logic
- More derived quantities and calculations
- More sophisticated plots and statistical tests
- Processing multiple files in parallel

## Navigation

- Previous: [Understanding the Mu2e Python Stack](01-UnderstandingTheStack.md)
- Next: [Next Steps and Resources](03-NextSteps.md)
- [Back to Main](README.md)
