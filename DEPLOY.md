# 小说写作助手 - 部署与发布指南

> 最后更新：2026-05-08

## 项目架构总览

```
┌─────────────────────┐      ┌──────────────────────┐
│   Flutter 客户端     │      │   Python AI 服务      │
│  (Web / Windows /   │─────▶│  (FastAPI, :8080)    │
│   macOS / Linux /   │      │  Ollama/NVIDIA 代理  │
│   Android / iOS)    │      └──────────────────────┘
│                     │
│  novel_writer       │      ┌──────────────────────┐
│  小说写作工具        │─────▶│   Go 认证服务          │
│                     │      │  (Gin, :8081)        │
│                     │      │  PostgreSQL + JWT     │
│                     │      └──────────────────────┘
│                     │
│                     │      ┌──────────────────────┐
│                     │─────▶│   Go 同步服务          │
│                     │      │  (Gin, :8082)        │
│                     │      │  PostgreSQL + GORM    │
│                     │      └──────────────────────┘
└─────────────────────┘
```

## 构建要求

### Flutter 客户端
- Flutter SDK: ^3.5.0
- Dart SDK: ^3.5.0
- 浏览器版：无需额外工具，Chrome / Edge / Safari 均可
- 桌面版所需工具：
  - Android Studio (Android 构建)
  - Xcode (iOS/macOS 构建，仅 macOS)
  - Visual Studio (Windows 构建，含 C++ 工作负载)
  - Linux 构建依赖（gtk3, x11, sqlite3）

### Python 后端
- Python 3.12+
- FastAPI + Uvicorn
- 可选：Ollama（本地 AI 推理）

### Go 后端
- Go 1.25+
- PostgreSQL 15+

---

## 一、本地开发环境搭建

### 1.1 Flutter 客户端

```bash
# 获取依赖
cd novel_writer
flutter pub get

# 运行（Windows 桌面）
flutter run -d windows

# 运行（macOS 桌面）
flutter run -d macos

# 运行（Linux 桌面）
flutter run -d linux

# 运行（Android）
flutter run -d android

# 运行测试
flutter test
```

### 1.2 Python AI 服务

```bash
cd backend/ai_service

# 创建虚拟环境
python -m venv venv
venv\Scripts\activate  # Windows
source venv/bin/activate  # Linux/macOS

# 安装依赖
pip install -r requirements.txt

# 配置环境变量（参考 .env.example）
# 至少需要设置一个 AI 模型的 API Key

# 启动服务
uvicorn main:app --reload --port 8080
```

### 1.3 Go 认证服务

```bash
cd backend/auth_service

# 构建
go build -o server ./cmd/server

# 需要 PostgreSQL 数据库
# 环境变量：
#   DATABASE_URL: postgres://postgres:password@localhost:5432/novel_writer
#   JWT_SECRET: <your-jwt-secret>

# 启动
./server
```

### 1.4 Go 同步服务

```bash
cd backend/sync_service

# 构建
go build -o server ./cmd/server

# 启动（共享数据库）
./server
```

---

## 二、发布构建

### 2.0 Web 浏览器版（推荐，零依赖）

```bash
# 添加 Web 支持（仅首次）
flutter create --platforms web .

# 本地运行
flutter run -d chrome

# 构建发布
flutter build web --release
```

输出路径：`build/web/`

部署方式：
- 直接上传到任意静态服务器（Nginx / Apache / GitHub Pages）
- Docker + Nginx 部署（约 5MB 静态文件）
- 支持 PWA 离线访问

```nginx
# Nginx 部署配置
server {
    listen 80;
    server_name novel-writer.example.com;
    root /var/www/novel-writer/build/web;
    index index.html;
    
    # 处理 Flutter Web 的路由
    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

### 2.1 Windows 桌面版

```bash
flutter build windows --release
```

输出路径：`build/windows/runner/Release/`

分发文件：
- `novel_writer.exe`
- `msvcp140.dll`
- `vcruntime140.dll`（如需要）
- `data/` 目录

可使用 [Inno Setup](https://jrsoftware.org/isinfo.php) 打包为安装程序。

### 2.2 macOS 桌面版

```bash
flutter build macos --release
```

输出路径：`build/macos/Build/Products/Release/`

分发：
- `novel_writer.app`（可直接分发）
- 使用 `dmg` 工具打包为 DMG 镜像

### 2.3 Linux 桌面版

```bash
flutter build linux --release
```

输出路径：`build/linux/x64/release/bundle/`

分发：可打包为 AppImage、Flatpak 或 Snap。

### 2.4 Android 版

```bash
# 生成密钥库（首次）
keytool -genkey -v -keystore upload-keystore.jks \
  -alias upload -keyalg RSA -keysize 2048 -validity 10000

# 配置 android/key.properties:
# storePassword=<密码>
# keyPassword=<密码>
# keyAlias=upload
# storeFile=../upload-keystore.jks

# 构建 APK
flutter build apk --release

# 构建 App Bundle（推荐 Google Play）
flutter build appbundle --release
```

### 2.5 iOS 版

```bash
# 需要 Xcode + Apple Developer 账号
flutter build ios --release --no-codesign

