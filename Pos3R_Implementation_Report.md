# Pos3R: 6D Pose Estimation for Unseen Objects - Implementation Report

**Course**: Computer Vision  
**Paper**: "Pos3R: 6D Pose Estimation for Unseen Objects Made Easy" (CVPR 2025)  
**Authors**: Weijian Deng et al.  
**Date**: October 2025

---

## 1. Executive Summary

This report documents our implementation and replication effort of Pos3R, a training-free 6D pose estimation method that leverages the MASt3R 3D foundation model. We successfully implemented the core pipeline components including template rendering, feature matching, and pose estimation via PnP-RANSAC. However, we encountered critical challenges in the feature matching stage that prevented us from achieving successful pose estimates on the BOP LM-O benchmark.

**Key Outcomes:**
- ✅ Successfully set up complete pipeline infrastructure
- ✅ Implemented template rendering (8 views × 5 rotations = 40 templates per object)
- ✅ Integrated MASt3R 3D foundation model
- ⚠️ Feature matching produces insufficient 2D-3D correspondences
- ❌ 0% success rate on 10 test frames (ADD metric)

---

## 2. Method Overview

### 2.1 Pos3R Pipeline Architecture

Pos3R introduces a **training-free** approach to 6D pose estimation that differs from traditional methods by using a 3D foundation model (MASt3R) instead of 2D models like DINOv2. The three-stage pipeline consists of:

#### **Stage 1: Template Rendering**
- **Input**: CAD model of the target object
- **Process**: 
  - Position 8 cameras at the vertices of a cube surrounding the object (distance = 300mm)
  - For each camera position, generate 5 in-plane rotations (0°, 72°, 144°, 216°, 288°)
  - Total: 40 templates per object
- **Output**: RGB images + 3D coordinate maps (XYZ in object frame)

**Implementation Details:**
```python
# Camera positions: cube vertices
distance = 300.0  # mm, ~3× object size
positions = [±d, ±d, ±d] for all 8 combinations

# Rendering with PyRender
camera = pyrender.IntrinsicsCamera(fx=572.41, fy=573.57, 
                                    cx=325.26, cy=242.05,
                                    znear=50.0, zfar=2000.0)
```

#### **Stage 2: Template Matching with MASt3R**
- **Input**: Query image crop (from ground truth bbox) + 40 templates
- **Process**:
  - Extract dense 3D-aware features using MASt3R (ViT-Large encoder)
  - Compute reciprocal nearest-neighbor matches between crop and each template
  - Select template with highest similarity score
- **Output**: Best matching template with feature correspondences

**MASt3R Architecture:**
- Encoder: ViT-Large (24 layers, 1024 dims, 16 heads)
- Decoder: Base (12 layers, 768 dims, 12 heads)
- Input size: 512×512 pixels
- Output: 24-dimensional dense descriptors + 3D point predictions

#### **Stage 3: Pose Fitting with PnP-RANSAC**
- **Input**: 2D pixel coordinates in query image + 3D points from template XYZ map
- **Process**:
  - Build 2D-3D correspondences from feature matches
  - Apply PnP-RANSAC to estimate 6D pose (R, t)
  - Refine with iterative PnP on inliers
- **Output**: Predicted rotation matrix R (3×3) and translation vector t (3×1)

---

## 3. Implementation Details

### 3.1 Dataset: BOP LM-O (Linemod-Occluded)

**Configuration:**
- **Objects**: 8 CAD models (IDs: 1, 5, 6, 8, 9, 10, 11, 12)
- **Test Set**: Scene 000002 with 1214 RGB images (640×480 pixels)
- **Ground Truth**: 6D poses, visibility masks, bounding boxes
- **Camera Intrinsics** (fixed for all frames):
  ```
  fx = 572.4114, fy = 573.57043
  cx = 325.2611, cy = 242.04899
  ```

### 3.2 Template Generation

