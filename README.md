SRS Lipid Droplet StarDist

Automated lipid droplet instance segmentation and quantitative analysis in stimulated Raman scattering (SRS) microscopy images using StarDist.

This repository provides a Python-based workflow for identifying individual lipid droplets in SRS images and extracting quantitative measurements including droplet number, individual droplet area, total lipid area, and droplet size distribution.

⸻

Overview

Stimulated Raman scattering (SRS) microscopy provides label-free chemical imaging of lipid-rich cellular structures. Quantitative analysis of lipid droplets, however, can require substantial manual segmentation and counting.

This project uses StarDist 2D instance segmentation to automatically identify individual lipid droplets from SRS microscopy images.

The analysis workflow is:

SRS microscopy image
        ↓
Image normalization
        ↓
StarDist 2D
        ↓
Instance segmentation
        ↓
Individual lipid-droplet masks
        ↓
Quantitative analysis
        ↓
Count + Area + Size

Because StarDist performs instance segmentation, neighboring lipid droplets can be represented as separate objects rather than as a single binary foreground region.

⸻

Example Workflow

Original SRS Image
        ↓
Manual Instance Annotation
        ↓
StarDist Training
        ↓
Trained Model
        ↓
New SRS Image
        ↓
Prediction
        ↓
Individual Lipid Droplets
        ↓
Quantification

Example images and prediction results will be provided in the examples/ directory.

⸻

Features

The current analysis pipeline supports:

* Automated lipid-droplet detection from SRS microscopy images
* StarDist 2D instance segmentation
* Individual lipid-droplet counting
* Individual droplet area measurement
* Total lipid-droplet area measurement
* Mean and median droplet area
* Equivalent droplet diameter
* Conversion from pixels² to µm² using microscope calibration
* Batch processing of TIFF images
* Export of instance-segmentation masks
* Prediction-overlay generation for quality control
* Per-droplet CSV output
* Per-image summary CSV output

⸻

Repository Structure

SRS_LipidDroplet_StarDist/
│
├── README.md
├── LICENSE
├── .gitignore
├── CITATION.cff
├── requirements.txt
│
├── notebooks/
│   ├── 01_training.ipynb
│   └── 02_inference.ipynb
│
├── model/
│   └── lipid_droplet_v1/
│
└── examples/
    ├── example_SRS.tif
    ├── example_annotation.tif
    └── example_prediction.png

⸻

Installation

A dedicated Conda environment is recommended.

1. Create the environment

conda create -n lipid_ai python=3.11 -y

Activate it:

conda activate lipid_ai

2. Install dependencies

python -m pip install tensorflow stardist csbdeep tifffile scikit-image pandas matplotlib numpy ipykernel

3. Add the environment to Jupyter

python -m ipykernel install --user --name lipid_ai --display-name "Python (lipid_ai)"

Open Jupyter Notebook or JupyterLab and select:

Python (lipid_ai)

as the notebook kernel.

⸻

Using the Pretrained Model

The trained model is stored in:

model/lipid_droplet_v1/

Load it using:

from stardist.models import StarDist2D
model = StarDist2D(
    None,
    name="lipid_droplet_v1",
    basedir="model"
)

⸻

Analyze an SRS Image

from tifffile import imread
from csbdeep.utils import normalize
img = imread("example_SRS.tif")
img_norm = normalize(
    img,
    1,
    99.8
)
labels, details = model.predict_instances(
    img_norm
)
print("Detected lipid droplets:", labels.max())

The resulting labels image is an instance-segmentation mask:

0 = background
1 = lipid droplet 1
2 = lipid droplet 2
3 = lipid droplet 3
...

⸻

Quantitative Analysis

Individual droplets can be measured using scikit-image.

from skimage.measure import regionprops_table
import pandas as pd
measurements = regionprops_table(
    labels,
    properties=[
        "label",
        "area",
        "equivalent_diameter_area",
        "centroid"
    ]
)
df = pd.DataFrame(measurements)

For an image calibration of:

pixel_size = 0.293  # µm/pixel

