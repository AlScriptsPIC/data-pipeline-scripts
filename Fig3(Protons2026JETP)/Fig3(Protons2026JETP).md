### <a id="jupyter-notebook-cells---interactive-navigation--parameter-inspection">Interactive Navigation & Parameter Inspection</a>
<details>
<summary><b>Tag: navigator1.0</b> – Interactive folder navigator and selector with following inspection of MainVars.py parameters and global saving</summary>

⚠️ **CORE MODULE NOTICE:** This cell manages global pipeline state. **Do not modify the internal code** during routine execution.
🔧 *Only enter this cell if you explicitly need to switch the normalization factor (e.g., change `Norm = fields['Mag']` to `fields['Elec']`). All folder selection and parameter inspection should be handled via the dropdown UI or downstream cells.*

**Description**:  
Provides an interactive dropdown menu to browse predefined simulation directories and dynamically load a `MainVars.py` configuration file upon selection. Automatically extracts key simulation parameters (`a`, `tau`, `D_o`) and computes a global normalization factor (`Norm`) based on physical constants defined in the target module. Designed for rapid parameter comparison across multiple simulation setups without manual file navigation.

**Key Features**:
- 🔍 Dynamic module loading via `importlib.util` (avoids namespace pollution)
- 📂 Real-time parameter extraction and validation
- 🔄 Global state tracking (`selected_folder`, `Norm`) for downstream cells
- 🛡️ Graceful error handling for missing files or undefined attributes

**Dependencies**:  
`ipywidgets`, `IPython.display`, `importlib.util`, `os`

**Usage Notes**:
- Ensure `MainVars.py` exists in each directory and exposes: `a`, `tau`, `D_o`, `ELECMASS`, `Omega`, `ELEMCHARGE`.
- `Norm` is updated globally and can be safely referenced in subsequent notebook cells.
- Unused imports and placeholder paths have been cleaned; `InOtherInstitute` is preserved as a commented template.

**Expected Output**:

</details>

