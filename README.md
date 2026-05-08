# 🖋️ 小说写作助手

AI 驱动的跨平台小说创作管理系统。

**技术栈**：Flutter (客户端) + FastAPI (AI 代理) + Go (认证/同步) + SQLite (本地存储) + PostgreSQL (云端)

## Windows 使用指南

### 🌐 浏览器版（即开即用）

### 一行命令启动

```powershell
cd F:\AI-WORK\novel-writer-source
flutter create --platforms web .   # 添加 Web 支持（仅首次）
flutter run -d chrome               # 在浏览器中运行
```

启动后浏览器自动打开，**不需要任何安装**，Chrome/Edge/Safari 都可以。

### Web 版特点

| 事项 | 说明 |
|------|------|
| ✅ 全功能可用 | 编辑器、章节管理、角色、世界观、时间线 |
| ✅ AI 功能 | 需启动 Python 后端（`backend/ai_service`） |
| ✅ 离线可用 | 数据存储在浏览器 IndexedDB |
| ✅ PWA 支持 | 可安装到桌面，像原生 App 一样使用 |
| ✅ 导出功能 | TXT/Markdown/EPUB 均可 |

> **注意**：Web 版使用 `sqflite_common_ffi_web` 替代本地 SQLite，首次加载稍慢，但功能一致。

### 构建静态网站

```powershell
flutter build web --release
# 输出到 build/web/，可直接部署到任何 Web 服务器
# 例如：GitHub Pages、Vercel、Nginx 等
```

---

## 1. 环境准备

你需要安装以下工具：

```powershell
# 1. 安装 Flutter SDK
# 下载 https://flutter.dev/docs/get-started/install/windows
# 解压到 C:\flutter，然后将 C:\flutter\bin 添加到系统 PATH

# 2. 安装 Visual Studio 2022
# 下载 https://visualstudio.microsoft.com/
# 安装时勾选 "使用 C++ 的桌面开发" 工作负载

# 3. 验证环境
flutter doctor

# 应看到所有项为绿色 ✔
# 如果缺 Android SDK，安装 Android Studio 即可
```

### 2. 运行桌面版（推荐）

```powershell
# 进入项目目录
cd F:\AI-WORK\novel-writer-source

# 获取依赖
flutter pub get

# 运行 Windows 桌面版
flutter run -d windows
```

启动后会看到：
1. **启动画面** → 自动跳转到项目列表
2. **新建项目** → 输入小说名称和描述
3. **写作工作台** → 三栏布局：章节树 | 编辑器 | 设定面板

### 3. 启动 AI 服务（可选）

如果需要 AI 辅助功能（续写/润色/一致性检查）：

```powershell
# 方式一：启动本地 AI 服务
cd backend\ai_service
python -m venv venv
venv\Scripts\activate
pip install -r requirements.txt
uvicorn main:app --reload --port 8080

# 方式二：直接使用本地 Ollama（推荐）
# 1. 下载安装 https://ollama.com
# 2. 拉取模型
ollama pull qwen2.5:7b
# 3. 配置 .env，Ollama 无需 API Key
```

### 4. 构建 Android APK

```powershell
# 需安装 Android Studio + Android SDK

# Debug 包（无需签名，直接可用）
flutter build apk --debug
# 输出：build\app\outputs\flutter-apk\app-debug.apk

# Release 包（需签名）
keytool -genkey -v -keystore android\app\upload-keystore.jks ^
  -alias upload -keyalg RSA -keysize 2044 -validity 10000
copy android\app\key.properties.template android\app\key.properties
# 编辑 key.properties，填入密码
flutter build apk --release
# 输出：build\app\outputs\flutter-apk\app-release.apk
```

### 5. 功能一览

| 功能 | 说明 |
|------|------|
| 📝 富文本编辑器 | flutter_quill，支持格式工具栏 + 500ms 防抖自动保存 |
| 📂 章节管理 | 树形结构，卷/章节层级，拖拽排序 |
| 👤 角色管理 | 角色档案 + 关系图谱 |
| 🌍 世界观管理 | 6 大分类：地理/历史/势力/规则/魔法/科技 |
| ⏳ 时间线管理 | 可视化时间轴 + 冲突检测 + 章节联动 |
| 🤖 AI 续写 | 多风格选择（正常/幽默/悬疑/浪漫/暗黑） |
| ✂️ AI 润色 | 精简/扩写/修辞/改写四种模式 |
| 🔍 AI 一致性检查 | 角色/世界观/时间线/剧情冲突检测 |
| 📤 导出 | TXT / Markdown / EPUB 三种格式 |
| ☁️ 云端同步 | 离线操作队列 + 自动同步 + 冲突解决 |

### 6. 快捷键

| 快捷键 | 功能 |
|--------|------|
| `Ctrl+S` | 保存当前章节 |
| `Ctrl+B` | 加粗 |
| `Ctrl+I` | 斜体 |
| `Ctrl+U` | 下划线 |

### 7. 项目架构

```
novel-writer-source/
├── lib/                    # Flutter 客户端源码
│   ├── models/            # 数据模型
│   ├── repositories/      # 数据访问层
│   ├── services/          # 业务服务
│   ├── providers/         # Riverpod 状态管理
│   ├── screens/           # 页面
│   └── widgets/           # 组件
├── backend/               # 后端服务
│   ├── ai_service/        # Python FastAPI AI 代理
│   ├── context_engine/    # 上下文索引引擎
│   ├── auth_service/      # Go 认证服务
│   └── sync_service/      # Go 同步服务
├── test/                  # 210+ 测试用例
└── PROGRESS.md            # 项目进度（21/21 完成 ✅）
```
