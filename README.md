# Role of Spatial Coherence in Single-Shot Lensless Image Reconstruction

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.24030562.svg)](https://doi.org/10.5281/zenodo.24030562)

This repository contains code for generating synthetic object libraries, simulating lensless measurements under partial spatial coherence, and reconstructing amplitude and phase objects using partially coherent and coherent forward models.

The code supports the workflow used in the paper **Role of Spatial Coherence in Single-Shot Lensless Image Reconstruction**, *Optics Letters* (2026), [doi:10.1364/OL.611778](https://doi.org/10.1364/OL.611778). The main computational components are the Generalized Van Cittert-Zernike Schell (GVS) propagator for partially coherent light, developed by Montoya et al. [[1]](#citation), a coherent angular spectrum model, and PyTorch-based inverse reconstruction routines.

The data generation notebooks use the authors' implementation of the GVS propagator, [`gvs_propagator`](https://github.com/mmonto952/gvs-propagator). It is not part of this repository and must be installed separately (see [Installation](#installation)).

## Repository structure

```text
.
├── generate_imgs.mlx
├── datagen_amp.ipynb
├── datagen_phase.ipynb
├── recon_amp.ipynb
└── recon_phase.ipynb
```

## File descriptions

`generate_imgs.mlx` generates a glyph-based synthetic object library in MATLAB. It creates four object classes: high-frequency high-density, high-frequency low-density, low-frequency high-density, and low-frequency low-density. The function saves the object images and an `object_library_metadata.csv` file containing spatial frequency and feature density scores.

`datagen_amp.ipynb` generates simulated amplitude object measurements. Each grayscale object is interpreted as an amplitude transmittance with zero phase. The object is padded, propagated using the partially coherent forward model, cropped back to the detector field of view, normalised, and saved as a simulated detector measurement.

`datagen_phase.ipynb` generates simulated phase object measurements. Each grayscale object is mapped to a phase delay between 0 and pi, and the object transmittance is represented as a unit modulus complex field. Measurements are generated at multiple detector distances, typically 2 mm and 4 mm.

`recon_amp.ipynb` performs amplitude reconstruction from simulated measurements. It reconstructs each object using both the partially coherent GVS model and the coherent model. The final reconstructions are saved as `amp_GVS.png` and `amp_COH.png`.

`recon_phase.ipynb` performs phase-only reconstruction from two detector plane measurements. It reconstructs the phase using both the partially coherent GVS model and the coherent model. The final phase reconstructions are saved as `phase_GVS.png` and `phase_COH.png`.

`datagen_amp.ipynb` and `datagen_phase.ipynb` take their partially coherent forward model (complex coherence factor, Schell propagation and angular spectrum GVS propagation) from the external `gvs_propagator` package [[1]](#citation). The reconstruction notebooks contain their own PyTorch implementation of the coherent and GVS forward models and do not import `gvs_propagator`.

## Installation

Create a Python environment (Python 3.11 to 3.14) and install the required packages.

```bash
pip install numpy scipy imageio matplotlib plotly dask opencv-python jupyter
```

Install PyTorch using the build appropriate for the available CPU or CUDA environment.

Install the GVS propagator [[1]](#citation) from its authors' repository, pinned to the commit this code was developed against.

```bash
pip install --no-deps "git+https://github.com/mmonto952/gvs-propagator@f505a41b5e86f5393e9d12429a9b2f05464b39c1"
```

`--no-deps` skips the package's own dependency list, which adds packages this workflow does not use and constrains the PyTorch version. The packages installed above cover everything the notebooks need from it.

The MATLAB object generation file requires MATLAB. The Image Processing Toolbox is needed for image resizing, rotation, filtering, and thresholding. The built in digit data option also requires the MATLAB digit dataset function used by the code. If that dataset is unavailable, provide a custom glyph folder through the configuration.

## Running the workflow

The data generation notebooks import `gvs_propagator`, so install it first (see [Installation](#installation)).

### 1. Generate synthetic objects

Open `generate_imgs.mlx` in MATLAB and run

```matlab
config.OutputRoot = 'path_to_output_folder';
config.GlyphFolder = 'path_to_optional_glyph_folder';
config.UseBuiltInDigits = true;
config.ImageSize = 2000;
config.ObjectsPerClass = 25;
config.RandomSeed = 22;

generate_imgs(config);
```

This creates

```text
glyph_object_library/
├── objects/
│   ├── HFHD/
│   ├── HFLD/
│   ├── LFHD/
│   └── LFLD/
└── object_library_metadata.csv
```

The Python notebooks expect object images to be addressable through `objects_root/class/object_id.png`. If the MATLAB-generated filenames are used directly, either rename the generated images or update the filename pattern in the notebooks.

### 2. Generate amplitude measurements

Open `datagen_amp.ipynb` and fill in the parameter placeholders. The required fields include image size, wavelength, source-to-object distance, source radii, camera pixel size, padding factor, propagation distance, object root, object classes, object IDs, output directory, and coherence folder names.

The notebook saves measurements using the layout

```text
amplitude_measurement_root/
└── class/
    └── object_id/
        └── source_radius_name/
            └── Simulated_2mm.png
```

### 3. Generate phase measurements

Open `datagen_phase.ipynb` and fill in the parameter placeholders. The notebook uses the same object library but converts each object into a phase transmittance.

The notebook saves measurements using the layout

```text
phase_measurement_root/
└── class/
    └── object_id/
        └── source_radius_name/
            ├── Simulated_2mm.png
            └── Simulated_4mm.png
```

### 4. Reconstruct amplitude objects

Open `recon_amp.ipynb` and set the measurement root, output folder, object classes, object IDs, source radii, coherence names, measurement filename, and optimisation parameters.

The notebook saves

```text
amplitude_reconstruction_output/
└── class/
    └── object_id/
        └── source_radius_name/
            ├── amp_GVS.png
            └── amp_COH.png
```

`amp_GVS.png` is reconstructed using the partially coherent GVS model. `amp_COH.png` is reconstructed using the coherent forward model.

### 5. Reconstruct phase objects

Open `recon_phase.ipynb` and set the measurement root, output folder, object classes, object IDs, source radii, coherence names, detector plane distances, and optimisation parameters.

The notebook saves

```text
phase_reconstruction_output/
└── class/
    └── object_id/
        └── source_radius_name/
            ├── phase_GVS.png
            └── phase_COH.png
```

`phase_GVS.png` is reconstructed using the partially coherent GVS model. `phase_COH.png` is reconstructed using the coherent forward model.

## Physical model summary

The partially coherent measurement model is the GVS propagator [[1]](#citation). It uses a spatial coherence function obtained from the Fourier transform of the source intensity distribution. For each source radius, a circular source mask is generated, its complex coherence factor is computed, and Schell propagation is applied to the coherent detector intensity. This gives a partially coherent detector intensity for the specified illumination coherence state.

The coherent comparison model uses the same object and propagation geometry but omits the spatial coherence filtering step. This allows direct comparison between reconstructions obtained with a coherence-aware forward model and reconstructions obtained under a fully coherent assumption.

## Main parameters to set

The most important parameters are

```text
wavelength
z_source_object
z_object_camera or propagation_distances
dx_camera
num_pixels_x and num_pixels_y
source_radii
coh_names
objects_root
meas_root
output_root or base_recon_dir
object_groups
object_ids
n_iters
learning_rate
crop_border or roi_border
```

The placeholder expressions marked by `## ... ##` are not executable Python. Replace them with valid Python values before running the notebooks.

## Output policy

The reconstruction notebooks are configured to save only the reconstructed object images. They do not save loss plots, comparison sheets, metric summaries, or predicted detector intensity images.

## Suggested GitHub cleanup

Before committing the repository, remove generated and machine-specific files such as

```text
__pycache__/
*.pyc
.ipynb_checkpoints/
*.mat
*.npy
large generated measurement folders
large reconstruction output folders
```

Generated object libraries, simulated measurements, and reconstruction outputs can be large. They are better stored separately or released through a data repository.

## Citation

If this code is used, please cite the associated paper

Ganesh M. Balasubramaniam, Xiao-Liu Chu, and Matthew R. Foreman, "Role of Spatial Coherence in Single-Shot Lensless Image Reconstruction," Opt. Lett. (2026). https://doi.org/10.1364/OL.611778

```bibtex
@article{Balasubramaniam2026SpatialCoherence,
  author  = {Balasubramaniam, Ganesh M. and Chu, Xiao-Liu and Foreman, Matthew R.},
  title   = {{Role of Spatial Coherence in Single-Shot Lensless Image Reconstruction}},
  journal = {Optics Letters},
  year    = {2026},
  doi     = {10.1364/OL.611778}
}
```

The code itself is archived on Zenodo. To cite it directly, use https://doi.org/10.5281/zenodo.24030562, which always resolves to the latest version. Version 1.0.0 is https://doi.org/10.5281/zenodo.24030563.

The partially coherent forward model is the Generalized Van Cittert-Zernike Schell (GVS) propagator. Please also cite

[1] Manuel Montoya, Maria J. Lopera, Yunfeng Nie, and David Blinder, "Generalized Van Cittert-Zernike Schell propagator: an efficient algorithm for simulating partially coherent light," Opt. Express 33, 46313-46325 (2025). https://doi.org/10.1364/OE.571673

```bibtex
@article{Montoya2025GeneralizedLight,
  author  = {Montoya, Manuel and Lopera, Maria J. and Nie, Yunfeng and Blinder, David},
  title   = {{Generalized Van Cittert-Zernike Schell propagator: an efficient algorithm for simulating partially coherent light}},
  journal = {Optics Express},
  volume  = {33},
  number  = {22},
  pages   = {46313--46325},
  year    = {2025},
  doi     = {10.1364/OE.571673}
}
```

## License

This code is licensed under the GNU General Public License v3.0. See [LICENSE](LICENSE).
