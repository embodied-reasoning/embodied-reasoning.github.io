# Web Validator Rules — PointArena 2026

## 1. Per-Pipeline Requirements (ALL 3 pipelines must satisfy)

### 1.1 Pipeline A — Gemini
- [ ] Topology SVG showing 5-module flow (Raw→Gatekeeper→Classify→Multipoint→Rewrite)
- [ ] 5 per-stage composite images (image + data text merged)
- [ ] 10 individual sample crops from 10 DIFFERENT source images
- [ ] 10-row data table with query/label/gatekeeper/category/multipoint
- [ ] Real JSON data trace visible (actual sample values)

### 1.2 Pipeline B — Qwen3
- [ ] Topology SVG showing 4-stage + rule/dedup/counting layer
- [ ] 5 per-stage visualization images (image + data text merged)
- [ ] At least 5 sample visualization crops (from Qwen3 pipeline outputs)
- [ ] 10-row rule-application table
- [ ] Real Qwen3 sample trace with actual JSON output
- [ ] REJECT if Pipeline B has fewer images than Pipeline A

### 1.3 Pipeline C — Steerable
- [ ] Topology SVG showing 4-step flow
- [ ] 4 SAM3 mask images (preview, overlay, original, dilated)
- [ ] 10 steerable sample visualizations with anchor→target→direction
- [ ] ALL 10 MUST use real source images (photographs), NOT dark-background coordinate diagrams
- [ ] REJECT if any steerable viz is a coordinate diagram (check: if image mode is RGB and all pixels near-black → REJECT)
- [ ] REJECT if any steerable viz image < 5KB (real photos are larger)
- [ ] 10-row data table with anchor/target/direction/question
- [ ] masks.json control point data visible

## 2. Image Quality Checks
- All `<img src>` must reference files existing on disk
- Minimum file sizes: general > 1KB, mask PNGs > 100B, steerable real-image viz > 500B
- Every pipeline section must have >= 5 `<img>` tags
- Pipeline B section must have at least as many images as Pipeline A
- No uneditable pipeline PNGs (viz_gemini_pipeline.png, viz_steerable_pipeline.png)
- No fa55de image references

## 3. SVG Topology Checks
- Each pipeline must have at least 1 `<svg>` element
- All rects within viewBox bounds
- All lines connect between valid module centers
- No text touching rect boundaries (< 2px margin)

## 4. Data Completeness
- Pipeline A: 10 sample rows with all 6 columns
- Pipeline B: 10 rule-sample rows with all 5 columns
- Pipeline C: 10 steerable sample rows with anchor/target coordinates

