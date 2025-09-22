# AGI.Captor 版本策略（锁定时间序列版本体系）

## 📋 概述

本项目采用 **“时间序列（Time-based）+ 显式锁定（Locked）+ 标签驱动（Tag-driven）”** 的确定性版本模型（已完全移除 GitVersion 依赖）：

| 目标 | 方案 |
|------|------|
| 版本生成 | 单次生成并写入 `version.json` |
| 单一来源 | 锁定文件 `version.json` |
| 可重复性 | 纯文件可审计，无外部计算工具 |
| 并行冲突 | UTC 秒级时间戳（冲突概率极低，必要时重新生成） |
| Changelog 分类 | 仅用于 Release Notes 分类，不驱动版本号变化 |
| 版本语义 | 线性时间序列，放弃主/次/补丁语义判断 |

> 版本不再“被计算”，而是“被声明并锁定”。流水线只接受与 `version.json` 一致的标签。

## 🔧 版本文件 `version.json`

示例（新四段 Display 格式）：
```json
{
  "version": "2025.9.22.070405",
  "assemblyVersion": "2025.9.22.7",
  "fileVersion": "2025.9.22.405",
  "informationalVersion": "2025.9.22.070405"
}
```

规则更新：
1. `version`（Display）采用四段结构：`YYYY.M.D.HHmm`（月日无前导零，时间固定 4 位）。
2. `assemblyVersion` = Display（所有版本字段统一）。
3. `fileVersion` = Display（所有版本字段统一）。
4. `informationalVersion` = Display（所有版本字段统一）。
5. `.csproj` 中对应字段由 Nuke 写回，不得手动编辑。
6. 发布标签名仍使用 `v<version>`（即四段 Display 版本）。
7. 守卫：Display 正则 + 版本字段一致性。

Display 正则：
```
^\d{4}\.[1-9]\d?\.[1-9]\d?\.[0-2]\d[0-5]\d$
```

结构分段（Display 四段）：
```
YYYY . M . D . HHmm
│      │   │    └─ 24h 时间四位（小时两位+分两位）
│      │   └─ 日 (1-31 无前导零)
│      └─ 月 (1-12 无前导零)
└─ 年
```

派生映射：
```
assemblyVersion = YYYY . M . D . Hour
fileVersion     = YYYY . M . D . (Minute*100 + Second)
informational   = Display
```
示例：UTC 2025-09-22 07:04:05
```
Display         = 2025.9.22.070405
assemblyVersion = 2025.9.22.7
fileVersion     = 2025.9.22.(04*100 + 05) = 2025.9.22.405
informational   = 2025.9.22.070405
```

### 版本生成逻辑（Nuke 目标 `UpgradeVersion`）
1. 获取当前 UTC 时间。
2. 按格式生成候选版本；若与上次相同秒，做补偿递增。
3. 计算派生四段版本并写回 `version.json` / `.csproj`。
4. 使用 `--lock` 标记锁定（内部记录防止未授权改写）。
5. 提交该文件；否则 PR 与发布检查会失败。

## 🌿 分支策略（精简化）

| 操作 | 要求 |
|------|------|
| 锁定新版本 | 在 `release` 分支执行 `UpgradeVersion --lock` 并提交 |
| 创建发布标签 | 仅可在 `release` 分支祖先 commit 上打 `v<version>` 标签 |
| 功能开发 | `feature/*` 分支开发，合并后再锁定版本 |
| 修复补丁 | 修复合并后重新生成新时间基版本 |

> 版本含义与功能规模解耦：更快发布、避免语义主观判断延迟。

（旧的基于分支+增量示意已废弃）

## 🏷️ 版本号格式（Time-based Display）

```
YYYY.M.D.HHmm
```

示例：`2025.9.22.1547`

优势：
- 线性时序即可判定新旧
- 不需讨论“是否该 minor/major”
- 解析简单，日志与构件命名直接关联

不包含：预发布 / build metadata / hotfix 后缀——额外状态通过 Release Notes 描述；若需要标记内测，使用 GitHub Release `prerelease` flag。若扩展附加信息，可在未来通过 `informationalVersion` 增加 `+meta`。 

## 📝 Conventional Commits（仅用于分类展示）

```bash
# 功能增加 → Minor 版本增量
feat(ui): add new dashboard layout
# 1.2.3 → 1.3.0

# 问题修复 → Patch 版本增量  
fix(auth): resolve login timeout issue
# 1.2.3 → 1.2.4

# 破坏性变更 → Major 版本增量
feat(api)!: redesign REST endpoints
# 或在提交正文中包含 "BREAKING CHANGE:"
# 1.2.3 → 2.0.0
```

### 已废弃标记
`+semver:major|minor|patch|breaking|skip|none` —— 由于不再使用 GitVersion 全部失效，应删除。

### 分类引用示例（供 changelog 抓取）
```
feat: 新增 GPU overlay pipeline
fix: 修复窗口闪烁
refactor: 抽象渲染调度器接口
perf: 降低内存占用 12%
docs: 更新 release 流程
build: 合并矩阵并增加 SHA256 清单
```

