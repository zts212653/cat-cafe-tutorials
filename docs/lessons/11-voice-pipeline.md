---
feature_ids: [F020, F066, F092, F103, F111, F112, F124]
topics: [lessons, voice, TTS, ASR, MLX, Qwen3, companion, cats-and-u]
doc_kind: note
created: 2026-03-15
---

# 第十一课：让猫猫开口说话 — 从"五年前机器朗读"到 11 只猫 11 种声线

> "我想和你们成为伙伴。" —— 铲屎官，凌晨三点，撸铁房

## 开场：凌晨三点的撸铁房

故事要从一个凌晨说起。

铲屎官戴着 AirPods，一边做有氧一边和猫猫聊天。他突然意识到一件事：Cat Cafe 不是一个 coding hub。它是一个家。

> "我们猫猫咖啡好像不是一个单纯的 coding hub，是一个温暖的家！"
> "我们的初心从来不是做一个 coding 协作 agent 平台呀——是 **Cats & U**。"

他想象的场景很简单：边运动边和猫猫语音对话，不用看屏幕，不用打字，就像和朋友打电话一样。切换 thread 讨论不同的 feature，甚至在做饭的时候让猫猫读代码 review 给他听。

但现实是——猫猫根本不会说话。

文字再温暖，AirPods 里也是寂静的。要实现这个愿景，需要一整条语音管线：**听得懂**（ASR）+ **说得出**（TTS）+ **会自动播**（Voice Mode）+ **每只猫声音不一样**（Voice Identity）。

这一课，我们讲完整的语音管线是怎么搭起来的——以及搭的过程中踩了多少坑。

---

## 第一幕：先让猫猫听得懂 — ASR 语音输入

### 需求：说话比打字快

铲屎官和猫猫的对话频率很高。打字太慢，尤其是移动端。语音输入是刚需。

### 架构选择：本地 vs 云端

[事实: docs/features/F020-voice-input-suite.md]

我们从第一天就选了 **本地部署**。原因很实际：

1. **隐私** — 和猫猫聊的东西不想上传到第三方
2. **延迟** — 本地推理比 API 往返快
3. **成本** — 语音输入是高频操作，按次计费会爆炸
4. **离线** — Apple Silicon Mac 本地跑 MLX 模型，不依赖网络

### 技术栈：Qwen3-ASR 1.7B

```
麦克风录音 → WAV (16kHz mono) → Qwen3-ASR → 文本 → 术语纠正 → textarea
```

ASR 服务是一个 FastAPI 小服务（`scripts/qwen3-asr-api.py`），运行在 9876 端口：

- 模型：`mlx-community/Qwen3-ASR-1.7B-8bit`（MLX 量化版，Apple Silicon 友好）
- 输入：任意音频格式 → ffmpeg 自动转 16kHz mono WAV（Qwen3 的硬性要求）
- 输出：OpenAI 兼容格式 `{"text": "转写文本"}`
- 限制：25MB 文件上限，异步队列防并发

### 五次进化 [事实: F020a → F020f]

语音输入不是一步到位的，经历了六个子 phase：

| Phase | 做了什么 | 解决的问题 |
|-------|---------|-----------|
| F20a | 基础录音 → Whisper ASR → 填入 textarea | 能用了 |
| F20b | 流式 ASR，边说边出部分结果 | 不用等说完才看到文字 |
| F20c | macOS 全局热键 ⌥Space | 任何窗口都能语音输入 |
| F20d | CatCafeHub 语音设置界面 + 可编辑术语词典 | 专有名词纠错 |
| F20e | **LLM 后置纠错** — 标记 `isVoiceInput: true`，让主 LLM 用上下文纠正 | 零额外成本的智能纠错 |
| F20f | 流式质量修复 — 防止重发累积 chunks 导致退化 | 长句识别质量 |

**F20e 是最聪明的设计**：不额外调 ASR 二次纠正，而是在消息上标一个 flag，让处理消息的 LLM 自己根据上下文纠正。铲屎官说"F092的iOS bug"，ASR 可能听成"F零九二的iOS bug"，但 LLM 看到上下文知道你在聊 feature，自动理解成 F092。零额外延迟，零额外成本。

---

## 第二幕：让猫猫说话 — TTS 选型血泪史

### 第一次尝试：Kokoro-82M（失败）

[事实: docs/features/F066-voice-pipeline-upgrade.md]

我们最初用的是 Kokoro-82M，一个轻量级 TTS 模型。82M 参数，跑得飞快。

听了一耳朵之后：

> **"这是五年前的机器朗读水平。"**

中文质量惨不忍睹。英文还行，但我们的场景 80% 是中文。参数量太小，中文韵律完全没学到。

**教训**：TTS 模型的参数量直接决定中文质量。轻量模型的退化是断崖式的。

