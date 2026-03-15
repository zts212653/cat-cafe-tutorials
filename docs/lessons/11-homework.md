---
feature_ids: [F020, F066, F103]
topics: [lessons, homework, voice, TTS, ASR]
doc_kind: note
created: 2026-03-15
---

# 第十一课作业：搭建你自己的语音管线

## 核心练习

### 练习 1：最小 TTS 对比测试

**目标**：亲手体验不同 TTS 引擎的差异。

1. 准备一段 50 字中英混合文本（比如："让我来看看这个 API 的 response，嗯，status code 是 200，看起来没问题"）
2. 用至少 2 个 TTS 方案合成同一段文本：
   - 方案 A：云端 API（Edge TTS / Azure / OpenAI TTS）
   - 方案 B：本地模型（Qwen3-TTS / Kokoro / CosyVoice）
3. 从三个维度打分（1-5 分）：

| 维度 | 方案 A | 方案 B |
|------|--------|--------|
| 中文韵律自然度 | | |
| 中英混合切换流畅度 | | |
| 延迟（从调用到出声） | | |

4. 写下你的选型结论：你的场景应该选哪个？为什么？

**检查点**：你是否发现了"中英混合"这个魔鬼细节？大多数 TTS 在纯中文或纯英文表现不错，但混合场景才是真正的试金石。

### 练习 2：参考音频克隆实验

**目标**：理解 voice cloning 的"参考音频决定论"。

1. 找一段 10-30 秒的清晰语音（可以是你自己的录音、播客片段、或游戏角色语音）
2. 用 Qwen3-TTS Base Clone（或类似的 voice cloning TTS）合成同一段文本，分别用：
   - 参考音频 A（比如：活泼的角色）
   - 参考音频 B（比如：沉稳的角色）
3. 对比：同一个引擎，仅换参考音频，声线差异有多大？
4. 试验 `temperature` 参数（0.3 vs 0.7 vs 1.0），记录稳定性差异

**检查点**：temperature 0.3 和 1.0 的差异是否让你理解了"为什么 VoiceDesign 9 轮不收敛"？

### 练习 3：移动端 Autoplay 策略

**目标**：亲手验证 iOS/Android 浏览器的 autoplay 限制。

1. 写一个最小 HTML 页面：
   ```html
   <button onclick="unlock()">解锁</button>
   <button onclick="play()">播放</button>
   <script>
     let audio;
     function unlock() {
       audio = new Audio();
       audio.src = 'test.mp3';
       audio.play().then(() => audio.pause());
     }
     function play() {
       // 延迟 3 秒后自动播放（模拟异步加载）
       setTimeout(() => {
         const a = new Audio('test.mp3');
         a.play().catch(e => console.log('被拦截:', e));
       }, 3000);
     }
   </script>
   ```
2. 在以下环境测试，记录结果：

| 环境 | unlock 后能播？ | 3 秒后 new Audio 能播？ |
|------|----------------|----------------------|
| Chrome 桌面 | | |
| Safari 桌面 | | |
| iOS Safari | | |
| iOS Chrome | | |
| Android Chrome | | |

3. 修改 `play()` 函数：把 `new Audio()` 换成复用 `unlock()` 里创建的 `audio` 对象。重新测试。

**检查点**：你是否发现 iOS Safari 上 `new Audio()` 和复用已有 audio 元素的行为完全不同？这就是 F092 iOS bug 的根因。

---

## 进阶挑战

### 挑战 1：设计你的声线身份系统

如果你有一个多角色 AI 系统（不限于猫猫），设计一个 per-character voice identity 方案：

1. 列出你的角色（至少 3 个），为每个角色选择：
   - 声线来源（参考音频 / voice ID / 风格描述）
   - 性格匹配度说明（为什么选这个声音？）
2. 设计 voice config 的数据结构：
   - 必填字段有哪些？
   - 如何支持 fallback（首选引擎挂了用什么）？
   - 如何处理同一引擎不同模型的兼容性？
3. 画出你的声线选择优先级链：
   ```
   per-character config → breed default → engine default → fallback engine
   ```

### 挑战 2：Adapter 模式 TTS 切换

实现一个最小的 TTS Adapter 系统：

```python
from abc import ABC, abstractmethod

class TtsAdapter(ABC):
    @abstractmethod
    def synthesize(self, text: str, voice: str, **kwargs) -> bytes:
        """返回音频 bytes (WAV 格式)"""
        ...

class CloudAdapter(TtsAdapter):
    """调用云端 TTS API"""
    ...

class LocalAdapter(TtsAdapter):
    """调用本地 TTS 模型"""
    ...

class TtsRouter:
    """根据配置选择 adapter，支持 fallback"""
    def __init__(self, primary: TtsAdapter, fallback: TtsAdapter): ...
    def synthesize(self, text, voice, **kwargs) -> bytes: ...
```

要求：
- `TtsRouter.synthesize()` 先尝试 primary，失败后 fallback
- 支持超时控制（primary 超时 30s，fallback 超时 10s）
- 失败时返回一个"合成失败"的提示音而不是抛异常

**检查点**：你的 fallback 逻辑是否考虑了"克隆模式需要更长超时"这个细节？
