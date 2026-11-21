# y700-ota — Y700 四代（TB322FC）OTA 与重签工作流

用 [DynoBox](https://github.com/miner7222/DynoBox) v0.3.6 的官方二进制，在 GitHub Actions
（`ubuntu-24.04`，x86_64）上完成：

1. 下载联想官方 Flash Tool 底包与官方 OTA 包；
2. 把 OTA 增量包应用到 259 底包的分区镜像上；
3. 按需重签 AVB（本仓库默认 **不重签**，见下文）；
4. 用 7-Zip 最高压缩率打成 1900 MiB 分卷，作为 GitHub Release 附件发布。

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
| `tag` | `v358` | Release 标签，产物发布到这里 |
| `resign` | `false` | 是否在应用 OTA 后重签 AVB。259 → 358 官方包保持 `false` |
| `key_name` | `testkey_rsa4096` | `resign=true` 时传给 DynoBox `--key` 的密钥（内置名或路径） |

工作流实际执行的命令（与本地等价）：

```bash
dynobox apply --input <259 镜像目录> --output out --repack 20269121142256-9170.zip
```

选中 `resign` 时追加：

```bash
--resign --key testkey_rsa4096
```

重签需要仓库 Secret `RESIGN_KEY_PEM`（RSA 私钥 PEM），工作流会写入临时文件并传给 `--key`。

## 产物

Release 附件：

- `TB322FC_259-358.7z.001` … 分卷（每卷 1900 MiB，GitHub 单附件上限 2 GiB）
- `SHA256SUMS.txt`

合并与解压：

```bash
cat TB322FC_259-358.7z.0* > TB322FC_259-358.7z
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
