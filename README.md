# IVUS–EM Reconstruction

Offline reconstruction software accompanying:

> **Learned ultrasound segmentation and deformable CT fusion for augmented reality endovascular surgery**  
> Dillon et al.

This repository performs learned intravascular ultrasound (IVUS) segmentation and three-dimensional reconstruction by combining IVUS images with synchronized electromagnetic (EM) tracking transforms. The command-line workflow can automatically download the associated datasets from Zenodo, run segmentation and TSDF fusion, display the reconstruction, and save reviewer-facing outputs.

## Repository contents

The files have the following roles:

- `run_mapping.py` — command-line entry point, dataset download, replay, reconstruction, visualization, and output export.
- `segmentation_helpers_runtime.py` — DeepLumen model definition, inference, and segmentation post-processing.
- `reconstruction_helpers_runtime.py` — reconstruction, point-cloud, TSDF, transform, and visualization utilities used at runtime.
- `aortascope_mapping_params.yaml` — mapping and model configuration.
- `calibration_parameters_ivus.yaml` — Dataset-specific calibration is loaded during replay.
- `requirements.txt` — pinned Python dependencies.

## Data

The associated IVUS–EM datasets are available from:

https://zenodo.org/records/20737792

The following named datasets are supported:

```text
patient_1
patient_2
patient_3
patient_4
patient_5
patient_6
patient_7
sheep_1
sheep_2
sheep_3
```

When a named dataset is requested and is not already present, `run_mapping.py` downloads the corresponding ZIP archive and extracts it automatically.


## System requirements

The software was tested in the following environment:

- Ubuntu/Linux
- Python 3.9
- A graphical desktop session capable of displaying Open3D and OpenCV windows
- A C++ compiler and CMake for building the Voxblox Python bindings
- Sufficient disk space for the selected Zenodo dataset and generated outputs

A CUDA-compatible GPU is optional for TensorFlow inference. TensorFlow uses an available compatible GPU when one is visible to the installed TensorFlow build; otherwise it runs supported operations on the CPU. CPU execution is expected to be slower.

The current script is not headless: it creates an Open3D visualization window and an OpenCV segmentation window. Run it from a local graphical session or a correctly configured remote graphical session.

## Python environment

Create and activate a Python 3.9 virtual environment:

```bash
python3.9 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
```

Install the pinned Python packages:

```bash
python -m pip install -r requirements.txt
```

The tested requirements are:

```text
numpy==1.26.4
open3d==0.18.0
PyYAML==6.0.1
opencv-python==4.9.0.80
tensorflow==2.16.1
scipy==1.13.1
```

The remaining imports used by the code are part of the Python standard library and do not require separate installation.

## Voxblox Python bindings

The reconstruction requires the Python bindings from:

https://github.com/PRBonn/voxblox_pybind

These bindings provide:

```python
from voxblox import FastTsdfIntegrator
```

Install the required Ubuntu build dependencies:

```bash
sudo apt-get update
sudo apt-get install -y \
    build-essential \
    cmake \
    libprotobuf-dev \
    protobuf-compiler \
    python3-dev \
    python3-pip \
    git
```

Clone the repository with its submodules and install it while the project virtual environment is active:

```bash
git clone --recurse-submodules https://github.com/PRBonn/voxblox_pybind.git
cd voxblox_pybind
make install
cd ..
```

Verify that the binding is available to the same Python interpreter used for this project:

```bash
python -c "from voxblox import FastTsdfIntegrator; print('Voxblox import successful')"
python -m pip show voxblox
```

The tested local installation reports:

```text
Name: voxblox
Version: 0.1
```

For a fixed archival release, record and pin the exact `voxblox_pybind` commit used for testing:

```bash
cd voxblox_pybind
git rev-parse HEAD
```

Then check out that commit before installation:

```bash
git checkout <COMMIT_HASH>
git submodule update --init --recursive
make install
```

## Model weights

The trained DeepLumen model weights are required but are not downloaded from Zenodo by the current script.

Provide the model in either of two ways:

1. Set `model_path` in `aortascope_mapping_params.yaml`.
2. Supply the weights on the command line with `--model`.

For example:

```bash
python run_mapping.py \
    --dataset-name patient_1 \
    --model /absolute/path/to/model.weights.h5
```

The command-line argument takes precedence over `model_path` in the YAML configuration.

Before sharing the repository, make sure the weights are available to reviewers through the private review package, a stable repository, or a permanent data archive. Do not commit large model files directly to ordinary Git history when they exceed GitHub's file-size limits.


## Quick start

From the repository root:

```bash
source .venv/bin/activate
python run_mapping.py --dataset-name patient_1
```

