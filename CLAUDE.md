# 隔空写字 - 手势交互系统

> 单文件 HTML 应用，通过摄像头 + MediaPipe 实现隔空手势写字、缩放、消除。

## 项目结构

```
shouzhixiezi/
├── air-writing.html   # 主程序（单文件，1118 行）
├── CLAUDE.md          # 本文件
└── .git/              # Git 仓库
```

## 技术栈

- **MediaPipe Hands** (`@mediapipe/hands@0.4`) — 手部 21 点关键点检测，最多 2 只手
- **MediaPipe Camera Utils** (`@mediapipe/camera_utils@0.3`) — 摄像头帧采集
- **Canvas 2D API** — 绘制 + 粒子系统，DPR 自适应高分辨率
- **纯 HTML/CSS/JS** — 无框架，零依赖构建工具

## 核心架构

### 坐标系统

```
世界坐标 ←→ 屏幕坐标
   ↓             ↓
 存储笔画    渲染 + 手势输入
```

- `screenToWorld(sx, sy)` — 屏幕像素 → 世界坐标（÷scale - offset）
- `worldToScreen(wx, wy)` — 世界坐标 → 屏幕像素（×scale + offset）
- `landmarkToScreen(lm)` — MediaPipe 归一化坐标 → 屏幕像素
  - 已处理视频 `object-fit: cover` 的缩放偏移（`getVideoDisplayRect()`）
  - x 轴翻转适配镜像摄像头

### 手势检测 (`detectGestures`)

| 手势 | 判定逻辑 | 触发功能 |
|------|---------|---------|
| 张开手掌 | 食指+中指+无名指+小指全部伸展 | 打开/关闭工具栏 |
| 食指指向 | 仅食指伸展，其余收拢 | 工具栏悬停选择 |
| 拇指+食指捏合 | 拇指尖与食指尖距离 < 0.055 | 隔空写字/擦除 |
| 食指+中指竖起 | 食指+中指伸展，其余收拢，指尖距 > 0.05 | 雪花消散 |
| 双手食指 | 两只手均为食指指向 | 缩放 |

### 状态机 (`STATE` 对象)

```
工具: pen / eraser
颜色: 7 种预设
笔刷: 3/6/12 px
视图: scale + offsetX/Y
笔画: strokes[] → {points, color, size, tool}
粒子: particles[] → {x, y, vx, vy, life, decay, ...}
UI: uiVisible, hoveredBtn, DWELL_MS=800
```

### 关键流程

1. **页面加载** → `waitForMediaPipe()` 轮询 `window.Hands` → `init()` 初始化摄像头和模型
2. **每帧处理** → Camera → MediaPipe Hands → `processHands(results)` → 手势判断 → 更新状态
3. **粒子动画** → `requestAnimationFrame(animate)` → `updateParticles()` 每帧更新
4. **缩放** → 双食指 → `startScaling` → `updateScaling` 每帧重绘 → 松手锁定

## API 端点（无后端）

所有逻辑在浏览器端运行，无后端服务。依赖两个外部 CDN：

```
https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils@0.3.1675466862/camera_utils.js
https://cdn.jsdelivr.net/npm/@mediapipe/hands@0.4.1675469240/hands.js
```

## 部署

- **平台**：GitHub Pages
- **仓库**：`github.com/dashuaibican/air-writing`
- **线上地址**：`https://dashuaibican.github.io/air-writing/air-writing.html`
- **HTTPS**：必须（`getUserMedia` 要求）
- **推送**：`git push origin master` 即可更新

## 修改注意事项

1. CDN 版本与 `locateFile()` 路径必须一致，否则 WASM/模型文件加载失败
2. 摄像头分辨率影响 MediaPipe 性能——当前 `ideal: 1280x720`，降低可提速
3. `isFingerExtended()` 用 `tip.y < pip.y - 0.02` 判断，依赖手部朝上姿态
4. 粒子采样步长自适应（`totalPixels > 4M → step=5`），防止大画布卡顿
5. 工具栏关闭后 `palmFrames = -25` 冷却期，值过大/过小影响体验
6. `object-fit: cover` 与 `getVideoDisplayRect()` 必须同步（CSS 和 JS 都用 cover）

## 测试方法

```bash
# 本地开发
直接浏览器打开 air-writing.html
# 需要允许摄像头权限

# 线上验证
open https://dashuaibican.github.io/air-writing/air-writing.html
```

- 键盘快捷键：`C` 清屏 `P` 笔 `E` 橡皮 `F` 全屏 `H` 切换提示
- 按 `F12` 打开控制台查看 MediaPipe 日志
