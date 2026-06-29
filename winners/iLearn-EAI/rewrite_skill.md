# Rewrite Skills — PointArena 5-Category Query Patterns

从原始数据集 5 类问题中提炼的每个类别的语言特征和重述规则。

---

## 1. Affordance (工具功能性使用)

**原始数据特征**:
- 简短命名工具/器具: "Point to the knife." "Point to the microphone."
- 目标由人类手部操作的通用工具定义
- 不包含使用场景描述（那是 Reasoning）

**重述规则 (rewrite_skill)**:
1. 保留 "Point to the" 开头
2. 必须加入功能性描述: "designed for", "used to", "suitable for", "intended for"
3. 加入交互动作: "holding", "gripping", "cutting", "drinking", "typing", "pressing"
4. 加入使用场景: "where you would", "that a person uses to"
5. 目标对象保持简短, 功能性描述加在后面

**模板**: `Point to the [object] — the [tool/instrument] designed for [functional use], where a person would [interaction action].`

**示例**:
- "Point to the knife." → "Point to the kitchen knife — the cutting tool designed for slicing and chopping, where you would grip the handle."
- "Point to the microphone." → "Point to the microphone — the audio device used to capture and amplify sound, suitable for speaking into."

---

## 2. Spatial Relation (空间几何关系)

**原始数据特征**:
- 包含方位词: "bottom left", "bottom right", "above", "on right side", "background"
- "Point to the [position] [object]."
- 空间锚点: "bottom left counter area", "bare elbow on right side"

**重述规则 (rewrite_skill)**:
1. 保留 "Point to the" 开头
2. 必须保留空间方位词 (left, right, above, below, between, nearest, farthest)
3. 加入空间锚点描述: "located", "positioned", "situated"
4. 加入参照物: "relative to", "with respect to", "in relation to"
5. 保持几何关系清晰

**模板**: `Point to the [object] [spatial_phrase] — located [position] relative to the [reference_object] in the scene.`

**示例**:
- "Point to the bottom left counter area." → "Point to the bottom left counter area — the surface region positioned in the lower-left quadrant of the kitchen scene."
- "Point to the bare area in front of rabbit left eye." → "Point to the bare area located directly in front of the rabbit's left eye, positioned relative to the animal's face."

---

## 3. Reasoning (语义推理/世界知识)

**原始数据特征**:
- 部分-整体关系: "red part of thermometer", "statue base"
- 物理状态: "shadow"
- 计数/识别: "one arm", "right hand"
- 需要背景知识推理

**重述规则 (rewrite_skill)**:
1. 保留 "Point to the" 开头
2. 加入因果/状态/部分-整体描述: "belonging to", "attached to", "casting by", "supporting"
3. 加入世界知识线索: "which is the", "that serves as", "functioning as"
4. 加入推理过程暗示: "based on", "determined by", "identified through"
5. 必须保留需要推理才能定位的认知瓶颈

**模板**: `Point to the [object] — the [part/feature] belonging to [whole], identified through [reasoning_cue].`

**示例**:
- "Point to the shadow." → "Point to the shadow — the dark area cast by the object blocking the light source, identified through understanding of illumination."
- "Point to the head." → "Point to the head — the uppermost body part of the person, identified as the anatomical structure containing the face and brain."

---

## 4. Object Reference (精确对象引用)

**原始数据特征**:
- 直接命名: "Point to the wall.", "Point to the right eye.", "Point to olive oil."
- UI/文本元素: "Point to the Red Arrpw."
- 细粒度定位: "Point to the top of item."

**重述规则 (rewrite_skill)**:
1. 保留 "Point to the" 开头
2. 加入精确描述符: "specific", "exact", "precise", "particular"
3. 加入区分特征: "distinguished from", "identifiable by", "recognizable as"
4. 加入视觉线索: "visible", "shown", "depicted", "displayed"
5. 保持对象名称为核心

**模板**: `Point to the [object] — the specific [object_type] shown in the image, identifiable by its [distinguishing_feature].`

**示例**:
- "Point to the right eye." → "Point to the right eye — the specific facial feature on the person's right side, distinguishable from the left eye by its position."
- "Point to the DOG NOSE." → "Point to the dog's nose — the prominent sensory organ visible on the animal's face, identifiable by its central position and dark coloration."

---

## 5. Counting (多实例枚举, Pipeline B 专用)

**原始数据特征**:
- 复数/不可数目标: "water", "Grass"
- 隐含多实例: "Point to all [X]"

**重述规则 (rewrite_skill)**:
1. 保留 "Point to all" 开头 (如果有多实例语言) 或 "Point to the" (单数目标)
2. 加入枚举提示: "each instance of", "every occurrence of", "all visible"
3. 保留目标的集合性质

**模板**: `Point to all [object_plural] — every visible instance of [object] present in the image.`
