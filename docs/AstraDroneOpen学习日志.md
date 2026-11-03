# AstraDroneOpen 学习日志

记录日期：2026 年 11 月 3 日

学习目标：在 Windows + WSL2 + Ubuntu 20.04 中搭建 AstraDroneOpen 开发环境，理解项目结构，并启动基础无人机仿真。

本文结合个人操作记录、排错过程和仓库文档整理。Windows 网络配置中遗忘的参数列为参考配置，不代表已经核实本机实际设置。

## 一、安装 Ubuntu 20.04，配置 WSL 网络

### 1. 安装环境

使用 WSL2 运行 Ubuntu 20.04。项目主要使用 ROS Noetic，并结合 PX4、Gazebo 等组件进行仿真。

在 Windows PowerShell 中可以查看发行版名称和 WSL 版本：

```powershell
wsl --list --verbose
wsl --version
```

如果尚未安装 Ubuntu 20.04，可以先查询可安装列表，再安装：

```powershell
wsl --list --online
wsl --install -d Ubuntu-20.04
```

已经安装的环境不需要重复安装。

### 2. `.wslconfig` 的位置和用途

文件位于 Windows 用户目录：

```text
C:\Users\你的Windows用户名\.wslconfig
```

也可以在 Windows PowerShell 中打开：

```powershell
notepad "$env:USERPROFILE\.wslconfig"
```

注意文件名是 `.wslconfig`，不要保存成 `.wslconfig.txt`。如果文件已有内容，应合并配置，保留已有参数，不要直接覆盖；已有 `[wsl2]` 节时，将参数加入该节。

可能对应本次配置思路的参数如下，需核对本机实际内容：

```ini
[wsl2]
networkingMode=mirrored
autoProxy=true
dnsTunneling=true
```

| 参数 | 作用 |
| --- | --- |
| `networkingMode=mirrored` | 使用镜像网络模式，改善 Windows 与 WSL 之间的网络互通 |
| `autoProxy=true` | 尝试让 WSL 使用 Windows 的 HTTP 代理信息 |
| `dnsTunneling=true` | 使用 DNS 隧道处理域名解析，提高部分网络环境下的兼容性 |

**需要修正的认识：这不是简单地把 Ubuntu IP 改成与 Windows 完全相同。镜像网络和自动代理是两个不同的功能。**

上述现代网络功能主要面向 Windows 11 22H2 及更高版本，并需要支持这些功能的 WSL。Windows 10 或旧版 WSL 不能照搬后就假定生效。自动代理也依赖 Windows 代理配置及其类型，不能保证所有应用、PAC 或 SOCKS 代理自动可用。

修改后，需要在 Windows PowerShell 中重启 WSL 才能应用：

```powershell
wsl --shutdown
```

该命令会停止所有 WSL 实例及其进程。应先保存文件并停止编译、安装和仿真，再执行，然后重新打开 Ubuntu。

