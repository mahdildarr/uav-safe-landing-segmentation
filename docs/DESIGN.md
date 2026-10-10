# Design

How the experiments are designed: the data, the networks and losses, the landing-zone search, and how each part is evaluated.

## 1. Problem

An unmanned aerial vehicle that has to land outside a prepared site (after a link loss, low battery or a delivery) must decide **where** it can touch down. From a downward-looking camera this is a dense prediction problem: every pixel must be assigned to a surface or an object, and the decision depends on small, rare classes (people, vehicles, poles) at least as much as on large ones (grass, pavement).

The project has two parts:

1. **Semantic segmentation** of high-resolution nadir images with a U-Net written from scratch and pretrained-encoder baselines.
2. **Safe landing-zone search**: explicit, explainable rules on top of the segmentation output, evaluated against the ground truth with landing-level metrics: not only pixel metrics.

## 2. Data

**Semantic Drone Dataset** (Graz University of Technology): 400 publicly available training images of 6000 × 4000 px, captured at nadir from 5–30 m above more than 20 houses, with pixel-accurate labels for 20 object classes plus auxiliary labels (bald tree, AR marker, conflicting). Licence: non-commercial use, no redistribution: the data is not included in the repositories.

**Landing taxonomy.** For a landing decision many source classes are equivalent (window, wall, roof, door → *structure*), while others need different safety margins. Source classes are mapped **by name** to eight decision-relevant classes:

| Landing class | Source classes | Role in the landing search |
|---|---|---|
| paved | paved area | safe surface |
| soft-ground | grass, dirt, gravel | safe surface |
| vegetation | vegetation, tree, bald tree | static hazard |
| water | water, pool | static hazard |
| structure | roof, wall, window, door, fence, fence pole | static hazard |
| obstacle | rocks, obstacle, AR marker | static hazard |
| person | person, dog | **dynamic hazard** (large clearance) |
| vehicle | car, bicycle | **dynamic hazard** (large clearance) |

Unlabelled and conflicting pixels are ignored in the loss and the metrics. The full 20-class setting is also supported.

**Resolution.** Images are resized once to 1536 × 1024 (masks with nearest-neighbour), which keeps a person at roughly 10–30 px. Training uses random 512 × 512 crops; evaluation uses the full 1536 × 1024 image.

**Augmentation.** Random scale (altitude change), flips and 90° rotations (a nadir view has no canonical "up"), colour jitter (lighting, season), mild noise and motion blur.

## 3. Splits: measuring near-duplicate leakage

Consecutive frames of a flight overlap, so a random image-level split can put almost the same view in training and evaluation.

* The **test set is fixed** for every experiment: contiguous blocks of 20 images (sorted by id), so neighbouring frames never straddle the test boundary.
* Only the train/validation division of the remaining images changes: **blocked** (validation made of whole blocks) or **random** (validation images drawn independently, most of them with overlapping neighbours in training).

Because the test set is identical in both modes, the difference between validation and test score directly measures how much a random split overstates performance, and whether it misleads model selection.

## 4. Networks

| Model | Encoder | Parameters | Purpose |
|---|---|---|---|
| **U-Net (from scratch)** | 4 down-sampling stages, 32 → 512 channels, BN, bilinear up-sampling, skip connections | ≈ 7.8 M | own implementation, no pretraining |
| U-Net + ResNet-34 | ImageNet-pretrained | ≈ 24 M | transfer-learning baseline |
| DeepLabV3+ + ResNet-50 | ImageNet-pretrained, atrous spatial pyramid pooling | ≈ 27 M | multi-scale context baseline |

Skip connections carry the fine spatial detail that thin objects (people, fence poles, bicycles) need; this is the main reason for the U-Net design in aerial scenes.

## 5. Class imbalance: losses

Ground classes cover most pixels; people and vehicles cover well under 1 %. Compared on the same model and split:

* cross-entropy (baseline), with and without median-frequency class weights;
* **Dice** loss (optimises overlap per class, insensitive to class size);
* **focal** loss (down-weights easy pixels);
* combinations CE + Dice and focal + Dice.

**Metrics:** per-class IoU, mean IoU over classes present, pixel accuracy. Training uses AdamW with a polynomial learning-rate schedule and mixed precision; the checkpoint with the best validation mIoU is kept; each configuration is trained with three seeds.

## 6. Landing-zone search

Given the class probabilities, a zone is a circle of diameter *footprint* that satisfies:

1. **Surface**: the whole footprint lies on a safe surface predicted with confidence ≥ 0.6 (uncertain pixels count as unsafe);
2. **Static clearance**: a margin from structures, vegetation, water and obstacles;
3. **Dynamic clearance**: a larger margin from people and vehicles.

Distances come from Euclidean distance transforms, so the search runs in milliseconds on a CPU. Feasible positions are ranked by clearance, mean confidence inside the footprint and distance from the image centre, and thinned with non-maximum suppression (top 3). Sizes are configured in metres and converted with the ground sampling distance (altitude, sensor width, focal length), or set directly in pixels.

**Landing metrics**: the same rules are applied to the ground-truth mask (with confidence 1), which defines where a zone is *admissible*. Proposed zones are then checked against it:

| Metric | Meaning |
|---|---|
| zone precision | share of proposed zones whose centre is admissible in the ground truth |
| hazard violation rate | share of proposed zones with a non-safe ground-truth pixel inside the footprint: the safety-critical number |
| availability recall | among images where the ground truth admits a zone, share where a safe zone is proposed |
| false-alarm rate | among images where no zone is admissible, share where one is proposed anyway |

Using one definition for proposal and verification keeps the metrics consistent: a zone that the rules would reject on the true labels is never counted as safe.

## 7. Deployment

The trained network is exported to ONNX and quantised to INT8 (static quantisation, calibrated on training images). For PyTorch FP32, ONNX Runtime FP32 and ONNX Runtime INT8 the project reports latency, frames per second, model size and **test mIoU at the export resolution**, so the speed gain is always shown together with its accuracy cost.

## 8. Limitations

* 400 images from one region and one season; generalisation to other terrain is not established.
* A single image carries no height information: a flat roof or a slope can look like safe ground. Depth or stereo input would be the next step.
* Pixel sizes depend on altitude, which varies within the dataset; metric sizes require the flight altitude.

## References

1. O. Ronneberger, P. Fischer, T. Brox, "U-Net: Convolutional Networks for Biomedical Image Segmentation," *MICCAI*, 2015.
2. L.-C. Chen et al., "Encoder-Decoder with Atrous Separable Convolution for Semantic Image Segmentation (DeepLabV3+)," *ECCV*, 2018.
3. T.-Y. Lin et al., "Focal Loss for Dense Object Detection," *ICCV*, 2017.
4. F. Milletari, N. Navab, S.-A. Ahmadi, "V-Net: Fully Convolutional Neural Networks for Volumetric Medical Image Segmentation," *3DV*, 2016 (Dice loss).
5. D. Eigen, R. Fergus, "Predicting Depth, Surface Normals and Semantic Labels with a Common Multi-Scale Convolutional Architecture," *ICCV*, 2015 (median-frequency balancing).
6. Semantic Drone Dataset, Institute of Computer Graphics and Vision, Graz University of Technology: https://ivc.tugraz.at/research-project/semantic-drone-dataset/
