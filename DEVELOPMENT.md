# 开发环境配置指南

本文档详细说明了如何在本地搭建 annual-lottery 项目的开发环境。

## 🚀 快速开始 (5分钟上手)

如果你已经安装了 Node.js 18+ 和 Git，可以快速开始：

```bash
# 1. 克隆项目
git clone https://github.com/luobin1016/annual-lottery.git
cd annual-lottery

# 2. 安装依赖 (选择其中一种)
pnpm install          # 推荐，需先安装: npm install -g pnpm
# 或
npm install --force   # 使用 npm

# 3. 启动开发服务器
pnpm dev             # 使用 pnpm
# 或
npm run dev          # 使用 npm

# 4. 打开浏览器访问 http://localhost:6719
```

**需要完整的多设备数据共享功能？** 额外启动后端服务：

```bash
# 新开一个终端
pnpm server:dev
# 或
npm run server:dev
```

---

## 目录

- [系统要求](#系统要求)
- [必需软件](#必需软件)
- [推荐的IDE和工具](#推荐的ide和工具)
- [项目克隆](#项目克隆)
- [环境配置](#环境配置)
- [启动项目](#启动项目)
- [常用开发命令](#常用开发命令)
- [常见问题](#常见问题)

## 系统要求

- **操作系统**: Windows 10/11, macOS 10.15+, 或 Linux (Ubuntu 18.04+)
- **内存**: 建议 8GB 以上
- **硬盘空间**: 至少 2GB 可用空间

## 必需软件

### 1. Node.js (必须)

**推荐版本**: Node.js 18.x 或 20.x (LTS 版本)

**安装方法**:
- 官网下载: [https://nodejs.org/](https://nodejs.org/)
- 建议下载 LTS (长期支持) 版本

**验证安装**:
```bash
node --version  # 应显示 v18.x.x 或 v20.x.x
npm --version   # 应显示 9.x.x 或 10.x.x
```

### 2. 包管理器

本项目支持两种包管理器，**推荐使用 pnpm**:

#### 选项 A: pnpm (推荐)

pnpm 是一个快速、节省磁盘空间的包管理器。

**安装**:
```bash
npm install -g pnpm
```

**验证**:
```bash
pnpm --version  # 应显示 8.x.x 或更高版本
```

#### 选项 B: npm (内置)

npm 随 Node.js 一起安装，无需额外安装。

**注意**: 如果使用 npm，安装依赖时需要使用 `--force` 标志:
```bash
npm install --force
```

### 3. Git (必须)

用于版本控制和克隆项目。

**安装**:
- Windows: [https://git-scm.com/download/win](https://git-scm.com/download/win)
- macOS: `brew install git` 或从官网下载
- Linux: `sudo apt-get install git` (Ubuntu/Debian)

**验证**:
```bash
git --version
```

## 推荐的IDE和工具

### 主要IDE

#### Visual Studio Code (强烈推荐)

**下载地址**: [https://code.visualstudio.com/](https://code.visualstudio.com/)

**必装插件**:

1. **Vue Language Features (Volar)** - Vue 3 官方支持
   - 插件ID: `Vue.volar`
   - 提供 Vue 3 语法高亮、智能提示等功能
   - **注意**: 如果之前安装了 Vetur，需要禁用它

2. **TypeScript Vue Plugin (Volar)**
   - 插件ID: `Vue.vscode-typescript-vue-plugin`
   - 让 TypeScript 认识 `.vue` 文件

**推荐插件**:

3. **ESLint** - 代码规范检查
   - 插件ID: `dbaeumer.vscode-eslint`

4. **Prettier - Code formatter** - 代码格式化
   - 插件ID: `esbenp.prettier-vscode`

5. **Tailwind CSS IntelliSense** - Tailwind CSS 智能提示
   - 插件ID: `bradlc.vscode-tailwindcss`

6. **Path Intellisense** - 路径自动补全
   - 插件ID: `christian-kohler.path-intellisense`

7. **Auto Rename Tag** - 自动重命名配对标签
   - 插件ID: `formulahendry.auto-rename-tag`

8. **GitLens** - Git 增强工具
   - 插件ID: `eamodio.gitlens`

**VS Code 配置**:

在项目根目录的 `.vscode/settings.json` 中已包含推荐配置。

#### 其他可选IDE

- **WebStorm**: JetBrains 出品的专业 Web 开发 IDE (付费)
  - 下载: [https://www.jetbrains.com/webstorm/](https://www.jetbrains.com/webstorm/)
  - 内置 Vue、TypeScript 支持

## 项目克隆

### 1. 克隆仓库

```bash
# 使用 HTTPS
git clone https://github.com/luobin1016/annual-lottery.git

# 或使用 SSH (需要配置 SSH 密钥)
git clone git@github.com:luobin1016/annual-lottery.git

# 进入项目目录
cd annual-lottery
```

### 2. 查看项目结构

```
annual-lottery/
├── src/                    # 前端源代码
│   ├── components/         # Vue 组件
│   ├── views/             # 页面视图
│   ├── stores/            # Pinia 状态管理
│   ├── assets/            # 静态资源
│   └── ...
├── server/                # 后端服务代码
│   ├── index.js           # 服务入口
│   ├── db.js              # 数据库配置
│   └── package.json       # 后端依赖
├── public/                # 公共静态文件
├── package.json           # 前端依赖配置
├── vite.config.ts         # Vite 构建配置
├── tsconfig.json          # TypeScript 配置
├── tailwind.config.js     # Tailwind CSS 配置
└── README.md              # 项目说明文档
```

## 环境配置

### 1. 安装前端依赖

在项目根目录执行:

```bash
# 使用 pnpm (推荐)
pnpm install

# 或使用 npm
npm install --force
```

**安装时间**: 首次安装可能需要 3-10 分钟，取决于网络速度。

### 2. 安装后端依赖

```bash
# 进入 server 目录
cd server

# 安装依赖
npm install --force

# 返回项目根目录
cd ..
```

或者使用快捷命令:

```bash
# 在项目根目录执行
pnpm server:install
# 或
npm run server:install
```

### 3. 环境变量配置 (可选)

项目根目录已包含环境变量文件:
- `.env` - 开发环境配置
- `.env.production` - 生产环境配置

默认配置通常无需修改，如需自定义后端服务地址，可编辑 `.env`:

```env
VITE_BASE_URL=http://localhost:3456
```

## 启动项目

### 方式一: 仅前端开发 (使用 localStorage)

如果只需要前端开发，不需要多设备数据共享:

```bash
# 使用 pnpm
pnpm dev

# 或使用 npm
npm run dev
```

访问: [http://localhost:6719](http://localhost:6719)

### 方式二: 前端 + 后端 (完整功能)

如果需要多设备数据共享，需要同时启动前后端:

#### 终端 1 - 启动后端服务

```bash
# 使用 pnpm
pnpm server:dev

# 或使用 npm
npm run server:dev

# 或手动启动
cd server
npm run dev
```

后端服务运行在: [http://localhost:3456](http://localhost:3456)

#### 终端 2 - 启动前端服务

```bash
# 使用 pnpm
pnpm dev

# 或使用 npm
npm run dev
```

前端服务运行在: [http://localhost:6719](http://localhost:6719)

## 常用开发命令

### 前端命令

```bash
# 开发模式 (带热重载)
pnpm dev
npm run dev

# 代码检查
pnpm lint
npm run lint

# 代码检查并自动修复
pnpm lint:fix
npm run lint:fix

# 运行单元测试
pnpm test
npm run test

# 运行测试 (带 UI 界面)
pnpm test:ui
npm run test:ui

# 构建生产版本
pnpm build
npm run build

# 构建可直接打开的 HTML 文件版本
pnpm build:file
npm run build:file

# 预览构建结果
pnpm preview
npm run preview
```

### 后端命令

```bash
# 开发模式 (自动重启)
pnpm server:dev
npm run server:dev

# 生产模式
pnpm server
npm run server

# 手动操作
cd server
npm run dev     # 开发模式
npm start       # 生产模式
```

## 开发工作流

### 1. 日常开发流程

```bash
# 1. 拉取最新代码
git pull origin main

# 2. 创建新分支 (可选)
git checkout -b feature/your-feature-name

# 3. 安装/更新依赖 (如果 package.json 有变化)
pnpm install

# 4. 启动开发服务器
pnpm dev

# 5. 进行代码修改...

# 6. 代码检查
pnpm lint:fix

# 7. 测试功能
pnpm test

# 8. 提交代码
git add .
git commit -m "描述你的修改"
git push origin feature/your-feature-name
```

### 2. 构建部署

```bash
# 构建生产版本
pnpm build

# 构建产物在 dist/ 目录
```

### 3. Docker 部署 (可选)

```bash
# 构建镜像
docker build -t annual-lottery .

# 运行容器
docker run -d --name annual-lottery -p 9277:80 annual-lottery

# 访问: http://localhost:9277
```

## 常见问题

### 1. 依赖安装失败

**问题**: `npm install` 或 `pnpm install` 报错

**解决方案**:
```bash
# 清理缓存
npm cache clean --force
pnpm store prune

# 删除 node_modules 和 lock 文件
rm -rf node_modules package-lock.json pnpm-lock.yaml

# 使用 --force 标志重新安装
npm install --force
# 或
pnpm install
```

### 2. 端口被占用

**问题**: `Port 6719 is already in use`

**解决方案**:
- 修改 `vite.config.ts` 中的端口号
- 或终止占用端口的进程:
  ```bash
  # Windows
  netstat -ano | findstr :6719
  taskkill /PID <PID> /F
  
  # macOS/Linux
  lsof -ti:6719 | xargs kill -9
  ```

### 3. Volar 插件不生效

**问题**: Vue 文件没有语法高亮或智能提示

**解决方案**:
1. 确保已安装 `Vue.volar` 插件
2. 禁用 Vetur 插件 (如果有安装)
3. 重启 VS Code
4. 使用 "Take Over Mode": 
   - Ctrl/Cmd + Shift + P
   - 输入 "TypeScript: Select TypeScript Version"
   - 选择 "Use Workspace Version"

### 4. TypeScript 类型错误

**问题**: 编辑器显示 TypeScript 错误

**解决方案**:
```bash
# 重新生成类型定义
pnpm dev  # 会自动生成 auto-imports.d.ts 和 components.d.ts
```

### 5. 后端数据库锁定

**问题**: `database is locked` 错误

**解决方案**:
```bash
# 停止所有服务
# 删除数据库锁文件
cd server
rm lottery.db-wal lottery.db-shm
# 重启服务
```

### 6. 构建失败

**问题**: `pnpm build` 失败

**解决方案**:
```bash
# 确保 TypeScript 类型检查通过
pnpm lint:fix

# 检查 Node.js 版本 (需要 18+ 或 20+)
node --version

# 清理构建缓存
rm -rf dist dist-file
pnpm build
```

### 7. 网络问题 (中国大陆用户)

**问题**: 依赖安装速度慢或失败

**解决方案**: 使用国内镜像源

```bash
# npm 使用淘宝镜像
npm config set registry https://registry.npmmirror.com

# pnpm 使用淘宝镜像
pnpm config set registry https://registry.npmmirror.com

# 恢复官方源
npm config set registry https://registry.npmjs.org
pnpm config set registry https://registry.npmjs.org
```

## 技术栈说明

了解项目使用的主要技术有助于开发:

- **Vue 3**: 渐进式 JavaScript 框架
- **TypeScript**: JavaScript 的超集，提供类型安全
- **Vite**: 下一代前端构建工具
- **Pinia**: Vue 3 官方状态管理库
- **Vue Router**: Vue.js 官方路由管理器
- **Tailwind CSS**: 实用优先的 CSS 框架
- **DaisyUI**: Tailwind CSS 组件库
- **Three.js**: 3D 图形库 (用于 3D 球体抽奖效果)
- **Express.js**: 后端 Web 框架 (Node.js)
- **SQLite**: 轻量级数据库 (通过 better-sqlite3)
- **Vitest**: Vue 生态的单元测试框架

## 学习资源

- [Vue 3 官方文档](https://cn.vuejs.org/)
- [TypeScript 中文文档](https://www.tslang.cn/)
- [Vite 中文文档](https://cn.vitejs.dev/)
- [Pinia 中文文档](https://pinia.vuejs.org/zh/)
- [Tailwind CSS 中文文档](https://www.tailwindcss.cn/)
- [Three.js 文档](https://threejs.org/docs/)

## 获取帮助

如果遇到问题:

1. 查看本文档的 [常见问题](#常见问题) 部分
2. 查看项目的 [README.md](./README.md)
3. 提交 [GitHub Issue](https://github.com/luobin1016/annual-lottery/issues)
4. 查看原项目的 [Issues](https://github.com/yongjiu8/annual-lottery/issues)

## 贡献代码

欢迎贡献代码! 请遵循以下步骤:

1. Fork 本仓库
2. 创建特性分支 (`git checkout -b feature/AmazingFeature`)
3. 提交更改 (`git commit -m 'Add some AmazingFeature'`)
4. 推送到分支 (`git push origin feature/AmazingFeature`)
5. 开启 Pull Request

更多贡献指南请查看 [CONTRIBUTING.md](./.github/CONTRIBUTING.md)

## 📋 快速参考卡片

### 环境要求速查表

| 软件 | 最低版本 | 推荐版本 | 说明 |
|------|---------|---------|------|
| Node.js | 18.x | 20.x LTS | 必需 |
| npm | 9.x | 10.x | 随 Node.js 安装 |
| pnpm | - | 8.x+ | 推荐，需单独安装 |
| Git | 2.x | 最新版 | 必需 |

### 常用命令速查表

| 命令 | 说明 |
|------|------|
| `pnpm install` | 安装所有依赖 |
| `pnpm dev` | 启动前端开发服务器 (端口 6719) |
| `pnpm server:dev` | 启动后端服务器 (端口 3456) |
| `pnpm build` | 构建生产版本 |
| `pnpm build:file` | 构建可直接打开的 HTML 版本 |
| `pnpm lint` | 代码检查 |
| `pnpm lint:fix` | 自动修复代码问题 |
| `pnpm test` | 运行单元测试 |
| `pnpm preview` | 预览构建结果 |

### VS Code 必装插件

- **Vue.volar** - Vue 3 语法支持 (必需)
- **Vue.vscode-typescript-vue-plugin** - TypeScript 支持 (必需)
- **dbaeumer.vscode-eslint** - 代码规范检查 (推荐)
- **bradlc.vscode-tailwindcss** - Tailwind CSS 智能提示 (推荐)

### 项目端口

| 服务 | 端口 | 说明 |
|------|------|------|
| 前端开发服务器 | 6719 | Vite 开发服务器 |
| 后端 API 服务器 | 3456 | Express + SQLite |

### 目录结构速览

```
annual-lottery/
├── src/                   # 前端源代码
│   ├── views/            # 页面
│   ├── components/       # 组件
│   ├── stores/           # 状态管理 (Pinia)
│   ├── router/           # 路由配置
│   └── utils/            # 工具函数
├── server/               # 后端服务
│   ├── index.js         # 服务入口
│   └── db.js            # 数据库配置
├── public/              # 静态资源
└── dist/                # 构建产物 (git ignored)
```

## 许可证

本项目采用 MIT 许可证 - 查看 [LICENSE](./LICENSE) 文件了解详情。