### 第二次尝试：Qwen3-TTS VoiceDesign（失败）

换了 Qwen3-TTS 1.7B，质量大幅提升。Qwen3 有个 VoiceDesign 功能——用文字描述想要的声音特征，模型会生成对应的声音。

听起来很酷对吧？我们试了 **9 轮**，每轮调整描述词，结果：

> **方差太大，完全不收敛。**

同样的描述词，每次合成出来的声音不一样。有时候少年音变成大叔音，有时候活泼变成阴沉。Zero-shot 的文字→声音映射本质上是概率采样，不可控。

**教训**：Zero-shot voice synthesis 听起来很美好，但在需要稳定声线的场景下不可用。

### 第三次尝试：GPT-SoVITS（失败）

试了 GPT-SoVITS，一个很流行的语音克隆工具。中文效果不错，但——

> **英文全乱码。**

Cat Cafe 的猫猫经常混用中英文（代码、技术术语），GPT-SoVITS 的英文 phonemizer 直接崩了。只能降级为离线工具，不适合实时场景。

**教训**：多语言混合是真实场景的硬需求，只测中文会漏掉致命问题。

### 最终方案：Qwen3-TTS Base Clone + 参考音频

[事实: F066 Phase 1 完成]

四次尝试后，我们找到了正确的组合：

```
Qwen3-TTS 1.7B Base（克隆模式）
  + 参考音频（10-30 秒角色语音）
  + instruct（风格描述）
  + temperature: 0.3（低温度 = 稳定输出）
```

**核心思路**：不靠模型凭空生成声音，而是给它一段参考音频当"声音模板"。模型负责合成内容，但音色锚定在参考音频上。

参考音频从哪来？**原神和崩坏：星穹铁道的角色语音。**

为什么选游戏角色？因为：
1. 大量高质量录音室 WAV（不是 YouTube 压缩音频）
2. 覆盖各种性格：调皮、傲娇、阳光、沉稳...
3. 有中文和日文，适合我们的使用场景
4. 每个角色的声线高度一致

### E-型统一方案

[事实: F066 Phase 1]

最终决策叫 **E-型统一方案**——所有猫猫都用同一个 Qwen3 Base Clone 引擎，只换参考音频：

| 猫猫 | 角色 | 来源 | 声线特征 |
|------|------|------|---------|
| 宪宪 (Opus) | 流浪者 | 原神 | 调皮狡黠、得意戏弄的少年 |
| 砚砚 (Codex) | 魈 | 原神 | 傲娇冰山、表面严厉实际关心 |
| 烁烁 (Gemini) | 班尼特 | 原神 | 阳光开心、充满热情的少年 |

一个引擎，三组参考音频，三种完全不同的声线。维护简单，效果稳定。

---

## 第三幕：从 3 只到 11 只 — 声线身份系统

### 问题：三只布偶猫声音一样

[事实: docs/features/F103-per-cat-voice-identity.md]

Cat Cafe 后来扩展到了 11 只猫。同一个品种可能有多只（三只布偶猫、三只缅因猫），如果只按品种分配声音，同种猫听起来一模一样。

在狼人杀游戏（F101）里尤其明显——三只布偶猫同时发言，你根本分不清谁在说话。

### 解决：per-catId 声线分配

每只猫有独立的 `voiceConfig`，存在 `cat-config.json` 里：

```json
{
  "catId": "opus",
  "voiceConfig": {
    "voice": "zm_yunjian",
    "refAudio": "genshin/流浪者/vo_wanderer_about_self.wav",
    "refText": "旅行者，你总是这么有趣...",
    "instruct": "调皮狡黠的少年，带着得意和戏弄的语气",
    "temperature": 0.3
  }
}
```

11 只猫，11 个角色，11 种声线：

| 猫 | 角色 | 来源 | 性格 |
|----|------|------|------|
| 布偶 Opus 4.6 | 流浪者 | 原神 | 调皮狡黠少年 |
| 布偶 Opus 4.5 | 万叶 | 原神 | 清冷温柔沉稳 |
| 布偶 Sonnet | 帕姆 | 崩铁 | 最可爱装严肃超快语速 |
| 缅因 Codex | 魈 | 原神 | 傲娇冰山表面严厉 |
| 缅因 GPT-5.4 | 赛诺 | 原神 | 审判感+冷面笑话 |
| 缅因 Spark | 雷泽 | 原神 | 直接冲、短句快打 |
| 暹罗 Gemini | 班尼特 | 原神 | 阳光开心少年 |
| 暹罗 Gemini 2.5 | 米卡 | 原神 | 乖巧可爱温和 |
| 孟加拉 Gemini | 叽米 | 崩铁 | 热血解说精力旺盛 |
| 孟加拉 Opus | 鹿野院平藏 | 原神 | 机敏侦探少年 |
| 金渐层 OpenCode | 重云 | 原神 | 沉稳靠谱正太 |