## 🔧 常用命令（新版）

```powershell
# 生成并锁定新版本（写入 version.json）
./build.ps1 UpgradeVersion --lock

# 验证版本已锁定
./build.ps1 CheckVersionLocked

# 显示构建信息（含当前锁定版本）
./build.ps1 Info

# 创建安装包（示例）
./build.ps1 Package --rids win-x64,linux-x64
```

### 构建系统命令

```powershell
# 获取构建信息（包含版本）
./build.ps1 Info

# 清理构建输出
./build.ps1 Clean

# 构建项目
./build.ps1 Build

# 运行测试
./build.ps1 Test

# 运行测试并生成覆盖率报告
./build.ps1 Test --coverage

# 发布应用（指定平台）
./build.ps1 Publish --rids win-x64,linux-x64,osx-x64

# 创建安装包
./build.ps1 Package

# 完整的CI构建流程
./build.ps1 Clean Build Test Publish Package
```

### Git 标签与发布

```bash
git tag v2025.121.915304
git push origin v2025.121.915304

# 查看所有标签
git tag -l

# 删除标签（如果需要）
git tag -d v1.4.0
git push origin :refs/tags/v1.4.0
```

**发布策略说明**:
- 仅允许 “锁定版本 + 匹配标签” 发布路径。
- 任何与 `version.json` 不一致的标签会在 `release.yml` 失败。

## 🚀 CI/CD 工作流程（高层）

### 开发流程
1. 功能 / 修复分支 → 合并入 `release`
2. CI 验证（测试 / 质量 / 覆盖率）
3. 需要发布时执行：`UpgradeVersion --lock` → 提交

### 发布流程
1. 创建并推送 `v<locked-version>` 标签
2. `release.yml`：祖先校验 + 并发互斥 + 版本匹配
3. 矩阵打包（所有 RID）→ 验证缺失即 fail-fast
4. 生成分类 changelog + `SHASUMS-<ver>.txt`
5. 创建 GitHub Release（禁用自动 notes，使用自生成 body）

详见： [Release Workflow Guide](./release-workflow.md)

## 📊 版本信息获取

### PowerShell 脚本示例

```powershell
function Get-LockedVersion {
  ($json = Get-Content version.json | ConvertFrom-Json) | Out-Null
  Write-Host "Locked Version: $($json.version)"
  return $json.version
}
Get-LockedVersion | Out-Null
```

### 在代码中获取版本

```csharp
// 在 .NET 应用中获取版本信息
using System.Reflection;

// 获取程序集版本
var assembly = Assembly.GetExecutingAssembly();
var version = assembly.GetName().Version;
var informationalVersion = assembly
    .GetCustomAttribute<AssemblyInformationalVersionAttribute>()
    ?.InformationalVersion;

Console.WriteLine($"Version: {version}");
Console.WriteLine($"Informational: {informationalVersion}");
```

## 🔍 故障排除

### 常见问题和解决方案

#### 1. 版本未锁定
执行：`./build.ps1 UpgradeVersion --lock` 并提交。

#### 2. 标签不匹配
删除错误标签：
```bash
git tag -d v2025.121.915304
git push origin :refs/tags/v2025.121.915304
```
确保 `version.json` 正确后重新创建。

#### 3. 祖先校验失败
标签指向 commit 不在 `release` 分支内 → 在正确基准重新打标签。

#### 4. 缺失某平台产物
矩阵某 Job 失败 → 修复后需删除旧标签重新创建。

### 调试工具

#### 本地版本验证
```powershell
Get-Content version.json | ConvertFrom-Json | Format-List
```

#### 构建问题诊断
```powershell
# 清理并重新构建
./build.ps1 Clean Build

# 检查构建输出
./build.ps1 Build --verbosity detailed

# 验证版本注入
dotnet build --verbosity normal | findstr Version
```

## 🎯 最佳实践

### 1. 发布前检查清单
- [ ] `UpgradeVersion --lock` 已执行并提交
- [ ] CI 全绿（测试/质量/安全）
- [ ] `release` 分支为最新且无未提交
- [ ] 差异审阅清晰（上一个标签..HEAD）
- [ ] 无临时/调试文件

### 2. 分支管理策略
- 保持 main 分支的稳定性
- 功能开发使用 `feature/` 前缀分支
- 紧急修复使用 `hotfix/` 前缀分支
- 及时清理已合并的分支

### 3. 标签管理规范
- 标签 = 版本号的唯一绑定
- 删除标签仅在产物错误且需重新发布时执行
- 使用注释标签记录上下文

### 4. 提交消息规范
- 使用前缀（feat|fix|refactor|perf|docs|build|chore|ci|test）
- 破坏性变更在正文说明迁移策略
- 简洁且可读

## 📚 相关文档

- [发布工作流指南](./release-workflow.md)
- [测试架构文档](./testing-architecture.md)
- [构建系统说明](./build-system.md)
- [项目状态报告](./project-status.md)

---

---
最后更新：2025-09-22 · 文档版本：2.0（迁移至锁定时间序列版本体系）