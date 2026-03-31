# QGroundControl 起飞控制链路解析（从按钮到 MAVLink）

> 目标：把 FlyView 中“Takeoff”按钮被点击后，到最终发出 MAVLink `COMMAND_LONG / MAV_CMD_NAV_TAKEOFF` 的完整调用链串起来。

## 1. 入口：FlyView 工具条里的 Takeoff 按钮

FlyView 工具条动作列表中直接包含 `GuidedActionTakeoff`：

- 文件：`src/FlyView/FlyViewToolStripActionList.qml`
- 关键点：`model` 内注册了 `GuidedActionTakeoff { }`

`GuidedActionTakeoff.qml` 只负责展示属性与 `actionID` 绑定：

- 文件：`src/FlyView/GuidedActionTakeoff.qml`
- 关键点：
  - `text: _guidedController.takeoffTitle`
  - `enabled: _guidedController.showTakeoff`
  - `actionID: _guidedController.actionTakeoff`

这说明 UI 层并不直接发 MAVLink，而是把“起飞”抽象成一个 `actionTakeoff`，交给 Guided 控制器。

## 2. 点击行为：统一进入 GuidedToolStripAction

`GuidedActionTakeoff` 继承自 `GuidedToolStripAction`。在 `onTriggered` 中统一做两件事：

1. 关闭其它 Guided UI
2. 调 `confirmAction(actionID)` 进入确认/参数流程

对应代码位于：

- `src/FlyView/GuidedToolStripAction.qml`

这一步很关键：**按钮点击并不立即下发飞控命令**，而是先走确认弹窗逻辑。

## 3. 控制器中台：GuidedActionsController

`GuidedActionsController.qml` 是 FlyView Guided 指令的“中台”。

### 3.1 action 编号与可见性条件

- `actionTakeoff: 3`
- `_activeVehicle` 绑定 `QGroundControl.multiVehicleManager.activeVehicle`
- `showTakeoff` 需要满足：
  - Guided 动作启用
  - 支持带高度或不带高度的起飞
  - 当前不在飞行中
  - 通过 `canTakeoff`（健康/解锁检查）

这样 UI 会根据飞行状态动态决定是否显示/可点击起飞按钮。

### 3.2 起飞滑条初始化

`setupSlider(actionCode)` 在 `actionTakeoff` 分支中配置高度滑条：

- 最小值来自 `_activeVehicle.minimumTakeoffAltitudeMeters()`
- 最大值来自 `flyViewSettings.guidedMaximumAltitude`
- 文案为 `Height (rel)`

这一步将 UI 输入统一成“相对高度”语义。

## 4. 确认弹窗执行：executeAction(actionTakeoff)

用户在确认弹窗里长按确认后，`GuidedActionConfirm.qml` 调用：

- `guidedController.executeAction(...)`

在 `GuidedActionsController.qml` 的 `executeAction` 中，起飞分支逻辑是：

- 如果 `supports.guidedTakeoffWithAltitude`：
  - 将滑条高度换算到米
  - 调 `_activeVehicle.guidedModeTakeoff(valueInMeters)`
- 否则：
  - 调 `_activeVehicle.startTakeoff()`

这形成了两个链路：

1. **Guided Takeoff（带高度）**
2. **StartTakeoff（飞控自定义起飞流程）**

## 5. QML -> C++ 桥：Vehicle 的 Q_INVOKABLE 接口

`Vehicle.h` 暴露了给 QML 调用的接口：

- `Q_INVOKABLE void guidedModeTakeoff(double altitudeRelative);`
- `Q_INVOKABLE void startTakeoff();`

在 `Vehicle.cc` 中实现：

- `guidedModeTakeoff`：先检查是否支持 guided mode，不支持就弹错误；支持则转发到 `_firmwarePlugin->guidedModeTakeoff(this, altitudeRelative)`。
- `startTakeoff`：直接转发 `_firmwarePlugin->startTakeoff(this)`。

到这里可以看到 QGC 架构原则：**Vehicle 只做状态/能力检查与分发，具体飞控差异下沉到 FirmwarePlugin。**

## 6. FirmwarePlugin 分发层（PX4/APM 分流）

