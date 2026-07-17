---
name: "android-audio-call-chain"
description: "Documents the complete Android AudioRecord call chain from Java to HAL, including error code -22 (BAD_VALUE) path and power state effects. Invoke when analyzing AudioRecord initialization failures or audio HAL issues."
---

# Android AudioRecord 调用链分析

本技能总结了 Android 系统中 AudioRecord 创建的完整调用链，从 Java 层到 HAL 层，包括错误码传递路径和电源状态影响。

## 一、完整调用链时序图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Java 层 (Application)                               │
├─────────────────────────────────────────────────────────────────────────────┤
│  AudioRecord.Builder().build()                                              │
│         │                                                                  │
│         ▼                                                                  │
│  AudioRecord.java constructor (frameworks/base/media/java/android/media/   │
│      AudioRecord.java#L322)                                                │
│         │                                                                  │
│         ▼                                                                  │
│  native_setup() [JNI调用]                                                  │
└───────────────────────────┬─────────────────────────────────────────────────┘
                            │
                            ▼ JNI层
┌─────────────────────────────────────────────────────────────────────────────┐
│                       JNI 层 (C++)                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│  android_media_AudioRecord_setup()                                         │
│  (frameworks/base/core/jni/android_media_AudioRecord.cpp#L296)             │
│         │                                                                  │
│         ▼                                                                  │
│  new AudioRecord() → lpRecorder->set()                                     │
└───────────────────────────┬─────────────────────────────────────────────────┘
                            │
                            ▼ Native层
┌─────────────────────────────────────────────────────────────────────────────┐
│                     Native 层 (C++) - libaudioclient                         │
├─────────────────────────────────────────────────────────────────────────────┤
│  AudioRecord::set() (frameworks/av/media/libaudioclient/AudioRecord.cpp#L373)│
│         │                                                                  │
│         ▼                                                                  │
│  AudioRecord::createRecord_l() (frameworks/av/media/libaudioclient/        │
│      AudioRecord.cpp#L825)                                                 │
│         │                                                                  │
│         ▼                                                                  │
│  audioFlinger->createRecord()                                               │
└───────────────────────────┬─────────────────────────────────────────────────┘
                            │
                            ▼ AudioFlinger
┌─────────────────────────────────────────────────────────────────────────────┐
│                     AudioFlinger 层                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│  AudioFlinger::createRecord()                                              │
│  (frameworks/av/services/audioflinger/AudioFlinger.cpp#L2099)              │
│         │                                                                  │
│         ▼                                                                  │
│  AudioSystem::getInputForAttr()                                            │
└───────────────────────────┬─────────────────────────────────────────────────┘
                            │
                            ▼ AudioSystem
┌─────────────────────────────────────────────────────────────────────────────┐
│                     AudioSystem 层                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│  AudioSystem::getInputForAttr()                                             │
│  (frameworks/av/media/libaudioclient/AudioSystem.cpp#L1102)                │
│         │                                                                  │
│         ▼                                                                  │
│  aps->getInputForAttr() [AudioPolicyService远程调用]                        │
└───────────────────────────┬─────────────────────────────────────────────────┘
                            │
                            ▼ AudioPolicyService
┌─────────────────────────────────────────────────────────────────────────────┐
│                   AudioPolicyService 层                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│  AudioPolicyService::getInputForAttr()                                      │
│         │                                                                  │
│         ▼                                                                  │
│  AudioPolicyManager::getInputForAttr()                                      │
│  (frameworks/av/services/audiopolicy/managerdefault/AudioPolicyManager.cpp │
│      #L2232)                                                               │
└───────────────────────────┬─────────────────────────────────────────────────┘
                            │
                            ▼ AudioPolicyManager
┌─────────────────────────────────────────────────────────────────────────────┐
│                   AudioPolicyManager 层                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│  mEngine->getInputDeviceForAttributes() (第2232行)                          │
│         │                                                                  │
│         ▼                                                                  │
│  Engine::getInputDeviceForAttributes()                                      │
│  (frameworks/av/services/audiopolicy/enginedefault/src/Engine.cpp#L717)    │
│         │                                                                  │
│         ▼                                                                  │
│  Engine::getDeviceForInputSource() (第746行)                               │
│  (frameworks/av/services/audiopolicy/enginedefault/src/Engine.cpp#L441)    │
└───────────────────────────┬─────────────────────────────────────────────────┘
                            │
                            ▼ Engine 返回设备或 nullptr
```

## 二、错误码 -22 (BAD_VALUE) 传递路径

### 错误产生点

当 `Engine::getDeviceForInputSource()` 返回 `nullptr` 时：

```cpp
// frameworks/av/services/audiopolicy/managerdefault/AudioPolicyManager.cpp#L2236-2239
if (device == nullptr) {
    ALOGW("getInputForAttr() could not find device for source %d", attributes.source);
    status = BAD_VALUE;  // -22
    goto error;
}
```

### 错误传递路径

```
Engine 返回 nullptr
    │
    ▼
AudioPolicyManager 返回 BAD_VALUE (-22)
    │
    ▼
AudioPolicyService (透传)
    │
    ▼
AudioSystem (透传)
    │
    ▼
AudioFlinger (透传)
    │
    ▼
AudioRecord::createRecord_l() 打印日志 (第831行)
    │
    ▼
AudioRecord::set() 返回 -22, 设置 mStatus = -22
    │
    ▼
JNI层转换为 AUDIORECORD_ERROR_SETUP_NATIVEINITFAILED (-20)
    │
    ▼
Java层记录日志, mState = STATE_UNINITIALIZED
```

### 关键日志点

| 日志 | 文件位置 | 含义 |
|-----|---------|------|
| `"AudioFlinger could not create record track, status: %d"` | AudioRecord.cpp#L831 | Native层错误日志 |
| `"getInputForAttr() could not find device for source"` | AudioPolicyManager.cpp#L2237 | 找不到设备 |
| `"getDeviceForInputSource() no device found for source"` | Engine.cpp#L594 | 设备查找失败 |
| `"Error code -20 when initializing native AudioRecord object"` | AudioRecord.java#L472 | Java层错误日志 |

## 三、设备查找流程 (Engine::getDeviceForInputSource)

### 查找优先级

对于 `AUDIO_SOURCE_MIC`（默认）：

```cpp
// frameworks/av/services/audiopolicy/enginedefault/src/Engine.cpp#L484-487
device = availableDevices.getFirstExistingDevice({
        AUDIO_DEVICE_IN_BLE_HEADSET, AUDIO_DEVICE_IN_WIRED_HEADSET,
        AUDIO_DEVICE_IN_USB_HEADSET, AUDIO_DEVICE_IN_USB_DEVICE,
        AUDIO_DEVICE_IN_BLUETOOTH_BLE, AUDIO_DEVICE_IN_BUILTIN_MIC});
```

### 最终回退

```cpp
// frameworks/av/services/audiopolicy/enginedefault/src/Engine.cpp#L593-598
if (device == nullptr) {
    device = availableDevices.getDevice(AUDIO_DEVICE_IN_STUB, ...);
    ALOGE_IF(device == nullptr, "getDeviceForInputSource() no default device defined");
}
```

## 四、设备查找成功后的处理流程

### 设备信息传递

```
Engine::getDeviceForInputSource() 返回 device
    │
    ▼
AudioPolicyManager::getInputForDevice(device, ...)
    │
    ├─► 查找 IOProfile (getInputProfile)
    ├─► 创建 AudioInputDescriptor
    ├─► 打开 HAL 输入流 (inputDesc->open)
    └─► 返回 audio_io_handle_t
            │
            ▼
AudioFlinger::openInput_l()
    │
    ├─► inHwHal->openInputStream() → HAL open_input_stream
    ├─► 创建 RecordThread
    └─► 返回 RecordHandle
            │
            ▼
AudioRecord 创建成功, mState = STATE_INITIALIZED
```

## 五、Audio HAL 打开设备流程

### 完整调用链

```
AudioInputDescriptor::open()
    │
    └─► AudioPolicyClient::openInput() → AudioFlinger::openInput() [Binder IPC]
            │
            └─► AudioFlinger::openInput_l()
                    │
                    └─► DeviceHalInterface::openInputStream() [虚函数]
                            │
                            ├─► DeviceHalLocal::openInputStream()
                            │       └─► mDev->open_input_stream() [本地 HAL]
                            │
                            └─► DeviceHalHidl::openInputStream()
                                    └─► mDevice->openInputStream() [HIDL HAL]
```

### QCOM HAL 入口

```cpp
// hardware/qcom/audio/hal/audio_hw.c#L6761
struct audio_module HAL_MODULE_INFO_SYM = {
    .common = {
        .tag = HARDWARE_MODULE_TAG,
        .id = AUDIO_HARDWARE_MODULE_ID,
        .methods = &hal_module_methods,  // { .open = adev_open }
    },
};

// 第6601行
adev->device.open_input_stream = adev_open_input_stream;
```

### HAL 加载机制

```
AudioFlinger::loadHwModule_l("primary")
    │
    └─► DevicesFactoryHalLocal::openDevice()
            │
            └─► load_audio_interface()
                    │
                    ├─► hw_get_module_by_class() → dlopen libaudio.primary.so
                    └─► audio_hw_device_open() → adev_open()
```

## 六、电源状态影响分析

### WAIT_FOR_VHAL 状态

在 `/packages/services/Car/service/src/com/android/car/power/CarPowerManagementService.java` 中：

```java
// 第333行 - 初始状态
onApPowerStateChange(CpmsState.WAIT_FOR_VHAL, CarPowerStateListener.WAIT_FOR_VHAL);

// 第482行 - S2R恢复TODO
// TODO(b/177478420): Restore Wifi, Audio, Location, and Bluetooth, if they are artificially
// modified for S2R.
```

### 影响链路

```
WAIT_FOR_VHAL 状态
    │
    ├─► VHAL 尚未响应
    ├─► Audio HAL 可能未初始化或依赖 VHAL
    ├─► mAvailableInputDevices 为空或不完整
    ├─► Engine::getDeviceForInputSource() 返回 nullptr
    └─► AudioPolicyManager 返回 BAD_VALUE (-22)
```

### Audio/麦克风控制

在 `PowerComponentHandler.java` 中：

```java
case PowerComponent.AUDIO:
case PowerComponent.MICROPHONE:
    // 音频和麦克风由 audio HAL 根据电源策略控制，框架层不直接控制
    return null;
```

## 七、关键数据结构传递

| 数据 | 来源 | 最终用途 |
|-----|------|---------|
| `device` | Engine::getDeviceForInputSource() | 查找 IOProfile，打开 HAL 流 |
| `inputId` | AudioInputDescriptor::open() | 查找 RecordThread |
| `selectedDeviceId` | device->getId() | 返回给客户端 |
| `portId` | PolicyAudioPort::getNextUniqueId() | 关联客户端和输入流 |
| `cblk` | RecordTrack::getCblk() | 共享内存控制块 |
| `RecordHandle` | new RecordHandle(recordTrack) | 远程控制录音轨道 |

## 八、错误码对照表

| 错误码 | 含义 | 产生位置 |
|-------|------|---------|
| `-22` (BAD_VALUE) | 无效参数/找不到设备 | AudioPolicyManager |
| `-20` (AUDIORECORD_ERROR_SETUP_NATIVEINITFAILED) | Native初始化失败 | JNI层转换 |
| `0` (SUCCESS) | 成功 | 各层正常返回 |

## 九、调试建议

### 检查 logcat

```bash
# 查找设备添加/移除日志
adb logcat | grep -E "setDeviceConnectionState|mAvailableInputDevices|openInput"

# 查找 AudioRecord 错误
adb logcat | grep -E "AudioFlinger could not create record track|getInputForAttr"

# 查找 HAL 打开失败
adb logcat | grep -E "open_input_stream|getDeviceForInputSource"
```

### 关键检查点

1. **设备列表**: `mAvailableInputDevices` 是否为空
2. **HAL 状态**: Audio HAL 是否成功初始化
3. **电源状态**: 是否在 `WAIT_FOR_VHAL` 状态下尝试录音
4. **权限检查**: 是否有 `RECORD_AUDIO` 权限

## 十、总结

本技能涵盖了 Android AudioRecord 创建的完整调用链，从 Java 层到 HAL 层，重点分析了：

1. **完整调用时序**: 10+ 层调用链路
2. **错误码传递**: -22 → -20 的转换过程
3. **设备查找**: Engine 层的设备选择逻辑
4. **HAL 调用**: QCOM HAL 的加载和调用机制
5. **电源影响**: WAIT_FOR_VHAL 状态对音频的影响

当遇到 AudioRecord 初始化失败、错误码 -22 或 -20、设备找不到等问题时，可参考此技能进行分析。