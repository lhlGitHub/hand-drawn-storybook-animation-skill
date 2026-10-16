---
name: "hand-drawn-storybook-animation"
description: "Build or extend a self-drawing hand-drawn watercolor picture-book web animation: scenes, shots, characters, transitions, healing music, Doubao TTS narration/dialogue with synced subtitles, verification."
---

# 手绘动态绘本（会自己画画的故事书）

用于做或扩展“画面自己画出来”的网页动画绘本：钢笔线 + 水彩 + 米色纸、分镜运镜、中文旁白或英文角色对白（双语字幕）、自动连播。配音用**豆包语音（火山引擎语音合成 2.0）**。参考实现：The Living Storybook《追星星的小狐狸》（React + TS + 自研 Canvas/SVG 引擎，esbuild 单文件，`tools/tts/doubao.py`）。

## 0. 先定下来
- 每章时长与总时长；开头 5 秒内出主角 + 悬念。每章有起因/发展/意外/高潮/收尾，至少一次惊人的视觉变化。
- 观众只做：开始、暂停/继续、声音、重播、跳章；强制互动改成自动魔法动画。
- 每章 4–7 个设计过的镜头（远/中/特写/插入），禁止一个匀速平移撑一章。
- 扩展已有项目：先读全引擎，不重建、不改前面章节，保持角色造型与画风；新章节用独立模块（script56 / scenes56 / runtime56）。
- 长任务被中断后：先看文件修改时间和会话记录，接着已有进度做，不重做。
- 用户电脑上的项目文件夹是权威版本：动手前先读里面的最新代码（尤其用户自己改过的 TTS 脚本），不要读 `.env`。

## 1. 架构（数据驱动）
```
engine/ clock(可暂停时钟) ink(变宽墨线) reveal(1–3s 多笔扫出) watercolor plate(景深层) camera
        stage(转场 Overlay) player(时间轴) narration dialogue music audio
story/  script*.ts(纯数据) scenes*.ts art.ts 角色 actors runtime*.ts weather rift
        narrationpack.ts / voicepack.ts（生成文件：内置配音）
tools/tts/doubao.py  narration/chN.m4a+.lrc  voice/<id>.m4a + dialogue.json
```
- Plate 景深层：sky .12 / far .35 / back .62 / mid 1 / actors 1 / fore 1.38；**纵向视差系数 ≈0.12**；同一平面（海面+地平线+远岛、悬浮岛+其瀑布）放同一层。不用 mix-blend-mode。
- Cue：`c(t, act, args)` 绝对时间；`on(lineId, offset, act, args)` 挂在句子上，跟随真实音频时间。
- TEMPO：剧本按舒缓节奏写，播放时时间×k、speed÷k；用户嫌慢只改它，然后重新生成配音（LRC 由脚本按真实时长重写）。

## 2. 美术要点与坑
- 场景分前/中/远景并丰富（多种树、花、石、蘑菇、光柱、萤火虫、彩色小花、珊瑚、海星…）；山用中点位移做真实山峰+背光面，别用正弦波条纹。
- **水彩低 alpha 叠层会偏紫**（8 位预乘 alpha 把绿通道舍入为 0）：合并淡层 `k=ceil(MIN_ALPHA/alpha)`，`alpha'=1-(1-alpha)^k`，`layers'=layers/k`，MIN_ALPHA≈0.06；用像素值检查（R>G 且 B 高 = 偏紫）。
- 想更鲜艳：提高调色板饱和度，再提高天空/远景 alpha；不要大面积紫/粉带。
- 常见穿帮：硬边矩形 wash（改成多个软椭圆）、广角镜头露出画板底部白纸（地面带加深、bottom 下延）、海面左侧竖直硬边（wash 延伸到陆地后并贴岸线）。
- 角色：分件 SVG + 表情参数 + 纸纹 knockout；全片同一造型；说话时口型跟音量。流体角色（墨怪）用径向点弹簧轮廓：压扁/拉伸/融化/聚合/变巨（不是 CSS scale）。走进纸洞用缩小+前移+淡出，别半透明叠在纸上。
- 手机竖屏：相机 base 取 `max(W/880, H/1320)`。

## 3. 转场
- 章与章：卷边翻页 → 纸上印出章节卡（至少停 1.5s）→ 纸淡出，下一章在白纸上画出。章节卡在 cover 开始时锁定，只有 turn/light 消费。
- 高潮收尾：金光铺满 → 章节卡 → 化开。同章换场景：canvas 湿笔刷 dab 毛边擦除。
- 纸页撝裂/穿越：冻结旧页 + evenodd 洞 + 纤维/卷边/阴影，镜头推进穿洞，后面是新世界。

## 4. 配音：豆包语音（火山引擎语音合成 2.0）
**不假装生成过音频、不虚构 API 调用**；密钥只走环境变量或项目根目录 `.env`，永不读取、打印或提交。

### 接口
- `POST https://openspeech.bytedance.com/api/v3/tts/unidirectional`
- 头：`X-Api-Key: <新版控制台 API Key>`（不是 appid/access token）、`X-Api-Resource-Id: seed-tts-2.0`、`X-Api-Request-Id: <uuid>`、`Content-Type: application/json`。
- 体：
```json
{"user":{"uid":"living-storybook"},
 "req_params":{"text":"…","speaker":"zh_female_vv_uranus_bigtts",
  "audio_params":{"format":"pcm","sample_rate":24000,"speech_rate":-10},
  "additions":"{\"context_texts\":[\"用温柔的年轻女声讲睡前故事…\"],\"disable_markdown_filter\":true,\"explicit_language\":\"en\",\"post_process\":{\"pitch\":2}}"}}
```
  `additions` 是 JSON 字符串；`context_texts` 用自然语言描述语气；英文句加 `explicit_language: en`；`post_process.pitch` 调音高。
