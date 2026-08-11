---
tags:
  - Android/Optimize
title: 内存抖动（Memory Churn）优化实践
---

**内存抖动（Memory Churn）** 是指在极短的时间内频繁地创建和销毁大量对象的现象。它通常是由代码编写不当引起（例如在循环体内、高频触发的方法或自定义 View 的 `onDraw()` 中不断分配内存），从而对程序的内存管理和 UI 渲染性能造成严重影响。

在 Android 中，垃圾回收（GC）机制虽然是自动的，但频繁的分配与回收会给垃圾回收器带来巨大压力，引发频繁的 GC。当 GC 运行时，系统需要暂停所有应用线程（即 **Stop The World (STW)**），这在主线程上就会直接表现为**界面卡顿、丢帧（Jank）**，甚至带来严重的耗电和设备发热问题。

---

## 1. 如何定位与检测内存抖动

在着手优化前，我们需要学会使用工具来精准定位内存抖动的具体代码位置：

### 1.1 Android Profiler (Memory Profiler)

使用 Android Studio 自带的 Profiler 进行内存分析：

1. **锯齿状波形**：在 Memory Profiler 图表中，如果看到内存占用曲线呈现出非常规律且密集的**锯齿状（Sawtooth）** 或“尖峰状”上升与骤降，这通常是典型的内存抖动特征。
2. **捕捉对象分配 (Record Allocations)**：点击 **Record Allocations** 录制一段内存分配记录。
3. **查看分配详情**：在录制结果中，按 **Allocation Count**（分配数量）或 **Size** 排序，重点观察哪些局部变量在短时间内产生了成千上万个实例。通过调用栈（Call Stack）可以直接双击跳转到具体的源码行。

![[memory_churn_profiler.png]]
### 1.2 Perfetto & Systrace

通过 Systrace 或 Perfetto 可以直观看到主线程上的 GC 活动：

* 观察 `PerformGroupGC` 或 `Blocking GC` 等标记。
* 如果在 `Choreographer#doFrame`（屏幕刷新回调）期间频繁穿插 GC 暂停，这证明内存抖动已经严重阻碍了主线程渲染。

---

## 2. 经典抖动场景与实战优化

### 场景一：自定义 View 的 `onDraw()` 中创建对象

`onDraw()` 会伴随界面滑动、动画或属性改变而被高频触发（通常每秒 60 次甚至 120 次）。在其中分配对象无异于“自掘坟墓”。

#### ❌ 错误示范

```kotlin
override fun onDraw(canvas: Canvas) {
    super.onDraw(canvas)
    
    // 错误：每次重绘都会分配新的 Paint 和 Rect 对象
    val paint = Paint().apply {
        color = Color.RED
        style = Paint.Style.STROKE
        strokeWidth = 5f
    }
    val rect = Rect(0, 0, width, height)
    
    canvas.drawRect(rect, paint)
    
    // 错误：高频进行字符串拼接
    val text = "Width: " + width + " Height: " + height
    canvas.drawText(text, 10f, 10f, paint)
}
```

#### ✅ 优化方案

将所有绘制和测量相关的辅助对象定义为**成员变量**，并在构造函数或 `onSizeChanged()` 中初始化，做到热点方法内“零分配”。

```kotlin
class MyCustomView @JvmOverloads constructor(
    context: Context, attrs: AttributeSet? = null, defStyleAttr: Int = 0
) : View(context, attrs, defStyleAttr) {

    // 1. 提取为成员变量，只在类初始化时创建一次
    private val paint = Paint().apply {
        color = Color.RED
        style = Paint.Style.STROKE
        strokeWidth = 5f
    }
    private val rect = Rect()

    override fun onSizeChanged(w: Int, h: Int, oldw: Int, oldh: Int) {
        super.onSizeChanged(w, h, oldw, oldh)
        // 2. 尺寸改变时更新 Rect 范围，避免在 onDraw 中创建
        rect.set(0, 0, w, h)
    }

    override fun onDraw(canvas: Canvas) {
        super.onDraw(canvas)
        
        // 3. 直接复用已有对象进行绘制
        canvas.drawRect(rect, paint)
    }
}
```

