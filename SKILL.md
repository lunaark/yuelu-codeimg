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
| **参考图/风格** | 有没有喜欢的设计风格、配色参考 | 让用户截图或描述 |
| **产品图/素材** | 如需展示具体产品，是否有现成图片 | 用户上传；没有则 AI 生成 |
| **尺寸/屏数** | 单张还是多屏长图 | 根据内容量建议，让用户确认 |
| **配色偏好** | 有无品牌色、喜欢的色调 | 用户说明 或 根据品类自动推荐 |

#### 沟通话术参考

- "你想做什么类型的图？（海报/长图/详情页/其他）"
- "有现成的文案吗？没有的话我帮你写，你确认后再出图"
- "有产品图或素材图吗？有的话直接发我就行；没有也没关系，AI 会自动帮你生成"
- "有喜欢的风格或配色参考吗？没有的话我根据内容自动推荐一套配色"

**关键：每个问题都要让用户知道「没有也行，AI 会帮你搞定」，降低使用门槛。**

**原则：用户给的越多，出图越精准；用户什么都没有，我们全帮他生成，但每步都确认。**

### 第一步：写 SVG/HTML 线框

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

### 第二步：截图

用 Chrome DevTools MCP 打开 HTML 文件并全页截图：

```
navigate_page → file:///path/to/screen.html
take_screenshot → fullPage: true, filePath: /path/to/screen-wireframe.png
```

### 第三步：压缩 + GPT 上色

```bash
# 压缩到 1024px 以内（API 限制）
sips -s format jpeg -Z 1024 -s formatOptions 70 screen-wireframe.png --out /private/tmp/small.jpg

# 调用 GPT Image 2 edits API
curl -s -m 120 -X POST "https://www.hfsyapi.cn/v1/images/edits" \
  -H "Authorization: Bearer $IMAGE_API_KEY" \
  -F model=gpt-image-2 \
  -F image=@/private/tmp/small.jpg \
  -F prompt="<上色提示词>" \
  -F size=<匹配尺寸>
```

**size 选择：**

| 线框比例 | API size 参数 |
|---------|--------------|
| 1080×1440（3:4） | 1024x1536 |
| 1080×1920（9:16） | 1024x1536 |
| 1080×1080（1:1） | 1024x1024 |

**上色提示词写法要点：**
- 描述目标风格（如"professional food photography"、"flat illustration"）
- 指明关键元素要保留（如"keep all Chinese text exactly"）
- 指明背景色调（确保相邻屏衔接）
- 如有产品图一致性需求，每屏提示词中描述同一产品外观

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

### 第四步：拼接长图

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
