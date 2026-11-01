# rea 分析记录（rea-agents v6.8.0）

> 分析对象：`fczj_207.apk`（com.bolian.motorhome 2.0.7，80,657,814 B）
> SHA-256：`7bc223711a9f33241ff47ffb3d53d2a418f7ba74ad8859b4fc9d1dfa324735d5`
> 执行时间：2026-11-01

## 1. 环境

| 项 | 值 |
| --- | --- |
| rea 版本 | 6.8.0（`npm i -g rea-agents`） |
| Node | **24.22.2**（Node 22 缺 `zstdDecompress` 导出，rea 直接报 SyntaxError，属已知坑位） |
| 可执行 | `/root/.nvm/versions/node/v22.13.1/bin/rea`（需用 Node 24 启动） |
| 分析引擎 | jadx / Ghidra / Hopper / IDA / Kuna **均未配置** |

`rea doctor` 结果：`environment_healthy: false`，8 项检查中 node、host 健康，4 个反编译引擎缺失（属未安装，非故障）。

## 2. 成功执行的命令

### 2.1 identify-flutter-build（有效）

```
rea identify-flutter-build apk/fczj_207.apk --format json
```

| 结论 | 值 |
| --- | --- |
| flutter_detected | true |
| ABI | arm64-v8a |
| libapp.so | 14,943,112 B，sha256 `ae26c66d…609a` |
| libflutter.so | 11,747,864 B，build_id `1e74ae0d…f508` |
| Dart snapshot hash | `0451907c2eaa8467e848c0067bfe8ed4` |
| 覆盖率 | partial |

**用途**：确认竞品为 Flutter 真机构建（非套壳、非 WebView 包），支撑「技术栈判定」一节。

### 2.2 inspect-apk-signature（有效，且有价值）

```
rea inspect-apk-signature apk/fczj_207.apk --format json
```

| 结论 | 值 |
| --- | --- |
| 签名方案 | 仅 **v2**（无 v1、无 v3、无 v3.1、无来源戳 source stamp） |
| 证书主体 | `C=ch, ST=beijing, L=beijing, O=xm, OU=xm, CN=zhu` |
| 证书有效期 | 2020-07-31 → 2047-12-17（RSA） |
| 证书指纹 SHA-256 | `4cf781e483255039583cff4f5a8297005b83f3542f1fe9bf2f690c4548a30e50` |
| 摘要校验 | `matches_apk: true`，`self_consistent: true` |
| 姿态 | 非 v1-only，无签名轮换，无来源戳 |

**解读**：自签名个人证书（`CN=zhu`、`O=xm`），不是企业级 EV 证书；未声明签名轮换，意味着**后续换证书将无法覆盖安装**。这与「小团队自研 + 应用市场分发」的特征一致，不构成山寨证据，但说明无法依赖平台签名做正版校验。

## 3. 失败执行的命令（如实记录）

| 命令 | 返回 | 原因 |
| --- | --- | --- |
| `rea analyze apk/fczj_207.apk` | `capability_unavailable` / `unsupported_provider` | APK 是 archive，Ghidra/Hopper/IDA/Kuna 只接可执行体，且均未配置 |
| `rea inspect-android-package` | `capability_unavailable` | 依赖 jadx-headless-mcp JAR（`REA_JADX_MCP_JAR` 未配置，rea 不代下载） |
| `rea inspect-dart-aot --abi arm64-v8a` | `unsupported_target` | 竞品为**统一快照**（`_kDartSnapshotText` 8.9MB + `_kDartSnapshotData` 5.9MB），rea 只支持 Dart 2.14 的**分离式 FullAOT（kind 3）**，明确报「observed unified Dart snapshot exports」 |

> 结论：rea 对 Flutter 现代构建（统一快照）的 AOT 还原能力有硬缺口；对 **dex/Java 层**能力完整，但本 APK 的 Java 层几乎为空（业务全在 Dart）。

## 4. 替代取证（补齐 rea 无法覆盖的部分）

| 目标 | 方法 | 产出 |
| --- | --- | --- |
| 页面清单 | 从 `libapp.so` 提取 Dart **源文件符号**（`motorhome/page/**/*.dart`），共 323 个业务源文件 | 129 个界面清单 |
| 界面文案 | 自写 UTF-16LE 扫描器（Dart TwoByteString），按关键词提取 ±40 字符上下文 | 取消规则、押金规则、发帖限制等原文 |
| 接口清单 | 提取 URL 路径常量并按前缀聚类 | 60+ 接口，10 个 api 文件归属 |
| 应用名/包名 | 二进制中定位 `AndroidManifest.xml` 的 label（UTF-16LE「房车之家」） | 正版判定依据 |

## 5. 复现命令

```bash
export NVM_DIR=/root/.nvm; . $NVM_DIR/nvm.sh; nvm use 24
R=/root/.nvm/versions/node/v22.13.1/bin/rea
node $R doctor
node $R identify-flutter-build apk/fczj_207.apk --format json
node $R inspect-apk-signature apk/fczj_207.apk --format json
```
