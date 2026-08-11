---
tags:
  - View/RecyclerView
title: RecyclerView DiffUtil 与局部刷新
date: 2026-08-05T11:52:00
---

在 Android 开发中，`RecyclerView` 是展示列表数据的核心组件。然而，面对频繁的数据更新（如点赞、评论、关注状态变更），如果处理不当，很容易导致列表闪烁、卡顿甚至动画丢失。

传统的 `notifyDataSetChanged()` 由于采用全局重绘，不仅会带来巨大的性能开销，还会破坏默认的动画效果。为了解决这一痛点，Android 官方推出了 **`DiffUtil`**。结合 `ListAdapter` 和 **Payload（载荷）** 机制，我们可以实现极其丝滑的局部精确刷新。

## 一、 DiffUtil 的核心原理

`DiffUtil` 是一个用于计算两个数据集差异的工具类，其核心依赖经典的 **Myers 差分算法**（Myers' Diff Algorithm）。它能够在后台线程高效地计算出旧列表到新列表的最小更新代价（最少的增、删、改、移操作）。

### 1. 核心回调方法

在自定义 `DiffUtil.ItemCallback` 时，我们需要实现以下两个关键方法：

- `areItemsTheSame(oldItem, newItem)`：判断两个对象是否是**同一个 Item**（通常比对唯一 ID）。决定了它们在视觉上是否代表同一个实体。
- `areContentsTheSame(oldItem, newItem)`：在 `areItemsTheSame` 返回 `true` 的前提下，进一步比对它们的**内容是否完全相同**。

### 2. Payload 局部刷新机制

当 `areItemsTheSame` 为 `true` 但 `areContentsTheSame` 为 `false` 时，`DiffUtil` 还允许我们重写 `getChangePayload()` 方法。 

通过返回一个自定义的标识（Payload），我们可以精准告诉 `RecyclerView`：**这次改变的只是某一个具体的字段（比如点赞状态），请只刷新对应的 View，而不要重新绑定整个 ViewHolder。**

## 二、 实战演练：以“帖子流点赞”为例

假设我们在做一个社交软件的帖子流，用户点击“点赞”时，如果引发整个 Item 重新绑定（导致图片闪烁、视频重载或打断点赞动画），用户体验会非常差。下面我们通过代码来看看如何完美结合 `DiffUtil` 和点赞动画。

### 1. 定义数据实体

```kotlin
data class Post(
    val id: String,          // 帖子唯一 ID
    val content: String,     // 帖子内容
    val imageUrl: String,    // 图片链接
    val isLiked: Boolean,    // 是否已点赞
    val likeCount: Int       // 点赞数量
)
```

### 2. 编写 DiffUtil.ItemCallback

```kotlin
import androidx.recyclerview.widget.DiffUtil

class PostDiffCallback : DiffUtil.ItemCallback<Post>() {
    
    // 1. 判断是否是同一个帖子
    override fun areItemsTheSame(oldItem: Post, newItem: Post): Boolean {
        return oldItem.id == newItem.id
    }

    // 2. 判断内容是否完全一致
    override fun areContentsTheSame(oldItem: Post, newItem: Post): Boolean {
        return oldItem == newItem
    }

    // 3. 计算差异并返回 Payload
    override fun getChangePayload(oldItem: Post, newItem: Post): Any? {
        if (oldItem.isLiked != newItem.isLiked || oldItem.likeCount != newItem.likeCount) {
            return "PAYLOAD_LIKE_CHANGED" 
        }
        return super.getChangePayload(oldItem, newItem)
    }
}
```

### 3 实现 ListAdapter 与局部点赞动画

借助官方封装的 `ListAdapter`（自带异步差分计算），我们在 Adapter 中重写带有 `payloads` 的 `onBindViewHolder`：

