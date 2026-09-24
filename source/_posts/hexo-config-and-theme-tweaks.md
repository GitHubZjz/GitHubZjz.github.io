---
title: Hexo 配置与主题定制踩坑实录：中文菜单、RSS、主题切换与头像处理
date: 2026-09-24 10:00:00
tags: [Hexo, 主题定制, i18n, 踩坑记录]
categories: [技术实践]
---

## 一、背景：搭完博客之后才发现的坑

上一篇讲了从 0 搭建 Hexo 博客的选型和部署规划。但真正开始用的时候才发现，**搭起来只是第一步，配置和主题定制才是花时间的地方**。

这篇文章把实际使用中遇到的几个典型问题记录下来，每个都带根因分析和解决方案，方便以后查阅，也希望能帮到遇到同样问题的人。

涉及的问题：

1. 语言改成中文后，顶栏菜单还是英文
2. RSS 订阅图标点开是空白
3. 主题怎么切换、怎么管理多个主题
4. 切换主题后站点配置不生效
5. 头像图片处理（去水印、裁剪）

---

## 二、中文菜单不生效：一个典型的 i18n 配置链路问题

### 2.1 现象

`_config.yml` 里已经设置 `language: zh-CN`，侧边栏的"归档""最新文章"都变成中文了，但顶栏导航还是 `Home Archives`。

### 2.2 根因分析

追踪模板渲染链路后发现，问题出在两层：

**第一层：模板没走 i18n 翻译函数**

landscape 主题的 `layout/_partial/header.ejs` 里，导航菜单这样渲染：

```ejs
<% for (var i in theme.menu){ %>
  <a class="main-nav-link" href="<%- url_for(theme.menu[i]) %>"><%= i %></a>
<% } %>
```

`<%= i %>` 直接输出 menu 配置的**键名**，没有调用 Hexo 的 `__()` 国际化翻译函数。而主题默认配置里 menu 的键名就是英文：

```yaml
# themes/hexo-theme-landscape/_config.yml
menu:
  Home: /
  Archives: /archives
```

所以无论你怎么改语言，顶栏永远是 `Home Archives`。

**第二层：深合并导致键叠加**

我第一反应是新建 `_config.landscape.yml` 覆盖 menu：

```yaml
menu:
  首页: /
  归档: /archives
```

结果顶栏变成了四个：`Home Archives 首页 归档`。

原因：**Hexo 对 `_config.主题名.yml` 和主题默认 `_config.yml` 做的是深合并（deep merge），不是覆盖**。对于 `menu` 这种字典对象，两个字典的键会叠加，而不是替换。

### 2.3 解决方案

最根本的做法是让模板走 i18n 翻译链路：

**第一步：把主题从 node_modules 复制到 themes/ 目录**

```powershell
Copy-Item -Recurse node_modules\hexo-theme-landscape themes\landscape
```

这样修改不会在 `npm install` 后丢失。

**第二步：改 header.ejs，让 menu 键名走翻译**

```ejs
<!-- 改前 -->
<a class="main-nav-link" href="<%- url_for(theme.menu[i]) %>"><%= i %></a>

<!-- 改后 -->
<a class="main-nav-link" href="<%- url_for(theme.menu[i]) %>"><%= __(i) %></a>
```

`__()` 是 Hexo 的 i18n 翻译函数，会根据当前语言查 `languages/zh-CN.yml`。

**第三步：在 zh-CN.yml 里补上 menu 键名的翻译**

```yaml
# themes/landscape/languages/zh-CN.yml
Home: 首页
Archives: 归档
```

**第四步：清空 `_config.landscape.yml`**，避免深合并叠加：

```yaml
# _config.landscape.yml
# 留空即可，让主题默认配置 + i18n 翻译生效
```

### 2.4 验证

```powershell
npx hexo clean
npx hexo g
```

检查 `public/index.html`，顶栏正确输出：

```html
<a class="main-nav-link" href="/">首页</a>
<a class="main-nav-link" href="/archives">归档</a>
```

### 2.5 小结

| 问题层 | 根因 | 解决 |
|--------|------|------|
| 模板层 | `<%= i %>` 没走 i18n | 改成 `<%= __(i) %>` |
| 配置层 | 深合并导致键叠加 | 不要在 override 里写 menu，靠 i18n 翻译 |

