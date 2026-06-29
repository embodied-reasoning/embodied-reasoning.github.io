# PointArena 2026 宣传网页 — 完整交接文档

## 1. 项目概述

为 PointArena 2026 竞赛（2nd Place, 77.2% Overall）制作成果展示网页，展示三条数据管道（Pipeline A/B/C）和模型创新。

**网页文件**: `web/index.html`（单文件，内嵌 CSS + SVG + JS）
**服务器**: `lv_qi@10.249.47.220` (SSH key: `~/.ssh/id_ed12234`)

---

## 2. 服务器连接

```
Host lv_qi_a100
  HostName 10.249.47.220
  User lv_qi
  IdentityFile ~/.ssh/id_ed12234
  IdentitiesOnly yes
```

### 2.1 关键目录

| 路径 | 内容 |
|------|------|
| `/mnt/data/lv_qi/PointArena/pointarena_extract/` | Pipeline A (Gemini) 脚本和输出 |
| `/mnt/data/lv_qi/PointArena/pointarena_extract/mix4_pointsabs_len1_10000_rule_llm/pixmo_points/` | Pipeline A 输出 JSON (2991个) |
| `/mnt/data/lv_qi/PointArena/clean_3types_local/qwen3_8b_three_types_20k_forward_v2/` | Pipeline B (Qwen3) 输出 |
| `/mnt/data/lv_qi/PointArena/clean_counting_local/` | Pipeline B Counting 管道 |
| `/mnt/data/lv_qi/PointArena/make_steerable1/` | Pipeline C (Steerable) 最新版 |
| `/mnt/data/lv_qi/PointArena/make_steerable1/sam_clean/outputs/mix10000_v2/07_final/accepted_samples.jsonl` | Steerable 最终输出 (10000条) |
| `/mnt/data/lv_qi/PointArena/make_steerable1/guide4/outputs_strict_smoke100_v2/02_sam_masks/` | SAM3 mask 输出 (含 preview/overlay/dilated) |
| `/mnt/data/lv_qi/PointArena/artifacts/raw/pixmo_points_full/images/` | 源图片 (SHA256 命名) |

### 2.2 常用命令

```bash
# 列出样本
ssh -i ~/.ssh/id_ed12234 lv_qi@10.249.47.220 "ls /path/to/dir/ | head -20"

# 下载单个文件
scp -i ~/.ssh/id_ed12234 lv_qi@10.249.47.220:<remote_path> <local_path>

# 服务器端提取数据
ssh ... "python3 -c '...' > /tmp/out.json"
scp ...:/tmp/out.json .
```

---

## 3. 三条管道算法概要

### 3.1 Pipeline A: Gemini API 4-Stage Agent (5-Class)

**源码**: `pointarena_extract/process_mix4_clean_classify_rewrite_same_session.py`

**流程**: Raw Record → Gatekeeper → 5-Way Classification → Multipoint Detection → Rewrite

**Stage 说明**:
- **Gatekeeper**: 验证是否为 pointing task，提取 core_target，拒绝纯 QA
- **5-Way Classification**: Affordance / Spatial Relation / Reasoning / Object Reference / Counting
- **Multipoint**: 判断 single vs multi-point (检测 all/several/each/every)
- **Rewrite**: 转换为 "Point to..." 规范格式，保留认知瓶颈线索

**容错**: parse_retries=2, max_retries=6, sample_retries=4, checkpoint-resume, ThreadPoolExecutor(32)

**输出 JSON 结构**:
```json
{
  "processing": {"status": "completed", "source_rank": N, "model": "gemini-3-flash-preview-thinking"},
  "record": {"query": "...", "points_abs": [[x,y]], "image_path": "...", "raw_meta": {"label": "..."}},
  "cleaning": {"keep": true, "core_target": "...", "attempts": [{"elapsed_sec": 4.33}]},
  "classification": {"parsed": {"category": "Object Reference", "thought": "..."}},
  "multipoint": {"parsed": {"multi_point": false, "reason": "..."}},
  "rewrite": {"parsed": null}
}
```

### 3.2 Pipeline B: Qwen3-8B Local 3-Class + Rules

**源码**: `clean_3types_local/process_mix4_clean_classify_rewrite_same_session_local.py`
**规则**: `clean_3types_local/three_type_rules.py`