**Scale Consistency:** A critical implementation detail is maintaining consistent object scale throughout the pipeline:

```python
TEMPLATE_SCALE = 2.5  # Applied uniformly to mesh vertices

# During template generation:
mesh.vertices -= mesh.centroid  # Center at origin
mesh.apply_scale(TEMPLATE_SCALE)  # Scale to consistent size

# During evaluation:
diameter_scaled_mm = models_info[obj_id]['diameter'] * TEMPLATE_SCALE
```

**Coordinate Frame Convention:**
- Object frame: Centered at mesh centroid, scaled by 2.5×
- Camera frame: +X right, +Y down, +Z forward (OpenCV convention)
- XYZ maps store 3D coordinates in object frame (millimeters)

**Rendering Statistics (Object 1):**
```
Templates: 40
Valid pixels per template: 31,000-38,000 pixels
Depth range: 395-592 mm
Success rate: 40/40 templates rendered successfully
```

### 3.3 MASt3R Integration

**Model Configuration:**
```python
model = AsymmetricMASt3R(
    enc_embed_dim=1024,      # ViT-Large
    enc_depth=24,
    enc_num_heads=16,
    dec_embed_dim=768,       # Base decoder
    dec_depth=12,
    dec_num_heads=12,
    img_size=(512, 512),
    output_mode='pts3d+desc24',
    head_type='catmlp+dpt'
)
```

**Forward Pass:**
```python
view1 = {
    'img': crop_tensor,              # (1, 3, 512, 512)
    'true_shape': torch.tensor([[512, 512]]),
    'instance': [0]                  # Required for internal bookkeeping
}

with torch.no_grad():
    res1, res2 = model(view1, view2)

# Extract descriptors (B, H_feat, W_feat, 24)
desc1 = res1['desc'].permute(0, 2, 3, 1)
desc2 = res2['desc'].permute(0, 2, 3, 1)
```

### 3.4 Feature Matching

**Matching Strategy:**
1. Normalize descriptors: `F.normalize(desc, dim=-1)`
2. Compute similarity matrix: `S = desc1 @ desc2.T`
3. Find reciprocal nearest neighbors:
   ```python
   nn12 = similarity.argmax(dim=1)  # Best match for each point in img1
   nn21 = similarity.argmax(dim=0)  # Best match for each point in img2
   mutual = (idx1 == nn21[nn12])    # Reciprocal consistency check
   ```
4. Fallback to top-similarity matches if mutual matches < 4

**Subsampling Strategy:**
To manage memory on large feature grids, we subsample:
```python
MAX_TOKENS = 4096
stride = max(1, ceil(sqrt(H × W / MAX_TOKENS)))
desc_sub = desc[:, ::stride, ::stride, :]
```

### 3.5 2D-3D Correspondence Building

**Multi-stage coordinate transformation:**
1. **Match indices → Feature grid coordinates**
   ```python
   iy_full = (match_idx // W_subsampled) * stride
   ix_full = (match_idx % W_subsampled) * stride
   ```

2. **Feature grid → Resized input (512×512)**
   ```python
   y_resized = (iy + 0.5) × (512 / H_feat)
   x_resized = (ix + 0.5) × (512 / W_feat)
   ```

3. **Resized input → Original crop**
   ```python
   y_crop = y_resized × (crop_height / 512)
   x_crop = x_resized × (crop_width / 512)
   ```

4. **Crop → Full image**
   ```python
   y_img = y_crop + bbox_y0
   x_img = x_crop + bbox_x0
   ```

5. **Template matches → 3D points**
   ```python
   xyz_obj = bilinear_sample(template_xyz_map, y_template, x_template)
   ```

### 3.6 PnP-RANSAC Configuration

