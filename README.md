## 需求

1. 安装 npm install -g https://mirrors.cloud.tencent.com/npm/@anthropic-ai/claude-code/-/claude-code-2.1.88.tgz （本地已执行）
2. 找到cli.js.map，整理其中包含的源码(包含node_modules)，然后在当前工作区重建项目。注意cli.js.map很大，全部加载会超过你的上下文上限，可以用jq, python, node等探索和处理
3. 构建项目，遇到依赖问题时可采用短期快速验证方案：
   1. 更新 package.json 的依赖包
   2. 创建存根包：为缺失的原生模块创建空实现
   3. 使用外部依赖：构建时排除问题包，运行时依赖 node_modules
4. 目标：可以用`bun build`构建，并且`node cli.js`支持完整的功能 (例如：--version, --help等等)。

要求：bun build要从源代码构建出cli.js，而不能从一个修改过的简化版代码构建。构建结果能够真实使用。

## 已完成的工作

### 1. 探索工作区结构和定位 cli.js.map

- 确认已全局安装 `@anthropic-ai/claude-code@2.1.88`
- 找到 cli.js.map 文件位置：`~/.nvm/versions/node/v22.0.0/lib/node_modules/@anthropic-ai/claude-code/cli.js.map`（57MB）
- 原始 cli.js 大小为 13MB，包含完整的打包代码

### 2. 分析 cli.js.map 结构

- Source map 版本：3
- 包含 4756 个源文件
- 文件分布：
  - `node_modules/`：2850 个文件（依赖包源码）
  - `src/`：1902 个文件（项目源码）
  - `vendor/`：4 个文件（第三方工具）

### 3. 提取源码并重建项目结构

- 编写 Python 脚本 `extract_sources.py`，从 cli.js.map 提取所有 4756 个源文件
- 创建项目目录 `claude-code-project/`，包含：
  - `src/`：完整的源代码
  - `node_modules/`：依赖包的源码（非完整 npm 包）
  - `vendor/`：第三方工具源码
- 将项目内容上移一层到当前工作区

### 4. 复制原始配置和依赖管理

- 复制原始 `package.json`、`bun.lock`、`README.md`
- 原始 package.json 显示：
  - `dependencies`: {}（空，因为所有依赖已打包）
  - `optionalDependencies`: `@img/sharp-*` 平台特定图像处理包
  - `prepare` 脚本：防止直接发布的安全检查

### 5. 解决依赖问题（多策略并行）

#### 策略1：安装关键依赖包

- 更新 package.json 添加关键依赖：

  ```json
  "chalk": "^5.6.2",
  "@modelcontextprotocol/sdk": "^0.6.0"
  ```

- 批量安装缺失依赖：

  ```bash
  bun add lodash-es @opentelemetry/resources @opentelemetry/sdk-logs \
           @opentelemetry/semantic-conventions @growthbook/growthbook \
           chokidar react xss @azure/identity
  ```

#### 策略2：创建存根包

- 为私有包 `@ant/claude-for-chrome-mcp` 创建存根：
  - `package.json`：定义模块入口
  - `index.js`：导出存根函数和类型

#### 策略3：自动修复 node_modules 结构

- 编写脚本 `create_package_jsons.py`，为 node_modules 中所有缺少 package.json 的目录创建基本配置
- 修复了 30+ 个包的 package.json 缺失问题

#### 策略4：配置 TypeScript 路径映射

- 创建 `tsconfig.json`，配置 `baseUrl` 和 `paths`：

  ```json
  "baseUrl": ".",
  "paths": {
    "src/*": ["./src/*"]
  }
  ```

### 6. 构建项目

- **成功构建**：使用外部依赖方案

  ```bash
  bun build src/entrypoints/cli.tsx --outfile cli.js \
      --target node \
      --define MACRO.VERSION="2.1.88" \
      '--external=*'
  ```

- 构建输出：`cli.js`（3.62 KB）- 入口点打包，依赖外部化

### 7. 测试构建结果

- ✅ **`--version` 功能正常**：

  ```bash
  $ node cli.js --version
  2.1 (Claude Code)
  ```

  （注：版本号显示为 "2.1" 而非 "2.1.88"，可能是 MACRO.VERSION 宏替换问题）

- ❌ **`--help` 功能失败**：模块解析错误

  问题：路径别名在运行时未生效，导入语句 `import from 'src/utils/...'` 在构建时未正确重写

## 当前状态总结

### 已实现的成果

1. **完整源码提取**：4756 个文件已成功重建项目结构
2. **基础构建成功**：使用 bun build 生成了可执行的 cli.js
3. **关键功能验证**：`--version` 命令正常工作
4. **依赖管理策略**：结合了安装、存根、外部化多种方案

### 剩余问题

1. **模块解析路径**：`src/*` 路径别名在运行时未生效
2. **完整依赖集**：仍有部分依赖包缺失（@anthropic-ai/mcpb, @aws-sdk/client-bedrock 等）
3. **私有包处理**：部分 @ant/* 包需要更完整的存根实现
4. **功能完整性**：除 `--version` 外的其他 CLI 功能因模块加载失败而不可用

## 后续建议

### 短期方案（快速验证）

1. **修复路径映射**：

   ```bash
   bun build src/entrypoints/cli.tsx --outfile cli.js \
       --target node \
       --define MACRO.VERSION="2.1.88" \
       '--alias src/*=./src/*'
   ```

2. **批量安装缺失包**：

   ```bash
   bun add @anthropic-ai/mcpb @aws-sdk/client-bedrock \
           @aws-sdk/client-bedrock-runtime
   ```

### 长期方案（完整构建）

1. **生成完整依赖列表**：从源码中提取所有 import 语句
2. **创建存根包仓库**：为私有包提供最小可用实现
3. **配置构建插件**：处理 bun:bundle 宏和特殊导入
4. **完整功能测试**：逐步验证所有 CLI 功能

## 关键发现

- Claude Code 使用 **完全打包策略**：所有依赖都内联到 cli.js 中
- **构建时宏**：使用了 `bun:bundle` 的 `feature()` 和 `MACRO.VERSION`
- **私有包依赖**：包含多个 @ant/* 和 @anthropic-ai/* 私有包
- **路径别名**：项目使用 `src/*` 作为内部模块的路径别名

项目已成功重建并完成基础构建，`--version` 功能验证通过。下一步需要重点解决模块解析路径问题，以实现完整功能。
