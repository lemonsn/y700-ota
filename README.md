# y700-ota — Y700 四代（TB322FC）OTA 与重签工作流

用 [DynoBox](https://github.com/miner7222/DynoBox) v0.3.6 的官方二进制，在 GitHub Actions
（`ubuntu-24.04`，x86_64）上完成：

1. 下载联想官方 Flash Tool 底包与官方 OTA 包；
2. 把 OTA 增量包应用到 259 底包的分区镜像上；
3. 按需重签 AVB（本仓库默认 **不重签**，见下文）；
4. 用 7-Zip 最高压缩率打成单个 7z（LZMA2、256 MB 字典、273 fast bytes、64 GiB 固实块、4 线程），作为 Actions artifact 上传。

## 数据来源

| 用途 | 地址 | 大小 / 校验 |
|---|---|---|
| 底包 259（Flash Tool 镜像） | `http://zsk-cdn.lenovows.com/知识库/Flash_tool_image/TB322_ZUXOS_1.5.10.259_Tool.7z` | 10,219,346,474 B；7z 密码 `FC(fv:SknR` |
| 官方 OTA 259 → 358 | `https://otanew-cdn.lenovo.com/firmware/20269121142256-9170.zip` | 319,935,310 B；A/B 增量包，43 个分区 |

OTA 包自带的元数据：

- `pre-device=TB322FC`
- `pre-build-incremental=TB322FC_CN_OPEN_USER_Q00041.1_W_ZUXOS_1.5.10.259_ST_260611`
- `post-build-incremental=TB322FC_CN_OPEN_USER_Q00041.1_W_ZUXOS_1.5.10.358_ST_260910`
- `ota-type=AB`

## 运行

Actions → **TB322FC 259 → 358** → Run workflow：

| 输入 | 默认 | 说明 |
|---|---|---|
| `tag` | `v358` | 仅用于 run 备注，不参与产物命名 |
| `resign` | `false` | 是否在应用 OTA 后重签 AVB。259 → 358 官方包保持 `false` |

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

## 产物

构建完成后在 run 页面底部的 **Artifacts** 下载：

- `TB322FC_259-358`：`TB322FC_259-358.7z` + `SHA256SUMS.txt`（单文件，压缩后 <10 GiB）
- 若压缩后 ≥10 GiB，自动切成 `part1/part2/part3` 三个 artifact（每份 ≤4.5 GiB），合并方式：

```bash
cat chunk-aa chunk-ab chunk-ac > TB322FC_259-358.7z
```

- `logs`：`apply.log` 与 `verify.log`，保留 7 天

artifact 保留 30 天，单个 artifact 上限 10 GiB，公共仓库不计存储配额。

下载的 artifact 是 zip，解压后得到上面的文件：

```bash
7z x TB322FC_259-358.7z
```

解压后是 Flash Tool 目录结构：物理分区 `*.img`、重打包的 `super_*.img` 分片和 `rawprogram*.xml`。

产物清单里的 `dynobox-manifest.json` 由仓库自带的 Ed25519 清单签名密钥签：

- 私钥 `keys/integrity_ed25519.pem`，公钥 `keys/integrity_ed25519.pub.pem`
- key id：`003c0da06205af934cb69a287fccb3c379b756a7d60b9f53ea0f171929e09ef4`（原始 32 字节公钥的 SHA-256）
- 校验：`dynobox verify --input out --trusted-integrity-key keys/integrity_ed25519.pub.pem`

DynoBox 另外内置了 `andtabcus` 发布身份的公钥，只有公钥、无私钥；其他签名者必须在
`verify` 时显式 pin 公钥才算 trusted。

## 关于 259 → 358 不重签

官方 OTA 是增量包，DynoBox 应用后会改写 `system`/`product`/`vendor` 等分区内容，但
`--resign` 未开启时不会更新 AVB 签名与描述符摘要。因此：

- 产物的 AVB 校验（`dynobox verify`）大概率报摘要不匹配，这是预期结果，工作流会记录
  实际退出码与日志，但不会因此中断打包；
- 需要可被 bootloader 正常校验的镜像时，用 `resign=true` 重签；换 key 后设备还需要
  接受该公钥的 `abl.elf`，否则会拒绝启动（DynoBox 会给出同样警告）。

## 免责声明

仅供学习与开发用途。修改固件可能导致设备变砖、数据丢失或安全能力下降，风险自负。
