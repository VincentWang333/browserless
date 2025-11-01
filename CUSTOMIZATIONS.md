# Browserless 自定义修改文档

## 📋 概述

本文档记录了对 [browserless](https://github.com/browserless/browserless) 项目 fork 的所有自定义修改。这些修改主要为了支持使用自编译的 Chromium 浏览器,并改进错误处理策略,提升服务可用性。

**主要目标**:
1. 支持通过环境变量配置自定义 Chromium 浏览器路径
2. 将浏览器二进制缺失从致命错误降级为警告,允许服务部分可用

**版本信息**:
- Browserless 版本: 2.38.1
- 首次自定义日期: 2025-11-01
- 维护分支: custom/main (或当前自定义分支)

---

## 🔍 修改清单

### 1. 自定义 Chromium 路径支持

#### 目标
允许使用自编译的 Chromium 浏览器代替 Playwright 默认下载的 Chromium,便于开发和测试特定版本或自定义构建的 Chromium。

#### 修改文件

##### 1.1 环境变量配置 (`.env.dev`)
- **位置**: 第 13-23 行
- **修改内容**: 添加 `CUSTOM_CHROMIUM_PATH` 环境变量配置及说明
- **标记**: `[CUSTOMIZED START/END]`

**使用示例**:
```bash
# macOS
CUSTOM_CHROMIUM_PATH=/Applications/Chromium.app/Contents/MacOS/Chromium

# Linux
CUSTOM_CHROMIUM_PATH=/usr/bin/chromium-browser

# Windows
CUSTOM_CHROMIUM_PATH=C:\Program Files\Chromium\Application\chrome.exe
```

##### 1.2 配置管理 (`src/config.ts`)
- **位置 1**: 第 187-194 行 - `customChromiumPath` 属性定义
- **位置 2**: 第 498-505 行 - `getCustomChromiumPath()` getter 方法
- **修改内容**:
  - 添加 protected 属性从环境变量读取自定义路径
  - 使用 `untildify()` 处理 `~` 符号
  - 提供公开的 getter 方法供其他模块访问
- **标记**: `[CUSTOMIZED START/END]`

##### 1.3 CDP 浏览器支持 (`src/browsers/browsers.cdp.ts`)
- **位置**: 第 56-66 行
- **修改内容**: 在 `ChromiumCDP` 构造函数中检查自定义路径,如果配置则覆盖默认 `executablePath`
- **逻辑**:
  ```typescript
  const customPath = config.getCustomChromiumPath();
  if (customPath) {
    this.executablePath = customPath;
    this.logger.info(`Using custom Chromium path: ${customPath}`);
  }
  ```
- **标记**: `[CUSTOMIZED START/END]`

##### 1.4 Playwright 浏览器支持 (`src/browsers/browsers.playwright.ts`)
- **位置**: 第 226-244 行
- **修改内容**: 为 `ChromiumPlaywright` 类添加完整构造函数,支持自定义路径
- **说明**: 虽然添加了支持,但当前使用自定义路径时 `ChromiumPlaywright` 会被跳过(见 1.5)
- **标记**: `[CUSTOMIZED START/END]`

##### 1.5 浏览器检测逻辑 (`src/utils.ts`)
- **位置**: 第 508-557 行
- **修改内容**:
  - 将 `availableBrowsers` 从 Promise 常量改为接受 `Config` 参数的函数
  - 检查自定义 Chromium 路径的可用性(优先于 Playwright 默认路径)
  - 当使用自定义路径时跳过 `ChromiumPlaywright`(因为自定义 Chromium 当前仅支持 CDP 协议)
- **关键逻辑**:
  ```typescript
  const customChromiumPath = config.getCustomChromiumPath();
  const chromiumPathToCheck = customChromiumPath || playwright.chromium.executablePath();

  if (chromiumExists && !customChromiumPath) {
    availableBrowsers.push(ChromiumPlaywright);
  }
  ```
- **标记**: `[CUSTOMIZED START/END]` + 行内 `[CUSTOMIZED]`

##### 1.6 函数调用更新
以下文件更新了 `availableBrowsers` 的调用方式,传入 `config` 参数:

- **`src/browserless.ts`** (第 278 行)
  - 标记: `[CUSTOMIZED]` 行内注释

- **`src/browsers/index.ts`** (第 89, 126 行)
  - 两处 `getProtocolJSON` 和 `getVersionJSON` 方法中
  - 标记: `[CUSTOMIZED]` 行内注释

- **`src/routes/management/http/meta.get.ts`** (第 98 行)
  - meta API 接口中
  - 标记: `[CUSTOMIZED]` 行内注释

---

### 2. 错误处理策略改进

#### 目标
将浏览器二进制文件缺失从致命错误(抛出异常导致服务无法启动)降级为警告(跳过对应路由但允许服务继续运行),提高服务可用性。

#### 修改文件

##### 2.1 路由加载逻辑 (`src/browserless.ts`)
- **位置**: 第 385-414 行
- **修改内容**:
  - 添加 `filterByBrowserAvailability` 过滤器函数
  - 将原来的 `throw new Error(...)` 改为 `this.logger.warn(...) + return false`
  - 在路由数组上应用过滤器,自动跳过不可用的浏览器路由

**原始行为**:
```typescript
throw new Error(`Couldn't load route "${route.path}" due to missing browser binary...`);
// 服务启动失败
```

**修改后行为**:
```typescript
this.logger.warn(`Skipping route "${route.path}" due to missing browser binary...`);
return false; // 跳过该路由,服务继续启动
```

- **标记**: `[CUSTOMIZED START/END]`

#### 影响
- Edge, Firefox, WebKit 等浏览器缺失时仅警告,不阻止服务启动
- 对应的路由(如 `/edge/*`, `/firefox/*`)会被自动跳过
- 提高开发环境友好度(可能只安装部分浏览器)

---

## 🏗️ 技术架构

### 配置流程

```
环境变量 (.env.dev)
    ↓
Config.customChromiumPath (读取并解析)
    ↓
Config.getCustomChromiumPath() (提供访问)
    ↓
浏览器实例构造时应用 (ChromiumCDP/ChromiumPlaywright)
    ↓
浏览器检测逻辑使用 (availableBrowsers)
```

### 浏览器可用性检测

```typescript
// 优先级: 自定义路径 > Playwright 默认路径
const chromiumPathToCheck = customChromiumPath || playwright.chromium.executablePath();

// 路径存在性检查
if (await exists(chromiumPathToCheck)) {
  availableBrowsers.push(ChromiumCDP);
  if (!customChromiumPath) {
    availableBrowsers.push(ChromiumPlaywright); // 仅在未使用自定义路径时添加
  }
}
```

### 路由加载策略

```typescript
// 过滤不可用的浏览器路由
httpRoutes.filter(filterByBrowserAvailability)
wsRoutes.filter(filterByBrowserAvailability)

// 警告但不抛出错误
logger.warn(`Skipping route "${route.path}"...`);
```

---

## 🔧 使用指南

### 配置自定义 Chromium

1. **设置环境变量**:
   ```bash
   # 在 .env.dev 或 .env 文件中
   CUSTOM_CHROMIUM_PATH=/path/to/your/chromium
   ```

2. **验证配置**:
   启动服务后查看日志:
   ```
   Using custom Chromium path: /path/to/your/chromium
   Starting new ChromiumCDP instance
   ```

3. **测试可用性**:
   ```bash
   curl http://localhost:3000/meta -H "Authorization: Bearer YOUR_TOKEN"
   ```
   检查响应中的 `chromium` 版本信息

### 查找所有自定义修改

使用以下命令快速定位所有标记:

```bash
# 查找所有 CUSTOMIZED 标记
grep -rn "CUSTOMIZED" src/ .env.dev

# 统计修改数量
grep -r "CUSTOMIZED" src/ .env.dev | wc -l

# 只查看完整注释块
grep -A 3 "CUSTOMIZED START" src/
```

---

## 📈 升级指南

### 合并上游更新时的注意事项

1. **检查冲突文件**:
   本文档列出的所有修改文件都可能与上游更新冲突,需要特别注意:
   - `src/config.ts`
   - `src/utils.ts`
   - `src/browserless.ts`
   - `src/browsers/browsers.cdp.ts`
   - `src/browsers/browsers.playwright.ts`
   - `src/browsers/index.ts`
   - `src/routes/management/http/meta.get.ts`

2. **保护自定义代码**:
   - 搜索所有 `[CUSTOMIZED]` 标记
   - 在合并冲突时确保保留这些标记的代码段
   - 如果上游修改了相同区域,需要手动集成自定义逻辑

3. **测试验证**:
   合并后必须测试以下功能:
   - ✅ 自定义 Chromium 路径配置是否仍然工作
   - ✅ 浏览器缺失时是否只警告不报错
   - ✅ 服务能否正常启动
   - ✅ ChromiumCDP 路由是否可用
   - ✅ 日志中是否显示 "Using custom Chromium path"

4. **更新本文档**:
   - 记录新的冲突及解决方案
   - 更新修改文件列表和行号
   - 添加新的自定义内容(如果有)

### 升级检查清单

- [ ] 备份当前自定义分支
- [ ] 查看上游更新日志(CHANGELOG)
- [ ] 识别潜在冲突文件
- [ ] 执行 merge/rebase 操作
- [ ] 解决冲突时保留 `[CUSTOMIZED]` 代码
- [ ] 重新编译: `npm run build`
- [ ] 运行测试: `npm test`(如果有)
- [ ] 启动服务并验证日志
- [ ] 测试自定义 Chromium 功能
- [ ] 更新本文档的版本信息和修改记录

---

## 🐛 已知问题和限制

### 1. ChromiumPlaywright 限制
- **现象**: 使用自定义 Chromium 路径时,`ChromiumPlaywright` 会被禁用
- **原因**: 当前实现假设自定义 Chromium 仅支持 CDP 协议(Puppeteer)
- **影响**: Playwright Chromium 相关的路由会被跳过
- **解决方案**: 如需支持,可以移除 `utils.ts` 中的跳过逻辑

### 2. 路径验证限制
- **现象**: 仅做基本的文件存在性检查
- **限制**:
  - 不验证文件是否可执行
  - 不检查 Chromium 版本兼容性
  - 不验证是否是有效的 Chromium 可执行文件
- **建议**: 启动前手动验证浏览器路径和版本

### 3. 跨平台路径差异
- **注意**: 不同操作系统的可执行文件路径格式不同
  - macOS: `/Applications/Chromium.app/Contents/MacOS/Chromium`
  - Linux: `/usr/bin/chromium` 或 `/usr/bin/chromium-browser`
  - Windows: `C:\Program Files\Chromium\Application\chrome.exe`
- **建议**: 在 CI/CD 中使用不同的环境配置文件

---

## 🔮 未来改进建议

### 短期优化
1. 添加 Chromium 版本检查和警告
2. 支持 ChromiumPlaywright 使用自定义路径
3. 添加路径有效性验证(可执行权限检查)
4. 提供更详细的错误信息和调试日志

### 长期规划
1. 扩展到其他浏览器(Firefox, WebKit)的自定义路径支持
2. 支持多个 Chromium 版本并存
3. 添加浏览器健康检查和自动重启机制
4. 集成到 Web UI 配置界面

---

## 📊 维护日志

| 日期 | 版本 | 修改内容 | 修改人 | 备注 |
|------|------|---------|--------|------|
| 2025-11-01 | 2.38.1 | 初始化自定义 Chromium 路径支持 | Vincent | 支持自编译 Chromium,添加环境变量配置 |
| 2025-11-01 | 2.38.1 | 错误处理策略改进 | Vincent | 浏览器缺失降级为警告,提高服务可用性 |
| 2025-11-01 | 2.38.1 | 添加 CUSTOMIZATIONS.md 文档 | Vincent | 建立自定义修改追踪机制 |

---

## 📚 相关资源

- **上游仓库**: https://github.com/browserless/browserless
- **上游文档**: https://docs.browserless.io/
- **Puppeteer 文档**: https://pptr.dev/
- **Playwright 文档**: https://playwright.dev/
- **Chromium 下载**: https://www.chromium.org/getting-involved/download-chromium/

---

## 📝 备注

### 代码搜索技巧

```bash
# 查看所有自定义修改摘要
grep -B 2 "Purpose:" src/**/*.ts | grep -A 1 "CUSTOMIZED"

# 查找特定功能的修改
grep -r "custom.*path" src/

# 统计自定义代码行数
grep -A 100 "CUSTOMIZED START" src/**/*.ts | wc -l
```

### Git 操作建议

```bash
# 查看所有自定义相关的 commits
git log --all --grep="CUSTOMIZED" --oneline

# 查看自定义代码的 diff
git diff upstream/main --word-diff-regex="CUSTOMIZED"

# 创建自定义修改的补丁
git diff upstream/main > custom-changes.patch
```

---

**最后更新**: 2025-11-01
**文档版本**: 1.0.0
**维护者**: Vincent Wang
