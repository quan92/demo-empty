# 实现报告：烟花动画 HTML 页面（wi-t1000374）

## 变更行为与实现依据

按已批准需求基线（`docs/wi-t1000374/requirements.md`，sha256:0b1ff9b6e1d07775e7a951b3b603bcb7d88b507df1df0b0b46ba164b6ddcfa4b）与架构基线（`docs/wi-t1000374/architecture-baseline.md`，sha256:c025a7c27d107bff06ed317cdd64f2867a1c855e065d6b8e78c752e52054bd66，simple-frontend 精简路线），交付单文件烟花动画页面：浏览器打开后无需任何操作即自动持续播放烟花动画。

## 实现范围

- 新增 `index.html`（工作区根目录，单文件，CSS 与 JavaScript 全部内联，零外部依赖、零网络请求、零持久化存储）。
- 未修改其他任何源码；无构建步骤、无后端、无测试工程（纯单文件静态页面，无可单测的独立业务模块）。

## 调用链 / 运行机制

1. 页面加载 → 内联 IIFE 脚本执行 → `resize()` 将 `<canvas id="sky">` 尺寸对齐视口并监听 `window.resize` 重设；
2. 启动时立即发射 3 发烟花弹（0ms / 400ms / 900ms），保证打开页面即可见动画；
3. `requestAnimationFrame(step)` 驱动主循环：
   - 每帧以半透明深色矩形覆盖画布形成余晖拖尾，并用 `lighter` 混合模式叠加发光；
   - 按随机节奏（35~80 帧）持续发射新烟花弹，约 35% 概率双发；
   - 烟花弹受重力上升，速度转为向下时调用 `explode()` 绽放为 70~130 个彩色粒子（主色 + 25% 辅色）向外扩散；
   - 粒子在重力与速度衰减作用下透明度递减，`alpha <= 0` 后移除；
4. 循环不停止，动画在观察期间持续播放。

## 架构基线符合性

- 单 HTML 文件、canvas 2D + `requestAnimationFrame`：与基线一致；
- 无路由、无表单、无交互控件、无用户输入：与基线一致；
- 不使用 localStorage / Cookie / IndexedDB / 网络请求，运行期内存对象列表随页面生命周期存在：与基线一致；
- 未扩展产品语义（无声音、无暂停/点击发射等未规定能力）。

## 开发侧验证证据

| checkId | 内容 | 命令 / 动作 | 工作目录 | 状态 | 退出码 |
| --- | --- | --- | --- | --- | --- |
| inline-script-syntax | 提取 `index.html` 内联 `<script>` 内容做 JavaScript 语法检查 | `sed -n '/^<script>$/,/^<\/script>$/p' index.html \| sed '1d;$d' > /tmp/wi-t1000374-fireworks.js && node --check /tmp/wi-t1000374-fireworks.js` | 工作区根目录 | passed（输出 SYNTAX_OK） | 0 |
| static-serve-reachable | 以静态服务器托管工作区根目录并请求页面 | `python3 -m http.server 4173 --directory <工作区根目录>`（注册后台作业 9b6beb48-e5bf-434c-91b3-46b399c80ae5）；`curl -s -o /dev/null -w "%{http_code} %{size_download}" http://localhost:4173/index.html` | 工作区根目录 | passed（HTTP 200，返回 4146 字节） | 0 |

## 应用预览地址（QA / 验收用）

- 预览 URL：`http://localhost:4173/index.html`（静态服务已由注册后台作业启动并确认 200 响应）；等效启动命令：`python3 -m http.server 4173 --directory <工作区根目录>`。
- 亦可按需求基线以 `file://` 方式直接在浏览器打开 `index.html`。
- 无需登录、无测试数据准备、无隔离配置。

## AC 覆盖状态（开发侧）

| AC | 开发侧状态 | 说明 |
| --- | --- | --- |
| AC-01 打开页面自动出现持续烟花动画 | 代码已实现，待 QA runtime-render 验收 | 实现阶段不执行浏览器 E2E / 视觉验收；页面加载即自动启动动画循环的代码路径已交付并通过语法与静态可达性自检。QA 验收动作：打开预览 URL，观察无需操作即出现烟花升空-绽放-消散且动画持续播放。 |

## 遗留风险

- 动画的视觉表现（色彩、绽放形态、流畅度）与"持续播放"结论需 QA 在浏览器中实际观察确认；本阶段仅提供代码实现与静态自检证据。
- 极低性能设备上粒子数量（单帧最多约数百个）可能影响帧率，需求未规定性能指标，未做额外优化。
