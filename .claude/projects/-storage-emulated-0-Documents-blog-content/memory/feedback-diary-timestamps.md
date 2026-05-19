---
name: diary-conventions
description: 日记模式的完整规范：时间戳、极简 frontmatter、不过度发挥
metadata:
  type: feedback
---

日记模式与博客文章规则不同，不要照搬文章规范。

**Frontmatter**：极简。title 直接用日期（无引号），tags 固定为 `diary`（纯字符串，不用数组格式 `[diary]`），**不加 description**。
```yaml
---
title: 2026-05-17
date: 2026-05-17
tags: diary
---
```

**时间戳**：每次写日记前用 `date '+%H:%M'` 获取系统时间，内容以 `**HH:MM**` 开头。同一天追加内容也标时间。

**写作**：只做轻微文笔润色和 Markdown 格式转换，不要扩写，不要深度分析，不要自行发挥。

**Why:** 用户多次纠正：tags 要统一用 `diary` 不要数组，不需要 description（那是博客文章的规范），时间要精确到分钟。

**How to apply:** 每次用户说"日记"时，先跑 `date '+%H:%M'`，然后严格按以上 frontmatter 模板输出，不要自行添加 description 或其他 tags。