**流程**: 同 Pipeline A 的 4-Stage + 3-class 限制 + 规则层 + 去重 + Counting 管道

**关键规则**:
- 18 个 affordance 关键词: "where you would", "used to hold", "to grip", "place to", "area for"
- 14 个 reasoning regex: "direction", "moving", "most likely", "background", "empty space"
- Multi-instance: "all"/"several"/"each"/"every" → drop/routing
- Existing-point: "existing point"/"cursor" → force Object Reference

**去重**: `build_exclusion_keys.py` → `exclusion_image_keys.json` (614K entries), nodup_reverse mode

**Counting 管道**: `build_counting_direct.py`, 20K samples/版本 (v2-v5)

### 3.3 Pipeline C: Steerable LLM-FREE

**源码**: `make_steerable1/sam_clean/run_build_mix10000_v2.py`

**4-Step 流程**:
1. **18-Rule Label Filter**: 过滤 point_records.parquet
2. **SAM3-Tracker ONNX**: 控制点 → binary mask → mask_dilate_px=5 → area_pixels
3. **700 Templates × 7 Groups**: 生成 anchor-target-direction candidates
4. **Connected-Component Verification**: path_width_px=3, 检查 dilated masks, 0 other objects → ACCEPTED

**5 个子类型 (Task Types)**:

| # | Type | Ratio | Samples | Pipeline Logic |
|---|------|-------|---------|----------------|
| 1 | move_until_axis_fixed | 85% | 8,500 | anchor → direction → target on same axis |
| 2 | nearest_among_objects | 5% | 500 | anchor → direction scan → nearest in path |
| 3 | nearest_axis_constrained | 5% | 500 | direction + axis constraint → nearest |
| 4 | move_then_axis_limited | 2.5% | 250 | move step → axis-limited search |
| 5 | move_until_from_other | 2.5% | 250 | axis from other object ref → target |

**Tiebreak 顺序**: nearest > nearest_axis > move_then > move_until_other > move_until_axis_fixed

---

## 4. 数据下载策略

### 4.1 选取 10 个不同图片的样本

```python
# 在服务器上遍历输出目录, 按 image_path 去重
images = defaultdict(list)
for fn in os.listdir(output_dir):
    d = json.load(open(fn))
    img = d['record']['image_path']
    if keep and cat != '?':
        images[img].append((fn, query, category))
    if len(images) >= 15: break

# 取前 10 个不同图片的样本
selected = list(images.items())[:10]
```

### 4.2 批量下载模式

```python
# 先提取 JSON 数据到 /tmp, 再 scp
ssh remote "python3 -c '...' > /tmp/samples.json"
scp remote:/tmp/samples.json local/

# 并行下载源图片 (每个样本一张)
subprocess.run(['scp', f'remote:{image_path}', f'img{i:02d}.jpg'])
```

### 4.3 图片质量检查

```python
arr = np.array(img)
brightness = arr.mean()
color_score = arr[:,:,0].std() + arr[:,:,1].std() + arr[:,:,2].std()
# brightness > 80, color_score > 150 → 色彩鲜艳，可用
# brightness < 50, color_score < 100 → 可能黑白，换掉
```

---

## 5. 网页布局规范

### 5.1 整体结构

```
┌──────────┬────────────────────────────────────┐
│ Sidebar  │  Main Content (scroll)              │
│ 250px    │                                     │
│ fixed    │  Hero + 4 stats                     │
│          │  Motivation + baseline table         │
│  TOC     │  Pipeline Overview SVG               │
│  links   │  ┌─────────────────────────────┐   │
│          │  │ Pipeline A                   │   │
│          │  │  Topology SVG                │   │
│          │  │  Per-Stage Rows (5 rows)     │   │
│          │  │  10 Sample Squares           │   │
│          │  │  Data Table + Rewrite        │   │
│          │  ├─────────────────────────────┤   │
│          │  │ Pipeline B                   │   │
│          │  │  Topology SVG                │   │
│          │  │  Rule Table (10 rows)        │   │
│          │  ├─────────────────────────────┤   │
│          │  │ Pipeline C                   │   │
│          │  │  5-Type Topology SVG         │   │
│          │  │  SAM3 Masks (4 images)       │   │
│          │  │  10 Steerable (2/type)       │   │
│          │  │  Sub-type Table + Data Table │   │
│          │  ├─────────────────────────────┤   │
│          │  │ AttnRes SVG                  │   │
│          │  │ ABC Encoding + Results       │   │
│          └─────────────────────────────────┘   │
└──────────┴────────────────────────────────────┘
```