The short form is equivalent:

```bash
python run_mapping.py -d patient_1
```

This command:

1. checks for `data/patient_1`;
2. downloads the archive from Zenodo if the dataset is absent;
3. verifies the archive checksum;
4. extracts and resolves the replay directory;
5. loads the model and calibration;
6. runs DeepLumen segmentation and IVUS–EM reconstruction;
7. displays the segmentation and 3D reconstruction;
8. saves the reconstruction outputs.

## Command-line options

Show all options with:

```bash
python run_mapping.py --help
```

### Run a named Zenodo dataset

```bash
python run_mapping.py --dataset-name patient_1
```

### Use an already extracted dataset

```bash
python run_mapping.py \
    --dataset-path /absolute/path/to/extracted/dataset
```

The supplied directory, or one of its subdirectories, must contain an accepted image folder and an accepted EM-transform folder.

### Use a different data root

```bash
python run_mapping.py \
    --dataset-name patient_1 \
    --data-root /absolute/path/to/data
```

### Override the mapping configuration

```bash
python run_mapping.py \
    --dataset-name patient_1 \
    --config /absolute/path/to/aortascope_mapping_params.yaml
```

### Override the model weights

```bash
python run_mapping.py \
    --dataset-name patient_1 \
    --model /absolute/path/to/model.weights.h5
```

### Choose the output directory

```bash
python run_mapping.py \
    --dataset-name patient_1 \
    --output-dir /absolute/path/to/results/patient_1
```

### Require an existing local dataset

Use `--no-download` to prevent network access:

```bash
python run_mapping.py \
    --dataset-name patient_1 \
    --no-download
```

The dataset must already exist under:

```text
<data-root>/patient_1/
```

### Redownload a dataset

```bash
python run_mapping.py \
    --dataset-name patient_1 \
    --force-download
```

This removes an existing cached archive when necessary, downloads a fresh copy, verifies its checksum, and re-extracts the selected dataset.

`--force-download` applies only to `--dataset-name`, not to `--dataset-path`.

## Outputs

By default, outputs are saved under:

```text
outputs/<dataset_name>/
```

For example:

```text
outputs/patient_1/
├── lumen_mesh.ply
├── lumen_point_cloud.ply
├── branch_point_cloud.ply
├── segmentation_preview.png
└── run_summary.json
```

The files contain:

- `lumen_mesh.ply` — reconstructed lumen surface mesh.
- `lumen_point_cloud.ply` — accumulated lumen point cloud.
- `branch_point_cloud.ply` — accumulated branch-region point cloud.
- `segmentation_preview.png` — most recent successful segmentation overlay.
- `run_summary.json` — compact run metadata.

Example `run_summary.json`:

```json
{
  "dataset": "patient_1",
  "runtime_seconds": 180.2,
  "model": "model.weights.h5",
  "voxel_size": 0.002,
  "outputs": {
    "lumen_mesh": "lumen_mesh.ply",
    "lumen_point_cloud": "lumen_point_cloud.ply",
    "branch_point_cloud": "branch_point_cloud.ply",
    "segmentation_preview": "segmentation_preview.png"
  }
}
```

The output summary intentionally does not report loaded, processed, or skipped-frame counts.

The script raises an error instead of writing a successful summary when it does not produce a required mesh, point cloud, or segmentation preview.

## Runtime

Runtime depends on:

- dataset length;
- CPU and GPU hardware;
- TensorFlow device availability;
- voxel size;
- visualization overhead;
- model and reconstruction configuration.

The exact end-to-end runtime for each successful run is written to `run_summary.json`.

Before reviewer release, record at least one measured reference runtime here:

```text
Reference dataset:
Tested hardware:
TensorFlow device:
Runtime:
```

Do not report an estimated value as a benchmark; use a completed run from the released code and environment.

## Configuration

The primary mapping parameters are stored in:

```text
aortascope_mapping_params.yaml
```

Important parameters include:

```text
gating
tsdf_map
voxel_size
hybrid_seg
conf_threshold
deeplumen_on
deeplumen_slim_on
deeplumen_lstm_on
endoanchor
model_path
vpC_map
orifice_center_map
machine
figure_mapping
```

For the reviewer workflow described here:

- `deeplumen_on` should enable the released DeepLumen model.
- `tsdf_map` should enable TSDF reconstruction.
- `model_path` must point to the released model weights unless `--model` is supplied.
- `machine` must match the acquisition format expected by the image-cropping code.
- `voxel_size` should match the value used to generate the reported reconstruction.

The released YAML file should contain the exact settings used for the manuscript results.

