# y700-ota — Y700 四代（TB322FC）OTA 与重签工作流

用 [DynoBox](https://github.com/miner7222/DynoBox) v0.3.6 的官方二进制，在 GitHub Actions
（`ubuntu-26.04`，x86_64）上完成：

1. 下载联想官方 Flash Tool 底包与官方 OTA 包；
2. 把 OTA 包应用到指定底包的分区镜像上（259 → 358，以及以 358 为底包的 086 / 018）；
3. 按需重签 AVB（本仓库默认 **不重签**，见下文）；
4. 用 7-Zip 最高压缩率打成单个 7z（LZMA2、256 MB 字典、273 fast bytes、64 GiB 固实块、4 线程），作为 Actions artifact 上传，再分块发布到 Release。

## 数据来源

| 用途 | 地址 | 大小 / 校验 |
|---|---|---|
| 底包 259（Flash Tool 镜像） | `http://zsk-cdn.lenovows.com/知识库/Flash_tool_image/TB322_ZUXOS_1.5.10.259_Tool.7z` | 10,219,346,474 B；7z 密码 `FC(fv:SknR` |
| 官方 OTA 259 → 358（9170） | `https://otanew-cdn.lenovo.com/firmware/20269121142256-9170.zip` | 319,935,310 B；A/B 增量包，43 个分区；目标 1.5.10.358_ST |
| 官方 OTA 358 → 086（3974） | `https://otanew-cdn.lenovo.com/firmware/20261012103144828-3974.zip` | 11,400,239,510 B；full 包；目标 **2.5.02.086_DEV**，Android 17 |
| 官方 OTA 358 → 018（1534） | `https://otanew-cdn.lenovo.com/firmware/20261111112627152-1534.zip` | 11,326,230,438 B；full 包；目标 **2.5.10.018_ST**，Android 17 |

9170 包自带的元数据：

- `pre-device=TB322FC`
- `pre-build-incremental=TB322FC_CN_OPEN_USER_Q00041.1_W_ZUXOS_1.5.10.259_ST_260611`
- `post-build-incremental=TB322FC_CN_OPEN_USER_Q00041.1_W_ZUXOS_1.5.10.358_ST_260910`
- `ota-type=AB`

## 运行

Actions → **TB322FC 259 → 358** → Run workflow：

| 输入 | 默认 | 说明 |
|---|---|---|
| `tag` | `v358` | 仅用于 run 备注，不参与产物命名 |
| `resign` | `false` | 是否在应用 OTA 后重签 AVB。官方包保持 `false` |

三个构建工作流：

| 工作流 | 作用 |
|---|---|
| `ota.yml` | 259 底包 + 9170 → 358 |
| `ota-086.yml` | 358 底包 + 3974 → 086 |
| `ota-next.yml` | 358 底包 + 指定 OTA → 任意版本（`version` 输入决定命名），018 用它 |

发布工作流：

| 工作流 | 作用 |
|---|---|
| `publish.yml` | 单个 artifact → 分块 → Release（可 `asset_name` 改名） |
| `publish-chunks.yml` | 多个 artifact 拼回 → 校验 → 重切 → Release |
| `publish-originals.yml` | 三个原始包一次发布 |
| `publish-ota-package.yml` | 把某个 OTA 原包追加到已有 Release |

工作流跑在 `ubuntu-26.04`（x86_64，glibc 2.43）。DynoBox 官方 v0.3.6 的 x86_64
二进制链接了 `GLIBC_2.43`；`ubuntu-24.04`（也就是当前的 `ubuntu-latest`）只有 2.39，
无法执行，所以不能用 `ubuntu-latest`。

工作流实际执行的命令（与本地等价）：

```bash
dynobox apply --input <259 镜像目录> --output out --repack 20269121142256-9170.zip
```

选中 `resign` 时追加：

```bash
--resign --key keys/testkey_rsa4096.pem
```

仓库自带的 `keys/testkey_rsa4096.pem` 取自
[showdo/AOSP-Testkey-BootSigner](https://github.com/showdo/AOSP-Testkey-BootSigner/blob/main/tools/pem/testkey_rsa4096.pem)
（4096 位 RSA，文件 SHA-256 `88bd0278559c5f54b709560987f2f5dd67f1afe42b2c6576a2d5fdf5fe0c6d69`），
与 DynoBox/avbtool-rs 内置的 `testkey_rsa4096` 密钥材料相同（PKCS#1/CRLF 与 PKCS#8/LF 的编码差异）。
工作流在 `resign=true` 时直接使用这份副本并校验哈希，不再支持外部 secret 覆盖。

## 已发布的 Release

| Release | 目标版本 | 渠道 | Android / SDK | SPL | 资产 |
|---|---|---|---|---|---|
| `v358-unsigned` | 2.5.10.358 | ST | 16 / 36 | 2026-07-05 | `TB322FC_358_images.7z.001`–`.007` + `SHA256SUMS.txt`，9.2 GiB |
| `v086-unsigned` | **2.5.02.086** | **DEV** | 17 / 37 | 2026-09-05 | `TB322FC_086_images.7z.001`–`.007` + `SHA256SUMS.txt`，9.97 GiB |
| `v018-unsigned` | **2.5.10.018** | **ST** | 17 / 37 | 2026-11-05 | `TB322FC_018_images.7z.001`–`.007` + `SHA256SUMS.txt`，9.89 GiB |
| `originals` | — | — | — | — | 259 底包 7 块、`20269121142256-9170.zip`、086 OTA 8 块、018 OTA 8 块，共 30.98 GiB |

版本关系：259 与 358 同属 ZUXOS 1.5.10（Android 16 / SDK 36），**358 就是 259 的迭代版本**
（同一分支上的下一个 ST 构建，9170 这个 305 MB 增量包只改动差异分区）；086 是 Android 17 的
ZUXOS 2.5.02 **DEV** 开发版；018 是 Android 17 的 ZUXOS 2.5.10 **ST** 稳定版，比 086 更新
（2.5.10 > 2.5.02，SPL 2026-11-05 > 2026-09-05）。086 与 018 都是 full 包，都从 358 底包应用。

三个镜像集都是「不重签」状态。合并与解压：

```bash
cat TB322FC_358_images.7z.0* > TB322FC_358_images.7z
7z x TB322FC_358_images.7z

# 259 底包带密码
cat TB322_ZUXOS_1.5.10.259_Tool.7z.0* > TB322_ZUXOS_1.5.10.259_Tool.7z
7z x -p'FC(fv:SknR' TB322_ZUXOS_1.5.10.259_Tool.7z
```

官方包对应关系：9170 = 259 → 358（增量，305 MB，同属 1.5.10 ST 的相邻构建）、
3974 = 358 → 086（full，11.40 GB、2.5.02.086_DEV）、1534 = 358 → 018（full，11.33 GB、2.5.10.018_ST）。

## 构建产物（Artifacts）

构建 run 页面底部的 **Artifacts** 里是中间产物，保留 30 天；发布到 Release 的是仓库里的长期副本。

- 压缩后 <10 GiB 时上传单个 artifact（含 `*.7z` 与 `SHA256SUMS.txt`）
- ≥10 GiB 时切成 `part1/part2/part3` 三个 artifact（每份 ≤4.5 GiB），用 `publish-chunks`
  工作流拼回并重新按 1.5 GiB 分块发布
- `logs`：`apply*.log` 与 `verify*.log`，保留 7 天

单个 artifact 上限 10 GiB，公共仓库不计存储配额。

产物清单里的 `dynobox-manifest.json` 由仓库自带的 Ed25519 清单签名密钥签：

- 私钥 `keys/integrity_ed25519.pem`，公钥 `keys/integrity_ed25519.pub.pem`
- key id：`003c0da06205af934cb69a287fccb3c379b756a7d60b9f53ea0f171929e09ef4`（原始 32 字节公钥的 SHA-256）
- 校验：`dynobox verify --input out --trusted-integrity-key keys/integrity_ed25519.pub.pem`

DynoBox 另外内置了 `andtabcus` 发布身份的公钥，只有公钥、无私钥；其他签名者必须在
`verify` 时显式 pin 公钥才算 trusted。

## 关于不重签

官方 OTA 应用后会改写 `system`/`product`/`vendor` 等分区内容，而 `--resign` 未开启时
不会重新签名。实际结果是：

- **动态分区被打包进 `super_*.img`**，`verify` 对它们退化为对 super 容器的签名校验，
  AVB 语义验收因此通过。358、086、018 三次构的 `dynobox verify` 退出码都是 0。
- 物理分区（`boot`、`vbmeta` 等）保持 OTA 包里的官方签名与描述符，full 包（086/018）
  整体替换时自洽。
- 但请你关心的是：**改了分区内容但没有重新签名后开机是否被 bootloader 拒**——如果
  后续用 `.dbp`、debloat 等方式再改分区字节，就必须用 `resign=true` 重签；换 key 后
  设备还需要接受该公钥的 `abl.elf`，否则拒绝启动。

## 免责声明

仅供学习与开发用途。修改固件可能导致设备变砖、数据丢失或安全能力下降，风险自负。
