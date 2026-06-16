---
title: Android 蓝牙状态监听
tags:
  - Android
  - Bluetooth
---
在Android系统中，当蓝牙的状态发生改变之后，系统会自动发送广播[BluetoothAdapter#ACTION_STATE_CHANGED](https://developer.android.com/reference/android/bluetooth/BluetoothAdapter#ACTION_STATE_CHANGED)。我们只需要注册一个BroadcastReceiver监听系统广播就可以实现对蓝牙状态的监听。

>[!Note] 权限说明
>
>- **Android 11 (API 30) 及以下**：需要 `android.permission.BLUETOOTH`。
>- **Android 12 (API 31) 及以上**：如果需要通过 `BluetoothAdapter` 获取状态，通常需要 `android.permission.BLUETOOTH_CONNECT` 权限。

`ACTION_STATE_CHANGED` 的 Intent 中包含两个重要的 Extra：

- `BluetoothAdapter.EXTRA_STATE`: 当前的新状态。
- `BluetoothAdapter.EXTRA_PREVIOUS_STATE`: 变化前的旧状态。

## 状态常量对照表

| 常量 (Int)                                  | 说明     |
| :---------------------------------------- | :----- |
| `BluetoothAdapter.STATE_OFF` (10)         | 蓝牙已关闭  |
| `BluetoothAdapter.STATE_TURNING_ON` (11)  | 蓝牙正在打开 |
| `BluetoothAdapter.STATE_ON` (12)          | 蓝牙已打开  |
| `BluetoothAdapter.STATE_TURNING_OFF` (13) | 蓝牙正在关闭 |

## 代码

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
  
class BluetoothBroadcastReceiver(  
    private var bluetoothStateChangeListener: BluetoothStateChangeListener? = null,  
) : BroadcastReceiver() {  
  
    override fun onReceive(context: Context?, intent: Intent?) {  
        val extra =  
            intent?.takeIf { it.action == BluetoothAdapter.ACTION_STATE_CHANGED }?.extras ?: return  
        val state = extra.getInt(BluetoothAdapter.EXTRA_STATE)  
        val oldState = extra.getInt(BluetoothAdapter.EXTRA_PREVIOUS_STATE)  
        bluetoothStateChangeListener?.onChange(oldState, state)  
        if (intent?.action == BluetoothAdapter.ACTION_STATE_CHANGED) {
            val state = intent.getIntExtra(BluetoothAdapter.EXTRA_STATE, BluetoothAdapter.ERROR)
            val oldState = intent.getIntExtra(BluetoothAdapter.EXTRA_PREVIOUS_STATE, BluetoothAdapter.ERROR)
            bluetoothStateChangeListener?.onChange(oldState, state)
        }
    }  
}  
  
@Stable  
fun interface BluetoothStateChangeListener {  
    fun onChange(oldState: Int, newState: Int)  
}  

@Composable  
fun BluetoothStateChange(onChangeListener: BluetoothStateChangeListener) {  
    val context = LocalContext.current  
    DisposableEffect(onChangeListener) {  
        val intentFilter = IntentFilter(BluetoothAdapter.ACTION_STATE_CHANGED)  
        val receiver = BluetoothBroadcastReceiver(onChangeListener)  
        ContextCompat.registerReceiver(  
            context, receiver, intentFilter, ContextCompat.RECEIVER_NOT_EXPORTED  
        )  
        onDispose {  
            context.unregisterReceiver(receiver)  
        }  
    }
}
```
