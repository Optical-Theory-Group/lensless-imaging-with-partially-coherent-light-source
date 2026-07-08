# Role of Spatial Coherence in Single-Shot Lensless Image Reconstruction

This repository contains code for generating synthetic object libraries, simulating lensless measurements under partial spatial coherence, and reconstructing amplitude and phase objects using partially coherent and coherent forward models.

The code supports the workflow used in the manuscript **Role of Spatial Coherence in Single Shot Lensless Image Reconstruction**. The main computational components are a generalised Van Cittert Zernike and Schell model for partially coherent propagation, a coherent angular spectrum model, and PyTorch-based inverse reconstruction routines.

## Repository structure

```text
.
├── generate_imgs.mlx
├── datagen_amp.ipynb
├── datagen_phase.ipynb
├── recon_amp.ipynb
├── recon_phase.ipynb
└── gvs_propagator
    ├── __init__.py
    ├── coherent.py
    ├── constants.py
    ├── masks.py
    ├── partially_coherent.py
    ├── utils.py
    └── notebooks
        ├── utils.py
        └── img
            └── usaf.png
```

The `__pycache__` folders and compiled `.pyc` files are not required for GitHub and should be excluded from version control.

## File descriptions

`generate_imgs.mlx` generates a glyph-based synthetic object library in MATLAB. It creates four object classes: high-frequency high-density, high-frequency low-density, low-frequency high-density, and low-frequency low-density. The function saves the object images and an `object_library_metadata.csv` file containing spatial frequency and feature density scores.

`datagen_amp.ipynb` generates simulated amplitude object measurements. Each grayscale object is interpreted as an amplitude transmittance with zero phase. The object is padded, propagated using the partially coherent forward model, cropped back to the detector field of view, normalised, and saved as a simulated detector measurement.

`datagen_phase.ipynb` generates simulated phase object measurements. Each grayscale object is mapped to a phase delay between 0 and pi, and the object transmittance is represented as a unit modulus complex field. Measurements are generated at multiple detector distances, typically 2 mm and 4 mm.

`recon_amp.ipynb` performs amplitude reconstruction from simulated measurements. It reconstructs each object using both the partially coherent GVS model and the coherent model. The final reconstructions are saved as `amp_GVS.png` and `amp_COH.png`.

`recon_phase.ipynb` performs phase-only reconstruction from two detector plane measurements. It reconstructs the phase using both the partially coherent GVS model and the coherent model. The final phase reconstructions are saved as `phase_GVS.png` and `phase_COH.png`.

`gvs_propagator/coherent.py` contains routines for coherent propagation, including angular spectrum, Fresnel, and shifted Fresnel propagation, as well as intensity computation for specified wavefronts.

`gvs_propagator/partially_coherent.py` contains partially coherent propagation routines, including complex coherence factor computation, Schell propagation, angular spectrum GVS propagation, Fresnel GVS propagation, and direct mutual intensity-based propagation functions.

`gvs_propagator/masks.py` contains utilities for generating source and aperture masks, including circular sources, pinholes, slits, and multi-slit masks.

`gvs_propagator/utils.py` contains general utilities for normalisation, centre cropping, grid generation, radial grid generation, image zooming, and spherical wavefront multiplication.

`gvs_propagator/constants.py` defines basic metric unit conversions.

`gvs_propagator/notebooks/utils.py` contains optional notebook display utilities. It is not required for the reconstruction pipeline.

## Installation

Create a Python environment and install the required packages.

```bash
pip install numpy scipy imageio matplotlib plotly dask opencv-python jupyter
```

Install PyTorch using the build appropriate for the available CPU or CUDA environment.

The MATLAB object generation file requires MATLAB. The Image Processing Toolbox is needed for image resizing, rotation, filtering, and thresholding. The built in digit data option also requires the MATLAB digit dataset function used by the code. If that dataset is unavailable, provide a custom glyph folder through the configuration.

## Running the workflow

Run all Python notebooks from the repository root so that imports from `gvs_propagator` resolve correctly.

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

The partially coherent measurement model uses a spatial coherence function obtained from the Fourier transform of the source intensity distribution. For each source radius, a circular source mask is generated, its complex coherence factor is computed, and Schell propagation is applied to the coherent detector intensity. This gives a partially coherent detector intensity for the specified illumination coherence state.

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

If this code is used, please cite the associated manuscript

```text
Role of Spatial Coherence in Single-Shot Lensless Image Reconstruction
```

## License

Add the intended license before public release.