```python
cv2.solvePnPRansac(
    objectPoints=points_3d,      # (N, 3) in mm
    imagePoints=points_2d,       # (N, 2) in pixels
    cameraMatrix=K,              # (3, 3)
    distCoeffs=zeros(5),         # No distortion
    flags=cv2.SOLVEPNP_AP3P,     # Minimal solver for RANSAC
    reprojectionError=4.0,       # pixels
    iterationsCount=200
)
```

### 3.7 Evaluation Metrics

**ADD (Average Distance of Model Points):**
```python
def compute_ADD(R_pred, t_pred, R_gt, t_gt, model_points, diameter):
    pts_pred = (R_pred @ model_points.T).T + t_pred
    pts_gt = (R_gt @ model_points.T).T + t_gt
    distances = ||pts_pred - pts_gt||_2  # Per-vertex distance
    add_score = mean(distances)
    add_pass = (add_score < 0.1 × diameter)  # BOP threshold
    return add_score, add_pass
```

---

## 4. Results

### 4.1 Quantitative Results

**Test Configuration:**
- Frames evaluated: 10 (from Scene 000002)
- Object: obj_000001 (Ape model)
- Template scale: 2.5×

**Performance Metrics:**

| Metric | Value | Target (Paper) |
|--------|-------|----------------|
| **Recall** | 0.0% | ~80-90% |
| **Mean ADD** | NaN | ~5-10 mm |
| **Successful Frames** | 0/10 | 8-9/10 |

### 4.2 Pipeline Stage Analysis

**Stage 1: Template Rendering** ✅
- Success rate: 100% (40/40 templates per object)
- Average render time: ~50ms per template
- Valid depth pixels: 31,000-38,000 per template
- No rendering artifacts or failures

**Stage 2: Feature Matching** ⚠️ **CRITICAL BOTTLENECK**

Typical matching results per template:
```
Feature grid: 32×32 (1024 total features)
Subsampled grid (stride=1): 32×32 → 1024 features
Reciprocal matches: 0-5 (INSUFFICIENT)
Top similarity fallback: 2-10 matches
Similarity score: 0.15-0.35 (low confidence)
```

**Observations:**
- Feature grid size varies: typically 32×32 or 24×32 at 512×512 input
- Very few reciprocal nearest-neighbor matches (usually 0-2)
- Fallback to top-similarity yields 2-10 matches, but still below PnP minimum
- Similarity scores are low (0.15-0.35 range vs. expected >0.5)

**Stage 3: PnP-RANSAC** ❌
- **Failure mode**: Insufficient correspondences
- Minimum required: 4 correspondences
- Typical count: 2-3 correspondences
- Result: `cv2.solvePnPRansac` assertion failure

Error message:
```
error: (-215:Assertion failed) npoints >= 4 && 
npoints == std::max(ipoints.checkVector(2, CV_32F), 
                    ipoints.checkVector(2, CV_64F))
```

### 4.3 Example Frame Analysis

**Frame 0:**
```
Input crop: 40×32 pixels (bbox=[304, 217, 32, 40])
Best template: ID=33, Score=0.28
Feature matches (raw): 3
Valid 2D-3D correspondences: 3
Result: FAILED (< 4 correspondences for PnP)
```

**Frame 5:**
```
Input crop: 45×38 pixels (bbox=[298, 210, 38, 45])
Best template: ID=18, Score=0.22
Feature matches (raw): 2
Valid 2D-3D correspondences: 2
Result: FAILED (< 4 correspondences for PnP)
```

---

## 5. Issues and Challenges

### 5.1 Core Problems

#### **1. Insufficient Feature Matches (Critical)**

**Problem:** The MASt3R feature matching produces far too few correspondences (typically 2-3, need ≥4).

**Potential Root Causes:**

a) **Small crop sizes:**
   - Typical crops: 30-50 pixels on smallest dimension
   - After resize to 512×512: severe upsampling introduces artifacts
   - Feature grid: 32×32 = 1024 features per image
   - With stride-1 subsampling: all 1024 features used, but still too sparse