# 然后使用 Xcode 进行签名和归档
# 或使用自动签名：
flutter build ios --release
```

---

## 三、生产部署

### 3.1 后端服务部署

#### Docker Compose（推荐）

```yaml
# docker-compose.yml
version: '3.8'
services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: novel_writer
      POSTGRES_PASSWORD: <password>
    volumes:
      - pgdata:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  auth-service:
    build: ./backend/auth_service
    environment:
      DATABASE_URL: postgres://postgres:password@postgres:5432/novel_writer
      JWT_SECRET: <jwt-secret>
    ports:
      - "8081:8081"
    depends_on:
      - postgres

  sync-service:
    build: ./backend/sync_service
    environment:
      DATABASE_URL: postgres://postgres:password@postgres:5432/novel_writer
    ports:
      - "8082:8082"
    depends_on:
      - postgres

  ai-service:
    build: ./backend/ai_service
    environment:
      OPENAI_API_KEY: <key>
      DEEPSEEK_API_KEY: <key>
    ports:
      - "8080:8080"
    depends_on:
      - postgres

volumes:
  pgdata:
```

### 3.2 环境变量配置

**Python AI 服务**（`backend/ai_service/.env`）：

```env
OPENAI_API_KEY=sk-xxx
ANTHROPIC_API_KEY=sk-ant-xxx
ZHIPU_API_KEY=xxx
DEEPSEEK_API_KEY=sk-xxx
APP_NAME=novel-writer-ai
MAX_CONTEXT_TOKENS=8192
```

**Go 认证服务**：

```env
DATABASE_URL=postgres://postgres:password@localhost:5432/novel_writer
JWT_SECRET=your-jwt-secret-32-chars-minimum
JWT_EXPIRATION=24h
AUTH_SERVER_PORT=:8081
ALLOWED_ORIGINS=*
```

**Go 同步服务**：

```env
DATABASE_URL=postgres://postgres:password@localhost:5432/novel_writer
SYNC_SERVER_PORT=:8082
ALLOWED_ORIGINS=*
```

---

## 四、发布检查清单

### 4.1 发布前检查

- [ ] 所有测试通过：`flutter test`（210+ 测试）
- [ ] 各页面 UI 无视觉问题
- [ ] 暗色/亮色主题切换正常
- [ ] 富文本编辑器功能正常
- [ ] 导出功能（TXT/Markdown/EPUB）可正常生成文件
- [ ] AI 续写/润色/一致性检查可调用
- [ ] 数据库迁移正常
- [ ] 版本号已更新（`pubspec.yaml`）

### 4.2 安全检查

- [ ] JWT Secret 已更换（非默认值）
- [ ] PostgreSQL 密码已更改
- [ ] CORS 域名已配置（非 `*`）
- [ ] 自定义 API Key 已妥善存储
- [ ] .env 文件未提交到版本控制

### 4.3 性能检查

- [ ] 富文本编辑器 500ms 防抖保存正常
- [ ] 大数据量下章节列表无卡顿
- [ ] AI 请求超时处理正常（15s 连接/60s 接收）
- [ ] 离线操作队列正常

---

## 五、应用市场提交流程

### 5.1 Microsoft Store（Windows）

1. 注册 Microsoft Partner Center 账号（$19 一次性费用）
2. 创建应用提交
3. 上传 MSIX 安装包
4. 填写应用描述、截图、隐私政策
5. 提交审核

### 5.2 Mac App Store（macOS）

1. 注册 Apple Developer Program（$99/年）
2. 在 Xcode 中配置签名和认证
3. 使用 Xcode Archive 创建构建
4. 通过 App Store Connect 提交

### 5.3 Google Play（Android）

1. 注册 Google Play Developer 账号（$25 一次性费用）
2. 创建应用列表
3. 上传 AAB 文件
4. 填写商店信息 - 标题/描述/截图/分类
5. 提交审核

### 5.4 App Store（iOS）

1. 需要 Mac 机器构建
2. 使用 Xcode 进行代码签名
3. 通过 App Store Connect 提交
4. 审核周期通常 1-3 天

---

## 六、版本号管理

`pubspec.yaml` 中的版本号格式：
```
version: 1.0.0+1
#         ^   ^
#         |   └── 构建号（Android内部版本/iOS构建号）
#         └────── 语义版本（主版本.次版本.修订）
```

版本更新策略：
- 主版本：重大重构或不兼容更改
- 次版本：新增功能
- 修订：Bug 修复

---

## 七、备份与恢复

### 7.1 数据库备份

```bash
# PostgreSQL 备份
pg_dump -U postgres novel_writer > backup_$(date +%Y%m%d).sql

# 恢复
psql -U postgres novel_writer < backup.sql
```

### 7.2 Flutter 构建缓存清理

```bash
flutter clean          # 清理构建缓存
flutter pub cache repair  # 修复依赖缓存
```

---

## 八、常见问题

### Q: Android 构建失败，提示 `flutter_quill` 版本冲突
A: `flutter_quill` 11.x 需要 Flutter SDK ^3.5.0，确保 SDK 版本匹配。

### Q: Windows 构建需要什么 C++ 环境
A: 安装 Visual Studio 2022，选择"使用 C++ 的桌面开发"工作负载。

### Q: 无法连接 Python AI 服务
A: 检查 `ApiClient.instance.setBaseUrl()` 中的 URL 配置，默认 `http://localhost:8080/api/v1/`。

### Q: PostgreSQL 连接失败
A: 确认 PostgreSQL 服务已启动，且 `DATABASE_URL` 配置正确。