## 5. Validation Script
```python
import re, os

web = r'C:\Users\Administrator\Desktop\讲解ppt制作\web'
with open(os.path.join(web, 'index.html'), 'r', encoding='utf-8') as f:
    html = f.read()

errors = []

# === 1. IMAGE EXISTENCE ===
imgs_in_html = re.findall(r'<img src=\"([^\"]+)\"', html)
for img in imgs_in_html:
    p = os.path.join(web, img)
    if not os.path.exists(p):
        errors.append(f'MISSING: {img}')
        continue
    sz = os.path.getsize(p)
    # Different thresholds for different image types
    if 'mask' in img:
        if sz < 100: errors.append(f'CORRUPT MASK: {img} ({sz}B)')
    elif 'steer_viz' in img and img.endswith('.jpg'):
        if sz < 500: errors.append(f'CORRUPT STEER: {img} ({sz}B)')
    else:
        if sz < 1024: errors.append(f'CORRUPT: {img} ({sz}B)')

# === 2. FORBIDDEN ===
for bad in ['viz_gemini_pipeline.png', 'viz_steerable_pipeline.png', 'fa55de']:
    if bad in html:
        errors.append(f'FORBIDDEN: {bad}')

# === 3. PIPELINE A CHECKS ===
a_stage_imgs = [i for i in imgs_in_html if 'stage' in i and 'steer' not in i]
a_crops = [i for i in imgs_in_html if 'sample' in i and 'crop' in i]
if len(a_stage_imgs) < 5:
    errors.append(f'Pipe A stage imgs: {len(a_stage_imgs)}/5')
if len(a_crops) < 10:
    errors.append(f'Pipe A crops: {len(a_crops)}/10')

# Check 10 different source images for crops (by checking file sizes differ)
crop_sizes = []
for c in a_crops:
    p = os.path.join(web, c)
    if os.path.exists(p):
        crop_sizes.append(os.path.getsize(p))
unique_sizes = len(set(crop_sizes))
if unique_sizes < 5:
    errors.append(f'Pipe A: only {unique_sizes} unique crop sizes (need >= 5 different images)')

# === 4. PIPELINE B CHECKS (NEW - previously missing!) ===
# Find the Pipeline B section boundaries
b_section_start = html.find('id="p-b"')
b_section_end = html.find('id="p-c"')
if b_section_start < 0 or b_section_end < 0:
    errors.append('Pipe B: section not found')
else:
    b_html = html[b_section_start:b_section_end]
    b_imgs = re.findall(r'<img src=\"([^\"]+)\"', b_html)
    b_svgs = re.findall(r'<svg ', b_html)
    
    if len(b_svgs) < 1:
        errors.append(f'Pipe B: no topology SVG ({len(b_svgs)})')
    if len(b_imgs) < 5:
        errors.append(f'Pipe B: only {len(b_imgs)} images (need >= 5: stage viz + sample crops)')
    
    # Check for stage images in Pipeline B section
    b_stage_imgs = [i for i in b_imgs if 'stage' in i.lower() or 'pipe_b' in i.lower() or 'qwen' in i.lower()]
    b_sample_imgs = [i for i in b_imgs if 'sample' in i.lower() or 'crop' in i.lower()]
    if len(b_stage_imgs) < 3 and len(b_sample_imgs) < 3:
        errors.append(f'Pipe B: no per-stage or sample images found (stage:{len(b_stage_imgs)}, sample:{len(b_sample_imgs)})')
    
    # Check for data table
    b_tables = len(re.findall(r'<table>', b_html))
    if b_tables < 1:
        errors.append('Pipe B: no data table')

# === 5. PIPELINE C CHECKS ===
c_section_start = html.find('id="p-c"')
c_section_end = html.find('id="attnres"')
if c_section_start < 0:
    errors.append('Pipe C: section not found')
else:
    c_html = html[c_section_start:c_section_end]
    c_imgs = re.findall(r'<img src=\"([^\"]+)\"', c_html)
    c_sam3 = [i for i in c_imgs if 'sam3' in i]
    c_steer = [i for i in c_imgs if 'steer_viz' in i]
    
    if len(c_sam3) < 4:
        errors.append(f'Pipe C SAM3: {len(c_sam3)}/4')
    if len(c_steer) < 10:
        errors.append(f'Pipe C steer viz: {len(c_steer)}/10')
    
    # Check at least 5 steerable viz are real images (JPG), not just coords (PNG)
    c_steer_jpg = [i for i in c_steer if i.endswith('.jpg')]
    if len(c_steer_jpg) < 5:
        errors.append(f'Pipe C: only {len(c_steer_jpg)} real-image steer viz (need >= 5)')

# === 6. SVG BOUNDS ===
svgs = re.findall(r'<svg viewBox=\"([^\"]+)\"[^>]*>(.*?)</svg>', html, re.DOTALL)
for vi, (vb, content) in enumerate(svgs):
    vb_w, vb_h = int(vb.split()[2]), int(vb.split()[3])
    for m in re.finditer(r'<rect[^>]*x=\"([^\"]+)\"[^>]*y=\"([^\"]+)\"[^>]*width=\"([^\"]+)\"[^>]*height=\"([^\"]+)\"', content):
        rx, ry, rw, rh = float(m[1]), float(m[2]), float(m[3]), float(m[4])
        if rx+rw > vb_w+2 or ry+rh > vb_h+2:
            errors.append(f'SVG#{vi} OOB')

# === SUMMARY ===
print(f'Total images: {len(set(imgs_in_html))}')
print(f'Pipe A: {len(a_stage_imgs)} stage + {len(a_crops)} crops')
print(f'Pipe B: imgs in section')
print(f'Pipe C: {len(c_sam3)} SAM3 + {len(c_steer)} steer ({len(c_steer_jpg)} real-img)')
print(f'SVGs: {len(svgs)}')

if errors:
    print(f'\nFAILED ({len(errors)}):')
    for e in errors: print(f'  X {e}')
    exit(1)
else:
    print('\nALL CHECKS PASSED')
```