area can be converted to µm² using:

df["area_um2"] = (
    df["area"] * pixel_size**2
)

Important: 0.293 µm/pixel is only an example. Users should enter the actual pixel calibration associated with their SRS acquisition.

⸻

Batch Analysis

The inference notebook can automatically process multiple SRS images.

For each image, the workflow can generate:

image_001_mask.tif
image_001_overlay.png
image_001_droplets.csv
image_002_mask.tif
image_002_overlay.png
image_002_droplets.csv
...
lipid_droplet_summary.csv

The summary includes measurements such as:

Measurement	Description
Droplet count	Number of detected lipid droplets
Total droplet area	Sum of segmented lipid-droplet area
Mean droplet area	Mean area of detected droplets
Median droplet area	Median area of detected droplets
Maximum droplet area	Largest detected droplet

⸻

Training

Training images and their corresponding manual annotations are used to train StarDist.

Manual masks use instance labels, rather than binary segmentation.

For example:

Background     = 0
Lipid droplet  = 1
Lipid droplet  = 2
Lipid droplet  = 3
...

A simplified training configuration is:

from stardist.models import Config2D, StarDist2D
config = Config2D(
    n_rays=32,
    grid=(1, 1),
    use_gpu=False,
    train_patch_size=(256, 256),
    train_batch_size=4,
    train_learning_rate=0.0003
)
model = StarDist2D(
    config,
    name="lipid_droplet_v1",
    basedir="model"
)

Training:

history = model.train(
    X_train,
    Y_train,
    validation_data=(X_val, Y_val),
    epochs=100,
    steps_per_epoch=50
)

Thresholds can subsequently be optimized using validation data:

model.optimize_thresholds(
    X_val,
    Y_val
)

See notebooks/01_training.ipynb for the complete workflow.

⸻

Annotation

Manual instance annotations can be generated using Labkit in Fiji/ImageJ.

Each lipid droplet should be annotated as an independent object.

The expected annotation format is an integer TIFF:

0 = background
1 = droplet 1
2 = droplet 2
3 = droplet 3
...

Binary masks in which all droplets have the same value are not appropriate for instance-segmentation training.

⸻

Quality Control

Automated segmentation results should always be visually inspected.

Users should examine prediction overlays for:

* missed lipid droplets,
* false-positive detections,
* merged neighboring droplets,
* incorrectly split droplets,
* inaccurate object boundaries,
* systematic failure to detect dim or small droplets.

Quantitative measurements should only be used after segmentation performance has been validated for the relevant imaging conditions.

⸻

Model Status

Current status: early research / pilot model

The current model was developed as an initial proof-of-concept for automated lipid-droplet segmentation in SRS microscopy images.

The training and validation datasets will be expanded as additional annotated SRS images become available.

Future versions are expected to include more extensive quantitative validation, including:

* Precision
* Recall
* F1 score
* Intersection over Union (IoU)
* Droplet-count error
* Droplet-area error
* Independent test datasets

Users should therefore validate the model on their own SRS imaging conditions before using the output for quantitative biological conclusions.

⸻

Reproducibility

Performance may depend on differences in:

* SRS imaging system
* Raman shift / imaging channel
* objective and magnification
* pixel size
* spatial resolution
* laser power
* image contrast
* signal-to-noise ratio
* sample preparation
* lipid-droplet morphology

For best performance, new images should be acquired under conditions similar to those represented in the training dataset.

⸻

Citation

If you use this repository, trained model, or analysis workflow in academic research, please cite this repository and the associated publication when available.

Citation metadata will be provided through:

CITATION.cff

A formal publication citation will be added following publication of the associated research.

⸻

License

The software in this repository is released under the MIT License.

See the LICENSE file for details.

Licensing terms for released microscopy datasets or other research data may be specified separately.

⸻

Acknowledgments

This project uses the StarDist framework for object detection and instance segmentation.

Please also cite the original StarDist publications when using StarDist in academic research.

⸻

Contact

For questions, issues, or suggestions, please open an issue in this GitHub repository.
