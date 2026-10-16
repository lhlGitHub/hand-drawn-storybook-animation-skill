# hand-drawn-storybook-animation · 手绘动态绘本 Skill

A Claude Skill for building self-drawing, hand-drawn watercolor picture-book animations that run in the browser: layered scenes, designed camera shots, character animation, page-turn / brush / paper-tear transitions, a gentle generative score, and Doubao (Volcengine) TTS narration and dialogue with bilingual subtitles — plus how to actually verify the result.

一个 Claude Skill：在浏览器里做“会自己画画的”手绘水彩动态绘本——分层场景、设计过的镜头、角色表演、翻页/笔刷/纸页撕裂转场、治愈系配乐、豆包语音配音与中英双语字幕，以及如何真正跑起来验证。

## Install · 安装

**Claude (claude.ai / desktop)**

1. Download this repo as a ZIP (or zip the `hand-drawn-storybook-animation` folder).
2. Customize → Skills → **+** → Create skill → **Upload a skill** → choose the ZIP.

**Claude Code**

```bash
git clone https://github.com/lhlGitHub/hand-drawn-storybook-animation-skill.git
cp -r hand-drawn-storybook-animation-skill/hand-drawn-storybook-animation ~/.claude/skills/
```

## Use · 使用

Just ask, e.g. “做一本会自己画画的绘本”, “给绘本加一章，用豆包配音”, or “Build a self-drawing storybook animation with bilingual subtitles”.

## Notes · 说明

- Doubao TTS needs your own Volcengine API key, kept in your project's `.env` — never commit it.
- 豆包配音需要你自己的火山引擎 API Key，放在自己项目的 `.env` 里，不要提交到仓库。