**核心原则：菜单文本应该靠翻译文件切换，而不是靠配置文件覆盖。**

---

## 三、RSS 订阅空白：缺插件

### 3.1 现象

顶栏 RSS 图标点击后 `Cannot GET /atom.xml`，页面空白。

### 3.2 根因

主题默认配置里有 `rss: /atom.xml`，但这个文件没人生成。Hexo 本身不带 RSS 生成器，需要单独装插件。

### 3.3 解决

```powershell
npm install hexo-generator-feed --save
```

在 `_config.yml` 末尾加配置：

```yaml
feed:
  type: atom
  path: atom.xml
  limit: 20
```

重新生成后 `public/atom.xml` 就出现了。浏览器打开会显示 XML 源码，这是正常的，RSS 阅读器能解析它。

### 3.4 隐藏 RSS 图标

如果暂时不想显示 RSS 入口，在主题覆盖配置里把 `rss` 设为空字符串：

```yaml
# _config.landscape.yml
rss: ''
```

`atom.xml` 照常生成，只是顶栏不显示图标。

---

## 四、主题切换：方法与多主题管理

### 4.1 切换步骤

三步走：

```powershell
# 1. 安装主题（npm 或 git clone 二选一）
npm install hexo-theme-butterfly
# 或
git clone https://github.com/xxx/hexo-theme-xxx.git themes/xxx

# 2. 复制到 themes/ 目录（方便自定义）
Copy-Item -Recurse node_modules\hexo-theme-butterfly themes\butterfly

# 3. 改 _config.yml
# theme: butterfly
```

然后清缓存重启：

```powershell
npx hexo clean
npx hexo server
```

### 4.2 多主题并存

themes/ 目录下可以同时放多个主题，切换时只改 `_config.yml` 的 `theme` 字段即可：

```
themes/
  ├── landscape/   # 默认主题
  ├── butterfly/   # Card UI 风格
  ├── next/        # 经典优雅
  ├── yun/         # 轻盈可爱
  └── chirpy/      # 极简风格
```

### 4.3 注意事项

**切换主题后，之前对旧主题的修改不会自动迁移。** 每个主题的模板结构和语言文件都不一样，需要针对新主题重新做中文适配。

不过像 Butterfly、NexT、Chirpy 这些主流主题本身内置完善的中文支持，切过去大概率已经是中文。

---

## 五、主题配置覆盖机制：三层配置的优先级

### 5.1 现象

切换到 chirpy 主题后，`_config.yml` 里设置的 `author: 曾志军`、`subtitle: 运气不是...` 全部不生效，页面上显示的是主题默认的 `Your Name`、`Sharing thoughts and knowledge`。

### 5.2 根因

Hexo 模板里有两个配置作用域：

| 作用域 | 来源 | 模板里怎么访问 |
|--------|------|---------------|
| 站点配置 | `_config.yml` | `config.xxx` |
| 主题配置 | `themes/主题名/_config.yml` | `theme.xxx` |

chirpy 模板用的是 `theme.xxx || config.xxx` 模式——**主题配置优先，主题配置为空时才回退到站点配置**。

问题在于 chirpy 主题的默认值几乎全是非空的：

```yaml
# themes/chirpy/_config.yml
author: "Your Name"           # 非空，遮住了 config.author
subtitle: "Sharing thoughts..." # 非空，遮住了 config.subtitle
```

注释说"Fallbacks to config.xxx if empty"，但值根本没设为空。

### 5.3 解决方案：用主题覆盖配置文件

Hexo 支持在站点根目录创建 `_config.主题名.yml`，它会**深合并覆盖**主题默认配置。

创建 `_config.chirpy.yml`，把冲突项清空：

```yaml
# _config.chirpy.yml
# 空字符串表示让模板回退到站点 _config.yml 的值
title: ""
subtitle: ""
description: ""
author: ""
language: ""
timezone: ""
url: ""

# 定制项直接写中文值
pagination:
  per_page: 10
  prev_text: "上一页"
  next_text: "下一页"

about:
  role: "软件工程师"
  description: "运气不是天降好运，是主动思考、持续行动，亲手创造的运气。"
```

### 5.4 三层配置覆盖关系

```
_config.yml (站点)            ← config.xxx
    ↓ 回退
themes/chirpy/_config.yml     ← theme.xxx （优先级最高）
    ↓ 被覆盖（deep merge）
_config.chirpy.yml            ← 你写的覆盖文件
```