### 5.2 Pipeline A 内部布局（模板 — B/C 参照此格式）

```
Pipeline A Section:
├── ey + st + sd (标题行)
├── Topology SVG (fc 容器)
├── Per-Stage Rows (5个 stage-row, 每个 flex: image 55% + text 45%)
├── 10 Balanced Samples (g2 网格, 每行2个)
├── Data Table (tw 容器, 10行 + rewrite列)
└── Caption paragraph
```

### 5.3 关键 CSS 类

| 类名 | 用途 |
|------|------|
| `.sidebar` | 固定左侧导航 (250px) |
| `.main` | 右侧主内容 (margin-left: 250px) |
| `.sec` | 内容章节 (max-width: 1100px, margin: 0 auto) |
| `.sec.alt` | 交替背景章节 |
| `.stage-row` | Per-stage 全宽行 (flex, image 55% + text 45%) |
| `.fc` | SVG 流程图容器 |
| `.ib` | 图片展示容器 |
| `.tw` | 表格容器 |
| `.g2/.g3/.g4` | 网格布局 |
| `.card` | 卡片 |

### 5.4 配色方案

```
--bg: #0D111A      主背景
--card: #161B28    卡片背景
--blue: #4F9CF5    Pipeline A / Spatial
--green: #34D399   Pipeline C / Affordance / 成功
--orange: #F59E4F  计数 / 输入数据
--purple: #A78BFA  AttnRes
--red: #F87171     Steerability / 错误
--cyan: #22D3EE    Pipeline B / Spatial Relation
--white: #E6E8EC   主文字
--gray: #AAABBB    次级文字
--muted: #707080   三级文字
```

---

## 6. 可视化方法

### 6.1 Per-Stage 图 — 纯标注无文字

**铁律**: Stage 图片必须是纯标注图（只含圈/框/箭头/符号），文字描述必须在 HTML 中。

```python
def make_stage_img(img, x, y, color, draw_fn):
    """创建纯标注 stage 图 (600x600, 无文字面板)"""
    im = img.copy().resize((600, 600), Image.LANCZOS)
    draw = ImageDraw.Draw(im)
    draw_fn(draw, x, y)
    # 彩色边框
    for i in range(4): draw.rectangle([i,i,599-i,599-i], outline=color, width=1)
    return im
```

**每步差异化标注**:

| Stage | 标注方法 | 颜色 |
|-------|---------|------|
| STAGE 0: Raw Input | 雷达搜索圈 (多层同心圆) | #F59E4F |
| STAGE 1: Gatekeeper | 边界框 + "TARGET LOCKED" | #34D399 |
| STAGE 2: Classification | 分类芯片 (高亮选中) | #4F9CF5 |
| STAGE 3: Multipoint | x1 标记 + xN 划线否决 | #A78BFA |
| STAGE 4: Rewrite | 等号箭头 + "NO CHANGE" | #34D399 |

### 6.2 样本正方形图 — 全图非裁剪

```python
def make_square(img, x, y, label, color):
    """全图正方形, 400x400, 十字准星 + 彩色边框 + 底部标签"""
    sq = max(w, h)
    canvas = Image.new('RGB', (sq, sq), '#1a1f2e')  # 暖深灰, 不纯黑
    ox, oy = (sq-w)//2, (sq-h)//2
    canvas.paste(img, (ox, oy))
    draw = ImageDraw.Draw(canvas)
    # 十字准星 (白色)
    cs = max(12, sq//50)
    draw.line([(px-cs,py),(px+cs,py)], fill='#FFFFFF', width=2)
    draw.line([(px,py-cs),(px,py+cs)], fill='#FFFFFF', width=2)
    # 彩色边框 4px
    draw.rectangle([0,0,sq-1,sq-1], outline=color, width=4)
    # 底部标签栏
    draw.rectangle([0,sq-26,sq,sq], fill='#0D111A')
    draw.text((8,sq-20), label, fill=color)
    return canvas.resize((400, 400))
```

### 6.3 Steerable Anchor-Target-Arrow

