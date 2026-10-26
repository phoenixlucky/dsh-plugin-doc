# DeepSeek Harness 插件制作与安装规范

[简体中文](README.md) | [English](README.en.md)

本文适用于希望在 DeepSeek Harness（DSH）中开发、分发和安装插件的项目。这里的“插件包”指可由 DSH profile 管理器安装的 npm 包；通常以 bundle 为分发边界，bundle 负责声明配置 patch，并可组合一个或多个 Cordis 插件。

## 1. 先确定插件形态

- **可安装 bundle：** 面向用户分发的完整功能包。包的 `package.json` 声明 `dsh.bundle.patch`，patch 插入或调整 Cordis 配置项。用户通过 DSH profile 管理器安装 bundle。
- **Cordis 插件：** 提供一项服务或功能的代码单元。它可以由 bundle 引用和组合；只有被打包为可安装 bundle 后，才适合按独立插件包向 Harness 用户分发。
- **客户端插件：** 需要浏览器侧代码时，按 DSH 的 `dsh.client` 配置声明入口、平台和注入依赖。将浏览器资源通过宿主提供的资源加载方式打包，不要依赖开发机上的绝对路径。

开发前先确认目标 DSH 版本、运行面（例如 Web 或 Desktop）、需要的宿主服务，以及安装后应出现的用户可见行为。不要只根据一个依赖版本推断 Harness 兼容版本。

## 2. 包名、版本和兼容性

`package.json` 是插件包身份与发布范围的唯一来源。

```json
{
  "name": "@scope/dsh-example",
  "version": "1.0.0",
  "type": "module",
  "engines": {
    "dsh": ">=0.2.1-rc.3 <0.3.0"
  },
  "dsh": {
    "bundle": {
      "patch": "./cordis.patch.yml"
    }
  }
}
```

- 包名使用稳定、唯一的 npm 名称，并在 `cordis.patch.yml`、入口元数据、市场索引、README 和安装命令中保持一致。改名后旧名称不会自动成为别名。
- `version` 是插件自身的 SemVer 版本；`engines.dsh` 才是兼容的 Harness 版本范围。依赖包版本也不能误当成插件版本。
- 每次发布都递增插件版本。发布过的版本不可覆盖；修复需要新版本号。
- 只声明经过验证的 Harness 版本范围。依赖的 DSH API 或 peer 版本应与该范围相符。
- 运行时导入放入 `dependencies`；由 Harness 提供的 API 放入 `peerDependencies`，构建需要的 peer 同时列入 `devDependencies`。

## 3. 代码与生命周期

- 遵循 Cordis 插件约定，使用明确的 `name`、`inject`、配置声明和 `apply` 入口。依赖服务应显式注入。
- 插件启用时创建的监听器、定时器、DOM 节点、样式表和注册项，都必须随插件卸载而清理；重复加载不能遗留重复状态。
- 客户端样式和资源限定在自己的主题或插件 id 范围内。使用 Harness 文档提供的主题变量和扩展点，不依赖生成的 CSS module 类名或无关页面的内部结构。
- 保留宿主已有的设置、持久化和交互行为。动画遵循 `prefers-reduced-motion`，布局为窄窗口设置合理限制。
- 可分发资源放在项目的资源目录中，使用语义化文件名并控制尺寸。资源必须包含在 npm tarball 中，运行时通过宿主资源加载器引用。
- 如需在文档、配置或补丁中引用包名，直接以 `package.json` 为准，避免改名后留下旧 id。

## 4. 发布清单必须完整

检查 `package.json` 的 `files`、`exports`、入口字段和 `dsh.bundle.patch`：

- `main`、`exports` 和 `types` 指向实际存在且准备发布的文件。
- `exports` 中每个目标都必须被纳入 tarball。没有计划分发源码时，不要导出 `src/*`。
- tarball 包含 bundle patch、运行入口、声明文件、locale、图标及代码引用的全部运行时资源。
- 不能依靠开发机上的文件、Harness 安装目录私有路径或未声明的传递依赖。
- `publishConfig.access: "public"` 只声明发布访问级别，不会自动发布包，也不能保证指定 registry 上已经存在该版本。