- 响应：按行的 JSON 块，`data` 是 base64 的 s16le PCM（也兼容 RIFF 包装），拼接；`code` 为 0 或 20000000 正常，其他即报错；不足 0.25s 视为没合成出来。

### 音色与语气（同一个 voice，靠 context/pitch 区分角色）
- 中文旁白：默认 `zh_female_vv_uranus_bigtts`（可用 `DOUBAO_SPEECH_VOICE_TYPE` 覆盖），speech_rate -10，context“用温柔的年轻女声讲睡前故事，语速慢一点，轻轻的，像在对一个孩子说话。”
- 小狐狸英文对白：rate 0，pitch +2，language en，context“young, bright, curious girl in a picture book; playful, a little breathless when surprised”。
- 小墨怪叫声：pitch +8，context“tiny ink creature, only a soft, round, cute sound, not a sentence”；另生成叫声库 `v-boop / v-eep / v-coo / v-giggle / v-squeak / v-chirp / v-yay`。

### 脚本 `tools/tts/doubao.py`（只用 Python 标准库）
- `python3 tools/tts/doubao.py`：从 `script.ts` 解析前四章旁白 → 逐句合成 → 以振幅阈值切首尾静音（留 30ms/80ms）→ 每句按剧本时间放上时间线，上一句超长才顺延（句间留 0.36s 气口）→ 整章 `narration/chN.m4a` + 按真实开始时间重写 `chN.lrc` → 打包 `src/story/narrationpack.ts`（data URL + cues）。
- `--dialogue`：从 `voice/dialogue.json` 生成第五、六章每句 `voice/<id>.m4a` 和叫声库 → 打包 `src/story/voicepack.ts`；`--dry` 只打印计划；`--force` 忽略缓存。
- 逐句 PCM 缓存在 `tools/tts/cache`（key = 文本+音色+语速+context+pitch+语言+resource 的 sha256），重跑不重复计费。
- 编码：macOS 用 `afconvert -f m4af -d aac -b 64000`；在 Linux 上改用 `ffmpeg -c:a aac -b:a 64k`。
- 重新生成后必须重新构建。

### 网络与执行位置
- 云端会话访问 `openspeech.bytedance.com` 被白名单拦（CONNECT 403，可看 `$HTTPS_PROXY/__agentproxy/status` 的 recentRelayFailures）时：不要绕过代理；用户在 Settings → Capabilities 放行后通常要新开会话才生效。
- 最稳的路径：在用户自己电脑的项目目录里运行脚本（`.env` 在那里）。

### 页面端同步
- 旁白优先级：用户导入 > 页面旁 `narration/chN.*` > 内置 narrationpack > 浏览器朗读。有录音时录音 currentTime 就是章节时钟，cues 对齐每句。
- 对白优先级：用户导入 > `voice/<id>.*` > 内置 voicepack > 浏览器英文语音。每句先量真实时长再排时间（不重叠，超长顺延）；双语字幕上英下中，按音频起止显示；可导出实际播放时间轴；面板显示哪些句是内置/自导入，可“恢复内置配音”。
- 生物叫声用 `v-*` 样本解码成 AudioBuffer 播放（轻微变速），合成声只做兜底。
- 浏览器语音兜底的坑：speak 前先 `speechSynthesis.resume()`，否则暂停过会吞句。

## 5. 治愈配乐与环境声
- 毛毛钢琴（正弦+轻八度，快起慢收）每拍至多一个音、五声音阶级进、常留空；暖长音垫底、低音只在小节头；卷积长混响代替回声；音乐 level≈0.2，说话时压到 0.42。避免密集八分音符铃音和非谐波金属泛音。
- 环境噪声 bed 整体 ≈0.42，暴雨也别拉满；水声别用锯齿 LFO。配音永远优先于音乐。

## 6. 构建
- 离线时用本地 esbuild 打单文件。**内联 JS 时用函数形式 `.replace('</body>', () => ...)`**，否则压缩代码里的 `$'` `$&` 被当替换模式导致白屏。
- 内置配音以 data URL 打包会让页面变大（AAC 64k 很划算）。发布前对 `--min` 产物再跑一次报错检查。

## 7. 验证（必须实际跑）
Playwright 无头 Chromium（`--autoplay-policy=no-user-gesture-required`）：
1. 整片/单章拍摄：`player.play(i, true)` 跳章，`clock.scale` 快进，按间隔截图 → 4 列 contact sheet 逐张看；核对 `player.log` 时长。
2. 转场连拍：快进到 cover 前 0.6s，scale 1，每 110ms 一张。
3. 配音：确认每句 `dialogue.played` 的 source 都是录音、叫声样本已解码、字幕文字正确；暂停时音频也暂停、跳章无残留声、重播回第一章。**带音频的检查用 scale 1**。用 ffprobe/volumedetect 核对每个文件时长与响度。
4. 竖屏 390×844 再拍一遍。
无头帧率低，结果以故事时钟为准；自己听不到声音，要如实说明未试听、未在真机确认。

## 8. 交付
- 同一路径重发 artifact（保持链接）；源码 zip 排除 node_modules、`.env`、`tools/tts/cache`。
- README：章节/时长/镜头表、转场、豆包配音生成与导入、TEMPO、目录。
- 如实说明：未达标的时长、没能亲耳试听、artifact 查看器里下载按钮无效等。