参考：[Microsoft WSL 高级配置](https://learn.microsoft.com/windows/wsl/wsl-config)、[WSL 网络](https://learn.microsoft.com/windows/wsl/networking)。

### 3. 验证下载访问

在 Ubuntu 中使用只读 Git 命令检查 GitHub 访问：

```bash
git ls-remote https://github.com/PX4/PX4-Autopilot.git HEAD
```

返回提交哈希表示本次 Git 访问成功，但不代表所有 PX4 子模块和依赖下载地址都可访问。后续应根据实际下载报错排查。

## 二、先安装并测试 gnome-terminal

在 Ubuntu 终端执行：

```bash
sudo apt update
sudo apt install -y gnome-terminal dbus-x11
```

遇到 sudo 密码提示时，输入当前 Ubuntu 用户的密码并按回车。密码输入时通常不显示字符或星号，这是正常现象。

测试新终端能否打开：

```bash
gnome-terminal
```

### 本次踩坑：PX4 编译窗口没有弹出

安装器停在：

```text
Waiting for Gazebo server (gzserver) to start...
```

项目脚本通过 `gnome-terminal` 打开另一个窗口，执行 PX4 编译，然后原窗口等待 `gzserver` 进程。没有安装终端程序、D-Bus 会话异常或图形显示不可用，都可能使新窗口打不开。

**缺少终端程序会影响这个安装脚本的自动启动流程，但不意味着 PX4 本身必须依赖 GNOME 终端才能编译。可以在已有终端手动执行构建。**

本次还出现了错误：

```text
Error constructing proxy for org.gnome.Terminal...
Could not connect: No such file or directory
```

这指向本地 D-Bus 会话连接问题，其中的 `proxy` 不是上网代理。可以尝试：

```bash
dbus-run-session -- gnome-terminal --wait
```

本次已经确认 `dbus-x11` 安装完成，但不能仅凭安装成功就断言图形会话问题已解决。WSL 还需要 WSLg 等图形应用支持。

## 三、了解项目文件结构

| 路径 | 主要用途 |
| --- | --- |
| `README.md` | 项目简介、安装与启动入口 |
| `docs/00-AstraDrone开发教程.md` | 入门教程与系统架构 |
| `docs/01-安装脚本详解.md` | 安装、构建脚本说明 |
| `scripts` | 安装和编译工具 |
| `scripts/run_sh` | 示例启动脚本 |
| `AstraDrone_ros1_ws` | ROS 1 工作空间 |
| `AstraDrone_ros2_ws` | ROS 2 工作空间 |
| `simulation` | 仿真模型、世界和仿真工作空间 |
| `third_party` | 第三方依赖库源码 |
| `hardware` | 硬件相关资料 |

基本认识：PX4 负责飞控，Gazebo 模拟无人机与物理环境，ROS 组织感知、规划、控制等程序之间的通信。

## 四、安装与构建顺序

以下以项目位于 `~/AstraDroneOpen`、电脑架构为 x86_64 为例。

### 1. 进入脚本目录

```bash
cd ~/AstraDroneOpen/scripts
chmod +x pc_installer_x86.bin build_sim_workspace_x86.bin build_AstraDrone_ros1_x86.bin
```

### 2. 首次运行一键安装器

```bash
./pc_installer_x86.bin
```

它不只是下载编译 PX4。按 README 说明，一键安装流程还会依次执行基础环境、PX4、第三方依赖及工作空间构建等阶段。

运行过程中应留意密码提示、下载失败、编译窗口输出和退出状态。

### 3. 按需单独构建仿真工作空间

```bash
cd ~/AstraDroneOpen/scripts
./build_sim_workspace_x86.bin
```

### 4. 按需单独构建 ROS 1 工作空间

```bash
cd ~/AstraDroneOpen/scripts
./build_AstraDrone_ros1_x86.bin
```

正确文件名是 `build_AstraDrone_ros1_x86.bin`，中间包含下划线，没有空格。

**这三个命令按“安装器 → 仿真构建 → ROS 构建”的次序理解，但不一定每次都要全部执行。** 如果一键安装器已经成功完成两个工作空间的构建，不必立即重复构建。单独脚本适用于相应阶段失败后的恢复或后续代码更新。

尽量在同一个 Ubuntu 终端中按阶段操作，前一步成功后再进入下一步。安装器需要其他窗口时，不要过早关闭；不要因为原窗口等待就反复重启整个安装器。

## 五、启动基本仿真

安装和工作空间构建完成后，在 Ubuntu 终端执行：

```bash
cd ~/AstraDroneOpen
source ~/.bashrc
source AstraDrone_ros1_ws/devel/setup.bash
bash scripts/run_sh/pc_example.sh
```

注意这里是项目目录下的 `scripts/run_sh/pc_example.sh`，不是系统根目录下的 `/scripts/run_sh/pc_example.sh`。

脚本会通过 tmux 启动 ROS Master、PX4 仿真 launch、Fast-LIO、Offboard 控制以及 QGC。它依赖安装器设置的 `astra`、`qgc` 等命令或别名和环境变量。

| 程序 | 作用 |
| --- | --- |
| QGC（QGroundControl） | 地面站，用于查看飞控状态、参数及飞行相关信息 |
| Gazebo | 展示无人机、场景和传感器等物理仿真 |
| RViz | 展示 ROS 中的位置、点云、轨迹等数据 |
| PX4 SITL | 在电脑中运行的飞控程序，不一定有独立图形窗口 |

用户记录：已观察到 QGC、Gazebo、RViz 等基本仿真窗口。

窗口出现说明部分程序已启动，但还需要检查模型是否加载、飞控连接是否建立，以及点云、定位等数据是否正常。`pc_example.sh` 的 Fast-LIO 启动参数包含 `rviz:=false`，因此 RViz 是否出现还取决于其他 launch 或手动启动步骤。

## 六、单独再次运行已编译的 PX4

如果只是重新启动 PX4 与 Gazebo，可以使用项目安装脚本中的命令：

```bash
cd ~/PX4-Autopilot
CMAKE_PREFIX_PATH=/usr/lib/x86_64-linux-gnu/cmake/opencv4 make px4_sitl gazebo
```

- `make`：调用构建规则，复用已有结果，必要时增量构建。
- `px4_sitl`：电脑上运行的软件在环飞控。
- `gazebo`：该项目脚本使用的仿真目标；不同 PX4 版本支持的目标可能不同。

不要同时用这个命令和 `pc_example.sh` 启动两套 PX4/Gazebo。单独手动启动时，在对应终端按 `Ctrl+C` 结束本次程序。

## 七、本次 PX4 模型生成报错

遇到的关键错误：

```text
ERROR: generation would overwrite changes
Changes should only be made to the template file
```

含义：模型生成器检测到 `.sdf` 与其上次生成的内容不同，为防止覆盖改动停止构建。后续应检查哪些修改需要保留，并将需要保留的修改维护在 `.sdf.jinja` 模板中。

涉及的模型：

- `iris_mid360_0`
- `iris_mid360_1`
- `iris_mid360_2`
- `iris_without_GPS`

已执行操作：将上述生成模型备份并移开，保留模板，尝试重新构建。

实际备份目录：

```text
/home/lee20/px4-sdf-backup.t4A12H
```

此前反馈再次编译仍报错，但没有提供新的错误日志，因此该错误的最终解决方式尚未记录。即使重新生成模型后构建成功，也要核对 Mid360 等定制传感器配置是否保留。

保存编译日志的方法：

```bash
cd ~/PX4-Autopilot
CMAKE_PREFIX_PATH=/usr/lib/x86_64-linux-gnu/cmake/opencv4 make px4_sitl gazebo > ~/px4-build.log 2>&1
```

命令运行时输出写入文件，终端暂时不显示进度。另开终端可实时查看：

```bash
tail -f ~/px4-build.log
```

查看日志时按 `Ctrl+C` 只会停止该终端的 `tail`。构建结束后查看末尾：

```bash
tail -n 50 ~/px4-build.log
```

## 八、鼠标在图形窗口内不可见

曾反馈 QGC、Gazebo、RViz 等多个窗口内看不见鼠标。多个程序共同出现该问题，应优先排查 WSL 图形显示和指针主题，而不是分别修改每个项目。

已提供过的尝试方法包括设置指针主题、大小，以及更新 WSL 并重启图形环境；本次尚未记录确认有效的方法。

若使用 WSL，在停止安装、编译和仿真后，可以在 Windows PowerShell 中更新和重启：

```powershell
wsl --update
wsl --shutdown
```

不要在任务仍运行时关闭 WSL，也不要把这个操作记为已经确认有效的修复方案。

## 九、当前进度与下一步

| 项目 | 当前记录 |
| --- | --- |
| WSL + Ubuntu 20.04 | 已按个人记录安装使用 |
| 网络与 Windows 代理共享 | 已有配置思路；需核对实际 `.wslconfig` 参数及生效状态 |
| 项目目录与工具用途 | 已初步理解 |
| 终端组件 | 已学习安装和 D-Bus 排查方式；窗口恢复结果需补充 |
| PX4 构建 | 已执行，并排查过模型覆盖保护错误 |
| 仿真与 ROS 工作空间构建 | 已记录相应脚本及执行流程；各阶段退出结果需补充 |
| 基本仿真窗口 | 用户反馈已出现 QGC、Gazebo、RViz |
| 模型报错最终解决过程 | 尚未补全 |
| 鼠标不可见问题 | 尚未确认解决 |
| 飞控连接、传感器数据、完整飞行流程 | 待进一步验证 |

下一步优先记录完整构建结果和模型错误的最终处理方法，再检查 QGC 飞控连接、Gazebo 模型及 RViz 数据；之后学习 ROS 节点、话题、消息和 launch 文件，并逐步阅读 `pc_example.sh` 中各条启动命令。
