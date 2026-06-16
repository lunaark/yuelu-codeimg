---
name: yuelu-codeimg
description: 代码控图——用 SVG/HTML 线框精确控制布局，再交给 GPT Image 2 上色渲染。适用于长图、海报、电商详情页、技能展示图等需要精确布局控制的图片生成场景。当用户说「代码控图」「代码画图」「线框上色」「SVG 生图」时触发。
---

# 代码控图 Skill

## 核心理念

**代码是画布，AI 是画笔。**

用 SVG/HTML 代码精确锁定布局（文字位置、元素尺寸、间距比例），再交给 GPT Image 2 进行风格化上色渲染。代码控制「在哪」，AI 控制「好看」。

优势：
- 布局精确可控，不会出现 AI 随意挪位置的问题
- 多屏长图风格统一，每屏独立可调
- 可复现、可迭代，改文案不用重新设计

## 前置依赖

- Chrome DevTools MCP（截图用）
- GPT Image 2 API（上色用）
- Python PIL（拼接用）

## 工作流

### 第零步：和用户沟通需求

**触发后第一件事是收集信息，不要直接开画。** 根据用户的场景主动引导：

#### 需要确认的信息

| 信息 | 说明 | 如何获取 |
|------|------|----------|
| **用途** | 海报？长图？详情页？技能展示？ | 直接问 |
| **文案内容** | 标题、正文、卖点等文字 | 用户提供 或 AI 根据主题生成后给用户确认 |
| **参考图/风格** | 有没有喜欢的设计风格、配色参考 | 用户截图/描述；没有则 AI 生成一张「风格参考图」并确认 |
| **产品图/素材** | 如需展示具体产品，是否有现成图片 | 用户上传；没有则 AI 生成 |
| **尺寸/屏数** | 单张还是多屏长图 | 根据内容量建议，让用户确认 |
| **配色偏好** | 有无品牌色、喜欢的色调 | 用户说明 或 根据品类自动推荐 |

#### 沟通话术参考

- "你想做什么类型的图？（海报/长图/详情页/其他）"
- "有现成的文案吗？没有的话我帮你写，你确认后再出图"
- "有产品图或素材图吗？有的话直接发我就行；没有也没关系，AI 会自动帮你生成"
- "有喜欢的风格或配色参考吗？有就发我；没有的话我先生成一张风格参考图，你确认后我按这个风格批量出图，保证多屏统一"

**关键：每个问题都要让用户知道「没有也行，AI 会帮你搞定」，降低使用门槛。**

**原则：用户给的越多，出图越精准；用户什么都没有，我们全帮他生成，但每步都确认。**

### 第一步：确立风格参考图（风格基准图）

**批量分屏出图前，先确立一张「风格参考图」作为整组图的视觉基准。** 所有屏的上色都向它对齐，这是多屏风格统一的关键——没有它，每屏各上各的色，拼起来必然花。

#### 来源

| 情况 | 做法 |
|------|------|
| 用户有风格参考 | 用户上传截图/图片，直接作为风格参考图（`style-ref.png`） |
| 用户没有风格参考 | **AI 自动生成一张**，给用户确认后再批量出图 |

#### AI 生成风格参考图

根据需求（品类、配色、调性）用 GPT Image 2 generations 接口生成一张风格样板：

```bash
curl -s -m 120 -X POST "https://www.hfsyapi.cn/v1/images/generations" \
  -H "Authorization: Bearer $IMAGE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-image-2",
    "prompt": "<风格描述：配色、光影、质感、插画/摄影风格、字体气质>，作为整组长图的视觉基准样板",
    "size": "1024x1024"
  }' | python3 -c "import sys,json,base64;d=json.load(sys.stdin)['data'][0];open('style-ref.png','wb').write(base64.b64decode(d['b64_json']) if 'b64_json' in d else __import__('urllib.request',fromlist=['x']).urlopen(d['url']).read())"
```

保存为 `style-ref.png`。这张图**不需要承载真实文案或精确布局**，只用来锁定「风格长什么样」——配色、光影、材质、留白、字体感觉。

**生成后必须给用户确认**："这是我定的风格基准，满意我就按这个风格批量出图；不满意我重新调。" 风格基准一旦定下，后面每屏都不再单独决定风格。

#### 提炼风格描述（复用前缀）

确认风格参考图后，提炼一段「风格描述」文字存为 `style-ref.md`，后续每屏上色提示词都复用这段固定前缀：

- **配色**：主色 / 辅色 / 背景色（具体色值）
- **光影**：柔光 / 硬光 / 方向 / 强度
- **质感**：扁平 / 拟物 / 颗粒感 / 通透
- **风格**：写实摄影 / 扁平插画 / 3D 渲染 / 国风等
- **字体气质**：现代无衬线 / 手写 / 衬线

图片基准（`style-ref.png`）+ 文字基准（`style-ref.md`）双管齐下，风格才锁得稳。

### 第二步：写 SVG/HTML 线框

每一屏写一个独立的 HTML 文件，内部用 SVG 精确定位所有元素。

**标准模板：**

```html
<!doctype html>
<html lang="zh-CN">
<head>
<meta charset="utf-8">
<style>
body{margin:0;background:#fff;display:flex;align-items:center;justify-content:center;min-height:100vh}
.wrap{width:1080px;height:1440px;position:relative}
</style>
</head>
<body>
<div class="wrap">
<svg viewBox="0 0 1080 1440" width="1080" height="1440" xmlns="http://www.w3.org/2000/svg">
  <!-- 背景 -->
  <rect width="1080" height="1440" fill="#FFFAF3"/>
  <!-- 布局元素 -->
</svg>
</div>
</body>
</html>
```