---

### 场景二：RecyclerView 列表滑动的绑定陷阱

`onBindViewHolder()` 会在列表快速滑动时，以每秒上百次的频率连续调用。如果在其中分配对象，瞬间就会导致内存暴涨。

#### ❌ 错误示范

```kotlin
override fun onBindViewHolder(holder: MyViewHolder, position: Int) {
    val item = itemList[position]
    holder.titleTextView.text = item.title

    // 错误 1：每次绑定都为 ItemView 创建一个新的 OnClickListener 匿名对象
    holder.itemView.setOnClickListener {
        navigateToDetail(item.id)
    }
    
    // 错误 2：高频解析复杂的日期格式化（SimpleDateFormat 不是轻量级对象）
    val sdf = SimpleDateFormat("yyyy-MM-dd HH:mm:ss", Locale.getDefault())
    holder.timeTextView.text = sdf.format(Date(item.timestamp))
}
```

#### ✅ 优化方案

* **方案 A**：在 `ViewHolder` 初始化（`onCreateViewHolder`）时一次性绑定监听器，通过 `bindingAdapterPosition` 动态获取当前位置的数据。
* **方案 B**：利用全局唯一的 `OnClickListener` 并配合 `Tag` 机制挂载数据。
* **对于复杂格式化**：将格式化器声明为静态（成员）变量，或提前在数据流（如 ViewModel / Repository 中）完成格式化处理，避免在滑动过程中进行数据解析。

```kotlin
class MyAdapter(private val itemList: List<Item>) : RecyclerView.Adapter<MyAdapter.MyViewHolder>() {

    // 1. 提取复用单例格式化工具，或者直接在数据映射阶段格式化
    private val dateFormatter = SimpleDateFormat("yyyy-MM-dd HH:mm:ss", Locale.getDefault())

    class MyViewHolder(itemView: View) : RecyclerView.ViewHolder(itemView) {
        val titleTextView: TextView = itemView.findViewById(R.id.titleTextView)
        val timeTextView: TextView = itemView.findViewById(R.id.timeTextView)
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): MyViewHolder {
        val view = LayoutInflater.from(parent.context).inflate(R.layout.item_layout, parent, false)
        val holder = MyViewHolder(view)
        
        // 2. 优化：只在 ViewHolder 创建时设置一次监听器
        holder.itemView.setOnClickListener {
            val position = holder.bindingAdapterPosition
            if (position != RecyclerView.NO_POSITION) {
                val item = itemList[position]
                navigateToDetail(item.id)
            }
        }
        return holder
    }

    override fun onBindViewHolder(holder: MyViewHolder, position: Int) {
        val item = itemList[position]
        holder.titleTextView.text = item.title
        
        // 3. 避免 new Date() 包装对象，直接重用 Date 或通过 String 缓存
        holder.timeTextView.text = dateFormatter.format(item.timestamp)
    }

    override fun getItemCount(): Int = itemList.size
    
    private fun navigateToDetail(id: String) { /* ... */ }
}
```

---

### 场景三：循环或高频逻辑中的“隐性对象分配”

许多内存抖动并非源于明显的 `new` 关键字，而是来自语言特性带来的“隐性对象”。

#### 1. 自动装箱/拆箱 (Autoboxing)

在计算、累加或高频回调中混用基本类型和包装类型，会产生海量的临时包装类对象。

```kotlin
// ❌ 错误：在循环中隐式装箱（生成大量的 Long 实例）
var sum = 0L
for (i in 0 until 1000000) {
    sum += i // 隐藏的 Long.valueOf() 调用
}

// ✅ 优化：全部使用原始基本数据类型 Long/Int，或者在 Kotlin 中注意避免 Nullable 类型引起装箱
```

#### 2. Kotlin 集合操作的链式调用

Kotlin 内置的 Collection 扩展函数非常方便，但在进行复杂的过滤和变换时，每一个中间步骤都会产生一个新的集合对象。