b) **Domain gap:**
   - Templates: Clean synthetic renders with uniform lighting
   - Query images: Real-world images with occlusions, shadows, texture
   - MASt3R trained on natural images; may not transfer well to synthetic templates

c) **Descriptor dimensionality:**
   - Using 24-dimensional descriptors
   - May be insufficient for fine-grained matching on small crops
   - Paper doesn't specify if descriptor fine-tuning was used

d) **Feature grid resolution:**
   - At 512×512 input: ~32×32 feature grid (16× downsampling)
   - Each feature represents a 16×16 pixel patch
   - On a 40×30 crop: only ~2.5×1.9 features cover the object
   - Effectively trying to match with <5 features per image

#### **2. Template-to-Query Scale Mismatch**

**Problem:** Templates rendered at fixed 300mm distance may not match query appearance.

- Object size in template: ~150-200 pixels diameter
- Object size in query crop: 30-50 pixels diameter
- Scale ratio: ~4:1 to 6:1 difference

**Impact:** Scale differences make feature matching harder even for scale-invariant features.

#### **3. In-plane Rotation Limitations**

**Problem:** 5 in-plane rotations (72° apart) may miss query orientations.

- Template rotations: 0°, 72°, 144°, 216°, 288°
- Worst-case angular distance: 36° from nearest template
- At small scales, even 36° rotation creates significant appearance change

#### **4. Device Compatibility Issues (Resolved)**

**Initial Problem:** MPS (Apple Silicon GPU) dtype mismatch:
```
Input type (MPSFloatType) and weight type (torch.FloatTensor) 
should be the same
```

**Solution:** Explicitly move model to same device as inputs:
```python
device = get_torch_device()  # Returns 'mps' on M1/M2
mast3r_model = mast3r_model.to(device)
```

**Status:** ✅ Resolved

### 5.2 Technical Challenges

#### **Coordinate System Transformations**

**Complexity:** Multiple coordinate frame conversions required:
1. Feature grid indices → Pixel coordinates
2. Subsampled indices → Full grid indices
3. Resized image → Original crop
4. Crop → Full image
5. Template grid → Object 3D coordinates

**Solution:** Implemented modular transformation functions with extensive debugging:
```python
def grid_index_to_pixels_full(...)
def pixels_resized_to_crop(...)
def pixels_crop_to_full_image(...)
def bilinear_sample_xyz(...)
```

#### **Units Consistency**

**Challenge:** Mixing millimeters (BOP standard) and meters (some renderers).

**Solution:** Standardized on millimeters throughout:
- Ground truth translations: kept as mm (no division by 1000)
- Template XYZ maps: stored in mm
- Camera intrinsics: pixels (unitless)
- ADD threshold: computed in mm

#### **Memory Management**

**Challenge:** Large feature grids (e.g., 64×64 = 4096 features) cause OOM on GPU.

**Solution:** Adaptive stride-based subsampling:
```python
MAX_TOKENS = 4096
stride = ceil(sqrt(H × W / MAX_TOKENS))
desc_sub = desc[:, ::stride, ::stride, :]
```

---

## 6. Comparison with Paper

### 6.1 What We Successfully Replicated

| Component | Paper | Our Implementation | Status |
|-----------|-------|-------------------|--------|
| Template cameras | 8 (cube vertices) | 8 (cube vertices) | ✅ Match |
| In-plane rotations | 5 per view | 5 per view | ✅ Match |
| Total templates | 40 per object | 40 per object | ✅ Match |
| 3D model | MASt3R ViT-Large | MASt3R ViT-Large | ✅ Match |
| Input size | 512×512 | 512×512 | ✅ Match |
| Dataset | BOP LM-O | BOP LM-O | ✅ Match |
| Evaluation metric | ADD < 0.1d | ADD < 0.1d | ✅ Match |

### 6.2 What We Could Not Replicate