```python
import os
import importlib.util
from IPython.display import display
import ipywidgets as widgets

# 🔧 CONFIGURATION: Change this value to switch normalization type
# Options: "Mag" (Magnetic) or "Elec" (Electric)
NORM_MODE = "Elec"  

# Predefined simulation directories to scan
FOLDER_LIST = [
    '   ',
    '/home/big4/castillo/conversion_a=74_tau_6_ne=0.40_2Jb_stop_window',
    '/home/big4/castillo/conversion_a=74_tau_6_ne=0.40_2Jb_stop_window/best_dumps',
    '/home/big4/castillo/conversion_a=74_tau_6_ne=0.40_2Jb_stop_window/tracking',
    '/home/big4/castillo/conversion_a=27_tau=20_ne=0.50_stop_window',
    '/home/big4/castillo/conversion_a=22_tau=30_ne=0.9_stop_window_ramp',
    '/home/big4/castillo/conversion_a=27_tau=20_ne=0.9_LowResolution'
]

# Global state variables (accessible by downstream cells)
selected_folder = None
Norm = 0.0

# Initialize interactive dropdown
folder_dropdown = widgets.Dropdown(
    options=FOLDER_LIST,
    description='Folder:',
    style={'description_width': 'initial'}  # Prevents label truncation
)

# Output container for dynamic prints
result_output = widgets.Output()

# Render widgets
display(folder_dropdown, result_output)

def on_folder_change(change):
    """Callback executed when the dropdown selection changes."""
    global selected_folder, Norm
    
    target_folder = change['new']
    main_vars_path = os.path.join(target_folder, 'MainVars.py')
    
    with result_output:
        result_output.clear_output()
        
        # Verify configuration file exists
        if not os.path.isfile(main_vars_path):
            print(f"⚠️ MainVars.py not found in: {target_folder}")
            return
        
        # Dynamically import the module without polluting the global namespace
        try:
            spec = importlib.util.spec_from_file_location("MainVars", main_vars_path)
            if spec is None or spec.loader is None:
                raise ImportError("Failed to resolve module specification.")
                
            module = importlib.util.module_from_spec(spec)
            spec.loader.exec_module(module)
            
            # Extract core simulation parameters
            a = module.a
            tau = module.tau
            D_o = module.D_o
            
            # Compute both normalization factors
            norm_elec = (module.ELECMASS * module.LIGHTSPEED * module.Omega) / module.ELEMCHARGE
            norm_mag  = (module.ELECMASS * module.Omega) / module.ELEMCHARGE
            
            # Apply selected normalization mode
            if NORM_MODE.upper() == "ELEC":
                Norm = norm_elec
            else:
                Norm = norm_mag  # Default fallback to Magnetic
            
            # Update global references for downstream cells
            selected_folder = target_folder
            
            # Display validated parameters
            print(f"a     = {a}")
            print(f"tau   = {tau}")
            print(f"D_o   = {D_o}")
            print(f"Norm  ({NORM_MODE.upper()}) = {Norm:.4e}")
            print(f"\n📁 Selected Folder: {selected_folder}")
            
        except AttributeError as e:
            print(f"❌ Missing expected variable in MainVars.py: {e}")
        except Exception as e:
            print(f"❌ Unexpected error loading MainVars.py: {e}")

# Bind callback to dropdown value changes
folder_dropdown.observe(on_folder_change, names='value')
```


    Dropdown(description='Folder:', options=('   ', '/home/big4/castillo/conversion_a=74_tau_6_ne=0.40_2Jb_stop_wi…



    Output()


### <a id="jupyter-notebook-cells---execution-context-validation">Execution Context Validation</a>
<details>
<summary><b>Tag: validator1.0</b> – Optional sanity check for selected folder and normalization factor</summary>

**Description**:  
A lightweight validation snippet that confirms whether a simulation directory has been properly selected via the interactive navigator. Displays the active folder path and the computed normalization factor (`Norm`), or prompts the user to make a selection if the global state remains uninitialized. Recommended as a quick checkpoint before executing downstream analysis, data loading, or plotting cells.

**Key Features**:
- ✅ Verifies global state (`selected_folder`, `Norm`) after widget interaction
- 🔔 Provides explicit console feedback to prevent silent failures
- 🛑 Safe-to-skip guard clause that improves notebook reproducibility

**Dependencies**:  
None (inherits `selected_folder` and `Norm` from `navigator1.0`)

**Usage Notes**:
- Run immediately after selecting a folder from the dropdown.
- If the warning appears, ensure the chosen directory contains a valid `MainVars.py` and that the callback executed without errors.
- Does not alter state; purely diagnostic.

**Expected Output**:

</details>

```python
# Optional sanity check: verify that the interactive selector has initialized global state
if selected_folder:
    print(f"✅ Working on folder: {selected_folder}/")
    print(f"   Normalization factor (Norm): {Norm:.4e}")
else:
    print("⚠️ No valid folder selected. Please choose a directory from the dropdown above.")
```

    ✅ Working on folder: /home/big4/castillo/conversion_a=74_tau_6_ne=0.40_2Jb_stop_window/tracking/
       Normalization factor (Norm): 3.2107e+12


### <a id="jupyter-notebook-cells---helmholtz-decomposition-3x3-time-evolution">Helmholtz Decomposition 3x3 Time Evolution</a>
<details>
<summary><b>Tag: helmholtz_plotter1.0</b> – 3x3 matrix plot of Helmholtz components (Total, Gradient, Solenoidal) across 3 time dumps</summary>

⚠️ **USER WARNING:** This cell generates complex 3x3 grids. **Ensure `xmin_limit` and `xmax_limit` lists** match the number of dumps (default 3). If your dump list has fewer/more elements, adjust the limit lists or pass scalars to apply the same limit to all columns.

**Description**:  
Generates a publication-ready 3x3 visualization comparing the Helmholtz decomposition of a field across three different time steps.
- **Rows**: Helmholtz components (Total, Gradient/Longitudinal, Solenoidal/Transverse).
- **Columns**: Temporal snapshots defined in `dumps`.
- **Layers**: Density background (grayscale) + Field overlay (color).
- **Features**: 
  - Independent X-axis limits per column.
  - Internal colorbars with automatic sub-ticks for high precision.
  - Clean labels with LaTeX notation ($E_{x}$, $E_{es}$, $E_{v}$).

**Key Features**:
- 📐 **Flexible Limits**: `xmin_limit=[60, 60, 60]` allows zooming into different regions for each time step independently.
- 🎨 **Smart Colorbars**: Auto-generated internal colorbars that adapt to the dynamic range of each subplot (`AutoMinorLocator`).
- 🏷️ **Publication Labels**: Subplots automatically labeled (a) through (i).

**Dependencies**:  
`h5py`, `numpy`, `matplotlib`, `os`.

**Usage Notes**:
- Requires `selected_folder` to be set via `navigator1.0`.
- Fourier data is expected in `selected_folder/FourierFieldsOptimized/`.
- `vrange_fourier` scales dynamically based on the maximum absolute value of the field in each plot.

**Expected Output**:
- 🖼️ 3x3 Plot Grid.
- 📋 Console print confirming save if `save_fig=True`.

</details>

```python
import os
import h5py
import numpy as np
import matplotlib.pyplot as plt
from matplotlib.ticker import AutoMinorLocator, MultipleLocator

def plot_9_subplots_helmholtz_time_comparison_limits_ticks(
    field="Elec",
    plane="xy",
    density="RhoEL",
    dumps=[5, 7, 12],
    component="x",
    helmholtz_types=["Total", "Gradient", "Solenoidal"],
    cmap_density="Greys",
    cmap_fourier="gist_rainbow_r",
    alpha_density=1.00,
    alpha_fourier=0.85,
    vrange_density=(0.0, 3.0),
    vrange_fourier=(-0.99, 0.99),
    figsize=(10.11, 10.5),
    show_colorbars=True,
    save_fig=False,
    plot_density=True,
    # Limits logic
    xmin_limit=[60, 60, 60],
    xmax_limit=[90, 90, 90],
    ymin_limit=-15,
    ymax_limit=15,
    tick_fontsize=13,
    cbar_tick_fontsize=12,
    # Colorbar styling
    cbar_left_pad=0.03,
    cbar_right_pad=0.03,
    cbar_left_anchor=(0.0, 0.5),
    cbar_right_anchor=(2.3, 0.5),
    cbar_panchor_left=(0, 0.5),
    cbar_panchor_right=(1, 0.5),
    cbar_shrink=0.85,
    # Shading/Border
    shade_density=True,
    azdeg=315,
    altdeg=45,
    vert_exag=0.5,
    blend_mode='overlay',
    line_border=True,
    line_position=80,
    line_color='green',
    line_width=0.5
):
    """
    Generates a 3x3 matrix of subplots comparing Helmholtz decomposition components across 3 dumps.
    
    - Rows: Helmholtz components (Total, Gradient, Solenoidal)
    - Cols: Time dumps
    - Features: Internal colorbars with sub-ticks, independent X-limits per column, clean subplot labels.
    """
    
    # Paths (requires selected_folder from navigator)
    folder_density = f"{selected_folder}/DataSlices/"
    folder_fourier = f"{selected_folder}/FourierFieldsOptimized/"

    helmholtz_notation = {
        "Total": r"$E_{x}$",
        "Gradient": r"$E_{es}$",
        "Solenoidal": r"$E_{v}$"
    }
    
    # sharex=False allows independent X limits per column
    fig, axes = plt.subplots(3, 3, figsize=figsize, sharex=False, sharey=True, squeeze=False)
    fig.suptitle(f"{field}$_{{{component}}}$ ({plane}) - {'Density ' + density + ' + ' if plot_density else ''}Helmholtz Decomposition", 
                 fontsize=16, y=0.985)

    # --- File Validation ---
    all_files_exist = True
    for dump in dumps:
        fourier_file = os.path.join(folder_fourier, f"sliceTotalGradSolen_Dump_{str(dump).zfill(3)}.h5")
        try:
            with h5py.File(fourier_file, 'r') as f:
                # Check existence of a known dataset
                _ = f[field + "MultiField_Total_x_xy"]
        except Exception as e:
            print(f"⚠️ Fourier file missing for dump={dump}: {e}")
            all_files_exist = False
            
        if plot_density:
            density_file = os.path.join(folder_density, f"slice_Dump_{str(dump).zfill(3)}.h5")
            try:
                with h5py.File(density_file, 'r') as f:
                    _ = f[density + "_" + plane][()]
            except Exception as e:
                print(f"⚠️ Density file missing for dump={dump}: {e}")
                all_files_exist = False
    
    if not all_files_exist:
        raise FileNotFoundError("Required HDF5 files missing. Check paths or file names.")

    subplot_labels = ['(a)', '(b)', '(c)', '(d)', '(e)', '(f)', '(g)', '(h)', '(i)']
    last_im2 = None

    # --- Plotting Loop ---
    for i, helmholtz in enumerate(helmholtz_types):
        for j, dump in enumerate(dumps):
            ax = axes[i, j]
            
            # 1. Load Density Data (Background)
            filename_density = os.path.join(folder_density, f"slice_Dump_{str(dump).zfill(3)}.h5")
            with h5py.File(filename_density, 'r') as f:
                if plot_density:
                    dataDensity = f[density + "_" + plane][()]
                bounds = f['globalGridGlobal'].attrs.get('vsLowerBounds') * 1.0e+6
                upper_bounds = f['globalGridGlobal'].attrs.get('vsUpperBounds') * 1.0e+6
                timeDump = f['time'].attrs.get('vsTime') * 1.0e+15

            xmin_file, ymin_file, _ = bounds
            xmax_file, ymax_file, _ = upper_bounds

            # Apply column-specific X limits
            xmin_plot = xmin_limit[j] if isinstance(xmin_limit, (list, tuple)) else xmin_limit
            xmax_plot = xmax_limit[j] if isinstance(xmax_limit, (list, tuple)) else xmax_limit
            
            # Fallbacks
            xmin_plot = xmin_plot if xmin_plot is not None else xmin_file
            xmax_plot = xmax_plot if xmax_plot is not None else xmax_file
            ymin_plot = ymin_limit if ymin_limit is not None else ymin_file
            ymax_plot = ymax_limit if ymax_limit is not None else ymax_file

            # 2. Plot Density
            if plot_density:
                ax.imshow(
                    dataDensity.T,
                    extent=[xmin_file, xmax_file, ymin_file, ymax_file],
                    aspect='equal',
                    cmap=cmap_density,
                    alpha=alpha_density,
                    vmin=vrange_density[0],
                    vmax=vrange_density[1],
                    zorder=1
                )

            # 3. Load Fourier Data (Overlay)
            filename_fourier = os.path.join(folder_fourier, f"sliceTotalGradSolen_Dump_{str(dump).zfill(3)}.h5")
            dset_name = f"{field}MultiField_{helmholtz}_{component}_{plane}"
            s_fourier = None
            
            try:
                with h5py.File(filename_fourier, 'r') as f_fourier:
                    s_fourier = f_fourier[dset_name][()]
            except Exception as e:
                print(f"⚠️ Fourier data unavailable for dump={dump}, {helmholtz}")

            # 4. Plot Fourier Overlay & Colorbar
            if s_fourier is not None and s_fourier.size > 0:
                Efabs = max(abs(s_fourier.min()), abs(s_fourier.max()))
                # Dynamic scaling
                vmin_f, vmax_f = (vrange_fourier[0] * Efabs, vrange_fourier[1] * Efabs) if Efabs != 0 else (-1e-10, 1e-10)
                
                im2 = ax.imshow(
                    s_fourier.T,
                    extent=[xmin_file, xmax_file, ymin_file, ymax_file],
                    aspect='equal',
                    cmap=cmap_fourier,
                    vmin=vmin_f,
                    vmax=vmax_f,
                    alpha=alpha_fourier,
                    zorder=2
                )
                last_im2 = im2

                # Internal Colorbar with Sub-ticks
                if show_colorbars:
                    cax = ax.inset_axes([0.12, 0.03, 0.76, 0.035])
                    cbar = fig.colorbar(im2, cax=cax, orientation='horizontal')
                    
                    # Ticks configuration
                    cbar.ax.xaxis.set_ticks_position('top')
                    cbar.ax.xaxis.set_label_position('top')
                    cbar.ax.tick_params(axis='x', labelsize=cbar_tick_fontsize, 
                                        top=True, bottom=False, labeltop=True, labelbottom=False, pad=2)
                    
                    # Add minor ticks (sub-ticks)
                    cbar.ax.xaxis.set_minor_locator(AutoMinorLocator(4))
                    cbar.ax.tick_params(axis='x', which='minor', 
                                        top=True, bottom=False, length=3, width=1, 
                                        color=(0, 0, 0, 0.5))
                    
                    # Style transparency
                    cbar.ax.spines[:].set_visible(False)
                    cax.set_frame_on(False)
                    cax.patch.set_facecolor('white')
                    cax.patch.set_alpha(0.75)

            # 5. Vertical Line Border
            if line_border:
                ax.axvline(x=line_position, color=line_color, linestyle='--', linewidth=line_width, zorder=3)

            # 6. Axes Limits & Ticks
            ax.set_xlim(xmin_plot, xmax_plot)
            ax.set_ylim(ymin_plot, ymax_plot)
            
            ax.tick_params(
                left=(j == 0),
                bottom=(i == 2),
                labelleft=(j == 0),
                labelbottom=(i == 2),
                labelsize=tick_fontsize,
                colors='black'
            )
            for spine in ax.spines.values():
                spine.set_visible(True)
            
            # 7. Labels
            # Subplot index (a)-(i)
            ax.text(0.05, 0.95, subplot_labels[i * 3 + j], transform=ax.transAxes,
                    fontsize=15, fontweight='normal', fontstyle='normal', color='black',
                    ha='left', va='top',
                    bbox=dict(boxstyle='round,pad=0.25', facecolor='none', edgecolor='none'))
            
            # Time label (only top row)
            if i == 0:
                ax.text(0.5, 0.98, f"t = {timeDump:.2f} fs",
                        transform=ax.transAxes, fontsize=12, fontweight='semibold',
                        color='white', ha='center', va='top',
                        bbox=dict(boxstyle='round,pad=0.2', facecolor='black', alpha=0.6, edgecolor='none'))

    # Final Layout
    plt.tight_layout(rect=[0.00, 0.00, 0.98, 0.96])
    plt.subplots_adjust(wspace=0.00, hspace=0.00)
    plt.show()

    if save_fig:
        fig_name = f"helmholtz_3x3_{dumps}_E{component}_{density}_{cmap_fourier}.png"
        fig.savefig(fig_name, dpi=150, bbox_inches='tight', facecolor='white')
        print(f"✅ Figure saved: {fig_name}")
```

### <a id="jupyter-notebook-cells---helmholtz-execution-example">Helmholtz Plot Execution Example</a>
<details>
<summary><b>Tag: helmholtz_runner1.0</b> – Concrete function call for 3-dump Helmholtz decomposition with custom zoom regions</summary>

⚠️ **USER WARNING:** This block configures a specific visualization run. **Ensure `xmin_limit` and `xmax_limit` lists match the length of `dumps`** (3 elements here). Mismatched list lengths will cause indexing errors or unexpected zoom behavior.

**Description**:  
Executes the `plot_9_subplots_helmholtz_time_comparison_limits_ticks()` engine to render a 3x3 grid comparing Helmholtz decomposition components across three temporal snapshots (`dumps=[16, 20, 29]`).
- **Density layer disabled**: `plot_density=False` renders only the field overlay for cleaner analysis of field structures.
- **Per-column zoom**: Independent X-axis limits track the evolving interaction region across time.
- **Auto-filename**: Output file encodes dump indices, component, density type, and colormap for organized versioning.

**Customization Guide**:
- 🔹 **`dumps`**: List of dump indices. Must have corresponding HDF5 files in `DataSlices/` and `FourierFieldsOptimized/`.
- 🔹 **`xmin_limit` / `xmax_limit`**: List of length `len(dumps)`. Use scalars (e.g., `xmin_limit=60`) to apply uniform limits.
- 🔹 **`plot_density`**: Set `True` to overlay density background; `False` for field-only visualization.
- 🔹 **`cmap_fourier`**: Colormap for field overlay. `"seismic"` for diverging red/blue, `"gist_rainbow_r"` for spectral.
- 🔹 **`save_fig`**: Toggle file export. Filename auto-generated inside the function.

**Dependencies**:  
Requires `plot_9_subplots_helmholtz_time_comparison_limits_ticks` definition, global `Norm` (from `navigator1.0`), and `selected_folder`.

**Expected Output**:
- 🖼️ 3x3 grid plot with field-only rendering, per-column zoom, and internal colorbars with sub-ticks.
- 💾 File saved as: `helmholtz_3x3_[16, 20, 29]_Ex_RhoEL_gist_rainbow_r.png` (example based on defaults).
- 📋 Console confirmation: `✅ Figure saved: ...`

**Execution Note**:
```python
# Adjust dumps/limits → Run cell → Verify console output confirms successful render & save
```
</details>

```python
# ▶▶▶ Execution block: Helmholtz decomposition time evolution (3 dumps)
plot_9_subplots_helmholtz_time_comparison_limits_ticks(
    # Temporal snapshots to visualize
    dumps=[16, 20, 29],
    
    # Visual style: Fourier field colormap
    cmap_fourier="seismic",  # Options: "seismic", "gist_rainbow_r", "RdBu_r"
    
    # Output configuration
    save_fig=True,                  # Export PNG with auto-generated filename
    plot_density=False,             # Show only field overlay (no density background)
    
    # Column-specific X-axis limits (one value per dump in `dumps` list)
    xmin_limit=[65, 75, 79],        # Left boundary for dumps 16, 20, 29 respectively
    xmax_limit=[95, 105, 109],      # Right boundary for dumps 16, 20, 29 respectively
    
    # Y-axis limits (shared across all subplots)
    # ymin_limit=-15, ymax_limit=15  # Defaults from function signature
)
```


    
![png](output_7_0.png)
    


    ✅ Figure saved: helmholtz_3x3_[16, 20, 29]_Ex_RhoEL_seismic.png