### 5.5 核心原则

**不要直接改 `themes/主题名/_config.yml`**，原因：

1. 主题升级时改动会丢
2. 用 git pull 更新主题时会产生冲突
3. 自己改了哪些配置不好追踪

正确做法是**只在 `_config.主题名.yml` 里写覆盖项**，主题原文件保持不动。

---

## 六、头像处理：去水印与裁剪

### 6.1 需求

用 AI 生成的头像右下角带"豆包AI生成"水印，需要去掉水印，裁剪成方形，缩放到合适尺寸。

### 6.2 方案

用 Python + Pillow 处理：

```python
from PIL import Image, ImageFilter

img = Image.open("avatar.png").convert("RGB")
W, H = img.size

# 1. 去水印：右下角区域用上方像素填充
wm_w = int(W * 0.22)
wm_h = int(H * 0.14)
wm_x0, wm_y0 = W - wm_w, H - wm_h
src_h = int(wm_h * 1.8)
src_y0 = max(0, wm_y0 - src_h)
patch = img.crop((wm_x0, src_y0, W, wm_y0))
patch = patch.resize((wm_w, wm_h), Image.LANCZOS)
patch = patch.filter(ImageFilter.GaussianBlur(radius=1.5))
img.paste(patch, (wm_x0, wm_y0))

# 2. 裁剪正方形居中
w, h = img.size
side = min(w, h)
cx, cy = w // 2, h // 2
img = img.crop((cx - side // 2, cy - side // 2, cx + side // 2, cy + side // 2))

# 3. 缩放到 512x512
img = img.resize((512, 512), Image.LANCZOS)

# 4. 保存
img.save("avatar.jpg", "JPEG", quality=92, optimize=True)
```

### 6.3 一个关键发现

处理完图片后发现，chirpy 主题的 CSS 里头像被 `border-radius: 50%` 裁成了圆形。这意味着：

- 图片方形区域**角上的内容在网页上看不到**
- 如果水印在角上，**不处理也看不到**

所以处理图片前先看主题 CSS 怎么显示头像，避免做无用功。

### 6.4 配置头像路径

把图片放到 `themes/chirpy/source/images/avatar.jpg`，Hexo 会自动把它复制到 `public/images/`。

在 `_config.chirpy.yml` 里配置：

```yaml
avatar: "/images/avatar.jpg"
```

---

## 七、总结：几个值得记住的原则

| 原则 | 说明 |
|------|------|
| **不要改 node_modules 或 themes 原文件** | 用 `_config.主题名.yml` 做覆盖，升级不丢改动 |
| **i18n 翻译靠翻译文件，不靠配置覆盖** | 菜单文本应该走 `__()` 函数，不是在配置里写中文键名 |
| **深合并是默认行为** | 字典类型的配置会叠加而非替换，覆盖时要注意 |
| **先看 CSS 再处理图片** | 头像被 `border-radius: 50%` 裁切的话，角上的水印本来就看不出来 |
| **插件要单独装** | RSS、sitemap、搜索等功能都需要对应的 generator 插件 |

## 八、涉及的工具和插件

| 工具/插件 | 用途 |
|-----------|------|
| `hexo-generator-feed` | 生成 RSS/Atom 订阅文件 |
| `hexo-renderer-pug` | pug 模板渲染（Butterfly、NexT 需要） |
| `hexo-renderer-sass` | sass 样式渲染 |
| `Pillow` (Python) | 图片处理（去水印、裁剪、缩放） |

## 九、当前主题配置结构

```
d:\projects\blog\
  ├── _config.yml              # 站点主配置
  ├── _config.chirpy.yml       # chirpy 主题覆盖配置
  ├── themes/
  │   ├── chirpy/              # 当前启用主题
  │   ├── landscape/           # 备用
  │   ├── butterfly/           # 备用
  │   ├── next/                # 备用
  │   ├── yun/                 # 备用（给小孩留的）
  │   └── gstyle/              # 备用
  └── source/
      └── _posts/
          ├── hexo-blog-from-zero.md         # 第一篇：选型与部署
          └── hexo-config-and-theme-tweaks.md # 本篇：配置与定制
```

下一篇会写部署上线：GitHub Pages 配置、自定义域名、HTTPS、CI 自动部署的完整流程。