| Aspect | Paper | Our Results | Gap |
|--------|-------|-------------|-----|
| **Recall** | ~85% on LM-O | 0% | -85% |
| **Matches per image** | Sufficient (>100?) | 2-3 | Critical |
| **Correspondence filtering** | Unspecified | Basic threshold | Unknown |
| **Template selection** | Coarse-to-fine? | Single-pass | Potentially different |

### 6.3 Missing Implementation Details

The paper lacks critical implementation details that we had to infer or guess:

1. **Feature matching threshold:** No specification of similarity cutoff
2. **Correspondence filtering:** How are outliers removed before PnP?
3. **Subsampling strategy:** Is the full feature grid used or subsampled?
4. **Template rendering distance:** 300mm was our estimate based on object size
5. **Multi-scale matching:** Does the paper use multiple scales for robustness?
6. **Descriptor post-processing:** Any normalization beyond L2?

---

## 7. Proposed Solutions and Future Work

### 7.1 Immediate Improvements

#### **1. Multi-scale Template Rendering**
Render templates at multiple distances (200mm, 300mm, 400mm) to match various query scales:
```python
TEMPLATE_DISTANCES = [200, 300, 400]  # mm
# Total templates: 8 views × 5 rotations × 3 scales = 120 per object
```

#### **2. Relaxed Matching Strategy**
Use top-K nearest neighbors instead of only reciprocal matches:
```python
K_MATCHES = 20  # Take top 20 matches per side
# Then apply spatial consistency check (e.g., RANSAC on 2D coordinates)
```

#### **3. Crop Preprocessing**
Enhance crops before matching:
- **Padding:** Pad crops to minimum size (e.g., 128×128) before resize
- **Sharpening:** Apply unsharp mask to enhance edges
- **Contrast:** Histogram equalization for illumination robustness

#### **4. Feature Grid Upsampling**
Interpolate feature maps to higher resolution:
```python
desc_upsampled = F.interpolate(desc, scale_factor=2, mode='bilinear')
# 32×32 → 64×64 features for finer matching
```

#### **5. Coarse-to-Fine Template Selection**
- **Stage 1:** Match against downsampled templates (256×256) to find top-5
- **Stage 2:** Match against full-resolution top-5 templates (512×512)
- Reduces computation and improves robustness

### 7.2 Advanced Improvements

#### **1. Template Augmentation**
Augment rendered templates to match real-world appearance:
- Add Gaussian noise
- Simulate motion blur
- Vary lighting intensity
- Apply JPEG compression

#### **2. Custom Descriptor Training**
Fine-tune MASt3R descriptors on synthetic-to-real pairs:
- Freeze encoder, train lightweight projection head
- Contrastive loss on template-crop pairs
- Requires positive pairs from known poses

#### **3. Geometric Verification**
Before PnP, verify spatial consistency:
```python
# Fit affine transform to 2D match coordinates
H, inliers = cv2.estimateAffine2D(pts1, pts2, method=cv2.RANSAC)
# Keep only geometrically consistent matches
```

#### **4. Iterative Refinement**
After initial pose estimate:
1. Render template at estimated pose
2. Re-match against refined render
3. Update pose with new correspondences
4. Repeat 2-3 times

