# 社区方案：把「放首歌」变成 B 站点播

> 本文描述一套已经在 MDZ-25-DA / S12A 上跑稳的**外挂功能**：喊一句「小爱同学，放首歌」，音箱不播小米曲库，而是去 B 站搜、选原唱、下载、解码、播放，并支持暂停 / 续播 / 下一首。
>
> 它**不是**本项目主线的一部分，也不改动本项目的任何代码。写在这里是因为它和「native-first」这套思路配合得很好：小爱答不上来的，交给外部能力。曲库就是一个典型的「答不上来」。

---

## 1. 用起来是什么样

```text
小爱同学，放首晴天        → 搜 B 站 → 选周杰伦原唱 → 播放
小爱同学，下一首          → 切到下一首候选，不重复
小爱同学，暂停 / 继续     → 就地暂停与续播
小爱同学，大声点          → 调的是我们自己的播放增益，不动系统音量
播完                      → 回到待机，可继续下一首
```

关键在于「选得对」：搜「邓紫棋 泡沫」时 B 站播放量最高的是综艺剪辑《假唱？呵！》，不是歌。只按播放量排序必然翻车，所以选曲是一套打分 + 过滤策略，不是取第一条。

---

## 2. 三层架构

```mermaid
flowchart LR
    A["小爱同学，放首歌"] --> B["音箱：native_first_client.sh
识别到音乐意图"]
    B --> C["HTTP → 局域网服务 :8123
/api/search"]
    C --> D["B 站 API
WBI 签名 + 搜索"]
    D --> E["选曲打分
时长 / 标题 / 播放量 / 黑名单"]
    E --> F["取播放地址
resolve_audio_url"]
    F --> G["ffmpeg → WAV
+ 增益"]
    G --> H["音箱 aplay -D default
出声"]
```

三层的职责：

| 层 | 跑在哪 | 干什么 |
|---|---|---|
| 意图层 | **音箱**（shell） | 接住「放首歌」这类请求，把关键词发给局域网服务，接回控制指令 |
| 能力层 | **一台常开的机器**（本文用 NAS） | B 站搜索、选曲、取流、转码、播放会话管理 |
| 出声层 | **音箱** | `aplay` 写 PCM，音量增益在服务端逐采样乘 |

**为什么要分三层**：音箱上只有 busybox + curl + aplay，没有 Python，也没有解码器。搜索签名、选曲打分、转码这些用 Python 写一天就好，塞进 shell 要写一周还容易错。分工之后，音箱侧只剩「转发关键词、按指令播放」两件事。

---

## 3. 关键技术点

### 3.1 B 站 WBI 签名（绕不开的第一关）

B 站搜索接口要求 `w_rid` 签名，算法固定：

```python
def _mixin_key(raw: str) -> str:
    return "".join(raw[i] for i in MIXIN_KEY_ENC_TAB)[:32]

def _sign(params: dict) -> str:
    img, sub = _wbi_keys()                    # 从 /x/web-interface/nav 的 wbi_img 取
    mixin = _mixin_key(img + sub)             # 64 位原串按固定表重排后取前 32 位
    params = dict(params)
    params["wts"] = int(time.time())
    query = urllib.parse.urlencode(sorted(params.items()))
    params["w_rid"] = hashlib.md5((query + mixin).encode()).hexdigest()
    return urllib.parse.urlencode(sorted(params.items()))
```

`img_key` / `sub_key` 从 `https://api.bilibili.com/x/web-interface/nav` 的 `data.wbi_img` 里取文件名（去掉扩展名），两者拼起来 64 位，再按 `MIXIN_KEY_ENC_TAB` 重排取前 32 位作为混入密钥。**这两个 key 会变**，所以每次启动或每轮请求前重新取一次，不要硬编码。

### 3.2 选曲策略（决定「好不好用」的地方）

```text
1. 时长过滤：45s ~ 600s（挡掉广告、整场直播、短片）
2. 标题必须含歌名（用 query 的最后一个词，长度 ≥ 2）
3. 播放量降序
4. 黑名单命中就跳过（"翻唱""鬼畜""纯音乐""钢琴版""DJ""伴奏"…）
5. 全被过滤则退化为「时长过滤后按播放量取第一」
```

再叠一层**原唱优先**：`recording_usable()` / `preferred_usable()` / `original_candidate()` 三个判定，按 `author`（UP 主是否官方号）、标题里的「翻唱 / cover / 现场 / live」等标记、以及是否来自歌手本人账号来打标，`search_smart()` 会优先返回 `original_candidate`。

实测收益：不加这层，「放首晴天」有相当概率给你一个钢琴版。

### 3.3 取流与解码

```text
缓存命中  → 直接用 <bvid>.mp3
未命中    → resolve_audio_url(bvid) 取 DASH 音频直链
         → ffmpeg -headers "Referer: https://www.bilibili.com/" \
                  -probesize 32768 -analyzeduration 0 \
                  -i <url> -vn -af <增益> -f wav -
```

`Referer` 必须带，否则 403。`-probesize 32768 -analyzeduration 0` 是为了压首声延迟：B 站直链是分片的，探测太大要等好几秒才出声。

### 3.4 出声路径：为什么是 `aplay -D default`

这台设备的音频拓扑有个硬约束：`mipns-xiaomi` 独占 `pcm1p`，直接 `aplay -D hw:0,1` 会 `Device busy`。唯一可用的出口是 `aplay -D default`。

所以：

- 播放只走 `aplay -D default`
- **音量不用 ALSA，也不用 `player_set_volume`**（后者会连带改写 `mysoftvol`，把系统音量基线搞乱）
- 增益在服务端做：ffmpeg 输出时 `-af volume=...`，或流式时逐采样乘系数
- 「小声点 / 大声点」改的是这个系数，与小米的语音音量互不影响

