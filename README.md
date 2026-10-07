# Batocera 移植到 Bozztek G03C1322（RK3288, bob）

把这套文件整个放进你的 fork `zjj3555/batocera.linux` 根目录（对应目录已排好，直接覆盖/新增即可），
然后跑 GitHub Actions 构建，把生成的 `batocera-*.img.gz` 用 **adb + dd** 整盘写进 eMMC 就能开机。

---

## 一、你的硬件（已通过 5.1 / 7.1.1 官方固件核对）

| 项目 | 值 |
|---|---|
| SoC | RK3288（rk3288-evb-act8846 参考设计） |
| 内存 | 2 GB（bootargs `vmalloc=512M`） |
| eMMC | 32 GB，8-bit（`/dev/block/mmcblk0`） |
| 显示 | **EDP 1920×1080**，工厂时序 **108 MHz**（不是常见的 148.5 MHz！） |
| 背光 | pwm0 + enable GPIO7_A2 |
| 面板电源 | vcc_lcd = GPIO7_A3 |
| 面板使能 | GPIO5_C2 |
| 以太网 | RK3288 GMAC，RGMII，100M，reset GPIO4_A7 |
| PMIC | ACT8846 @ i2c0/0x5a + syr827/syr828 |
| 触摸 | wdt_ts_i2c @ i2c4/0x2c（**mainline 无驱动，暂不工作**） |
| TF 卡槽 | 无（所以只能整盘写 eMMC） |

## 二、文件放哪（镜像 fork 目录结构）

| 本目录下的文件 | 放到 fork 的位置 |
|---|---|
| `board/batocera/rockchip/rk3288/linux_patches/0001-boards-rk3288-g03c.patch` | **新增**（内核补丁：加 G03C 设备树 + EDP 面板时序） |
| `configs/batocera-rk3288.board` | **覆盖**（DTS 改为只构建 `rk3288-g03c`） |
| `board/batocera/rockchip/rk3288/tinkerboard/create-boot-script.sh` | **覆盖**（boot 里拷入 rk3288-g03c.dtb） |
| `board/batocera/rockchip/rk3288/tinkerboard/boot/extlinux.conf` | **覆盖**（FDT 指向 rk3288-g03c.dtb） |
| `.github/workflows/build-g03c.yml` | **新增** |
| `dev-src/`（rk3288-g03c.dts + gen_patch.py） | **不要提交**，只是给维护者看/重新生成补丁用的源码 |

> `0001-boards-rk3288-g03c.patch` 已把 dts 内容和 panel-edp.c 改动打包好，`dev-src/` 里的 dts 只是给人读的副本。

> 原理：Batocera 的 rk3288 用一个 U-Boot multiboard + genimage 产出整盘镜像。我们**复用 tinker 的 U-Boot**
>（它已在你的 G03C 上验证能启动内核），只把**内核 DTB** 换成 G03C 专属的。extlinux 显式加载 `rk3288-g03c.dtb`，
> 内核用 EVB-act8846 的 PMIC/以太网/eMMC 配置 + G03C 的 EDP 面板。

## 三、设备树要点（rk3288-g03c.dts，已含在补丁里）

基于 mainline `rk3288-evb.dtsi`（G03C 就是 EVB-act8846 参考设计），只做三处 G03C 覆盖：
1. **面板**：`compatible = "bozztek,g03c-1080p"`，使能脚 `GPIO5_C2`，电源 `vcc_lcd`（GPIO7_A3）。
   108 MHz 时序写进 `drivers/gpu/drm/panel/panel-edp.c`（mainline 的 lg,lp079qx1 是 1536×2048，不是你的屏）。
2. **禁掉 TF（sdmmc）**：G03C 没有 TF 卡槽。
3. 保持 EVB 的 ACT8846 PMIC / RGMII 以太网 / 8-bit eMMC / 2GB 内存。

## 四、构建

**方式 A：GitHub Actions（推荐）**
1. 把上面的文件提交推送到你的 fork。
2. 仓库 → **Actions** → 左侧选 **Build Batocera G03C (rk3288)** → **Run workflow**。
3. 构建约 2–6 小时（首次要下载 buildroot 子模块 + 全部源码，十几 GB）。完成后在 Actions 的 **Artifacts** 里下载 `batocera-g03c`，解压得到 `batocera-*.img.gz`。

**方式 B：本地 Docker 构建（Linux/macOS）**
```bash
git clone https://github.com/zjj3555/batocera.linux
cd batocera.linux
git submodule update --init --recursive
make pull-docker-image
make rk3288-build
# 产物：output/rk3288/images/batocera/images/g03c/batocera-*.img.gz
```

## 五、刷进 eMMC（无 TF 卡槽 → adb + dd 整盘写）

> 关键点：RKDevTool 写不进 LBA0（MBR 引导区），所以用 **Android 侧 adb 整盘 dd**，这个流程你已验证过。

1. **先刷回 Android 5.1 救砖包**（RKDevTool → 升级固件 → `D:\rk3288\G03C_rk3288(bob)_update.img`），等它开机、网络 adb 可用。
2. 连 adb（`adb connect <盒子IP>:5555`）→ `adb root`。
3. 把镜像推到盒子：
   ```bash
   adb push batocera-*.img /sdcard/g03c.img
   ```
4. 整盘写入（注意 5.1 的 dd 不支持 `bs=4M`/`conv=fsync`）：
   ```bash
   adb shell "dd if=/sdcard/g03c.img of=/dev/block/mmcblk0 bs=1048576"
   ```
5. `adb reboot`，拔电重启。**首次约 1–2 分钟**扩展系统分区。

## 六、首次开机看什么

- **LAN 灯亮 + 活动灯闪** = U-Boot 和内核起来了（你已见过）。
- **路由器 DHCP 出现新设备 / 能 SSH**（`root` / `linux`）＝启动链通、以太网正常。
  > LibreELEC tinker 之前没 IP，是因为 tinker 的 PHY 配置和 G03C 不符；本 DTB 用的是 EVB 的 RGMII 配置，和 G03C 工厂一致，应该能拿到 IP。
- **屏幕亮 + 显示 Batocera 界面** = EDP 面板时序对了（108 MHz）。这是本次移植的核心验证点。
- 触摸暂不工作（wdt 驱动缺失）——用 USB 手柄/键盘操作即可，符合游戏系统定位。

## 七、如果屏幕仍黑

按优先级排查：
1. 接 HDMI 试一下（EVB 里 HDMI 仍是 enable 的，若 G03C 有 HDMI 口能看到输出，说明内核起来了，只是 EDP 面板问题）。
2. 把 `panel-edp.c` 里 `bozztek_g03c_1080p_mode` 的 `clock` 改成 `148500`（标准 1080p），重跑构建——看是不是这个面板其实要 148.5 MHz。
3. 确认背光：`/sys/class/backlight/*/brightness` 手动调到 255 试试。

## 八、回滚

随时用 RKDevTool 刷回 `G03C_rk3288(bob)_update.img`（5.1），一切恢复如初。
