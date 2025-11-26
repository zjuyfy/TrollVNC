# TrollVNC 触摸随机化功能验证指南

## 问题分析

### 为什么 VNC 点击容易被检测？

钉钉等应用的反自动化检测系统会检查以下参数：

1. **触摸压力值 (pathPressure)** - 真人按压力度会变化
2. **触摸半径 (pathMajorRadius)** - 手指接触面积会变化
3. **触摸时间间隔** - 真人点击时间间隔不规律
4. **触摸位置微小偏移** - 真人点击同一位置时会有轻微偏差
5. **触摸速度曲线** - 手指移动速度不是匀速
6. **设备 ID 和事件源** - 检测事件是否来自真实硬件

### 当前实现的功能

✅ **已实现**：
- 触摸压力随机化 (0.0 - 0.3)
- 触摸半径随机化 (4.0 - 8.0 像素)
- 设备集成标志 (`kIOHIDEventFieldDigitizerIsDisplayIntegrated`)
- 内置设备标志 (`kIOHIDEventFieldIsBuiltIn`)

❌ **未实现（可能被检测的原因）**：
- 触摸时间间隔随机化
- 触摸位置微小抖动（±1-2 像素）
- 触摸速度曲线模拟
- 多点触控时的关联性检测

## 验证步骤

### 1. 编译带日志版本

```bash
cd /mnt/data3/zyp/TrollVNC
git add src/STHIDEventGenerator.mm
git commit -m "Add touch randomization debug logging"
git push origin main
```

等待 GitHub Actions 编译完成。

### 2. 安装并启用随机化

1. 安装新编译的 IPA
2. 打开 **设置 → TrollVNC**
3. 找到 **"Randomize Touch Parameters"（随机化触摸参数）**
4. **打开这个开关** ✅

### 3. 查看日志验证

#### 方法 1：使用系统日志（推荐）

```bash
# SSH 连接到 iPad
ssh root@<iPad-IP>

# 实时查看 TrollVNC 日志
log stream --predicate 'process == "trollvncserver"' --level debug

# 或者使用 grep 过滤随机化日志
log stream --predicate 'process == "trollvncserver"' | grep TouchRandomize
```

#### 方法 2：使用 TrollVNC 内置日志查看器

1. 设置 → TrollVNC
2. 点击底部 **"View Logs"**
3. 查找包含 `[TouchRandomize]` 的行

### 4. 测试点击

通过 VNC 连接后，点击屏幕，你应该看到类似日志：

```
[TouchRandomize] Finger 0: radius=6.34 (randomized)
[TouchRandomize] Finger 0: pressure=0.187 (randomized)
```

**如果看到这些日志，说明随机化已生效！**

**如果没有日志，可能的原因：**
- ❌ 开关没有打开
- ❌ VNC 服务没有重启（重启设备或杀掉进程）
- ❌ 使用了旧版本

### 5. 对比测试

#### 测试 A：关闭随机化
```bash
# 设置中关闭 "Randomize Touch Parameters"
# 查看日志：不应该有 [TouchRandomize] 消息
# 压力值固定为 0.0，半径固定为 5.0
```

#### 测试 B：开启随机化
```bash
# 设置中打开 "Randomize Touch Parameters"
# 查看日志：每次点击的压力和半径都不同
# 压力值在 0.0-0.3 之间随机
# 半径在 4.0-8.0 之间随机
```

## 进一步增强反检测能力

如果开启随机化后仍然被检测，可以添加以下功能：

### 功能 1：点击位置微小抖动

在点击位置添加 ±1-2 像素的随机偏移：

```objective-c
// 在 touchDownAtPoints 中添加
CGFloat jitterX = (arc4random_uniform(5) - 2) / 10.0; // ±0.2 像素
CGFloat jitterY = (arc4random_uniform(5) - 2) / 10.0;
locations[index].x += jitterX;
locations[index].y += jitterY;
```

### 功能 2：点击时间间隔随机化

修改 `fingerLiftDelay` 为随机值：

```objective-c
// 当前是固定 0.05 秒
// 改为 0.03-0.08 秒随机
NSTimeInterval randomDelay = 0.03 + (arc4random_uniform(50) / 1000.0);
```

### 功能 3：触摸时长随机化

真人点击时长通常在 50-150ms：

```objective-c
NSTimeInterval touchDuration = 0.05 + (arc4random_uniform(100) / 1000.0);
```

## 检测原理分析

### 钉钉可能使用的检测方法

1. **统计分析**：收集 100 次点击，分析压力值分布
   - 真人：正态分布，标准差较大
   - 机器：固定值或分布异常

2. **时序分析**：分析点击间隔
   - 真人：不规律，有思考时间
   - 机器：等间隔或过于规律

3. **设备指纹**：检查事件源 ID
   - 真人：来自硬件设备
   - 机器：可能来自软件模拟

4. **行为模式**：
   - 真人：点击位置有微小偏差
   - 机器：每次点击完全相同的坐标

### 建议的完整解决方案

```objective-c
// 综合所有随机化因素
typedef struct {
    BOOL positionJitter;      // 位置抖动
    BOOL pressureRandomize;   // 压力随机化 ✅ 已实现
    BOOL radiusRandomize;     // 半径随机化 ✅ 已实现
    BOOL timingRandomize;     // 时间随机化
    BOOL velocityVariation;   // 速度变化
} AntiDetectionConfig;
```

## 快速问题排查

### Q: 如何确认功能已启用？

```bash
# 查看配置文件
cat /var/mobile/Library/Preferences/com.82flex.trollvnc.plist | grep RandomizeTouch

# 应该看到：
# <key>RandomizeTouch</key>
# <true/>
```

### Q: 修改后需要重启吗？

是的，修改设置后需要：
1. 断开 VNC 连接
2. 重新连接（会读取新配置）
3. 或者重启 TrollVNC 服务

### Q: 如何临时测试？

```bash
# 使用命令行参数临时启用
trollvncserver -r

# 查看是否生效
log stream --predicate 'process == "trollvncserver"' | grep "Touch randomization"
# 应该看到: Touch randomization enabled
```

## 总结

1. ✅ 确认开关已打开
2. ✅ 查看日志验证随机化生效
3. ✅ 每次点击应该有不同的压力和半径值
4. ⚠️ 如果仍被检测，可能需要添加更多随机化因素（位置、时间、速度等）

需要我帮你实现更多反检测功能吗？