```python
# Anchor (蓝圈) + Target (红圈) + Arrow (绿线+箭头)
draw.ellipse([ax-r,ay-r,ax+r,ay+r], outline='#4F9CF5', width=3)
draw.ellipse([tx-r,ty-r,tx+r,ty+r], outline='#F87171', width=3)
draw.line([(ax,ay),(tx,ty)], fill='#34D399', width=3)
# 箭头头部
angle = math.atan2(ty-ay, tx-ax)
draw.line([(tx,ty),(tx-al*cos(angle-0.5),ty-al*sin(angle-0.5))], fill='#34D399', width=3)
```

### 6.4 铁律总结

1. **图文分离**: 图片禁止内嵌文字面板，文字必须 HTML 渲染
2. **全图非裁剪**: 样本图展示完整源图，非局部裁剪
3. **每步差异化**: Stage 图必须有独特的视觉符号
4. **彩色优先**: 选 brightness>80, color_score>150 的图片
5. **禁止纯黑底**: 坐标图 (深色底+两个点) = "没有渲染"
6. **正方形 400x400**: 所有样本图统一尺寸
7. **锚点可见**: 锚点必须在图像内 (margin > 15%)

---

## 7. 重述 (Rewrite) 规则

### 7.1 五类 Skill (from rewrite_skill.md)

每个类别的重述必须匹配对应的语言模式:

| 类别 | 核心模式 (regex) |
|------|-----------------|
| Affordance | `designed for\|used to\|suitable for\|intended for` + `holding\|gripping\|cutting\|drinking\|where you would` |
| Spatial Relation | `located\|positioned\|situated` + `relative to\|in the\|below\|above\|front of\|behind` |
| Reasoning | `belonging to\|identified as\|determined by\|which is the\|functioning as` |
| Object Reference | `specific\|prominent\|visible\|identifiable\|recognizable\|distinguishable` |

每个重述必须 `len(rewrite) > len(query) + 10` (充分充实)

### 7.2 Steerable 语言风格

- 192/200 样本包含 "blue point"
- Type 1-3: 必须包含 "blue point"
- Type 4-5: 可使用真实物体名作锚点
- 常见开头: "Use the blue point as...", "Take the blue point as...", "Starting from...", "Beginning at..."
- 常见结尾: "...of it", "...to the right/left of it"

---

## 8. 验证器 (review.md)

### 8.1 检查项

1. **文件完整性**: 所有 `<img src>` 引用文件存在且 > 最小阈值 (mask > 100B, 一般 > 500B)
2. **章节完整性**: 9 个 section 全存在 (hero, motivation, p-overview, p-a, p-b, p-c, attnres, abc, results)
3. **SVG 边界**: 所有 rect 在 viewBox 内 (right < vb_w+2, bottom < vb_h+2)
4. **Stage 图纯图片**: 宽高比 < 1.5 (无合成文字面板)
5. **Stage 行存在**: `.stage-row` >= 4 (图文分离)
6. **分类均衡**: >= 3 个类别覆盖
7. **重述校验**: 每个 rewrite 匹配对应类别 skill 模式
8. **重述充实**: `len(rewrite) > len(query) + 10`
9. **禁止项**: "null (no rewrite needed)", "Coord diagram", "fa55de", viz_gemini_pipeline.png, viz_steerable_pipeline.png
10. **Steerable 全部 JPG**: 10/10 steer_viz 是 JPG 实图 (非 PNG 坐标图)
11. **Pipeline C 语言风格**: 问题匹配 steerable 模式
12. **锚点可见**: 样本锚点 margin > 10%

### 8.2 运行验证

```bash
python -c "
import re, os, json
from PIL import Image
import numpy as np

web = r'C:\Users\Administrator\Desktop\讲解ppt制作\web'
with open(os.path.join(web, 'index.html'), 'r', encoding='utf-8') as f:
    html = f.read()
errors = []

# 执行所有检查...

if errors:
    print(f'FAILED ({len(errors)})')
    for e in errors: print(f'  X {e}')
else:
    print('ALL CHECKS PASSED')
"
```

---

## 9. 避坑指南 (历史碰壁记录)

### 9.1 图片相关