发布前在干净检出中完成构建，并查看真实打包清单：

```sh
pnpm install --frozen-lockfile
pnpm run build
pnpm pack --dry-run
```

逐项确认输出中的入口、patch、依赖元数据和资源都齐全；检查 `exports` 没有指向 tarball 中不存在的文件。包发生变化后重新生成 tarball，并用目标 registry 查询精确包名和版本。

## 5. 发布到 registry

1. 确认目标 registry、scope 权限、包名和即将发布的版本。
2. 在发布前检查 tarball 文件清单，并确认没有密钥、`.npmrc`、缓存、私有配置或无关源码。
3. 发布到用户实际配置的 registry。公共 scoped 包需设置合适的公开访问权限。
4. 从另一台环境或干净 profile 验证精确版本可查询、可安装、可启用；仅看到发布命令成功不足以证明用户能安装。
5. 记录包名、版本、支持的 DSH 范围和安装方式，并更新对应 README 或市场索引。

查询示例：

```sh
pnpm view @scope/dsh-example@1.0.0 version --registry=https://registry.npmjs.org/
```

Registry 返回 `404` 时，先核对包名、版本和 registry，并确认该版本确实已发布。更换镜像源无法修复拼错、改名或尚未发布的包。若出现认证提示，单独检查 registry 权限；不要把 token 粘贴到日志或 issue。

## 6. 安装、启用与卸载

使用 DSH 插件管理器管理 profile 中的 bundle：

```sh
# 安装 registry 中的精确版本
dsh plugin --profile <profile> add @scope/dsh-example@1.0.0

# 从本地目录安装；Windows 使用绝对路径
dsh plugin --profile <profile> add E:\work\dsh-example

# 从 Git 安装；发布文档应优先固定提交或 release tag
dsh plugin --profile <profile> add git+https://github.com/example/dsh-example.git#<commit>

# 卸载
dsh plugin --profile <profile> remove @scope/dsh-example
```

在 profile 目录里直接执行 `pnpm add` 只会修改 npm 依赖，可能不会同步 DSH 的 bundle 选择和兼容性管理；面向用户的安装说明应使用 `dsh plugin`。开发时可以用本地绝对路径或 `file:` spec，避免把某台电脑的路径写成通用安装方式。

安装后确认 bundle 出现在目标 profile 的插件列表中、状态为启用，并检查插件实际功能。卸载后确认插件注册的条目和资源已撤销，用户数据按插件文档说明保留或删除。

## 7. 安装失败排查

| 现象 | 优先检查 |
|---|---|
| Registry `404` | `package.json.name`、请求的版本、是否已发布、当前 registry 是否正确。不要先反复切换镜像。 |
| `401` / `403` | scope 登录状态、registry 的 token 和包访问权限。隐藏或轮换泄漏的 token。 |
| 兼容性拒绝 | DSH 实际版本、`engines.dsh`、Harness peer 依赖范围和插件发布说明。 |
| 安装成功但插件未加载 | 是否通过 `dsh plugin` 管理、profile 是否正确、bundle patch 与入口是否在包内、依赖是否完整。 |
| 本机可用、另一台失败 | 使用 tarball 文件清单复核是否漏发生成文件；检查是否引用本机路径、未发布依赖或默认分支上不稳定的 Git 内容。 |

向维护者报告问题时提供：完整包名和版本、DSH 版本、安装方式、registry 主机名、错误码和去除 token 后的日志。不要发送 Authorization header、npm token 或完整 `.npmrc`。

## 8. 每次发布前的检查表

- [ ] 包名在 manifest、patch、代码、README 和市场元数据中一致。
- [ ] 插件自身版本与 `engines.dsh` 及依赖版本分别正确。
- [ ] 构建成功；tarball 包含所有导出文件、patch 和运行时资源。
- [ ] `exports` 不指向未发布文件；安装不依赖开发机状态。
- [ ] 目标 registry 能查询精确的 `name@version`。
- [ ] 使用 Harness 插件管理器，在干净 profile 中完成安装、启用和卸载验证。
- [ ] 更新发布说明和安装文档；错误日志不包含凭据。
