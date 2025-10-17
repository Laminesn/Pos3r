# Pos3R: 6D Pose Estimation Replication

A complete implementation replicating the Pos3R paper: **"Pos3R: 6D Pose Estimation for Unseen Objects Made Easy"** (CVPR 2025) by Weijian Deng et al.

## 📋 Overview

This project implements a 3D-based method for 6D object pose estimation that works with unseen objects without requiring training on specific objects. The approach leverages MASt3R, a 3D foundation model, for robust feature extraction and matching.

## 🎯 Key Features

- **Template-based Pose Estimation**: 40 templates per object (8 cameras × 5 rotations)
- **MASt3R Integration**: Uses a 3D foundation model for feature extraction
- **BOP Benchmark Evaluation**: Tested on LM-O (Linemod-Occluded) dataset
- **PnP-RANSAC Pose Fitting**: Robust pose estimation from 2D-3D correspondences
- **Complete Pipeline**: From rendering to evaluation

## 🏗️ Project Structure

```
.
├── Pos3R_Replication.ipynb    # Main implementation notebook
├── mast3r/                     # MASt3R model and dependencies
├── lmo/                        # LM-O dataset
│   ├── models/                 # 3D object models
│   └── test_all/               # Test images
├── templates/                  # Rendered templates for each object
├── results/                    # Evaluation results
└── docs/
    ├── 6D_Pose_Estimation.md
    ├── IMPLEMENTATION_READY.md
    └── PROJECT_STATUS.md
```

## 📦 Dataset

**LM-O (Linemod-Occluded)**
- 8 objects with significant occlusion
- 1,214 test frames
- Ground truth poses, masks, and camera intrinsics provided
- BOP format compatible

## 🔧 Setup

### Requirements
- Python 3.8+
- PyTorch
- OpenCV
- PyRender
- NumPy, SciPy

### Installation

```bash
# Clone the repository
git clone [your-repo-url]
cd [repo-name]

# Install dependencies (if requirements.txt is provided)
pip install -r mast3r/requirements.txt
```

### Dataset Setup
Download the LM-O dataset from the [BOP Benchmark](https://bop.felk.cvut.cz/datasets/) and place it in the `lmo/` directory.

### Model Checkpoint
The MASt3R checkpoint should be placed in `mast3r/checkpoints/`:
- `MASt3R_ViTLarge_BaseDecoder_512_catmlpdpt_metric.pth` (2.6GB)

## 🚀 Usage

Open and run the Jupyter notebook:

```bash
jupyter notebook Pos3R_Replication.ipynb
```

The notebook is organized into clear sections:
1. Setup & Dependencies
2. Dataset Preparation
3. Template Rendering
4. MASt3R Feature Extraction
5. Image Matching & Template Selection
6. Pose Fitting with PnP-RANSAC
7. Evaluation on BOP Benchmark
8. Visualization & Results

## 📊 Implementation Details

### Template Rendering
- 8 camera positions distributed on a hemisphere
- 5 in-plane rotations per camera position
- 40 templates total per object
- Includes RGB images and 3D point clouds

### Feature Extraction
- Uses MASt3R (3D foundation model)
- Extracts dense 3D-aware features
- Enables robust matching across viewpoints

### Pose Estimation
- Template matching via feature similarity
- 2D-3D correspondence establishment
- PnP-RANSAC for robust pose fitting
- Refinement using iterative optimization

## 📈 Results

The implementation follows the methodology from the original paper and evaluates on the BOP benchmark metrics:
- VSD (Visible Surface Discrepancy)
- MSSD (Maximum Symmetry-Aware Surface Distance)
- MSPD (Maximum Symmetry-Aware Projection Distance)

## 📚 References

```
@inproceedings{deng2025pos3r,
  title={Pos3R: 6D Pose Estimation for Unseen Objects Made Easy},
  author={Deng, Weijian and others},
  booktitle={CVPR},
  year={2025}
}
```

## 📄 License

This project is for educational and research purposes. Please refer to the original paper and MASt3R license for usage restrictions.

## 🙏 Acknowledgments

- Original Pos3R paper authors
- MASt3R team for the foundation model
- BOP Challenge organizers for the benchmark and datasets

---

**Note**: This is a replication project for educational purposes. For the official implementation, please refer to the original authors' repository.