## Using a local dataset

An already extracted dataset may be supplied with:

```bash
python run_mapping.py \
    --dataset-path /absolute/path/to/dataset \
    --model /absolute/path/to/model.weights.h5
```

A minimal expected layout is:

```text
dataset/
├── image_numpys/
│   ├── image_0.npy
│   ├── image_1.npy
│   └── ...
├── EM_data/
│   ├── transform_0.npy
│   ├── transform_1.npy
│   └── ...
└── calibration_parameters_ivus.yaml
```

Alternate accepted image and transform folder names are listed in the Data section.

Image and transform files are sorted naturally by their numeric frame indices. The number of image files must equal the number of transform files, and every transform must have shape `(4, 4)`.

## GPU and CPU execution

No command-line device flag is required for normal TensorFlow behavior.

To inspect available TensorFlow devices:

```bash
python - <<'PY'
import tensorflow as tf

print("GPUs:", tf.config.list_physical_devices("GPU"))
print("CPUs:", tf.config.list_physical_devices("CPU"))
PY
```

When no compatible GPU is visible, TensorFlow uses the CPU for supported operations. A machine having an NVIDIA GPU is not sufficient by itself; the TensorFlow installation, drivers, and runtime libraries must also expose the device to TensorFlow.

For reproducible CPU testing, run in a new shell with:

```bash
CUDA_VISIBLE_DEVICES=-1 python run_mapping.py --dataset-name patient_1
```

CPU-only execution should be tested before release when CPU support is claimed.

## Verification

Verify the Python source files compile:

```bash
python -m py_compile \
    run_mapping.py \
    segmentation_helpers_runtime.py \
    reconstruction_helpers_runtime.py
```

Verify the main imports:

```bash
python - <<'PY'
import cv2
import numpy
import open3d
import scipy
import tensorflow
import yaml
from voxblox import FastTsdfIntegrator

print("All required imports succeeded.")
PY
```

Check the command-line interface:

```bash
python run_mapping.py --help
```

Run one complete dataset and confirm that all five output files are created.

## Troubleshooting

### `ModuleNotFoundError: No module named 'voxblox'`

The compiled Voxblox bindings are not installed for the active Python interpreter.

Check:

```bash
which python
python -m pip show voxblox
python -c "from voxblox import FastTsdfIntegrator"
```

Rebuild `voxblox_pybind` while the intended virtual environment is active.

### Model weights not found

Supply an existing model path:

```bash
python run_mapping.py \
    --dataset-name patient_1 \
    --model /absolute/path/to/model.weights.h5
```

Also verify `model_path` in `aortascope_mapping_params.yaml`.

### Calibration file not found

Make sure `calibration_parameters_ivus.yaml` is present beside the Python scripts and that the selected dataset contains its acquisition-specific calibration file when required.

### Open3D or OpenCV window errors

The current implementation requires a graphical display. Confirm that the `DISPLAY` environment variable is configured:

```bash
echo "$DISPLAY"
```

Run locally from a graphical desktop session or configure X forwarding correctly. A headless server requires code changes or a virtual display.

### Dataset exists but has the wrong structure

A named dataset directory must contain both an accepted image folder and an accepted transform folder. Remove the incomplete directory or rerun:

```bash
python run_mapping.py \
    --dataset-name patient_1 \
    --force-download
```

### TensorFlow does not detect the GPU

Inspect TensorFlow visibility:

```bash
python -c "import tensorflow as tf; print(tf.config.list_physical_devices('GPU'))"
```

An empty list means TensorFlow will use the CPU. Verify the TensorFlow installation and system GPU runtime separately.

### Output-saving error

The script saves outputs only after a successful run produces:

- a non-empty lumen mesh;
- a non-empty lumen point cloud;
- a non-empty branch point cloud;
- a segmentation preview.

Review the preceding console output and configuration when any required geometry is empty.



## Citation

When using this software or the associated datasets, cite the accompanying manuscript and the Zenodo record.

Manuscript citation:

```text
Dillon, T. M. et al. Learned ultrasound segmentation and deformable CT fusion
for augmented reality endovascular surgery. Manuscript under review.
```


## Intended use

This software is research code and is not a medical device. It is not intended for clinical diagnosis, treatment, procedural decision-making, or patient care.

## License and review status

This repository is currently prepared for confidential editorial and peer review.

No open-source license is granted unless and until public-release and licensing terms are approved by the relevant institutional intellectual-property office. Review access does not grant permission for redistribution, commercial use, sublicensing, or creation of derivative products beyond what is necessary for manuscript evaluation.

Third-party dependencies remain subject to their respective licenses.
