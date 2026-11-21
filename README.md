# y700-ota — Y700 四代（TB322FC）OTA 与重签工作流

用 [DynoBox](https://github.com/miner7222/DynoBox) v0.3.6 的官方二进制，在 GitHub Actions
（`ubuntu-24.04`，x86_64）上完成：

1. 下载联想官方 Flash Tool 底包与官方 OTA 包；
2. 把 OTA 增量包应用到 259 底包的分区镜像上；
3. 按需重签 AVB（本仓库默认 **不重签**，见下文）；
4. 用 7-Zip 最高压缩率打成单个 7z（LZMA2、256 MB 字典、273 fast bytes、64 GiB 固实块、2 线程），作为 Actions artifact 上传。

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
| `key_name` | — | 已弃用：重签固定使用 `keys/testkey_rsa4096.pem`（或 `RESIGN_KEY_PEM` secret） |

工作流实际执行的命令（与本地等价）：

```bash
dynobox apply --input <259 镜像目录> --output out --repack 20269121142256-9170.zip
```

选中 `resign` 时追加：

```bash
--resign --key keys/testkey_rsa4096.pem
```

仓库自带的 `keys/testkey_rsa4096.pem` 是 AOSP 公共测试密钥（4096 位 RSA，来自
[showdo/AOSP-Testkey-BootSigner](https://github.com/showdo/AOSP-Testkey-BootSigner)），
与 DynoBox/avbtool-rs 内置的 `testkey_rsa4096` 是同一把。需要换成自己的密钥时，
配置仓库 Secret `RESIGN_KEY_PEM`（RSA 私钥 PEM），工作流会优先使用它。

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

## 关于 259 → 358 不重签

官方 OTA 是增量包，DynoBox 应用后会改写 `system`/`product`/`vendor` 等分区内容，但
`--resign` 未开启时不会更新 AVB 签名与描述符摘要。因此：

- 产物的 AVB 校验（`dynobox verify`）大概率报摘要不匹配，这是预期结果，工作流会记录
  实际退出码与日志，但不会因此中断打包；
- 需要可被 bootloader 正常校验的镜像时，用 `resign=true` 重签；换 key 后设备还需要
  接受该公钥的 `abl.elf`，否则会拒绝启动（DynoBox 会给出同样警告）。

## 免责声明

仅供学习与开发用途。修改固件可能导致设备变砖、数据丢失或安全能力下降，风险自负。