1. **不要用 `fa55de` 图片**: 该样本标签与实物不匹配，网站中永久禁止
2. **不要用纯黑底坐标图**: 深色底 + 两个点的 steerable 图 = "没有渲染"
3. **不要合成文字到 JPEG**: 文字必须 HTML 渲染，不能 Python PIL 写进图
4. **不要 2 列 stage 图**: 每行一张 stage，不是 g2 网格
5. **SAM3 mask PNG 很小 (300B)**: 是正常的，二进制 mask 压缩率高
6. **steerable 源图只渲染一次**: 先下载原始图，再标注。不要在图上有图
7. **锚点在边界外**: 选样本时检查 `y/h` 和 `x/w`，margin < 10% 就换

### 9.2 HTML 编辑

1. **不要用 Bash -c 嵌入长 HTML**: 用 Python 脚本 + `Write` 工具
2. **替换时注意重复 section**: 多次 insert 会产生垃圾代码，用 Python 精确操作
3. **Edit 工具需要精确匹配**: 缩进、空格、换行必须完全一致
4. **section 标签容易丢失**: 替换 HTML 时检查 `<section ` 开头是否完整

### 9.3 Pipeline 展示

1. **Pipeline B 容易缺可视化**: 验证器必须检查 B 有 >= 5 images
2. **Pipeline C 子类型必须展示**: 5 个 task type 都要有样本 (每类 2 个)
3. **Rewrite 不能显示 null**: 必须用 rewrite_skill 生成真实重述
4. **10 样本必须均衡**: 不能 7/10 都是 Object Reference

### 9.4 SVG 拓扑图

1. **底部模块容易太小**: 最小高度 30px, 字体 >= 12px
2. **viewBox 要留余量**: 最大 rect bottom + 10px < vb_h
3. **箭头 marker ID 要唯一**: 不同 SVG 用不同 ID (aa, bb, cc, arx 等)

### 9.5 数据下载

1. **SSH 可能超时**: 大目录遍历加 `--maxdepth`, 单个命令限制 30s
2. **先提取再下载**: 服务器端 python3 提取数据到 /tmp, 再 scp
3. **按 image_path 去重**: 确保 10 个样本 = 10 张不同图片

---

## 10. 本地文件结构

```
web/
├── index.html              # 主网页 (单文件)
├── review.md               # 验证器规则
├── rewrite_skill.md        # 重述技能文档
├── exp.md                  # 经验文档
├── guide.md                # 本文件 (交接文档)
├── balanced_samples.json   # 10 个均衡样本数据
├── samples10.json          # Pipeline A 原始样本
├── qwen_samples.json       # Pipeline B 原始样本
├── accepted_steerable_samples.jsonl  # Pipeline C 全部样本
├── sam3_masks.json         # SAM3 mask 数据
│
├── img00.jpg - img09.jpg   # Pipeline A 源图 (10张)
├── qwen_img00.jpg - qwen_img09.jpg  # Pipeline B 源图 (10张)
├── steer_raw_00.jpg - steer_raw_09.jpg  # Pipeline C 源图 (10张)
│
├── stage0_input.jpg - stage4_rewrite.jpg  # Pipeline A stage 图
├── qwen_stage0_input.jpg - qwen_stage4_rewrite.jpg  # Pipeline B stage 图
│
├── sample01_crop.jpg - sample10_crop.jpg  # Pipeline A 样本正方形
├── qwen_sample01_crop.jpg - qwen_sample10_crop.jpg  # Pipeline B 样本正方形
├── steer_sub_00.jpg - steer_sub_09.jpg    # Pipeline C 样本正方形
│
├── steer_viz01.jpg - steer_viz10.jpg      # Pipeline C 旧版 (保留)
├── steer_img1.jpg, steer_img2.jpg         # 早期 steerable 源图
│
├── sam3_preview.png        # SAM3 预览
├── sam3_overlay.png        # SAM3 叠加
├── sam3_mask_original.png  # SAM3 原始 mask
├── sam3_mask_dilated.png   # SAM3 膨胀 mask
├── sam3_semantic.png       # SAM3 语义图
│
├── arch_attnres.png        # AttnRes 架构图 (已被 SVG 替代)
├── arch_abc_encoding_v4.png # ABC 编码图
│
├── sample_gemini_output.json  # Gemini 样本示例
├── sample_steerable_output.json  # Steerable 样本示例
├── s1.json - s10.json      # 早期 Gemini 样本 (已废弃)
└── qwen_sample1.json, qwen_sample2.json  # 早期 Qwen3 样本
```
