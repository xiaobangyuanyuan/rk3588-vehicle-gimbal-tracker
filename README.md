# RK3588 车辆识别与双轴云台 Linux 客户端

V4L2 摄像头、RKNN
YOLOv8 推理、SDL 实时画面和检测框，并把云台 UDP、任务模式和终端交互统一
进一个进程。启动后只运行一个可执行文件：`./app`。

## 已实现

- `/dev/video11` 单平面 NV12 摄像头采集，摄像头、推理、UDP、控制台各自独立线程。
- RK3588 NPU YOLOv8 推理；显示全部检测框，车辆目标由单目标跟踪器持续跟踪。
- 修复历史后处理代码中置信度索引错误。
- 不依赖 RGA 做预处理，避免 `RGA_COLORFILL fail` 刷屏；终端不输出逐帧日志。
- 保留现有 STM32 UDP 协议，不要求修改 MCU 报文。
- UDP 报文校验：源 IP/端口、长度、magic、版本、消息类型、payload 长度、序列号和 ACK 对应关系。
- 100 ms 心跳，300 ms 遥测/ACK 超时；统计丢包、乱序、非法包和 ACK 状态。
- 链路失效或 MCU watchdog 触发时自动退出跟踪/手动模式并进入 HOLD。
- Linux 端角度范围和跟踪速度可运行时修改；MCU 端不增加角度限制。
- 模式：HOLD、TRACK、MANUAL；TRACK 可在运行时选择 COCO 80 类目标，退出前发送 DISARM。
- 模型和标签按 `app` 所在目录解析，不依赖启动时的工作目录。
- 构建脚本不会删除 `model/` 或已安装的模型。

## 目录

```text
rk3588_gimbal_linux_client/
├── CMakeLists.txt
├── build-linux.sh
├── include/
├── src/
├── model/
│   ├── coco_80_labels_list.txt
│   └── yolov8n_rk3588_int8.rknn   # 需要放入，历史仓库不含此文件
└── third_party/rknpu2/
```

## 板卡依赖

```bash
sudo apt update
sudo apt install -y build-essential cmake libsdl2-dev
```

RKNN 头文件和 ARM64 `librknnrt.so` 已随工程提供。

## 第一次构建

进入工程根目录，也就是能看到 `build-linux.sh` 的目录：

```bash
cd ~/rk3588_gimbal_linux_client
chmod +x build-linux.sh
```

历史仓库没有模型。你的板卡已经有模型时，可让脚本自动复制：

```bash
GIMBAL_MODEL_PATH=~/work/yuntai/model/yolov8n_rk3588_int8.rknn \
./build-linux.sh -t rk3588 -b Release
```

之后模型已经保留，正常重新编译即可：

```bash
./build-linux.sh -t rk3588 -b Release
```

脚本不执行 `rm -rf install`，不会删除 `install/rk3588_linux/model/`。

## 运行

```bash
cd ~/rk3588_gimbal_linux_client/install/rk3588_linux
./app
```

程序默认参数：

- 摄像头：`/dev/video11`
- Linux 绑定 IP：`192.168.10.1`
- STM32 IP：`192.168.10.2`
- STM32 命令端口：`5000`
- Linux 遥测端口：`5001`

如需改启动参数：

```bash
./app --camera /dev/video11 \
      --bind-ip 192.168.10.1 --stm32-ip 192.168.10.2 \
      --command-port 5000 --telemetry-port 5001
```

只验证视觉功能：

```bash
./app --no-gimbal
```

查看全部选项：

```bash
./app --help
```

## 交互命令

程序启动后，在同一个终端输入：

```text
status
mode track car
mode track person
mode TRACK traffic_light
mode hold
mode manual 10 -20
track on
track off
classes
speed 35
speed m1 25
speed m2 45
range m1 -70 55
range m2 -110 110
manual 10 -20
m1 15
m2 -30
hold
disarm
exit
```

说明：

- `mode track CLASS` 会清除旧目标，立即按新类别重新搜索并锁定；命令不区分大小写。
- `CLASS` 支持类别英文名或 0～79 的 COCO ID，空格可写成空格、下划线或连字符，例如 `traffic light`、`traffic_light`。
- `classes` 显示当前模型支持的全部 80 类；`mode track` 或 `track on` 沿用上一次选择的类别。
- `speed` 单位为度/秒，只影响 Linux 跟踪算法的角速度上限。
- `range` 是 Linux 命令生成范围；没有改 MCU 端限幅。
- `m1`/`m2` 单轴命令需要新鲜遥测，或者先用 `manual M1 M2` 同时给出两轴。
- `exit`、窗口关闭、`Esc`、`Q`、`Ctrl+C` 都会结束程序；正常退出会 HOLD/DISARM。
- SDL 窗口中 `G` 切换 TRACK/HOLD，`H` 进入 HOLD。

## 状态与安全行为

`status` 会显示当前模式、两轴速度、Linux 范围、遥测年龄、接收/丢包/乱序/非法包计数、ACK 年龄、watchdog 和电机位置/目标。

协议本身没有 CRC 字段，因此本工程不会擅自增加 CRC 破坏兼容性。当前校验覆盖包头、大小、来源、类型、版本、序列和 ACK 关联；如果以后 MCU 与 Linux 同时升级协议，再统一增加 CRC。

当遥测或 ACK 超过 300 ms、或者 MCU 上报 watchdog 触发时，控制线程会自动切换 HOLD，不会继续发送跟踪目标。

## 常见问题

### 提示模型不存在

```bash
cp ~/work/yuntai/model/yolov8n_rk3588_int8.rknn \
   ~/rk3588_gimbal_linux_client/model/
./build-linux.sh -t rk3588 -b Release
```

也可直接复制到 `install/rk3588_linux/model/` 后运行。程序会明确报文件路径，不会因为空模型路径段错误。

### UDP 无法绑定

确认 Linux 网口确实配置了 `192.168.10.1`，并且没有另一个旧 UDP 客户端占用 5001：

```bash
ip addr
ss -lunp | grep ':5001'
```

UDP 不可用时视觉仍可运行，但云台状态显示 OFFLINE。