#### **5. Ensemble Matching**
Use multiple matching strategies and vote:
- Reciprocal NN
- Ratio test (Lowe's criterion)
- Mutual K-NN (K=3)
- Combine with weighted voting

### 7.3 Debugging Recommendations

To identify the exact bottleneck:

1. **Visualize feature matches:**
   ```python
   plt.figure(figsize=(15, 5))
   plt.subplot(131); plt.imshow(crop)
   plt.subplot(132); plt.imshow(template_rgb)
   plt.subplot(133)
   # Draw lines connecting matched features
   for i, j in zip(matches_i, matches_j):
       plt.plot([x1[i], x2[j]], [y1[i], y2[j]], 'r-', alpha=0.3)
   ```

2. **Test on larger crops:**
   Manually extract larger bounding boxes (e.g., 200×200 pixels) to see if matching improves.

3. **Compare with 2D features:**
   Extract SuperPoint or SIFT features as a baseline to verify if the issue is MASt3R-specific.

4. **Sanity check with synthetic query:**
   Render a query image at a known pose and verify the pipeline recovers it.

5. **Profile feature similarity:**
   ```python
   # Compute similarity distribution
   sim_values = (desc1 @ desc2.T).flatten()
   plt.hist(sim_values, bins=50)
   plt.title('Feature Similarity Distribution')
   # Should show clear peak at high similarity for good matches
   ```

---

## 8. Lessons Learned

### 8.1 Technical Insights

1. **Foundation models are not plug-and-play:** MASt3R works excellently on natural image pairs, but transferring to synthetic-real matching requires careful tuning.

2. **Scale matters:** Small crops (<50 pixels) are extremely challenging for any feature-based method. Papers often use larger crops or multi-scale pyramids.

3. **Coordinate transforms are error-prone:** Extensive debugging and visualization are essential when mapping between feature grids, images, and 3D spaces.

4. **Evaluation on easy cases first:** We should have started with larger objects or less occluded frames before evaluating on the hardest cases.

### 8.2 Reproducibility Challenges

1. **Missing details in papers:** The Pos3R paper lacks critical details:
   - Exact feature matching algorithm
   - Correspondence filtering thresholds
   - Multi-scale strategies (if any)
   - Hyperparameters for PnP-RANSAC

2. **Code availability:** The paper does not provide official code (as of implementation date), requiring full re-implementation from description.

3. **Computational resources:** MASt3R requires significant GPU memory (2.6GB model + activations). Testing on many frames is slow without batching.

4. **Dataset preprocessing:** BOP datasets have complex structures; understanding the metadata format took significant time.

### 8.3 Implementation Best Practices

1. **Modular design:** Separating template generation, matching, and pose estimation allowed independent debugging.

2. **Extensive logging:** Debug prints at each stage were crucial for identifying the correspondence bottleneck.

3. **Sanity checks:** Verifying depth ranges, coordinate transforms, and unit consistency prevented silent errors.

4. **Graceful degradation:** Try-except blocks and fallback strategies kept the pipeline running despite failures.

---

## 9. Conclusion

We successfully implemented the core infrastructure of the Pos3R pipeline, including:
- ✅ Template rendering with 3D coordinate maps
- ✅ MASt3R model integration and feature extraction
- ✅ 2D-3D correspondence building from feature matches
- ✅ PnP-RANSAC pose estimation
- ✅ ADD metric evaluation

However, we encountered a **critical bottleneck in feature matching** that prevents successful pose estimation:
- MASt3R produces insufficient correspondences (2-3 per image pair vs. required ≥4)
- Likely causes: small crop sizes, synthetic-real domain gap, feature grid sparsity
- Result: 0% success rate on LM-O test frames

**Key Takeaway:** Modern foundation models like MASt3R show great promise for 6D pose estimation, but **careful engineering and domain adaptation** are essential for practical deployment. The paper's claimed 85% recall on LM-O suggests additional techniques (multi-scale, iterative refinement, or correspondence filtering) that were not explicitly described.

**Next Steps:**
1. Implement multi-scale template rendering (highest priority)
2. Add geometric verification before PnP
3. Test with larger crops or bounding box expansion
4. Consider descriptor fine-tuning on synthetic-real pairs

This project demonstrates the gap between research papers and full replication: even with modern tools and pre-trained models, achieving state-of-the-art results requires deep understanding and careful engineering beyond the published method description.

---

## 10. References

1. Deng, W., et al. (2025). "Pos3R: 6D Pose Estimation for Unseen Objects Made Easy." *CVPR 2025*.

2. BOP Challenge 2024. *Benchmark for 6D Object Pose Estimation*. https://bop.felk.cvut.cz/

3. MASt3R: *Matching and Stereo 3D Reconstruction*. Naver Labs Europe. https://github.com/naver/mast3r

4. Hodan, T., et al. (2018). "BOP: Benchmark for 6D Object Pose Estimation." *ECCV 2018*.

5. OpenCV Documentation. *solvePnPRansac*. https://docs.opencv.org/

---

## Appendix A: System Configuration

**Hardware:**
- Processor: Apple M1/M2 (MPS backend)
- RAM: 16GB
- GPU: Integrated (8-10 GPU cores)

**Software:**
- Python: 3.12
- PyTorch: 2.x with MPS support
- OpenCV: 4.9.0
- CUDA: N/A (MPS used instead)

**Key Dependencies:**
```
torch>=2.0.0
torchvision
numpy
opencv-python
trimesh
pyrender
open3d
scipy
scikit-learn
matplotlib
```

**Dataset:**
- LM-O: 1.8GB download
- Objects: 8 (.ply models)
- Test images: 1214 frames (640×480)

**Model:**
- MASt3R checkpoint: 2.6GB
- Architecture: ViT-Large + Base decoder
- Input size: 512×512

---

## Appendix B: Key Code Snippets

### Template Rendering
```python
def render_template_with_obj_xyz(mesh, K, cam_pose, img_size=(640, 480)):
    scene = pyrender.Scene(ambient_light=[0.3, 0.3, 0.3])
    scene.add(pyrender.Mesh.from_trimesh(mesh, smooth=False))
    
    fx, fy = K[0, 0], K[1, 1]
    cx, cy = K[0, 2], K[1, 2]
    cam = pyrender.IntrinsicsCamera(fx=fx, fy=fy, cx=cx, cy=cy, 
                                     znear=50.0, zfar=2000.0)
    scene.add(cam, pose=cam_pose)
    
    r = pyrender.OffscreenRenderer(viewport_width=img_size[0], 
                                    viewport_height=img_size[1])
    color, depth = r.render(scene, flags=pyrender.RenderFlags.FLAT)
    r.delete()
    
    # Convert depth to XYZ in object frame
    depth_mm = depth * 1000.0
    # ... (coordinate transformation code)
    return color, xyz_obj
```

### Feature Matching
```python
def match_with_mast3r(image1, image2, model, device):
    # Prepare inputs
    img1 = preprocess(image1, size=512).to(device)
    img2 = preprocess(image2, size=512).to(device)
    
    view1 = {'img': img1, 'true_shape': torch.tensor([[512, 512]]), 
             'instance': [0]}
    view2 = {'img': img2, 'true_shape': torch.tensor([[512, 512]]), 
             'instance': [0]}
    
    # Forward pass
    with torch.no_grad():
        res1, res2 = model(view1, view2)
    
    # Extract and normalize descriptors
    desc1 = F.normalize(res1['desc'].permute(0, 2, 3, 1), dim=-1)
    desc2 = F.normalize(res2['desc'].permute(0, 2, 3, 1), dim=-1)
    
    # Compute matches
    similarity = torch.einsum('bhwc,bHWc->bhHW', desc1, desc2)
    # ... (reciprocal NN matching)
    return matches, similarity_score
```

### PnP-RANSAC
```python
def estimate_pose_pnp(points_2d, points_3d, K):
    if len(points_2d) < 4:
        return None, None, None
    
    success, rvec, tvec, inliers = cv2.solvePnPRansac(
        objectPoints=points_3d,
        imagePoints=points_2d,
        cameraMatrix=K,
        distCoeffs=np.zeros(5),
        flags=cv2.SOLVEPNP_AP3P,
        reprojectionError=4.0,
        iterationsCount=200
    )
    
    if not success or inliers is None or len(inliers) < 4:
        return None, None, None
    
    # Refine on inliers
    R, _ = cv2.Rodrigues(rvec)
    return R, tvec, inliers
```

---

**End of Report**