```kotlin
// ❌ 错误：每一步（filter、map）都会生成一个全新的 List 集合
val result = list.filter { it.active }
                 .map { it.name }
                 .take(5)

// ✅ 优化：针对大数据量或频繁执行的操作，转换为 Sequence 惰性计算（类似 Java Streams）
val result = list.asSequence()
                 .filter { it.active }
                 .map { it.name }
                 .take(5)
                 .toList() // 仅在最后一步生成一个最终 List
```

---

## 3. 高阶防抖设计模式与工具

### 3.1 对象池（Object Pool）模式

对于在运行时不得不大量创建、生命周期极短的对象，使用**对象池**复用它们是最佳方案。Android SDK 本身就提供了 `Pools` 帮助类：

```kotlin
// 声明一个容量为 10 的 Paint 对象池
private val paintPool = Pools.SimplePool<Paint>(10)

fun acquirePaint(): Paint {
    // 如果池子里有，直接复用；没有，则创建新对象
    val paint = paintPool.acquire() ?: Paint()
    paint.reset() // 重置状态，保持纯净
    return paint
}

fun releasePaint(paint: Paint) {
    // 使用完毕后，回收到对象池中
    paintPool.release(paint)
}
```

#### 💡 官方经典案例：`Handler` 中的 `Message` 复用机制

在 Android 框架层中，对象池模式被极其广泛地应用于核心高频通信场景。最经典的例子莫过于异步消息机制中的 `Message` 类。

在开发中，官方强烈建议不要直接使用 `new Message()`，而是使用 `Message.obtain()` 或 `handler.obtainMessage()` 来获取消息实例。这背后的原因在于：

* **单链表池设计**：`Message` 内部维护了一个静态的单链表作为对象池（最大容量为 50）。
* **获取复用**：当调用 `Message.obtain()` 时，会先从链表头部取出已存在的 `Message` 实例，清除其状态后返回，从而避免了在堆上频繁申请新内存。
* **自动回收**：当 `Looper` 消费完该消息后，系统会自动调用 `recycleUnchecked()` 方法重置并回收到该链表池中。

正是得益于这套机制，即使在复杂的界面手势滑动或高频传感器事件中发送大量消息，也不会引发因 `Message` 对象产生的内存抖动。

### 3.2 优先使用 Android 专属高效集合

传统的 `HashMap` 键值对存储会因为装箱引入临时 `Map.Entry` 对象和大量的包装类。Android 系统提供了针对内存优化的高效数据结构：

* **`SparseArray<E>`**：用原始 `int` 作为键，避免了 `Integer` 的装箱，内部通过双数组二分查找实现。
* **`LongSparseArray<E>`** / **`ArrayMap<K, V>`**：适用于数据量较小（通常千级以内）的场景，大幅节省内存空间并减少垃圾收集的触发。

---

## 4. 总结

优化内存抖动的核心思路是 **“避免高频分配”** 与 **“实现对象复用”**。在实际项目开发中，我们可以总结出以下几条性能优化黄金法则：

1. **渲染热点路径“零分配”**：`onDraw`、`onMeasure` 等高频渲染和布局计算方法是禁区，必须严禁出现 `new` 关键字及隐藏的字符串拼接。
2. **列表绑定逻辑做重构**：将监听器绑定提升至 `ViewHolder` 初始化（创建）期；对于日期、数字等数据的复杂格式化操作应提前在后台线程处理，避免在滑动绑定过程中进行解析。
3. **警惕语言特性开销**：在 Kotlin 中进行大数据集合过滤和变换时，善用 `asSequence()` 进行惰性求值；在密集计算循环中注意使用基本数据类型，防止自动装箱造成不必要的堆对象分配。
4. **灵活引入复用机制**：对于频繁申请释放的临时对象，应效仿 Android 官方 `Message.obtain()` 的设计，引入 `Object Pool` 或直接选用更节省内存的 `SparseArray` 系列数据容器。

通过在日常开发中规避这些高频分配陷阱，配合 Android Profiler 的锯齿状波形排查，能够极大地减轻 GC 压力，为用户带来更丝滑稳定的界面交互体验。