`FirmwarePlugin.h` 定义虚函数：

- `guidedModeTakeoff(...)`
- `startTakeoff(...)`

默认实现在 `FirmwarePlugin.cc` 中是“不支持”提示；真实行为由 PX4/APM 插件 override。

### 6.1 PX4 路径

`PX4FirmwarePlugin::guidedModeTakeoff` 关键流程：

1. 读取当前 AMSL 高度（若未知则报错）
2. `takeoffAltAMSL = 当前AMSL + 相对起飞高度`
3. 调 `vehicle->sendMavCommand(..., MAV_CMD_NAV_TAKEOFF, ..., param7 = takeoffAltAMSL)`
4. 监听 `mavCommandResult`，命令被接受后触发 arm（若未解锁）

`PX4FirmwarePlugin::startTakeoff` 则更偏“模式驱动”：

1. 切到 takeoff flight mode
2. 执行 arming

### 6.2 ArduPilot 路径

`APMFirmwarePlugin::_guidedModeTakeoff` 关键流程：

1. 仅多旋翼/VTOL 允许 guided takeoff
2. 校验 AMSL 可用
3. 起飞高度取 `max(minimumTakeoffAltitudeMeters, 用户输入高度)`
4. 先切 Guided 模式，再 arm
5. 发 `MAV_CMD_NAV_TAKEOFF`（`param7 = relative altitude`）

`startTakeoff` 则优先走 Takeoff 模式 + arm 流程。

## 7. 真正发包：Vehicle::sendMavCommand -> _sendMavCommandWorker

无论 PX4/APM，最终都收敛到 `Vehicle::sendMavCommand`：

1. 进入 `_sendMavCommandWorker(...)`
2. 做去重、链路存在性、超时/重试策略等处理
3. 加入命令队列 `_mavCommandList`
4. 在 `_sendMavCommandFromList` 中组包：
   - 默认 `COMMAND_LONG`（`mavlink_command_long_t`）
   - 或按需 `COMMAND_INT`
5. 调 `sendMessageOnLinkThreadSafe` 从当前主链路发出

对起飞命令来说，常见是 `COMMAND_LONG + MAV_CMD_NAV_TAKEOFF`，`param7` 承载目标高度（不同飞控插件决定是 AMSL 还是相对高度语义）。

## 8. 一张调用链总图（Takeoff 按钮）

```text
FlyViewToolStripActionList
  -> GuidedActionTakeoff (actionID = actionTakeoff)
  -> GuidedToolStripAction.onTriggered
  -> GuidedActionsController.confirmAction(...)
  -> GuidedActionConfirm.onActivated
  -> GuidedActionsController.executeAction(actionTakeoff)
      -> Vehicle.guidedModeTakeoff(height) 或 Vehicle.startTakeoff()
      -> FirmwarePlugin(PX4/APM).guidedModeTakeoff / startTakeoff
      -> Vehicle.sendMavCommand(MAV_CMD_NAV_TAKEOFF 或模式切换/解锁相关命令)
      -> Vehicle::_sendMavCommandWorker
      -> Vehicle::_sendMavCommandFromList
      -> mavlink_msg_command_long_encode_chan / mavlink_msg_command_int_encode_chan
      -> sendMessageOnLinkThreadSafe
      -> 飞控接收并回 ACK (COMMAND_ACK)
```

## 9. 调试建议（实战）

1. 开 `GuidedActionsControllerLog` 看 UI 状态机（showTakeoff、vehicle armed/flying）。
2. 打开 `VehicleLog` 看命令发送与重试日志（`Sending ...`、timeout）。
3. 抓 MAVLink 包确认 `MAV_CMD_NAV_TAKEOFF` 参数（尤其 `param7`）。
4. 对比 PX4/APM 差异时，重点看：
   - 起飞前是否强制切模式
   - arm 的时机（命令前/后）
   - 高度参数语义（AMSL vs Relative）

---

如果你愿意，我可以下一篇继续写 **“Land / RTL / Pause / Goto 的完整链路对照表（含命令参数矩阵）”**，把 `executeAction` 里所有 case 一次性梳理成可检索文档。
