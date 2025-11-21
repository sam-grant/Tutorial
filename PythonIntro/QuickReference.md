# Mu2e Python Stack - Quick Reference

A one-page cheat sheet for common operations.

## Environment Setup

```bash
# On gpvms
mu2einit
pyenv ana

# On EAF
mamba activate mu2e_env

# Local
conda activate ana_v2.4.0
```

## Import Essentials

```python
import awkward as ak
import numpy as np
from pyutils.pyprocess import Processor
from pyutils.pyselect import Select
from pyutils.pycut import CutManager
from pyutils.pyplot import Plot
```

## Load Data

```python
# Single file
proc = Processor()
data = proc.process_data(
    file_name="file.root",
    branches=["trk", "trkqual"]
)

# Multiple files
data = proc.process_data(
    file_list_path="files.txt",
    branches=["trk", "trkqual"],
    max_workers=4
)

# SAM dataset
data = proc.process_data(
    defname="sam-definition",
    branches=["trk", "trkqual"],
    max_workers=4
)
```

## Inspect Data

```python
from pyutils.pyprint import Print

printer = Print()
printer.print_n_events(data, n_events=1)

# Check structure
print(data.type)

# Count tracks
print(f"Events: {len(data)}")
print(f"Total tracks: {ak.sum(ak.num(data.trk))}")
```

## Apply Cuts

```python
selector = Select()

# Particle ID
is_electron = selector.is_electron(data)
is_positron = selector.is_positron(data)

# Geometry
is_downstream = selector.is_downstream(data)

# Track quality
good_qual = ak.any(data.trkqual.result > 0.8, axis=-1)

# Apply mask
selected = data[is_electron & good_qual]
```

## Cut Flow

```python
cuts = CutManager()

cuts.add_cut("electron", "e- candidate", is_electron)
cuts.add_cut("quality", "qual > 0.8", good_qual)
cuts.add_cut("momentum", "p > 100",
             ak.any(data.trk.mom.mag > 100, axis=-1))

# Generate flow
flow = cuts.create_cut_flow(data)
print(cuts.format_cut_flow(flow))

# Apply all cuts
final_mask = cuts.combine_cuts()
selected = data[final_mask]
```

## Plot

```python
plotter = Plot()

# 1D histogram
mom = ak.flatten(data.trk.mom.mag)
plotter.plot_1D(
    array=mom,
    nbins=100, xmin=0, xmax=110,
    xlabel="Momentum [MeV/c]",
    title="Track momentum"
)

# Overlay
hists = {
    "e-": electron_mom,
    "e+": positron_mom
}
plotter.plot_1D_overlay(
    hists_dict=hists,
    nbins=100, xmin=0, xmax=110,
    norm_by_area=True
)

# 2D histogram
plotter.plot_2D(
    x=mom, y=qual,
    nbins_x=50, nbins_y=50,
    xmin=0, xmax=110,
    ymin=0, ymax=1,
    xlabel="Momentum", ylabel="Quality"
)
```

## Awkward Operations

```python
# Count items per event
n_tracks = ak.num(data.trk)

# Check if ANY track passes
any_pass = ak.any(data.trk.mom.mag > 100, axis=-1)

# Check if ALL tracks pass
all_pass = ak.all(data.trk.mom.mag > 100, axis=-1)

# Get maximum per event
max_mom = ak.max(data.trk.mom.mag, axis=-1)

# Flatten jagged array
all_mom = ak.flatten(data.trk.mom.mag)

# Combine fields
combined = ak.zip({
    "mom": data.trk.mom.mag,
    "qual": data.trkqual.result
})
```

## Vector Operations

```python
from pyutils.pyvector import Vector

vec = Vector()

# Get 3D vectors
mom_vec = vec.get_vector(data.trksegs, 'mom')
pos_vec = vec.get_vector(data.trksegs, 'pos')

# Calculate magnitude
mag = vec.get_mag(data.trksegs, 'mom')

# Manual calculations
mom_x = data.trksegs.mom.fCoordinates.fX
mom_y = data.trksegs.mom.fCoordinates.fY
mom_t = np.sqrt(mom_x**2 + mom_y**2)
```

## Common Patterns

### Event-level vs Track-level

```python
# WRONG: Track-level mask (jagged!)
mask = data.trk.mom.mag > 100

# RIGHT: Event-level mask (boolean per event)
mask = ak.any(data.trk.mom.mag > 100, axis=-1)
```

### Flatten before plotting

```python
# WRONG: Can't plot jagged array
plotter.plot_1D(data.trk.mom.mag)

# RIGHT: Flatten first
plotter.plot_1D(ak.flatten(data.trk.mom.mag))
```

### Combining masks

```python
# AND
combined = mask1 & mask2

# OR
combined = mask1 | mask2

# NOT
combined = ~mask1

# Complex
combined = is_electron & (good_qual | high_mom)
```

## Save Results

```python
import pandas as pd

# Cut flow
df = cuts.format_cut_flow(flow)
df.to_csv("cutflow.csv", index=False)

# Arrays
np.save("momenta.npy", ak.to_numpy(ak.flatten(mom)))

# Summary table
summary = pd.DataFrame({...})
summary.to_csv("summary.csv", index=False)
```

## Get Help

```python
# In Python
help(Processor)
help(cuts.add_cut)

# Check available methods
dir(plotter)

# Examine structure
print(data.type)
print(data.fields)
```

## Slack Channels

- `#analysis-tools` - User questions
- `#analysis-tools-devel` - Development
- `#python-interest` - General Python

## Documentation

- pyutils: `pyutils/README.md`
- EventNtuple: [github.com/Mu2e/EventNtuple](https://github.com/Mu2e/EventNtuple)
- Awkward: [awkward-array.org](https://awkward-array.org)
- Mu2e Wiki: [mu2ewiki.fnal.gov](https://mu2ewiki.fnal.gov)