**常用尺寸（宽度统一 1080px）：**

| 用途 | 高度 | viewBox |
|------|------|---------|
| 标准内容屏 | 1440px | 0 0 1080 1440 |
| 内容密集屏（FAQ、多卡片） | 1920px | 0 0 1080 1920 |
| CTA/收尾屏 | 1080px | 0 0 1080 1080 |
| 首屏/封面 | 1440px | 0 0 1080 1440 |

**线框规则：**
- 用 `<rect>` 做区域占位，用 `stroke-dasharray` 虚线框标记图片占位区
- 用 `<text>` 写实际文案（GPT 上色会保留文字内容）
- 用 `fill` 标记颜色倾向（GPT 会参考但不完全照搬）
- 相邻屏的边缘颜色要匹配，避免拼接色差

### 第三步：截图

用 Chrome DevTools MCP 打开 HTML 文件并全页截图：

```
navigate_page → file:///path/to/screen.html
take_screenshot → fullPage: true, filePath: /path/to/screen-wireframe.png
```

### 第四步：压缩 + GPT 上色

```bash
# 压缩到 1024px 以内（API 限制）
sips -s format jpeg -Z 1024 -s formatOptions 70 screen-wireframe.png --out /private/tmp/small.jpg

# 风格参考图也压缩一份
sips -s format jpeg -Z 1024 -s formatOptions 70 style-ref.png --out /private/tmp/style-ref.jpg

# 调用 GPT Image 2 edits API —— 同时传入线框 + 风格参考图（多图输入）
curl -s -m 120 -X POST "https://www.hfsyapi.cn/v1/images/edits" \
  -H "Authorization: Bearer $IMAGE_API_KEY" \
  -F model=gpt-image-2 \
  -F image[]=@/private/tmp/small.jpg \      # 第一张：线框，控布局 \
  -F image[]=@/private/tmp/style-ref.jpg \  # 第二张：风格参考图，控风格 \
  -F prompt="<上色提示词>" \
  -F size=<匹配尺寸>
```

**带风格参考图上色（保证多屏一致）：** 把 `style-ref.jpg` 随线框一起传入 edits 接口，提示词里明确「保留第一张图的布局和所有中文文字，按第二张图的配色、光影、质感和整体风格上色」。每屏都引用同一张风格参考图，整组长图风格就稳。

> 若中转接口不支持多图输入（`image[]`），退化为单图：只传线框，但把第一步提炼的「风格描述」`style-ref.md` 写进每屏提示词的固定前缀，靠文字层面对齐风格。

**size 选择：**

| 线框比例 | API size 参数 |
|---------|--------------|
| 1080×1440（3:4） | 1024x1536 |
| 1080×1920（9:16） | 1024x1536 |
| 1080×1080（1:1） | 1024x1024 |

**上色提示词写法要点：**
- 固定前缀复用 `style-ref.md` 的风格描述（每屏一字不差，这是风格统一的根）
- 描述目标风格（如"professional food photography"、"flat illustration"）
- 指明关键元素要保留（如"keep all Chinese text exactly"）
- 指明背景色调（确保相邻屏衔接）
- 如有产品图一致性需求，每屏提示词中描述同一产品外观
- 有风格参考图时，提示词明确「按第二张参考图的风格上色」

**API 返回处理：**
```python
import json, base64, urllib.request
r = json.load(sys.stdin)
d = r['data'][0]
if 'b64_json' in d:
    img = base64.b64decode(d['b64_json'])
elif 'url' in d:
    img = urllib.request.urlopen(d['url']).read()
```

### 第五步：拼接长图

```python
from PIL import Image

images = [Image.open(f) for f in file_list]
target_w = images[0].width

resized = []
for img in images:
    if img.width != target_w:
        new_h = int(img.height * target_w / img.width)
        img = img.resize((target_w, new_h), Image.LANCZOS)
    resized.append(img)

total_h = sum(img.height for img in resized)
result = Image.new('RGB', (target_w, total_h), (255, 255, 255))

y = 0
for img in resized:
    result.paste(img, (0, y))
    y += img.height

result.save('output.png', quality=95)
```

## 关键经验

### 屏间衔接
相邻屏的顶部/底部背景色必须匹配。如果 A 屏底部是 `#FFFAF3`，B 屏顶部渐变也要从 `#FFFAF3` 开始。线框阶段就要规划好。

### 风格一致性（批量分屏的核心）
批量出多屏时，风格统一靠的不是「每屏都写得很像」，而是**所有屏对齐同一张风格参考图**：
1. 出图前先确立 `style-ref.png`（用户给的，或 AI 生成后用户确认的）。
2. 每屏上色都把 `style-ref.png` 作为第二张图传入 edits 接口，提示词固定前缀复用 `style-ref.md` 的风格描述。
3. 风格基准定下后不要中途换——某一屏想微调风格，先改基准图，再整组重出，别只改一屏。
4. 拼接后若发现某屏跳脱，多半是它没引用风格参考图或提示词前缀漏了，补上重出该屏即可。

### 产品一致性
多屏出现同一产品时：
1. 先单独生成一张产品主图
2. 在每屏的上色提示词中用相同文字描述产品外观
3. 关键特征要具体（"glass bottle with white cap, golden orange juice inside"）

### 批量加速
多屏可以并行处理——每屏的截图、压缩、API 调用互不依赖，用 `run_in_background` 并行发送。

### 不要在线框里放过渡文字
类似"▼ 下一篇"的过渡文字会在拼接后显得多余，只在最后一屏放收尾信息。

## 配置

读取 `config.env` 获取 API 密钥。
