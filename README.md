# Safe Landing-Zone Detection for UAVs with Semantic Segmentation

Where can a drone land safely when it has to come down outside a prepared site? This project answers the question from a single downward-looking image: a **CNN segments every pixel** of the scene, and an explicit, explainable **landing-zone search** finds touchdown spots that keep clear of buildings, trees and water — and, with a larger margin, of people and vehicles. The landing decision itself is evaluated against the ground truth, not only the pixel accuracy.

<p align="center"><img src="docs/assets/pipeline.svg" width="100%" alt="Pipeline"></p>

> **Status:** Work in progress — the pipeline is implemented and tested; results will be added once the experiments on the Semantic Drone Dataset are completed.
> This repository presents the project and its design. **The source code is kept in a private repository** and can be shared on request.

## Why this project

Emergency landing after a link loss or low battery, package delivery and autonomous inspection all need the same capability: understanding the ground below at pixel level and turning that understanding into a safe decision. The hard part is not the large classes (grass, pavement) but the small, rare ones — a person or a bicycle covers well under 1 % of the image — and the fact that a segmentation score does not directly say whether a chosen landing spot is safe.

## What is inside

| | |
|---|---|
| **Networks** | U-Net written from scratch (≈ 7.8 M params) · U-Net + ResNet-34 · DeepLabV3+ + ResNet-50 (ImageNet-pretrained) |
| **Landing taxonomy** | dataset classes → 8 decision-relevant classes: safe surfaces, static hazards, dynamic hazards |
| **Class imbalance** | cross-entropy · Dice · focal loss · median-frequency class weights |
| **Landing search** | confidence-aware safe surface, static and dynamic clearances (distance transforms), ranking + NMS, metric sizes from altitude and camera |
| **Landing metrics** | zone precision · hazard violation rate · availability recall · false-alarm rate |
| **Rigour** | fixed blocked test set, random vs blocked validation to measure near-duplicate leakage, three seeds, unit and end-to-end tests |
| **Deployment** | ONNX export, INT8 quantisation, latency / FPS reported together with mIoU |

## Design

The full design — data and taxonomy, splits, architectures, losses, the landing-zone rules and metrics, deployment and limitations — is in **[docs/DESIGN.md](docs/DESIGN.md)**.

## Results

Result tables will be published here when the experiments are completed.

| Model | Params | Test mIoU | Person IoU | Zone precision | Hazard violation | INT8 FPS (CPU) |
|---|---|---|---|---|---|---|
| U-Net (scratch) | 7.8 M | _TBD_ | _TBD_ | _TBD_ | _TBD_ | _TBD_ |
| U-Net + ResNet-34 | 24.4 M | _TBD_ | _TBD_ | _TBD_ | _TBD_ | _TBD_ |
| DeepLabV3+ + ResNet-50 | 26.7 M | _TBD_ | _TBD_ | _TBD_ | _TBD_ | _TBD_ |

## Tech stack

Python · PyTorch · segmentation-models-pytorch · Albumentations · SciPy · ONNX / ONNX Runtime · Google Colab (T4 GPU)

## Author

**Şule Özdamar** — Computer Engineering, İstanbul Arel University · [github.com/mahdildarr](https://github.com/mahdildarr)

Data: [Semantic Drone Dataset](https://ivc.tugraz.at/research-project/semantic-drone-dataset/), Institute of Computer Graphics and Vision, Graz University of Technology. The dataset is not redistributed here.