```kotlin
import android.view.LayoutInflater
import android.view.ViewGroup
import androidx.recyclerview.widget.ListAdapter
import androidx.recyclerview.widget.RecyclerView

class PostAdapter : ListAdapter<Post, PostAdapter.PostViewHolder>(PostDiffCallback()) {

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): PostViewHolder {
        val view = LayoutInflater.from(parent.context).inflate(R.layout.item_post, parent, false)
        return PostViewHolder(view)
    }

    // 全量绑定（没有 Payload 时触发）
    override fun onBindViewHolder(holder: PostViewHolder, position: Int) {
        val post = getItem(position)
        holder.bindFull(post)
    }

    // 局部绑定（有 Payload 时触发）
    override fun onBindViewHolder(
        holder: PostViewHolder, 
        position: Int, 
        payloads: MutableList<Any>
    ) {
        if (payloads.isEmpty()) {
            super.onBindViewHolder(holder, position, payloads)
        } else {
            val post = getItem(position)
            for (payload in payloads) {
                if (payload == "PAYLOAD_LIKE_CHANGED") {
                    // 只刷新点赞 UI 并开启动画，图片和长文本绝对不闪烁！
                    holder.updateLikeUI(post.isLiked, post.likeCount, animate = true)
                }
            }
        }
    }

    inner class PostViewHolder(itemView: android.view.View) : RecyclerView.ViewHolder(itemView) {
        
        fun bindFull(post: Post) {
            // 绑定文本、加载图片等全量操作
            updateLikeUI(post.isLiked, post.likeCount, animate = false)
        }

        fun updateLikeUI(isLiked: Boolean, likeCount: Int, animate: Boolean) {
            // 更新点赞按钮状态和数字文本
            // ivLikeButton.isSelected = isLiked
            // tvLikeCount.text = likeCount.toString()

            // 如果是用户点击触发的局部更新，播放平滑的点赞动效
            if (animate) {
                playLikeAnimation()
            }
        }

        private fun playLikeAnimation() {
            // 示例：执行点赞按钮的弹性放大动画
            ivLikeButton.animate().cancel()
            ivLikeButton.scaleX = 1.0f
            ivLikeButton.scaleY = 1.0f
            
            ivLikeButton.animate()
                .scaleX(1.3f)
                .scaleY(1.3f)
                .setDuration(150)
                .withEndAction {
                    ivLikeButton.animate()
                        .scaleX(1.0f)
                        .scaleY(1.0f)
                        .setDuration(150)
                        .start()
                }
                .start()
        }
    }
}
```


### 4. 实现AsyncListDiffer

如果无法继承ListAdapter实现，需要继承自定义的Adapter，官方提供的AsyncListDiffer 就是核心底层支撑。

```kotlin
class CustomPostAdapter : RecyclerView.Adapter<CustomPostAdapter.PostViewHolder>() {
    // 内部组合 AsyncListDiffer
    private val differ = AsyncListDiffer(this, PostDiffCallback())

    fun submitList(list: List<Post>) = differ.submitList(list)

    override fun getItemCount(): Int = differ.currentList.size

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): PostViewHolder {
        // ...
    }

    override fun onBindViewHolder(holder: PostViewHolder, position: Int) {
        holder.bindFull(differ.currentList[position])
    }

    override fun onBindViewHolder(
        holder: PostViewHolder, 
        position: Int, 
        payloads: MutableList<Any>
    ) {
        if (payloads.isEmpty()) {
            super.onBindViewHolder(holder, position, payloads)
        } else {
            val post = differ.currentList[position]
            for (payload in payloads) {
                if (payload == "PAYLOAD_LIKE_CHANGED") {
                    holder.updateLikeUI(post.isLiked, post.likeCount, animate = true)
                }
            }
        }
    }
    // ...
}

```
## 三、 总结

通过 `DiffUtil` + `Payload` + `ListAdapter` 的组合拳，我们在 Android 列表开发中可以获得以下优势：

1. **性能卓越**：差分计算自动在后台线程执行，避免阻塞主线程。
2. **体验极佳**：通过局部刷新（Payload），屏幕中未发生改变的复杂元素（如大图、视频流）保持静止，彻底消除了“全屏闪烁”的糟糕体验。
3. **动画完美承载**：点赞、收藏等带有交互动效的场景能够流畅执行，不会被数据刷新打断。
