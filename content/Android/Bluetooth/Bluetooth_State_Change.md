---
title: Android 蓝牙状态监听：从 BroadcastReceiver 到 Jetpack Compose 的优雅实现
tags:
  - Android/Bluetooth
  - Compose
---

在开发 Android 蓝牙相关应用时，实时感知系统的蓝牙开关状态是一个极其高频且重要的需求。例如：当用户在系统控制中心关闭蓝牙时，应用应当立即停止蓝牙扫描以节省电量，并提示用户开启蓝牙；当蓝牙重新开启后，应用又需要自动恢复先前的连接或扫描工作。

Android 系统在蓝牙状态发生改变时，会向全系统发送一个广播：[`BluetoothAdapter.ACTION_STATE_CHANGED`](https://developer.android.com/reference/android/bluetooth/BluetoothAdapter#ACTION_STATE_CHANGED)。本文将详细阐述如何通过注册 `BroadcastReceiver` 来监听蓝牙状态的变化，并演示如何将其优雅地集成到现代 Android 的 Jetpack Compose 声明式 UI 体系中。

---

## 1. 核心概念与状态常量

当系统广播 `ACTION_STATE_CHANGED` 触发时，其携带的 `Intent` 会附带两个重要的 Extra 整数参数：

- **`BluetoothAdapter.EXTRA_STATE`**：蓝牙变化后的**当前新状态**。
- **`BluetoothAdapter.EXTRA_PREVIOUS_STATE`**：蓝牙变化前的**历史旧状态**。

通过这两个参数，我们不仅能知道当前蓝牙是开是关，还可以捕获到它们之间的过渡状态。

### 蓝牙状态对照表

蓝牙的生命周期状态主要有以下 4 种：

| 状态常量 (Int) | 对应值 | 说明 | 业务建议 |
| :--- | :--- | :--- | :--- |
| `BluetoothAdapter.STATE_OFF` | 10 | 蓝牙已彻底关闭 | 停止所有蓝牙操作，并在 UI 上展示“蓝牙已关闭”的提示。 |
| `BluetoothAdapter.STATE_TURNING_ON` | 11 | 蓝牙正在打开中 | 可用于在 UI 上显示加载动画（Connecting...）。 |
| `BluetoothAdapter.STATE_ON` | 12 | 蓝牙已彻底打开 | 此时可以安全地启动蓝牙扫描、建立 GATT 连接等。 |
| `BluetoothAdapter.STATE_TURNING_OFF` | 13 | 蓝牙正在关闭中 | 提前释放蓝牙相关的物理资源，停止未完成的任务。 |

---

## 2. 权限声明与安全适配

根据 Android 系统的版本演进，蓝牙状态的获取和监听有不同的权限要求：

1. **Android 11 (API 30) 及以下**：
   在 `AndroidManifest.xml` 中声明传统蓝牙权限即可：
   ```xml
   <uses-permission android:name="android.permission.BLUETOOTH" />
   ```

2. **Android 12 (API 31) 及以上**：
   Android 12 引入了更细粒度的蓝牙运行时权限。如果你的应用需要通过 `BluetoothAdapter` 去获取当前状态或操作蓝牙，必须动态申请连接权限：
   ```xml
   <uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />
   ```
   > [!IMPORTANT]
   > 仅仅监听 `ACTION_STATE_CHANGED` 广播本身在大部分 Android 系统版本上不需要动态权限，但只要你想在状态变为 `STATE_ON` 后调用 `BluetoothAdapter` 的任何方法（如开始扫描），就必须拥有 `BLUETOOTH_CONNECT` 权限。

3. **Android 14 (API 34) 广播安全限制**：
   在 Android 14 及以上版本中，针对非系统广播，应用在注册动态广播接收器时，必须显式指定 `RECEIVER_EXPORTED` 或 `RECEIVER_NOT_EXPORTED` 标识。由于蓝牙状态变化广播属于系统广播，通常我们可以安全地声明为 `ContextCompat.RECEIVER_NOT_EXPORTED` 以确保应用内的接收安全。

---

## 3. 代码实现

我们将整个实现划分为三部分：
1. **`BroadcastReceiver` 的封装**：负责拦截系统广播并提取状态。
2. **状态变化监听接口**：用于解耦业务逻辑。
3. **Jetpack Compose 适配层**：将监听器的生命周期与 Compose 视图绑定，防止内存泄漏。

### 完整实现代码

```kotlin
import android.bluetooth.BluetoothAdapter
import android.content.BroadcastReceiver
import android.content.Context
import android.content.Intent
import android.content.IntentFilter
import androidx.compose.runtime.Composable
import androidx.compose.runtime.DisposableEffect
import androidx.compose.runtime.Stable
import androidx.compose.ui.platform.LocalContext
import androidx.core.content.ContextCompat

/**
 * 蓝牙状态改变监听器接口
 */
@Stable
fun interface BluetoothStateChangeListener {
    fun onChange(oldState: Int, newState: Int)
}

/**
 * 自定义广播接收器，用于捕获蓝牙开关状态的变化
 */
class BluetoothBroadcastReceiver(
    private var bluetoothStateChangeListener: BluetoothStateChangeListener? = null,
) : BroadcastReceiver() {

    override fun onReceive(context: Context?, intent: Intent?) {
        if (intent == null || intent.action != BluetoothAdapter.ACTION_STATE_CHANGED) {
            return
        }

        // 提取新旧状态，若提取失败则使用 ERROR 作为默认值
        val newState = intent.getIntExtra(BluetoothAdapter.EXTRA_STATE, BluetoothAdapter.ERROR)
        val oldState = intent.getIntExtra(BluetoothAdapter.EXTRA_PREVIOUS_STATE, BluetoothAdapter.ERROR)

        // 过滤掉异常值后回调给监听者
        if (newState != BluetoothAdapter.ERROR && oldState != BluetoothAdapter.ERROR) {
            bluetoothStateChangeListener?.onChange(oldState, newState)
        }
    }
}

/**
 * Jetpack Compose 辅助组件：自动注册与反注册蓝牙状态监听器
 *
 * @param onChangeListener 状态改变时的回调闭包
 */
@Composable
fun BluetoothStateChangeListenerEffect(onChangeListener: BluetoothStateChangeListener) {
    val context = LocalContext.current
    
    DisposableEffect(onChangeListener) {
        val intentFilter = IntentFilter(BluetoothAdapter.ACTION_STATE_CHANGED)
        val receiver = BluetoothBroadcastReceiver(onChangeListener)
        
        // 注册广播接收器，适配 Android 14+ 导出安全性要求
        ContextCompat.registerReceiver(
            context, 
            receiver, 
            intentFilter, 
            ContextCompat.RECEIVER_NOT_EXPORTED
        )
        
        // 当 Composable 退出组合或 key 发生改变时，自动执行反注册，杜绝内存泄漏
        onDispose {
            context.unregisterReceiver(receiver)
        }
    }
}
```

---

## 4. 实战应用与最佳实践

### 在 Compose 页面中监听状态

你可以非常简单地在任何 `@Composable` 页面中调用上述 Effect：

```kotlin
@Composable
fun BluetoothMonitorScreen() {
    var bluetoothState by remember { mutableStateOf("未知") }

    BluetoothStateChangeListenerEffect { oldState, newState ->
        bluetoothState = when (newState) {
            BluetoothAdapter.STATE_OFF -> "已关闭"
            BluetoothAdapter.STATE_TURNING_ON -> "正在开启..."
            BluetoothAdapter.STATE_ON -> "已开启"
            BluetoothAdapter.STATE_TURNING_OFF -> "正在关闭..."
            else -> "未知状态"
        }
    }

    Column(
        modifier = Modifier.fillMaxSize(),
        verticalArrangement = Arrangement.Center,
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Text(text = "当前蓝牙状态: $bluetoothState", style = MaterialTheme.typography.h6)
    }
}
```

### 避坑指南与设计细节

1. **首次加载状态缺失问题**：
   `BroadcastReceiver` 只有在蓝牙状态**发生变化时**才会收到通知。如果用户打开应用时蓝牙已经是开启状态，广播接收器是不会收到任何回调的。
   * **解决方案**：在页面初始化时，应当通过 `BluetoothAdapter.getDefaultAdapter()?.state`（或 `BluetoothManager`）主动获取一次蓝牙的当前初始状态，然后再配合 `BluetoothStateChangeListenerEffect` 进行后续的状态变更追踪。

2. **内存泄露防护**：
   务必确保在 `onDispose` 中执行了 `context.unregisterReceiver(receiver)`。如果在 Activity 或 Fragment 销毁后没有正确注销 Receiver，会导致 Context 泄露，系统也会抛出 `ReceiverNotRegisteredException` 异常。

3. **线程问题**：
   `onReceive` 默认在应用的主线程（UI 线程）执行。因此，千万不要在 `onReceive` 或者 `BluetoothStateChangeListener` 的回调中执行耗时的 I/O 操作或耗时的蓝牙扫描初始化。如果有这类需求，应通过 `CoroutineScope` 将其调度至后台线程（如 `Dispatchers.IO`）。

