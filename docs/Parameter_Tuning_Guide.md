# MoS₂ Analysis Parameter Tuning & Troubleshooting Guide

## Table of Contents
1. [Quick Start Parameter Sets](#quick-start-parameter-sets)
2. [Parameter Deep Dive](#parameter-deep-dive)
3. [Systematic Tuning Workflow](#systematic-tuning-workflow)
4. [Common Issues & Solutions](#common-issues--solutions)
5. [Advanced Tuning Strategies](#advanced-tuning-strategies)
6. [Validation & Quality Assessment](#validation--quality-assessment)
7. [Performance Optimization](#performance-optimization)
8. [Troubleshooting Checklist](#troubleshooting-checklist)

---

## Quick Start Parameter Sets

### 🎯 Recommended Sets for Different Scenarios

#### **Ultra-Sensitive (Current Default)**
*For maximum detection - research/discovery mode*
```python
ULTRA_SENSITIVE = {
    'intensity_threshold': 140,
    'min_internal_area': 30,
    'min_area_ratio': 0.005,  # 0.5%
    'intensity_drops': [2, 5, 8, 12, 18, 25],
    'edge_methods': [(10,30), (15,45), (20,60), (25,75), (5,25), (8,35)]
}
# Expected: 80-100% detection, some false positives
```

#### **Balanced (High Quality Images)**
*For clean samples with good contrast*
```python
BALANCED = {
    'intensity_threshold': 145,
    'min_internal_area': 40,
    'min_area_ratio': 0.008,  # 0.8%
    'intensity_drops': [3, 6, 10, 15, 20],
    'edge_methods': [(15,45), (20,60), (25,75)]
}
# Expected: 60-80% detection, fewer false positives
```

#### **Conservative (Publication Ready)**
*For high-precision, manually validated results*
```python
CONSERVATIVE = {
    'intensity_threshold': 150,
    'min_internal_area': 60,
    'min_area_ratio': 0.015,  # 1.5%
    'intensity_drops': [5, 10, 15],
    'edge_methods': [(20,60), (25,75)]
}
# Expected: 40-60% detection, high precision
```

#### **Hyper-Sensitive (Noisy/Difficult Images)**
*For challenging samples or low-contrast images*
```python
HYPER_SENSITIVE = {
    'intensity_threshold': 135,
    'min_internal_area': 20,
    'min_area_ratio': 0.002,  # 0.2%
    'intensity_drops': [1, 2, 4, 6, 8, 10, 15, 20, 30],
    'edge_methods': [(5,25), (8,35), (10,30), (15,45), (20,60), (25,75)]
}
# Expected: ~100% detection, many false positives
```

---

## Parameter Deep Dive

### 🔍 **Stage 1 Parameters (Flake Detection)**

#### `intensity_threshold` (Critical)
**Default: 140** | **Range: 100-180** | **Impact: ⭐⭐⭐⭐⭐**

```python
# What it does: Separates flake material from background
binary = (gray < intensity_threshold).astype(np.uint8) * 255
```

**Tuning Guidelines:**
- **Lower (120-130)**: More inclusive, catches faint flakes
- **Higher (150-160)**: More selective, only obvious flakes
- **Sweet Spot**: Image mean - 20 to 30

**Diagnostic Code:**
```python
# Check your image's intensity distribution
import matplotlib.pyplot as plt
import numpy as np

# Load your image
gray = cv2.cvtColor(img, cv2.COLOR_RGB2GRAY)
mean_intensity = gray.mean()
std_intensity = gray.std()

print(f"Mean intensity: {mean_intensity:.1f}")
print(f"Std intensity: {std_intensity:.1f}")
print(f"Suggested threshold: {mean_intensity - 25:.0f} to {mean_intensity - 15:.0f}")

# Plot histogram
plt.hist(gray.flatten(), bins=50, alpha=0.7)
plt.axvline(140, color='r', label='Current threshold')
plt.axvline(mean_intensity - 20, color='g', label='Suggested')
plt.legend()
plt.show()
```

#### `min_flake_area` & `max_flake_area`
**Default: 200-15000 pixels** | **Impact: ⭐⭐⭐**

**Tuning Guidelines:**
```python
# Calculate area in real units (if known)
pixel_size_um = 0.1  # Example: 100nm per pixel
min_area_um2 = min_flake_area * (pixel_size_um ** 2)
print(f"Minimum flake area: {min_area_um2:.2f} μm²")
```

---

### 🎯 **Stage 2 Parameters (Multilayer Detection)**

#### `min_internal_area` (High Impact)
**Default: 30 pixels** | **Range: 10-100** | **Impact: ⭐⭐⭐⭐**

**Physical Interpretation:**
```python
# Minimum twisted domain size
pixel_size_nm = 100  # Example
min_domain_size = np.sqrt(min_internal_area) * pixel_size_nm
print(f"Minimum domain diameter: {min_domain_size:.0f} nm")
```

**Tuning Strategy:**
- **Smaller (10-20)**: Catches tiny domains, more noise
- **Larger (50-80)**: Only significant domains, fewer detections
- **Literature Guide**: Typical twisted domains are 100-1000 nm

#### `min_area_ratio` (Critical)
**Default: 0.005 (0.5%)** | **Range: 0.001-0.02** | **Impact: ⭐⭐⭐⭐⭐**

**Understanding the Impact:**
```python
# Example calculation for a 1000-pixel flake
flake_area = 1000
min_structure_area_pct = min_area_ratio * 100
min_structure_pixels = flake_area * min_area_ratio

print(f"For a {flake_area}-pixel flake:")
print(f"Minimum internal structure: {min_structure_pixels:.1f} pixels ({min_structure_area_pct:.1f}%)")
```

**Tuning Guidelines:**
- **0.001-0.003**: Ultra-sensitive, research mode
- **0.005-0.008**: Balanced sensitivity
- **0.01-0.02**: Conservative, publication-ready

#### `intensity_drops` (Fine-tuning)
**Default: [2, 5, 8, 12, 18, 25]** | **Impact: ⭐⭐⭐⭐**

**Understanding Intensity Analysis:**
```python
# Diagnostic: Check intensity variations in your flakes
def analyze_flake_intensity(roi_gray, roi_mask):
    masked_pixels = roi_gray[roi_mask > 0]
    mean_int = masked_pixels.mean()
    std_int = masked_pixels.std()
    
    print(f"Flake mean intensity: {mean_int:.1f}")
    print(f"Standard deviation: {std_int:.1f}")
    
    # Suggested intensity drops based on std
    suggested_drops = [
        std_int * 0.2, std_int * 0.5, std_int * 1.0, 
        std_int * 1.5, std_int * 2.0
    ]
    print(f"Suggested drops: {[int(d) for d in suggested_drops]}")
```

**Customization Strategies:**
- **High contrast samples**: [5, 10, 15, 20, 25]
- **Low contrast samples**: [1, 2, 3, 5, 8, 12]
- **Unknown samples**: Use diagnostic code above

---

## Systematic Tuning Workflow

### 📊 **Step-by-Step Optimization Process**

#### **Phase 1: Stage 1 Optimization**

```python
# 1. Test intensity threshold range
def optimize_stage1(image_path):
    results = {}
    thresholds = range(120, 161, 5)  # Test 120, 125, 130, ..., 160
    
    for thresh in thresholds:
        pipeline = UltraSensitiveMoS2Pipeline()
        pipeline.intensity_threshold = thresh
        
        img_rgb, gray, binary, flakes = pipeline.stage1_detect_flakes(image_path)
        
        results[thresh] = {
            'flake_count': len(flakes),
            'avg_area': np.mean([f['area'] for f in flakes]) if flakes else 0,
            'avg_solidity': np.mean([f['solidity'] for f in flakes]) if flakes else 0
        }
    
    # Plot results
    thresholds_list = list(results.keys())
    flake_counts = [results[t]['flake_count'] for t in thresholds_list]
    
    plt.figure(figsize=(10, 6))
    plt.plot(thresholds_list, flake_counts, 'bo-')
    plt.xlabel('Intensity Threshold')
    plt.ylabel('Number of Flakes Detected')
    plt.title('Stage 1 Optimization')
    plt.grid(True)
    
    # Find optimal threshold (look for plateau)
    optimal_idx = np.argmax(flake_counts)
    optimal_threshold = thresholds_list[optimal_idx]
    print(f"Suggested threshold: {optimal_threshold}")
    
    return results
```

#### **Phase 2: Stage 2 Sensitivity Tuning**

```python
# 2. Optimize multilayer detection sensitivity
def optimize_stage2(image_path, fixed_threshold):
    pipeline = UltraSensitiveMoS2Pipeline()
    pipeline.intensity_threshold = fixed_threshold
    
    # Get Stage 1 results
    img_rgb, gray, binary, flakes = pipeline.stage1_detect_flakes(image_path)
    print(f"Stage 1: Found {len(flakes)} flakes")
    
    # Test different sensitivity levels
    sensitivity_levels = {
        'conservative': {
            'min_area_ratio': 0.015,
            'min_internal_area': 60,
            'intensity_drops': [5, 10, 15]
        },
        'balanced': {
            'min_area_ratio': 0.008,
            'min_internal_area': 40,
            'intensity_drops': [3, 6, 10, 15, 20]
        },
        'sensitive': {
            'min_area_ratio': 0.005,
            'min_internal_area': 30,
            'intensity_drops': [2, 5, 8, 12, 18, 25]
        },
        'ultra_sensitive': {
            'min_area_ratio': 0.002,
            'min_internal_area': 20,
            'intensity_drops': [1, 2, 4, 6, 8, 10, 15, 20, 30]
        }
    }
    
    results = {}
    for level_name, params in sensitivity_levels.items():
        # Apply parameters
        for param, value in params.items():
            setattr(pipeline, param, value)
        
        # Run Stage 2
        multilayer_flakes = pipeline.stage2_ultra_sensitive_detection(img_rgb, gray, flakes)
        
        detection_rate = len(multilayer_flakes) / len(flakes) * 100 if flakes else 0
        total_structures = sum(len(f.get('internal_structures', [])) for f in multilayer_flakes)
        
        results[level_name] = {
            'multilayer_count': len(multilayer_flakes),
            'detection_rate': detection_rate,
            'total_structures': total_structures,
            'avg_structures_per_flake': total_structures / len(multilayer_flakes) if multilayer_flakes else 0
        }
        
        print(f"{level_name}: {len(multilayer_flakes)} multilayer flakes ({detection_rate:.1f}%), "
              f"{total_structures} total structures")
    
    return results
```

#### **Phase 3: Method-Specific Optimization**

```python
# 3. Analyze method effectiveness
def analyze_method_effectiveness(multilayer_flakes):
    method_stats = {}
    
    for flake in multilayer_flakes:
        for structure in flake.get('internal_structures', []):
            method = structure.get('detection_method', 'unknown')
            base_method = method.split('_')[0]  # Get base method name
            
            if base_method not in method_stats:
                method_stats[base_method] = {
                    'count': 0,
                    'total_area': 0,
                    'total_confidence': 0,
                    'areas': [],
                    'confidences': []
                }
            
            method_stats[base_method]['count'] += 1
            method_stats[base_method]['total_area'] += structure['area']
            method_stats[base_method]['areas'].append(structure['area'])
            
            confidence = structure.get('confidence', 0)
            method_stats[base_method]['total_confidence'] += confidence
            method_stats[base_method]['confidences'].append(confidence)
    
    # Calculate statistics
    for method, stats in method_stats.items():
        if stats['count'] > 0:
            stats['avg_area'] = stats['total_area'] / stats['count']
            stats['avg_confidence'] = stats['total_confidence'] / stats['count']
            stats['area_std'] = np.std(stats['areas'])
            
            print(f"{method.title()}: {stats['count']} structures")
            print(f"  Avg area: {stats['avg_area']:.1f} ± {stats['area_std']:.1f} pixels")
            print(f"  Avg confidence: {stats['avg_confidence']:.3f}")
    
    return method_stats
```

---

## Common Issues & Solutions

### ❌ **Issue 1: Very Low Detection Rate (<10%)**

**Symptoms:**
- Only 1-2 multilayer flakes detected
- Expected 10+ from visual inspection
- Most obvious multilayers missed

**Diagnostic Questions:**
1. Are flakes being detected in Stage 1?
2. What's the intensity contrast in your images?
3. Are the missed structures very small or faint?

**Solutions:**

```python
# Solution A: More aggressive parameters
pipeline.min_area_ratio = 0.002          # From 0.005 to 0.002
pipeline.min_internal_area = 20           # From 30 to 20
pipeline.intensity_drops = [1,2,3,5,8,12,18,25]  # Add very low drops

# Solution B: Check intensity threshold
# Run diagnostic code from Parameter Deep Dive section

# Solution C: Enable hyper-sensitive edge detection
pipeline.edge_methods = [
    (5, 20),   # Hyper-sensitive
    (8, 25),   # Ultra-sensitive  
    (10, 30),  # Very sensitive
    (15, 45),  # Sensitive
    (20, 60),  # Medium
    (25, 75)   # Conservative
]
```

### ❌ **Issue 2: Too Many False Positives**

**Symptoms:**
- Detection rate >90% but obvious noise
- Tiny specks detected as multilayer structures
- Unrealistic structure counts (>50 per flake)

**Solutions:**

```python
# Solution A: More conservative parameters
pipeline.min_area_ratio = 0.01            # From 0.005 to 0.01
pipeline.min_internal_area = 50            # From 30 to 50

# Solution B: Add shape validation
def validate_structure_shape(contour):
    area = cv2.contourArea(contour)
    perimeter = cv2.arcLength(contour, True)
    if perimeter == 0:
        return False
    
    circularity = 4 * np.pi * area / (perimeter * perimeter)
    return circularity > 0.1  # Reject very elongated shapes

# Solution C: Reduce intensity sensitivity
pipeline.intensity_drops = [5, 10, 15, 20]  # Remove very low drops
```

### ❌ **Issue 3: Inconsistent Results Between Images**

**Symptoms:**
- Good detection on some images, poor on others
- Same sample type, different performance
- Parameters seem image-dependent

**Solutions:**

```python
# Solution A: Adaptive thresholding
def adaptive_intensity_threshold(gray):
    # Use Otsu's method as starting point
    _, binary_otsu = cv2.threshold(gray, 0, 255, cv2.THRESH_BINARY + cv2.THRESH_OTSU)
    otsu_threshold = cv2.threshold(gray, 0, 255, cv2.THRESH_BINARY + cv2.THRESH_OTSU)[0]
    
    # Adjust based on image statistics
    mean_intensity = gray.mean()
    suggested_threshold = min(otsu_threshold, mean_intensity - 20)
    
    return int(suggested_threshold)

# Solution B: Normalize images before analysis
def normalize_image_contrast(img):
    # Convert to LAB color space
    lab = cv2.cvtColor(img, cv2.COLOR_RGB2LAB)
    l, a, b = cv2.split(lab)
    
    # Apply CLAHE to L channel
    clahe = cv2.createCLAHE(clipLimit=3.0, tileGridSize=(8,8))
    l = clahe.apply(l)
    
    # Merge and convert back
    enhanced = cv2.merge([l, a, b])
    enhanced_rgb = cv2.cvtColor(enhanced, cv2.COLOR_LAB2RGB)
    
    return enhanced_rgb
```

### ❌ **Issue 4: Poor Twist Angle Accuracy**

**Symptoms:**
- Angles outside expected range (0-60°)
- Many angles near 0° or 60°
- Inconsistent measurements

**Solutions:**

```python
# Solution A: Improve structure quality before angle calculation
def filter_structures_for_angles(internal_structures):
    filtered = []
    for structure in internal_structures:
        # Require minimum vertices for good angle calculation
        if len(structure['approx']) >= 4:
            # Require reasonable area
            if structure['area'] > 50:
                # Require good shape (not too elongated)
                hull = cv2.convexHull(structure['contour'])
                hull_area = cv2.contourArea(hull)
                solidity = structure['area'] / hull_area if hull_area > 0 else 0
                
                if solidity > 0.5:
                    filtered.append(structure)
    
    return filtered

# Solution B: Multiple angle calculation methods
def robust_angle_calculation(vertices):
    angles = []
    
    # Method 1: Centroid to farthest point
    centroid = np.mean(vertices, axis=0)
    distances = np.linalg.norm(vertices - centroid, axis=1)
    apex_idx = np.argmax(distances)
    angle1 = np.arctan2(vertices[apex_idx][1] - centroid[1], 
                       vertices[apex_idx][0] - centroid[0])
    angles.append(np.degrees(angle1) % 360)
    
    # Method 2: Principal component analysis
    pca_vertices = vertices - centroid
    cov_matrix = np.cov(pca_vertices.T)
    eigenvalues, eigenvectors = np.linalg.eig(cov_matrix)
    primary_direction = eigenvectors[:, np.argmax(eigenvalues)]
    angle2 = np.arctan2(primary_direction[1], primary_direction[0])
    angles.append(np.degrees(angle2) % 360)
    
    # Return median of methods
    return np.median(angles)
```

---

## Advanced Tuning Strategies

### 🧠 **Adaptive Parameter Selection**

```python
class AdaptiveMoS2Pipeline(UltraSensitiveMoS2Pipeline):
    def __init__(self):
        super().__init__()
        self.auto_tune = True
    
    def analyze_image_properties(self, img_rgb):
        """Analyze image to suggest optimal parameters"""
        gray = cv2.cvtColor(img_rgb, cv2.COLOR_RGB2GRAY)
        
        # Image statistics
        mean_int = gray.mean()
        std_int = gray.std()
        
        # Contrast analysis
        contrast = std_int / mean_int
        
        # Noise level estimation
        noise_level = cv2.Laplacian(gray, cv2.CV_64F).var()
        
        # Suggested parameters based on analysis
        if contrast < 0.1:  # Low contrast
            self.intensity_threshold = max(120, mean_int - 30)
            self.min_area_ratio = 0.003
            self.intensity_drops = [1, 2, 3, 5, 8, 12, 18]
        elif contrast > 0.2:  # High contrast
            self.intensity_threshold = min(160, mean_int - 15)
            self.min_area_ratio = 0.008
            self.intensity_drops = [5, 10, 15, 20]
        else:  # Medium contrast
            self.intensity_threshold = mean_int - 20
            self.min_area_ratio = 0.005
            self.intensity_drops = [2, 5, 8, 12, 18, 25]
        
        if noise_level > 1000:  # Noisy image
            self.min_internal_area = max(40, self.min_internal_area)
        
        print(f"Image analysis - Contrast: {contrast:.3f}, Noise: {noise_level:.1f}")
        print(f"Suggested threshold: {self.intensity_threshold}")
        print(f"Suggested area ratio: {self.min_area_ratio}")
```

### 📈 **Multi-Scale Analysis**

```python
def multi_scale_detection(self, roi_gray, roi_mask, offset_x, offset_y):
    """Run detection at multiple scales to catch different sized features"""
    all_structures = []
    
    # Scale factors to test
    scale_factors = [0.8, 1.0, 1.2]
    
    for scale in scale_factors:
        if scale != 1.0:
            # Resize ROI
            new_height, new_width = int(roi_gray.shape[0] * scale), int(roi_gray.shape[1] * scale)
            scaled_roi = cv2.resize(roi_gray, (new_width, new_height))
            scaled_mask = cv2.resize(roi_mask, (new_width, new_height))
        else:
            scaled_roi = roi_gray
            scaled_mask = roi_mask
        
        # Adjust parameters for scale
        scaled_min_area = self.min_internal_area * (scale ** 2)
        
        # Run detection methods with scaled parameters
        # ... (detection code with scaled parameters)
        
        # Scale contours back to original size
        for structure in scale_structures:
            if scale != 1.0:
                structure['contour'] = structure['contour'] / scale
        
        all_structures.extend(scale_structures)
    
    return all_structures
```

---

## Validation & Quality Assessment

### ✅ **Automated Quality Metrics**

```python
def calculate_quality_metrics(results):
    """Calculate quality metrics for parameter validation"""
    metrics = {}
    
    # Detection consistency
    detection_rates = [r['analysis_summary']['detection_rate_percent'] for r in results]
    metrics['detection_rate_mean'] = np.mean(detection_rates)
    metrics['detection_rate_std'] = np.std(detection_rates)
    
    # Structure size consistency
    all_areas = []
    for result in results:
        for multilayer in result['multilayer_details']:
            for structure in multilayer['internal_structures']:
                all_areas.append(structure['area'])
    
    if all_areas:
        metrics['structure_area_mean'] = np.mean(all_areas)
        metrics['structure_area_std'] = np.std(all_areas)
        metrics['area_coefficient_variation'] = metrics['structure_area_std'] / metrics['structure_area_mean']
    
    # Angle consistency
    all_angles = []
    for result in results:
        for angle_data in result['twist_angle_data']:
            all_angles.append(angle_data['average_twist'])
    
    if all_angles:
        metrics['angle_mean'] = np.mean(all_angles)
        metrics['angle_std'] = np.std(all_angles)
        metrics['angle_range'] = max(all_angles) - min(all_angles)
    
    # Quality score (0-1, higher is better)
    quality_score = 0.0
    
    # Penalize very low or very high detection rates
    ideal_detection_rate = 0.7  # 70%
    detection_penalty = abs(metrics['detection_rate_mean'] / 100 - ideal_detection_rate)
    quality_score += max(0, 0.4 - detection_penalty)  # Max 0.4 points
    
    # Reward consistency
    if metrics['detection_rate_std'] < 20:  # Less than 20% variation
        quality_score += 0.3
    
    # Reward reasonable structure sizes
    if 'area_coefficient_variation' in metrics and metrics['area_coefficient_variation'] < 2.0:
        quality_score += 0.2
    
    # Reward reasonable angles
    if 'angle_range' in metrics and metrics['angle_range'] < 50:  # Less than 50° range
        quality_score += 0.1
    
    metrics['quality_score'] = quality_score
    
    return metrics

def recommend_parameters(quality_metrics, current_params):
    """Recommend parameter adjustments based on quality metrics"""
    recommendations = []
    
    if quality_metrics['detection_rate_mean'] < 30:  # Too low
        recommendations.append("Decrease min_area_ratio (more sensitive)")
        recommendations.append("Decrease min_internal_area")
        recommendations.append("Add more aggressive intensity_drops")
    
    elif quality_metrics['detection_rate_mean'] > 90:  # Too high, likely false positives
        recommendations.append("Increase min_area_ratio (less sensitive)")
        recommendations.append("Increase min_internal_area")
        recommendations.append("Remove very low intensity_drops")
    
    if quality_metrics.get('detection_rate_std', 0) > 30:  # Inconsistent
        recommendations.append("Consider adaptive thresholding")
        recommendations.append("Normalize image contrast before analysis")
    
    if quality_metrics.get('angle_range', 0) > 50:  # Unrealistic angle range
        recommendations.append("Improve structure quality filters")
        recommendations.append("Use more conservative parameters")
    
    return recommendations
```

---

## Performance Optimization

### ⚡ **Speed Optimization**

```python
# Optimize for speed without losing too much accuracy
FAST_PARAMETERS = {
    'intensity_threshold': 140,
    'min_internal_area': 40,        # Slightly larger (fewer candidates)
    'min_area_ratio': 0.008,        # Less sensitive (fewer false positives)
    'intensity_drops': [5, 10, 15], # Fewer levels
    'edge_methods': [(15,45), (25,75)], # Only 2 methods
    'skip_template_matching': True   # Skip slowest method
}

# Process images in parallel
from multiprocessing import Pool
import functools

def process_image_fast(image_path, parameters):
    pipeline = UltraSensitiveMoS2Pipeline()
    for param, value in parameters.items():
        if hasattr(pipeline, param):
            setattr(pipeline, param, value)
    
    return pipeline.process_ultra_pipeline(image_path)

def batch_process_optimized(image_paths, parameters, n_cores=4):
    process_func = functools.partial(process_image_fast, parameters=parameters)
    
    with Pool(n_cores) as pool:
        results = pool.map(process_func, image_paths)
    
    return results
```

### 💾 **Memory Optimization**

```python
# For processing many large images
def memory_efficient_pipeline(image_path):
    """Process image with minimal memory footprint"""
    
    # Process image in tiles if very large
    img = cv2.imread(image_path)
    if img.shape[0] * img.shape[1] > 4096 * 4096:  # Very large image
        return process_image_tiled(img)
    
    # Normal processing with memory cleanup
    pipeline = UltraSensitiveMoS2Pipeline()
    
    # Stage 1
    results_stage1 = pipeline.stage1_detect_flakes(image_path)
    
    # Clear large intermediate arrays
    del pipeline  # Free pipeline memory
    
    # Only keep essential data for Stage 2
    essential_data = {
        'image': results_stage1[0],
        'gray': results_stage1[1],
        'flakes': results_stage1[3]
    }
    
    # Recreate pipeline for Stage 2
    pipeline = UltraSensitiveMoS2Pipeline()
    multilayer_results = pipeline.stage2_ultra_sensitive_detection(
        essential_data['image'], 
        essential_data['gray'], 
        essential_data['flakes']
    )
    
    # Continue with reduced memory footprint...
    return multilayer_results
```

---

## Troubleshooting Checklist

### 🔧 **Before Running Analysis**

- [ ] **Image Quality Check**
  - [ ] Images are in focus
  - [ ] Good contrast between flakes and substrate
  - [ ] Minimal dust/contamination
  - [ ] Consistent illumination

- [ ] **File Format Check**
  - [ ] Images are in supported format (PNG, JPG, TIFF)
  - [ ] File paths are correct
  - [ ] Images are not corrupted

- [ ] **Parameter Validation**
  - [ ] intensity_threshold appropriate for your images
  - [ ] min_area_ratio not too restrictive
  - [ ] intensity_drops cover expected range

### 🔍 **During Analysis**

- [ ] **Stage 1 Validation**
  - [ ] Check if expected number of flakes detected
  - [ ] Visually inspect Stage 1 results
  - [ ] Verify flake shapes look reasonable

- [ ] **Stage 2 Monitoring**
  - [ ] Monitor detection rate per flake
  - [ ] Check for excessive false positives
  - [ ] Verify internal structures look realistic

- [ ] **Error Handling**
  - [ ] Check for contour operation errors
  - [ ] Monitor memory usage for large images
  - [ ] Verify all detection methods are working

### ✅ **After Analysis**

- [ ] **Results Validation**
  - [ ] Detection rate in expected range (30-90%)
  - [ ] Twist angles realistic (0-60°)
  - [ ] Structure sizes reasonable
  - [ ] Visual inspection matches automated results

- [ ] **Quality Assessment**
  - [ ] Run quality metrics calculation
  - [ ] Check consistency across images
  - [ ] Validate against known samples if available

### 🚨 **Emergency Troubleshooting**

**If analysis completely fails:**
```python
# Minimal working example
pipeline = UltraSensitiveMoS2Pipeline()
pipeline.min_area_ratio = 0.02  # Very conservative
pipeline.min_internal_area = 100  # Large structures only
pipeline.intensity_drops = [10, 20]  # Simple detection

# Try with single detection method
pipeline.disable_edge_detection = True
pipeline.disable_hierarchy_detection = True
pipeline.disable_template_matching = True
# Only use intensity-based detection
```

**If getting strange errors:**
```python
# Enable debug mode
pipeline.debug_mode = True
pipeline.verbose = True

# Check intermediate results
img_rgb, gray, binary, flakes = pipeline.stage1_detect_flakes(image_path)
print(f"Stage 1 successful: {len(flakes)} flakes")

# Test single flake
if flakes:
    test_flake = [flakes[0]]  # Test with just first flake
    multilayer = pipeline.stage2_ultra_sensitive_detection(img_rgb, gray, test_flake)
    print(f"Stage 2 test successful: {len(multilayer)} multilayer")
```

This comprehensive parameter tuning guide should help you optimize the MoS₂ analysis pipeline for your specific samples and requirements!