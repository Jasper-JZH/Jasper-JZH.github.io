---
title: PrimeTween学习笔记
date: 2024-06-14 16:17:29
tags: [Unity, PrimeTween, 学习笔记]
---

# 前言
之前一直用的是DoTween，在看一个DoTween的B站视频时，看到评论区有人提到了PrimeTween，且提到了“高性能”，因此就找到了[Github上的仓库](https://github.com/KyryloKuzyk/PrimeTween)，被其高性能（是DoTween的几倍性能）与“0GC”吸引，于是通读了[文档](https://github.com/KyryloKuzyk/PrimeTween?tab=readme-ov-file#getting-started)并简单实践了一下。下面有针对性的记录一些个人感觉印象比较深刻，以及可能比较常用的点：

# Tween
- 使用`Tween.`来调用Tween的静态方法以创建一个Tween动画
- Tween是一个结构体，可以用变量接收并[查看Tween的状态或控制其行为](https://github.com/KyryloKuzyk/PrimeTween?tab=readme-ov-file#controlling-tweens)，例如：
``` C#
Tween m_Tween = Tween.PositionX(...);
if(m_Tween.isAlive && m_Tween.isPaused)
{
	m_Tween.Stop();
}
```
- Tween几乎可以驱动任何Unity中元素的变化，如UI，UIToolKit，Material，Camera，Transform等
- PrimeTween中不支持对Tween的正向/逆向播放，因此如果想实现类似于窗口的弹出/收回，UI组件的渐隐/渐现的”对称“效果，需要停止当前并用新的Tween来播放新的对称动画。例如：
```C#
[SerializeField] RectTransform window;
public void SetWindowOpened(bool isOpened) {
    Tween.UIAnchoredPositionY(window, endValue: isOpened ? 0 : -500, duration: 0.5f);
}
```

## CallBack
- Tween支持类似DoTween的回调，例如：
``` C#
Tween.PositionX(...).OnComplete(()=>Debug.Log("Tween Complete!"))
```
 - 但需要注意的是，这种情况会引入匿名函数的引用**带来GC**，[解决方法](https://github.com/KyryloKuzyk/PrimeTween?tab=readme-ov-file#zero-allocations-with-delegates)如下：
	 - 使用一个具体的参数来表示该委托，例如：
	 ```C#
	 Tween.PositionX(...)
		 .OnComplete(target : this, target.Log("Tween Complete!"));
		 // 注意这里的target表示该脚本自身，所以要自身实现Log方法
		 // 应该target参数处传其他引用进去也可以的（TODO 测试下）
	 ```

## [Delay](https://github.com/KyryloKuzyk/PrimeTween?tab=readme-ov-file#delays)
- 实用且常用的延迟执行，例如：
```C#
Tween.Delay(duration: 1f, ()=>Debug.Log("xxx"));
```

## [Cycles](https://github.com/KyryloKuzyk/PrimeTween?tab=readme-ov-file#cycles)
- 实现动画循环，例如:
```C#
Tween.PositionY(transform, endValue: 10, duration: 0.5f, cycles: 2, cycleMode: CycleMode.Yoyo);// 循环2次，Yoyo(来回模式)
```

# Sequence
- 实现一组动画与回调的播放控制，支持重叠播放，并行播放，顺序播放等。例如：
```C#
Sequence.Create(cycles: 10, CycleMode.Yoyo)
    // PositionX and Scale tweens are 'grouped', so they will run in parallel
    .Group(Tween.PositionX(transform, endValue: 10f, duration: 1.5f))
    .Group(Tween.Scale(transform, endValue: 2f, duration: 0.5f, startDelay: 1))
    // Rotation tween is 'chained' so it will start when both previous tweens are finished (after 1.5 seconds)
    .Chain(Tween.Rotation(transform, endValue: new Vector3(0f, 0f, 45f), duration: 1f)) 
    .ChainDelay(1)
    .ChainCallback(() => Debug.Log("Sequence cycle completed"))
    // Insert color animation at time of '0.5' seconds
    // Inserted animations overlap with other animations in the sequence
    .Insert(atTime: 0.5f, Tween.Color(image, Color.red, duration: 0.5f));
```
- 除了Sequnce来实现一组有序动画的控制，PrimeTween还支持协程与异步
## [Coruntine](https://github.com/KyryloKuzyk/PrimeTween?tab=readme-ov-file#coroutines)
- 在协程中使用`.ToYieldInstruction()`来等待动画完成后，继续执行后面的代码。类似于等待一段时间的条件，只不过这里的条件是当前动画完成，例如：
```C#
IEnumerator Coroutine() {
    Tween.PositionX(transform, endValue: 10f, duration: 1.5f);
    yield return Tween.Scale(transform, 2f, 0.5f, startDelay: 1).ToYieldInstruction();
    yield return Tween.Rotation(transform, new Vector3(0f, 0f, 45f), 1f).ToYieldInstruction();
    // Non-allocating alternative to 'yield return new WaitForSeconds(1f)'
    yield return Tween.Delay(1).ToYieldInstruction(); 
    Debug.Log("Sequence completed");
}
```

## [Async/Await](https://github.com/KyryloKuzyk/PrimeTween?tab=readme-ov-file#asyncawait)
- PrimeTween支持C#中的async/await语法，从而使用同步的形式编写异步代码，有效避免陷入“地狱回调”。注意这里并不是通过多线程实现的。用法举例：
```C#
async void AsyncMethod() {
    Tween.PositionX(transform, endValue: 10f, duration: 1.5f);
    await Tween.Scale(transform, endValue: 2f, duration: 0.5f, startDelay: 1);
    await Tween.Rotation(transform, endValue: new Vector3(0f, 0f, 45f), duration: 1f);
    // Non-allocating alternative to 'await Task.Delay(1000)' that doesn't use 'System.Threading'
    // Animations can be awaited on all platforms, even on WebGL
    await Tween.Delay(1); 
    Debug.Log("Sequence completed");
}
```

## [Custom tweens](https://github.com/KyryloKuzyk/PrimeTween?tab=readme-ov-file#custom-tweens)
- PrimeTween中支持”自定义动画“，支持的值类型有`float, Color, Vector2/3/4, Quaternion, and Rect`，可以针对以上值实现自定义的变化，例如：
```C#
float floatField;
Color colorField;

// Animate 'floatField' from 0 to 10 in 1 second
Tween.Custom(0, 10, duration: 1, onValueChange: newVal => floatField = newVal);

// Animate 'colorField' from white to black in 1 second
Tween.Custom(Color.white, Color.black, duration: 1, onValueChange: newVal => colorField = newVal);
```


# TimeScale
- PrimeTween可以在不改变Unity内置TimeScale的情况下实现指定动画的倍速控制，或调整全局倍速，例如：
```C#
// 平滑变速指定Tween或Sequence
Tween.TweenTimeScale(Tween/Sequence tween, ...)

// 全局倍速控制（改变Unity的TimeScale）
Tween.GlobalTimeScale(...)

```

# [Speed-based anim](https://github.com/KyryloKuzyk/PrimeTween?tab=readme-ov-file#speed-based-animations)
- 使用`Tween._AtSpeed`实现按指定速度来匀速执行动画，PrimeTween会自动按指定的速度计算时间。
- 需要注意的是，当需要链式`.Chain`使用基于速度的动画时，需要指定每段动画的开始位置，例如：
```C#
Tween.LocalPositionAtSpeed(transform, endValue: midPos, speed)
    // Set 'startValue: midPos' to continue the movement from the 'midPos' instead of the initial 'transform.position'
    .Chain(Tween.LocalPositionAtSpeed(transform, startValue: midPos, endValue: endPos, speed));
```


# Debugging tweens
- 在运行时，PrimeTween会维护一`PrimeTweenManager`在DonDestoryOnLoad中，可以在Inspector面板中点击查看所有Tween的状态。

# [Migrating from DoTween to PrimeTween](https://github.com/KyryloKuzyk/PrimeTween?tab=readme-ov-file#migrating-from-dotween-to-primetween)
- 支持一键转化，也可以对照API表格手动更新