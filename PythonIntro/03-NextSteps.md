# Next Steps and Resources

Congratulations on completing your first Mu2e Python analysis! This section points you toward resources for deepening your knowledge and tackling more complex analyses.

## Advancing Your Skills

### 1. Explore the pyutils Examples

The `pyutils` repository contains detailed tutorial notebooks covering advanced topics:

**Location:** `pyutils/examples/notebooks/`

| Notebook | What You'll Learn | When to Use |
|----------|-------------------|-------------|
| `pyutils_basics.ipynb` | Core functionality review | Reinforcing fundamentals |
| `pyutils_on_EAF.ipynb` | Remote data access from EAF | Working on EAF platform |
| `pyutils_multifile.ipynb` | Parallel processing, SAM datasets | Production analysis |
| `pyplot_demo.ipynb` | Advanced plotting features | Publication-quality figures |
| `pycut_demo.ipynb` | Cut flow optimization | Complex selection studies |

**How to access:**
```bash
cd pyutils/examples/notebooks
jupyter notebook
```

### 2. Master Awkward Array

`awkward` is central to HEP analysis in Python. Invest time in understanding it deeply.

**Essential resources:**
- [Awkward Array Documentation](https://awkward-array.org/)
- [Awkward Array Tutorial](https://awkward-array.org/doc/main/getting-started/index.html)
- [Scikit-HEP Tutorial](https://hsf-training.github.io/hsf-training-scikit-hep-webpage/)

**Key concepts to master:**
- Jagged/ragged arrays
- Broadcasting operations
- Reducers (`ak.sum`, `ak.any`, `ak.all`, `ak.max`, etc.)
- Masking and filtering
- Flattening strategies
- Type system

**Common patterns:**
```python
# Count items in each event
n_per_event = ak.num(data.trk)

# Check if ANY track in an event passes a cut
any_pass = ak.any(data.trk.mom.mag > 100, axis=-1)

# Check if ALL tracks in an event pass a cut
all_pass = ak.all(data.trk.mom.mag > 100, axis=-1)

# Get the maximum value per event
max_per_event = ak.max(data.trk.mom.mag, axis=-1)

# Flatten to a 1D array
all_momenta = ak.flatten(data.trk.mom.mag)

# Zip multiple arrays together
combined = ak.zip({"mom": data.trk.mom.mag, "qual": data.trkqual.result})
```

### 3. Learn About EventNtuple Structure

Understanding the EventNtuple data format is crucial.

**Resources:**
- [EventNtuple Documentation](https://github.com/Mu2e/EventNtuple)
- [EventNtuple Branch Guide](https://mu2ewiki.fnal.gov/wiki/EventNtuple)

**Key branches to know:**
```
evtinfo.*          → Event metadata (run, subrun, event number)
trk.*              → Reconstructed track parameters
trksegs.*          → Track segment information at surfaces
trkqual.*          → Track quality MVA results
trkhits.*          → Individual hit information
clu.*              → Calorimeter cluster information
crvcoincs.*        → CRV coincidence information
trkmcsim.*         → MC truth for tracks (MC only)
```

**Exploring branches interactively:**
```python
import uproot

file = uproot.open("yourfile.root")
tree = file["EventNtuple/ntuple"]

# List all branches
print(tree.keys())

# Check a specific branch structure
print(tree["trk"].typename)
```

### 4. Processing Multiple Files

Real analyses process many files. Learn parallel processing:

**Basic file list:**
```python
from pyutils.pyprocess import Processor

proc = Processor()

# From a text file containing file paths
data = proc.process_data(
    file_list_path="files.txt",
    branches=["trk", "trkqual"],
    max_workers=8  # Parallel processing
)
```

**From SAM dataset:**
```python
# Query SAM and process files in parallel
data = proc.process_data(
    defname="your-sam-definition",
    branches=["trk", "trkqual"],
    max_workers=8
)
```

**Custom processing per file:**
```python
def process_one_file(file_name):
    """Custom processing for each file"""
    proc = Processor()
    data = proc.process_data(file_name=file_name, branches=["trk"])

    # Do something with this file's data
    n_electrons = ak.sum(selector.is_electron(data))

    return {"file": file_name, "n_electrons": n_electrons}

# Process all files with custom function
proc = Processor()
results = proc.process_data(
    file_list_path="files.txt",
    custom_process_func=process_one_file,
    max_workers=8
)

# results is a list of dictionaries
for result in results:
    print(f"{result['file']}: {result['n_electrons']} electrons")
```

**Using the Skeleton class:**

For complex analyses, use the `Skeleton` template:

```python
from pyutils.pyprocess import Skeleton, Processor

class MyAnalysis(Skeleton):
    def __init__(self, cuts_config, verbosity=1):
        super().__init__(verbosity)
        self.cuts_config = cuts_config
        self.results = []

    def process_file(self, file_name):
        """Process one file"""
        proc = Processor()
        data = proc.process_data(file_name=file_name, branches=["trk", "trkqual"])

        # Apply your analysis
        # ... return results ...

        return result

# Use it
analysis = MyAnalysis(cuts_config={...})
analysis.file_list_path = "files.txt"
analysis.max_workers = 8
results = analysis.execute()
```

### 5. Advanced Cut Flow Analysis

`CutManager` has powerful features for systematic studies:

**Grouping cuts:**
```python
cuts = CutManager()

# Add cuts with groups
cuts.add_cut("electron", "e- candidate", mask1, group="pid")
cuts.add_cut("positron", "e+ candidate", mask2, group="pid")
cuts.add_cut("p_min", "p > 100", mask3, group="kinematics")
cuts.add_cut("p_max", "p < 110", mask4, group="kinematics")

# Toggle entire groups
cuts.toggle_group({"kinematics": False})
```

**Systematic variations:**
```python
# Save baseline
cuts.save_state("baseline")

# Try variation 1: looser quality cut
cuts.toggle_cut({"quality": False})
flow1 = cuts.create_cut_flow(data)

cuts.restore_state("baseline")

# Try variation 2: different momentum window
cuts.toggle_cut({"p_min": False, "p_max": False})
cuts.add_cut("p_signal", "103 < p < 105", new_mask)
flow2 = cuts.create_cut_flow(data)

# Compare efficiencies
```

**Combining multiple cut flows (for multiprocessing):**
```python
def process_file(file_name):
    # ... load data, apply cuts ...
    return cuts.create_cut_flow(data)

# Process many files
flows = []
for file in file_list:
    flows.append(process_file(file))

# Combine all cut flows
combined = cuts.combine_cut_flows(flows)
print(cuts.format_cut_flow(combined))
```

### 6. Advanced Plotting

`pyplot.Plot` supports many advanced features:

**Styling:**
```python
# Use custom matplotlib style
plotter = Plot(style_path="my_style.mplstyle")

# Or use the Mu2e style (default)
plotter = Plot()
```

**Error bars:**
```python
plotter.plot_1D(
    array=data,
    nbins=50, xmin=0, xmax=100,
    error_bars=True  # Show Poisson errors
)
```

**Logarithmic scales:**
```python
plotter.plot_1D(
    array=data,
    nbins=50, xmin=0.1, xmax=1000,
    log_x=True,
    log_y=True
)
```

**Graphs with error bars:**
```python
x = np.array([1, 2, 3, 4, 5])
y = np.array([2.1, 3.9, 6.2, 7.8, 10.1])
yerr = np.array([0.2, 0.3, 0.4, 0.3, 0.5])

plotter.plot_graph(
    x=x, y=y, yerr=yerr,
    xlabel="x", ylabel="y",
    col='red'
)
```

**Complex overlays:**
```python
# Multiple graphs with different colors and markers
graphs = {
    "Data": {"x": x_data, "y": y_data, "yerr": yerr_data},
    "MC": {"x": x_mc, "y": y_mc, "yerr": yerr_mc},
    "Prediction": {"x": x_pred, "y": y_pred}
}

plotter.plot_graph_overlay(
    graphs=graphs,
    xlabel="x", ylabel="y",
    legend_position='upper left'
)
```

### 7. Machine Learning Integration

The Mu2e environment includes ML tools:

**PyTorch example:**
```python
import torch
import torch.nn as nn

# Your track data as features
features = ak.to_numpy(ak.flatten(data.trk.mom.mag))

# Build a simple network
model = nn.Sequential(
    nn.Linear(n_features, 64),
    nn.ReLU(),
    nn.Linear(64, 32),
    nn.ReLU(),
    nn.Linear(32, 1),
    nn.Sigmoid()
)

# Train on your data...
```

**Scikit-learn example:**
```python
from sklearn.ensemble import RandomForestClassifier
import pandas as pd

# Convert awkward to pandas for sklearn
df = pd.DataFrame({
    'mom': ak.to_numpy(ak.flatten(data.trk.mom.mag)),
    'nactive': ak.to_numpy(ak.flatten(data.trk.nactive)),
    'quality': ak.to_numpy(ak.flatten(data.trkqual.result)),
})

# Train classifier
clf = RandomForestClassifier()
clf.fit(df[features], labels)
```

## Working on Different Platforms

### EAF (Elastic Analysis Facility)

**Advantages:**
- Web-based Jupyter interface
- Elastic scaling for large jobs
- Direct access to tape storage
- Easy collaboration

**Setup:**
```bash
# One-time: create symlink
ln -s /cvmfs/mu2e.opensciencegrid.org/env/ana/current ~/.conda/envs/mu2e_env

# Activate
mamba activate mu2e_env
```

**See:** [EAF Tutorial](../EAF/README.md)

### GPVMs (General Purpose Virtual Machines)

**Advantages:**
- Full shell access
- Integration with Mu2e software stack
- Direct grid job submission

**Setup:**
```bash
mu2einit
pyenv ana
```

### Local Development

**Advantages:**
- Work offline
- Use your own tools and IDE
- Fast iteration

**Setup:**
```bash
# Install pyutils in your environment
conda create -n my_analysis python=3.11
conda activate my_analysis
pip install git+https://github.com/Mu2e/pyutils.git

# Or use provided environment
conda env create -f environment.yml
```

## Common Analysis Patterns

### Pattern 1: Quick Data Check
```python
# Load just a few events to check structure
from pyutils.pyprocess import Processor
from pyutils.pyprint import Print

proc = Processor()
data = proc.process_data(file_name="file.root", branches=["trk"])

printer = Print()
printer.print_n_events(data, n_events=3)
```

### Pattern 2: Cut Optimization
```python
# Scan cut values to optimize efficiency
from pyutils.pycut import CutManager
import numpy as np

qualities = np.linspace(0.5, 0.95, 10)
efficiencies = []

for qual_cut in qualities:
    cuts = CutManager()
    cuts.add_cut("quality", f"qual > {qual_cut}",
                 ak.any(data.trkqual.result > qual_cut, axis=-1))
    flow = cuts.create_cut_flow(data)
    efficiencies.append(flow[-1]['efficiency'])

# Plot efficiency vs cut value
plt.plot(qualities, efficiencies)
plt.xlabel("Quality cut")
plt.ylabel("Efficiency")
plt.show()
```

### Pattern 3: Compare MC to Data
```python
# Load both MC and data
data_mc = proc.process_data(file_name="mc.root", branches=["trk"])
data_real = proc.process_data(file_name="data.root", branches=["trk"])

# Apply same cuts to both
cuts = CutManager()
# ... define cuts ...
mc_selected = data_mc[cuts.combine_cuts()]
data_selected = data_real[cuts.combine_cuts()]

# Compare distributions
hists = {
    "MC": ak.flatten(mc_selected.trk.mom.mag),
    "Data": ak.flatten(data_selected.trk.mom.mag)
}

plotter.plot_1D_overlay(hists, nbins=100, xmin=95, xmax=110,
                        xlabel="Momentum [MeV/c]")
```

### Pattern 4: Truth Matching (MC only)
```python
from pyutils.pymcutil import MC

# Load MC truth branches
data = proc.process_data(file_name="mc.root",
                         branches=["trk", "trkmcsim"])

# Use MC utilities
mc_util = MC()
primary_codes = mc_util.count_particle_types(data)

# Filter for conversion electrons
is_ce = (primary_codes == 167)  # CE code
ce_data = data[is_ce]
```

## Troubleshooting Resources

### Documentation
- **pyutils README:** `pyutils/README.md` (comprehensive module docs)
- **Mu2e Wiki:** [https://mu2ewiki.fnal.gov](https://mu2ewiki.fnal.gov)
- **EventNtuple:** [https://github.com/Mu2e/EventNtuple](https://github.com/Mu2e/EventNtuple)
- **Awkward Array:** [https://awkward-array.org](https://awkward-array.org)

### Getting Help
- **Slack:**
  - `#analysis-tools` - User questions and discussion
  - `#analysis-tools-devel` - Development and advanced topics
  - `#python-interest` - General Python questions
- **GitHub Issues:** [https://github.com/Mu2e/pyutils/issues](https://github.com/Mu2e/pyutils/issues)
- **Office Hours:** Check Mu2e calendar for analysis help sessions

### Common Questions

**Q: How do I process files on the grid?**
A: Use SAM datasets with `proc.process_data(defname="...")`. For very large jobs, submit grid jobs that run your Python script.

**Q: My analysis is slow. How do I speed it up?**
A:
1. Only load branches you need
2. Use parallel processing (`max_workers`)
3. Apply cuts early to reduce data size
4. Profile your code to find bottlenecks

**Q: Can I use ROOT alongside pyutils?**
A: Yes! Use `pyenv rootana` to get ROOT + PyROOT. You can mix ROOT and pyutils in the same script.

**Q: How do I contribute to pyutils?**
A: See the developer section in `pyutils/README.md`. Fork, make changes, submit a pull request!

**Q: What if a feature I need is missing?**
A:
1. Check if it exists in the underlying packages (awkward, numpy, etc.)
2. Ask on Slack - it might exist and you just didn't know
3. Request it on GitHub Issues
4. Implement it yourself and contribute!

## Best Practices

### Code Organization
```
my_analysis/
├── config/
│   ├── cuts.yaml          # Cut definitions
│   └── samples.yaml       # Input samples
├── data/
│   └── file_lists/        # File lists
├── notebooks/
│   └── exploration.ipynb  # Interactive analysis
├── scripts/
│   ├── process_data.py    # Main analysis
│   └── make_plots.py      # Plotting
├── output/
│   ├── plots/             # Generated plots
│   └── results/           # CSV, ROOT files
└── README.md
```

### Version Control
- Track your analysis code in git
- Tag releases corresponding to presentations/publications
- Document major changes
- Share code with collaborators via GitHub

### Reproducibility
- Record environment version: `conda list > environment.txt`
- Save exact cut definitions
- Document file lists / SAM definitions
- Include random seeds for ML

### Performance
- Profile before optimizing: `python -m cProfile script.py`
- Use `awkward` operations instead of Python loops
- Process files in parallel when possible
- Cache intermediate results

## Your Analysis Checklist

Starting a new analysis? Use this checklist:

- [ ] Define physics goal and signal region
- [ ] Identify necessary EventNtuple branches
- [ ] Set up analysis repository with version control
- [ ] Create file list or SAM definition
- [ ] Implement selection cuts incrementally
- [ ] Verify cuts with visual inspection
- [ ] Generate and review cut flow
- [ ] Calculate derived quantities
- [ ] Create diagnostic plots at each step
- [ ] Compare to MC (if available)
- [ ] Estimate systematics
- [ ] Document everything
- [ ] Share code with collaborators for review

## Conclusion

You now have the foundation to conduct sophisticated analyses with the Mu2e Python stack. Remember:

- **Start simple** - Get something working, then add complexity
- **Inspect often** - Use `Print` and plots to check your data
- **Ask for help** - The community is friendly and responsive
- **Contribute back** - Share your tools and improvements

Happy analyzing!

## Navigation

- Previous: [Your First Analysis: A Complete Walkthrough](02-FirstAnalysis.md)
- [Back to Main](README.md)
