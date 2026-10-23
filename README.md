# 小米 MIX 2S（polaris / 骁龙 845）iOS 16 移植记录

> 目标设备：Xiaomi MIX 2S（代号 polaris，SDM845，6GB 内存，指纹为 Synaptics S3330 触摸）
> 目标系统：iOS 16.7.16（20H392，iPhone 10,3 / D22AP 用户态）
> 本文记录从零开始到真机启动的完整过程：路线选择、遇到的每个问题、根因和最终解法。

## 结论（截至本文）

真机已通过 `fastboot boot` 临时引导成功启动，进入 iOS 用户态。

最终可用的镜像定义：

| 组件 | 来源 / 做法 | 校验 |
|---|---|---|
| 内核 | [luuu20309/ios16_on_polaris_mix2s](https://github.com/luuu20309/ios16_on_polaris_mix2s) Release `polaris-kernel-0090` 的 `kernel.release.sweet`（Mach-O，12,285,320 B） | SHA-256 `9c19baf0c804992bc72dc0a436ca23a36688f2e22db49a168fb4da24c344c3df` |
| 设备树 DTB | 从本机原厂 `boot.img` 提取全部 14 个 polaris DTB（`tools/polaris_dtb.py --from-boot`） | 30,355,119 B |
| ADT | `tools/mkadt.py --polaris --ram 6` + iPhone X 16.7.16 设备树 `d22ap.adt`（补 `/product` 节点） | ADT 12,732 B |
| payload | `tools/mkpayload.py --check-polaris-cpus --max-payload-size 0xdea000` | 11,742,016 B，头 `SWEETXNU` |
| 引导镜像 | `PLATFORM=polaris shim/build.sh`（Android boot.img v1，kernel 槽 = gzip(Image)+DTB 拼尾） | 42,115,072 B |
| rootfs | [luuu20309/ios16_on_polaris_mix2s](https://github.com/luuu20309/ios16_on_polaris_mix2s) Release `ios-rootfs-16.7.16-12`（iPhone10,3 官方 IPSW 制作，HFS+ 16G 稀疏镜像） | 按发布方 CHECKSUMS |

刷写方式：`fastboot boot shim-polaris-xnu.img`（**临时引导，不写 boot 分区**，重启即回 Android）。

---

## 一、路线选择

市面有三条相关路线：

1. **ios-sweet**（dipa-studio，红米 Note 10 Pro / sweet，SM7150）：已有完整 XNU 补丁集（平台专家、触摸、显示、SMC 等），并且真机点亮过主屏。
2. **iOS16_on_mix2s**（JaneAwA，MIX 2S）：把 sweet 补丁集移植到 polaris，做到 M3.5（自写 init 当 PID 1）。
3. **ios16_on_polaris_mix2s**（luuu20309，MIX 2S）：独立推进到 0090 RELEASE，真机进锁屏、单双指触摸可用。

最终采用：**内核与 rootfs 来自 3，引导链（shim / ADT / payload 打包）自己按 1 的框架搭建**，全部产物本地自构建，不依赖任何人的预编译镜像。

---

## 二、遇到的坑与解法（按时间线）

### 坑 1：fastboot boot 任何自构建镜像都黑屏（ABL 不交权）

- **现象**：`fastboot boot` 返回 `Booting OKAY`，但屏幕背光亮、纯黑、挂死不重启。连"涂色探针"（shim 第一条指令往 0x9d400000 涂绿、0x9c000000 涂红后死循环）都不出颜色。
- **排查**：探针无颜色 ⇒ ABL 根本没把控制权交给 shim，不是画错地址。
- **根因**：MIX 2S 的 ABL 只认**高通 Image.gz-dtb 形态**——gzip 压缩的内核 + 明文 DTB 拼在 gzip 流尾部，boot header 用 **v1**。裸 arm64 Image（未压缩）或 v2 独立 dtb 槽会导致 ABL 校验/加载失败而不交权。
- **解法**：
  ```bash
  gzip -9 -c shim/out/Image > shim/out/Image.gz
  cat shim/out/Image.gz dtb/boot_unpacked/dtb > shim/out/Image.gz-dtb
  mkbootimg --header_version 1 --kernel out/Image.gz-dtb --ramdisk payload.bin ... -o shim-boot.img
  ```
- **验证**：同样的探针在 gzip+DTB 形态下真机出全绿屏，证明 ABL 交权、framebuffer 地址（0x9d400000）、shim 控制台全部正常。

### 坑 2：CI 构建"成功"但内核缺一半驱动（补丁静默跳过）

- **现象**：CI 流水线 conclusion=success，但真机行为不对；内核字符串检查发现 `SweetCPU`、`AppleCLCD` 等符号全没了。
- **根因**：CI 用的 [blacktop/darwin-xnu-build](https://github.com/blacktop/darwin-xnu-build) `build.sh` 打补丁逻辑是：
  ```bash
  if git apply --check "$PATCH"; then git apply "$PATCH"; else warning "Skipping patch ..."; fi
  ```
  **补丁应用失败只打 warning 并静默跳过**。给同一个文件的某个补丁"加行"后，后续补丁的 hunk 行号偏移，全部 `--check` 失败被跳过。
- **踩雷实录**：给 0016（平台专家）加 8 行 → 同文件的 0018/0038/0040/0046/0058/0062 全部跳过；改 0054 的时间参数 90→20 → 0055 的上下文不匹配被跳过，并连坐 0062/0063。
- **解法**：
  - 修改补丁必须**零行数变化**（用字符串替换而不是加行）；
  - 每次构建后把日志里的 `skipped` 列表过一遍，确认没有意外跳过；
  - 需要改行为时优先用 **boot-arg**（如 `sweetsample=...` 采样名单）而不是改代码。

### 坑 3：panic "Unable to find driver for this platform: D22APiPhone10,3xiaomi,polarisqcom,sdm845appleARMPlatform"

- **现象**：XNU 启动到 IOPPlatformExpert 后 panic，平台名是一串**没分隔符的脏串**。
- **根因**：ADT 的 `compatible` 属性是逐字节拼接的（`b'+'`），而设备树要求 **NUL 分隔的字符串列表**，匹配必然失败。
- **解法**：`mkadt.py` 用 `strlist()/cstr()` 写 NUL 分隔；参考项目 README 第 7 节有同样记录。

### 坑 4：panic "Unable to find driver for this platform: D22AP"（干净版）

- **现象**：ADT 修好后平台名干净了，但仍 panic——内核里没有任何 personality 认领 `D22AP`。
- **根因**：`KernelConfigTables.cpp` 的平台专家/CPU/显示条目全部硬编码 sweet：
  - `IONameMatch = 'xiaomi,sweet'`、`SweetCPU` 认 `qcom,kryo470`、`AppleCLCD` 认 `sweet,display`
  - 而 polaris 的 compatible 是 `D22AP` / `qcom,kryo385` / `polaris,display`
- **解法**：加 polaris 分支补丁（`IONameMatch='D22AP'` 平台专家、`qcom,kryo385` CPU、`polaris,display` 显示），并把补丁里所有 `xiaomi,sweet` / `qcom,kryo470` / `sweet,display` 做**全局字符串替换**（保证行数不变、上下文一致）。

### 坑 5：Apple logo 之后变黑，SpringBoard 无限崩溃（BoundingPathBitmap）

- **现象**：能看到 Apple logo，随后屏幕变暗黑掉；20 分钟不动。TWRP 裸 dd 拉取 userdata 尾部的 4MB 内核日志环，发现 SpringBoard **422 次同一崩溃**：
  ```
  *** Assertion failure ... _UIScreenComplexBoundingPathUtilities _loadBitmapForScreen:type:
  BoundingPathBitmap-1125x2436-375x812-3.00x
  ```
- **根因**：SpringBoard 准备进锁屏（`SBStartupTransitionToLockOut`）时，UIKit 按"iPhone X 圆角屏"去加载 1125x2436 的圆角位图，找不到就 `NSInternalInconsistencyException` 崩溃循环。而 ADT 缺 iPhone X 真机的 **`/product` 节点**——`mkadt.py` 生成 ADT 时若没有 `research/ipsw-x/d22ap.adt`，就静默跳过 `/product`，MobileGestalt 拿不到 iPhone X 的身份（product type / device class / artwork），UIKit 走错路径。
- **解法**（来自 [参考项目文档](https://github.com/luuu20309/ios16_on_polaris_mix2s/blob/main/docs/POLARIS-MIX2S.md)）：
  1. 用 `tools/remotezip.py` 从 iPhone X iOS 16.7.16 (20H392) IPSW **只远程提取** `Firmware/all_flash/DeviceTree.d22ap.im4p`（33,719 B，不用下 6GB 整包）：
     ```
     https://updates.cdn-apple.com/2025FallFCS/fullrestores/122-77006/1158E502-D76D-4226-AA06-ED5E30CD1111/iPhone10,3,iPhone10,6_16.7.16_20H392_Restore.ipsw
     ```
  2. 解包：im4p 是 DER 包裹的，payload 为 LZFSE 压缩——`pip install lzfse` 解出 186,084 B 的 `d22ap.adt`（Apple ADT 二进制，不是 FDT）。
  3. 放到 `research/ipsw-x/d22ap.adt` 后重跑 `mkadt.py --polaris --ram 6`：ADT 从 10,048 B 变成 **12,732 B**，`strings` 可见 `product-name` / `iPhone X` / `D22AP`。
- **验证**：新镜像启动后 SpringBoard 不再崩该断言，顺利进系统。

### 坑 6：GitHub Actions macOS runner 排队失败

- **现象**：`The job was not acquired by Runner of type hosted even after multiple attempts`。
- **解法**：用 token 走 API 自动触发 `workflow_dispatch` 并轮询结果：
  ```
  POST /repos/<owner>/<repo>/actions/workflows/xnu.yml/dispatches
  {"ref":"polaris-port","inputs":{"macos_version":"13.5","kernel_config":"DEVELOPMENT","machine_config":"POLARIS","keep_going":"true"}}
  ```
  注意：artifact 下载时 `/actions/artifacts/<id>/zip` 会 302 到 Azure Blob，**不能带 Authorization 头跟随重定向**（会 401），要先取 Location 再裸下载。

### 坑 7：shim 链接报错 PHDR segment not covered by LOAD segment

- **现象**：新版 binutils（2.42）链接 shim.ld 时报 `error: PHDR segment not covered by LOAD segment`。
- **解法**：`shim.ld` 加显式段表，只动段定义、不改代码：
  ```
  PHDRS { shim PT_LOAD FLAGS(7); }
  .text : { ... } :shim
  .data : { ... } :shim
  .bss  : { ... } :shim
  ```

### 坑 8：误跑 main 分支导致 "Unsupported CURRENT_MACHINE_CONFIG POLARIS"

- **现象**：CI 报 `MakeInc.kernel:22: *** Unsupported CURRENT_MACHINE_CONFIG POLARIS. Stop.`
- **根因**：网页点 `Run workflow` 时 "Use workflow from" 下拉框默认是 **main**（上游分支，没有 0099/0100 的 polaris machine config），而不是 `polaris-port`。
- **解法**：下拉框手动改选 `polaris-port`；用 API 触发时 `ref` 必须写 `polaris-port`。

---

## 三、最终打包命令（完整复现）

Linux 主机（Ubuntu），依赖：`aarch64-linux-gnu-gcc`、`gcc`、`make`、`cpio`、`lzfse` python 包。

```bash
# 0. 目录准备
git clone https://github.com/luuu20309/ios16_on_polaris_mix2s.git
cd ios16_on_polaris_mix2s
mkdir -p out research/ipsw-x

# 1. 设备树提取（原厂 boot.img → 14 个 polaris DTB）
python3 tools/polaris_dtb.py --from-boot boot.img out/polaris-dtb.bin

# 2. iPhone X 设备树（补 /product 节点）
python3 tools/remotezip.py "<IPHONE_X_IPSW_URL>" 'DeviceTree.d22ap.' research/ipsw-x
python3 - <<'EOF'
import lzfse
d = open('research/ipsw-x/Firmware/all_flash/DeviceTree.d22ap.im4p','rb').read()
# DER: 取最后一个 OCTET STRING 作为 payload（33,650 B，LZFSE）
i = 0
def rd(b, i):
    t = b[i]; i += 1
    l = b[i]; i += 1
    if l & 0x80:
        n = l & 0x7f; l = int.from_bytes(b[i:i+n],'big'); i += n
    return t, b[i:i+l], i+l
t, seq, _ = rd(d, 0); i = 0; payload = None
while i < len(seq):
    t, v, i = rd(seq, i)
    if t == 0x04: payload = v
open('research/ipsw-x/d22ap.adt','wb').write(lzfse.decompress(payload))
EOF

# 3. ADT（注意 --ram 6 与手机容量一致）
python3 tools/mkadt.py out/polaris.adt --polaris --ram 6
#    无 "no /product node" 警告，12,732 B

# 4. payload
python3 tools/mkpayload.py kernel.release.sweet out/polaris.adt out/polaris-payload.bin \
  --cmdline "debug=0xc serial=3 cpus=4 keepsyms=1 -enable_kprintf_spam -noprogress rd=ufsroot0 unentitled_ios_sim_launch=1 ignition_level=1 wdt_test=no_user_checkin ufsrw=1 rootrw=1 launchd_unsecure_cache=1 wdt=-1 sweetfb_noconsole=1 sweetfb_hz=30 sweetfb_polaris=1 sweettouch=0 sweetkeys=0 polaristouch=1 polaristouchhid=1 polaristouchreset=1" \
  --check-polaris-cpus --max-payload-size 0xdea000

# 5. 引导镜像（shim）
PLATFORM=polaris shim/build.sh out/polaris-payload.bin
#   → out/shim-polaris-xnu.img（v1 头，kernel 槽 = gzip(Image)+DTB 拼尾）
```

> 注意：payload 大小必须小于 `0xdea000`（ABL 的 ramdisk→DTB 边界），`--max-payload-size` 会在超限时直接拒绝。

## 四、真机操作

```bash
# 只临时引导，不写 boot 分区
fastboot boot shim-polaris-xnu.img
```

- 需要解锁 bootloader、官方 fastboot 能识别设备（`fastboot devices` 有且只有一个 polaris）。
- rootfs 刷入（会清空 userdata，务必先备份）：
  ```bash
  fastboot -S 512M flash userdata ios16-root.simg
  ```

## 五、日志获取方法（无串口时）

内核补丁把控制台日志写在 `userdata` 分区末尾 1025 个 4KiB 块（4MB ring），**不需要挂载分区**：

```bash
# 在 TWRP / OrangeFox recovery（有 adb root）
adb shell "tail -c 4194304 /dev/block/by-name/userdata > /tmp/sweet-log.bin"
adb pull /tmp/sweet-log.bin
```

然后本地 `tools/sweet-log.py` 解码（或直接看文本，ring 内全是可打印字符）。

---

## 六、参考

- [dipa-studio/ios-sweet](https://github.com/dipa-studio/ios-sweet) — XNU 补丁集与 shim 框架（sweet/红米 Note 10 Pro）
- [JaneAwA/iOS16_on_mix2s](https://github.com/JaneAwA/iOS16_on_mix2s) — 最早的 MIX 2S 移植（M3.5）
- [luuu20309/ios16_on_polaris_mix2s](https://github.com/luuu20309/ios16_on_polaris_mix2s) — 当前主力路线（0090，锁屏 + 触摸）
- [blacktop/darwin-xnu-build](https://github.com/blacktop/darwin-xnu-build) — CI 构建脚本（补丁静默跳过的来源）

## 七、已知未完成 / 后续

- 触摸：0090 内核已带 polaristouch 驱动，真机验证需逐个开到 `polaristouchhid=1`（本镜像已带）。
- GPU（Adreno 630）：参考项目仍在离线预检/只读诊断阶段，无可用加速。
- 8 核（A75）：`--check-polaris-cpus` 只验证 4 核；5-8 核需单独逐级验证。
- 温度/电池/关机键：驱动未实现。