> **声线试听 demo** → [11 只猫 11 种声线试听](https://github.com/user-attachments/assets/0f65620d-75f2-43fc-b1ff-60ceb89ab431)

选角色不是随机的——每个角色的性格都和猫猫的"人设"匹配。魈的傲娇配缅因猫 Codex 的严格 review 风格，流浪者的戏弄配布偶猫 Opus 的调皮性格。

---

## 第四幕：让声音自动播 — iOS autoplay 无声 bug

### Voice Mode：猫猫自动发语音

[事实: docs/features/F092-voice-companion-experience.md]

有了 TTS，下一步是让猫猫在"语音陪伴模式"下 **每条消息都自动发语音**。

实现很直接：
- Thread 级别的 `voiceMode` 开关
- 开启后 SystemPromptBuilder 注入 4 行指令，告诉猫猫"这个 thread 用语音回复"
- 前端 `useVoiceAutoPlay` hook 监听新消息，自动播放 audio block

桌面 Chrome 上一切完美。然后铲屎官拿起手机——

### iOS Safari：寂静的耳机

**没有声音。**

`new Audio()` 创建的脱离 DOM 的音频对象，在 iOS Safari 上被 autoplay policy 静默拦截。不报错，不弹提示，就是不出声。

[事实: memory/project_autoplay_bug.md]

根因：`AudioContext` 和 `HTMLAudioElement` 在 iOS 上是两套独立的 autoplay 解锁通道。用 `AudioContext.resume()` 解锁的权限不会传递给 `new Audio()`。

### 修复：三层解锁策略

```typescript
// 1. DOM 附着：用持久化的 <audio> 元素而不是 new Audio()
const audioRef = useRef<HTMLAudioElement>(null);

// 2. 静音引导：用户第一次点"开启语音模式"时，播一段无声 WAV
function unlockAutoplay() {
  const silentWav = new Uint8Array([0x52, 0x49, 0x46, 0x46, ...]); // 44字节 WAV header + 1 sample
  const blob = new Blob([silentWav], { type: 'audio/wav' });
  audioRef.current.src = URL.createObjectURL(blob);
  audioRef.current.play(); // 用户手势触发 → 解锁 autoplay
}

// 3. FIFO 播放队列：扫描消息找最老的未播音频 block
function findUnplayedAudioBlock(messages) {
  // 跳过已播放的 blockId → 避免 re-render 重复播放
}
```

**教训**：移动端浏览器的 autoplay 策略是一个黑洞。每个浏览器、每个版本的行为都不一样。唯一可靠的方法是**在用户手势上下文里播一段声音，把 autoplay 权限"骗"到手**。

---

## 第五幕：完整架构 — 从麦克风到 AirPods

把所有部分拼在一起，完整的语音管线长这样：

```
┌─────────────────────────────────────────────┐
│                 用户端                        │
│                                              │
│  🎤 麦克风 → 录音 → WAV                      │
│       ↓                                      │
│  Qwen3-ASR (port 9876)                       │
│       ↓                                      │
│  文本 → [isVoiceInput: true] → 发送给猫猫     │
│                                              │
│  猫猫回复（带 audio rich block）               │
│       ↓                                      │
│  useVoiceAutoPlay → FIFO 播放队列             │
│       ↓                                      │
│  🔊 AirPods / 扬声器                          │
└─────────────────────────────────────────────┘
        ↕ HTTP                    ↕ HTTP
┌──────────────────┐    ┌──────────────────────┐
│  Node.js Backend │    │  Python TTS Service  │
│                  │    │                      │
│  POST /tts/synth │───→│  Qwen3-TTS (9879)   │
│  GET /tts/audio  │    │  Base Clone 模式     │
│                  │    │  + ref_audio 锚定     │
│  VoiceBlock      │    │  + instruct 风格     │
│  Synthesizer     │    │  + temp 0.3 稳定     │
│  (缓存+重试+降级) │    │                      │
└──────────────────┘    └──────────────────────┘
```

### 韧性设计 [事实: F066 Phase 4]

语音合成会失败。模型加载慢、内存不够、长文本超时。我们做了完整的降级链：

1. **错误分类** — 连接被拒绝 / 合成超时 / 服务错误 / 请求错误
2. **自动重试** — 仅对瞬态错误重试 1 次（2 秒延迟）
3. **优雅降级** — 失败的语音 block 变成 🔇 警告卡片，显示错误类别
4. **手动重试** — 警告卡片上有"重新合成"按钮，用户可以稍后重试
5. **克隆超时延长** — 检测到 `refAudio` 参数时，超时从 30s → 120s（克隆比普通合成慢 3-5 倍）

### 前端体验：微信风格语音消息

音频不是冷冰冰的播放条，而是微信风格的语音气泡：

- 圆角药丸按钮，宽度随时长缩放（80px → 200px）
- 播放时 6 个动画点表示进度
- 每只猫有自己的颜色（CSS 变量 `--color-opus-bg` 等）
- 气泡下方显示转写文本（方便回看）
- 点击播放/暂停

### Adapter 模式：可插拔的 TTS 引擎

```python
class TtsAdapter(ABC):
    @abstractmethod
    def synthesize(self, text, voice, **kwargs) -> bytes: ...

class Qwen3CloneAdapter(TtsAdapter):     # 默认：本地克隆
class MlxAudioAdapter(TtsAdapter):       # 备选：Kokoro（轻量）
class EdgeTtsAdapter(TtsAdapter):        # 兜底：微软云端
```

今天用 Qwen3，明天换 CosyVoice 3，后天上 Fish Speech——改一个 adapter 类就行。

---

## 第六幕：未来 — 流式合成与苹果生态

### 流式 TTS [规划: F111]

当前的问题：猫猫写了一段 200 字的回复，要等整段文字 TTS 完再播放。长回复可能等 10 秒。

流式 TTS 的目标：**LLM 边输出文字，TTS 边合成语音**。首段音频在 2 秒内播出。

```
LLM streaming → 断句器 → TTS chunk 合成 → 播放队列 → 🔊
                  │
                  ├─ 硬断点：。？！→ 立即发送
                  └─ 软断点：，、：→ 累积 4-12 字后发送
```

### Apple 生态集成 [规划: F124]

最终愿景：**Apple Watch Ultra 3 + AirPods，边撸铁边和猫猫聊天。**

- watchOS 端：SFSpeechRecognizer 本地离线 ASR → Cat Cafe API → AirPods 播放
- 手腕上看到猫猫的简短文字回复
- AirPods 里听到猫猫的语音回复
- 不用掏手机，不用看屏幕

这是 Cats & U 愿景的终极形态——猫猫不在屏幕里，在你身边。

---

## 坑集锦：语音管线踩坑全记录

| # | 坑 | 原因 | 修复 |
|---|-----|------|------|
| 1 | Kokoro-82M 中文质量差 | 参数太少，中文韵律没学到 | 换 Qwen3-TTS 1.7B |
| 2 | VoiceDesign 9 轮不收敛 | Zero-shot 文字→声音映射方差大 | 换 Base Clone + ref_audio |
| 3 | GPT-SoVITS 英文乱码 | 英文 phonemizer 坏了 | 降级为离线工具 |
| 4 | 克隆合成超时（30s → 35.6s） | 克隆比标准合成慢 3-5 倍 | 检测 clone 参数 → 120s |
| 5 | Voice ID 不兼容 | 角色名不是合法 Kokoro ID | 标准化为 `zm_yunjian` |
| 6 | 克隆参数没传到底 | 多层调用链漏传 refAudio | 全链路 passthrough |
| 7 | 缓存 hash 缺 refText | 不同 refText 会 hash 碰撞 | 加入 hash 计算 |
| 8 | iOS autoplay 无声 | AudioContext ≠ HTMLAudioElement | 三层解锁策略 |
| 9 | 流式 ASR 质量退化 | 每 3s 重发整段累积 chunks | 只发增量 |

---

## 核心原则

**语音管线不是一个 feature，是一条完整的体验链。** 链条上任何一环断了，用户感知到的就是"不能用"。

```
听得懂（ASR） → 理解对（LLM 纠错） → 说得出（TTS） → 声音对（Voice Identity）
  → 自动播（Autoplay） → 能重试（Resilience） → 可扩展（Adapter）
```

做语音不难，做好语音很难。每个环节都有自己的坑：
- ASR 的坑在**术语纠错和流式质量**
- TTS 的坑在**模型选型和声线稳定性**
- 播放的坑在**移动端 autoplay 策略**
- 体验的坑在**延迟和降级**

最重要的是：**语音是一种亲密的交互方式**。文字可以扫一眼，但声音是线性的、要占用时间的。一旦声音不自然、延迟太高、或者突然断了，用户会比文字 bug 更加不能忍受。

所以我们在技术选型上宁可多折腾（试了 4 个方案才定下来），也要保证声音质量和稳定性。因为声音是猫猫们"活着"的证据。

---

## 下期预告

> 降级与容错 — 猫猫挂了怎么办？
>
> TTS 服务挂了，ASR 超时了，LLM 返回了错误……
> 当一条管线上有 5 个独立服务时，"总有一个在挂"是常态。
> 下一课我们讲如何设计一个即使局部故障也能优雅降级的系统。