### 3.5 播放控制：单 owner

暂停 / 续播 / 下一首最怕多个人同时写播放状态。做法是**只允许一个 owner**持有会话：

```
/api/play     → 转发到内部 :8124 的会话管理，登记当前曲目
/api/control  → 暂停 / 继续 / 下一首 / 停止
/api/track    → 查询当前状态
/api/similar  → 基于当前曲目推荐（用于「再来首类似的」）
```

音箱侧只发指令，不自己维护队列。这样即使音箱客户端重启，播放会话也不会错乱。

---

## 4. 门槛：真的需要 NAS 吗

**不需要 NAS，但需要「一台常开的机器 + 一个解码器」。**

真正的技术门槛不是 NAS，是 **AAC 解码**：

- B 站音频是 DASH / AAC（m4a）
- `aplay` 只吃 PCM / WAV
- 音箱上没有 ffmpeg，也没有 Python

只要跨过这一条，剩下的都能塞进音箱。

### 4.1 方案 A：全部塞进音箱（无外置机器）

| 要素 | 怎么做 | 代价 |
|---|---|---|
| HTTP + 签名 | `curl` + busybox `md5sum`，shell 实现 WBI | 约 100 行 shell，可维护 |
| 解码 | 静态 aarch64 解码器：精简版 ffmpeg（只留 aac/mp3 demuxer+decoder、wav muxer）**约 2–4 MB**；或 `faad2` 静态（约 150 KB） | 占 `/data`，需要交叉编译 |
| 播放 | `aplay -D default` | 无 |
| 选曲打分 | shell 里做简化版（时长 + 标题 + 播放量） | 黑名单与打分逻辑要重写 |

上游项目的 `TTS_ENGINE=device` 已经证明这条路是通的（设备上跑静态 Rust 二进制做 EdgeTTS）。同理，一个 2–4 MB 的静态解码器完全可以放进去。

### 4.2 方案 B：任意一台常开设备

不一定是 NAS。树莓派、软路由、旧安卓手机（Termux）、常年不关的台式机、甚至另一台小爱（如果它有 Python）都行。跑的就是本文那个 800 行左右的 Python 服务，资源占用极低。

这也是视频里那个项目的形态：`TTS_ENGINE=server` 就是「有台机器提供 TTS」的意思，属于被接受的设计。

### 4.3 方案 C：让音箱自己播 URL

有些固件的 `mediaplayer` 能直接播 HTTP 音频流，若能喂给它 B 站直链（带 Referer），解码这一步就省了。**这一条还没在 S12A 上验证过**，属于待试项：值得先花十分钟试一次，成了就能省掉整个解码器。

---

## 5. 已知限制

- **B 站登录态会过期**。搜索本身不需要登录，但部分音频取流会受影响。cookies 建议定期检查 `isLogin`，过期后重新导入。
- **选曲永远有主观性**。打分策略能挡掉大部分翻唱/剪辑，但「好听」这件事没法用播放量衡量，最终还是要人听一次。
- **不能边播边打断**。当前实现里，LLM 播报和 B 站播放是两个状态，播放中喊「小爱同学」能否打断取决于固件的中断行为，没有做专门处理。
- **首声延迟** 受 B 站分片大小影响。`-probesize 32768` 是调出来的值，换固件可能要重调。

---

## 6. 和本项目的关系

这套东西和 `native-first` 是互补的，不是竞争：

| 本项目负责 | 本文负责 |
|---|---|
| 判断「小爱答不上来」，把话转给 LLM | 判断「这是要听歌」，把活转给曲库 |
| LLM 回答 → TTS → 播报 → 续听 | B 站搜索 → 解码 → 播放 → 暂停/续播/下一首 |
| `UNSUPPORTED_PATTERNS` 失败文案拦截 | 音乐意图识别 + 播放会话管理 |

两者共用同一套「接管 → 处理 → 回到待机」骨架。`aivs_speech_guard` 那套「监听 `SpeechSynthesizer/Speak` → 命中规则 → 冻结 mediaplayer → 接管」的手法，同样可以拿来判断「小爱说'我找不到这首歌'」，然后切到 B 站分支。

反过来，本文的分层思路（音箱只做意图与出声，能力放外面）也适用于本项目里你想外挂的其他技能：天气之外的新闻、播客、有声书，都是同一个形状。

---

## 7. 复现清单

想自己搭一套，按这个顺序：

1. 先确认音箱上：`curl` 在不在、`aplay -D default` 能不能出声、`/data` 有多少空间
2. 找一台常开机器，把搜索 + 选曲 + 转码 + 播放会话做成 HTTP 服务（Python 起步最快）
3. 音箱侧写一个薄客户端：识别到音乐意图 → 调 `/api/search` → 拿 `bvid` → 调 `/api/play`
4. 跑通「搜得对」再谈「播得顺」：先用 `curl` 手动调接口，看返回的标题对不对
5. 最后接进开机自启动。**这一步最容易被忽略，也最容易导致「过几天功能就没了」**：`/etc` 下的改动如果只是 tmpfs 覆盖，设备一重启就全部回原厂。自启动逻辑一定要落在持久分区（`/data`），并且有一个能被系统真正调用的入口。

---

## 8. 致谢

- 本项目的 native-first 骨架与 `aivs_speech_guard` 拦截思路
- [open-xiaoai](https://github.com/idootop/open-xiaoai) 的 `/data/init.sh` 自启动方案
- 以及所有被播放量排序坑过、最后决定自己写选曲策略的人
