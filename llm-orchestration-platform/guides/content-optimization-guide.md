# 内容优化功能使用指南

## 功能概述

内容优化模块可以将你的原始文本内容优化改写，使其更适合在不同社交媒体平台发布。支持多种平台和风格，自动生成标题、标签建议。

## 支持的平台

| 平台 | 代码 | 特点 |
|------|------|------|
| 小红书 | XIAOHONGSHU | 年轻女性为主，注重生活方式和美学，喜欢emoji和分段 |
| 抖音 | DOUYIN | 短视频配文，需要吸引眼球，引发互动，适合口语化表达 |
| TikTok | TIKTOK | 国际化受众，简洁有力，注重趣味性和话题性 |
| 微博 | WEIBO | 简洁明快，适合热点话题，字数限制 |
| 微信公众号 | WECHAT | 深度内容，逻辑清晰，适合长文 |

## 支持的风格

| 风格 | 代码 | 说明 |
|------|------|------|
| 专业严谨 | PROFESSIONAL | 使用专业术语，逻辑严密，数据支撑 |
| 轻松随意 | CASUAL | 口语化表达，亲切自然，贴近生活 |
| 幽默风趣 | HUMOROUS | 运用幽默元素，轻松活泼，引人发笑 |
| 情感共鸣 | EMOTIONAL | 触动情感，引发共鸣，真诚温暖 |
| 励志激励 | INSPIRATIONAL | 积极向上，鼓舞人心，传递正能量 |
| 潮流时尚 | TRENDY | 紧跟潮流，使用网络热词，年轻化表达 |

## 支持的内容类型

- PRODUCT_REVIEW：产品测评
- TUTORIAL：教程攻略
- LIFESTYLE：生活分享
- KNOWLEDGE：知识科普
- STORY：故事叙述
- OPINION：观点评论

## API 使用示例

### 1. 基础优化

```bash
curl -X POST http://localhost:8080/api/v1/content/optimize \
  -H "Content-Type: application/json" \
  -d '{
    "originalContent": "今天试用了一款新的护肤品，效果还不错，推荐给大家。",
    "platform": "XIAOHONGSHU",
    "style": "CASUAL",
    "contentType": "PRODUCT_REVIEW"
  }'
```

响应示例：

```json
{
  "optimizedContent": "姐妹们！今天要给你们分享一个宝藏护肤品💎\n\n用了一周下来，真的被惊艳到了！\n\n✨ 肤感超级好，不油不腻\n✨ 吸收速度快，不搓泥\n✨ 第二天起来皮肤水润润的\n\n真的是我今年发现的最好用的护肤品之一！强烈推荐给和我一样追求高性价比的姐妹们～\n\n你们用过吗？评论区聊聊～",
  "suggestedTitles": [
    "💎宝藏护肤品分享！用完皮肤水润到发光",
    "姐妹们！这款护肤品真的绝了",
    "实测一周｜这款护肤品值得无限回购"
  ],
  "suggestedTags": [
    "#护肤分享",
    "#好物推荐",
    "#美妆测评",
    "#护肤心得",
    "#种草"
  ],
  "optimizationNotes": "针对小红书平台特点，增加了emoji表情，采用分段式结构，使用口语化表达，添加互动引导",
  "platform": "小红书",
  "style": "轻松随意",
  "tokenUsage": {
    "promptTokens": 450,
    "completionTokens": 280,
    "totalTokens": 730
  }
}
```

### 2. 抖音风格优化

```bash
curl -X POST http://localhost:8080/api/v1/content/optimize \
  -H "Content-Type: application/json" \
  -d '{
    "originalContent": "分享一个快速做早餐的方法，只需要10分钟就能做出营养丰富的早餐。",
    "platform": "DOUYIN",
    "style": "HUMOROUS",
    "contentType": "TUTORIAL",
    "targetAudience": "上班族"
  }'
```

### 3. 生成多个版本

```bash
curl -X POST http://localhost:8080/api/v1/content/optimize \
  -H "Content-Type: application/json" \
  -d '{
    "originalContent": "今天学到了一个时间管理的技巧，分享给大家。",
    "platform": "WEIBO",
    "style": "INSPIRATIONAL",
    "count": 3
  }'
```

### 4. 获取支持的平台列表

```bash
curl http://localhost:8080/api/v1/content/platforms
```

### 5. 获取支持的风格列表

```bash
curl http://localhost:8080/api/v1/content/styles
```

### 6. 获取支持的内容类型

```bash
curl http://localhost:8080/api/v1/content/content-types
```

## 使用场景示例

### 场景1：小红书产品测评

**原始内容**：
```
这款面膜用起来还可以，补水效果不错，价格也合理。
```

**优化后（轻松随意风格）**：
```
姐妹们！今天要给你们安利一款超好用的面膜💕

💧 补水效果：满分！敷完脸蛋水嫩嫩
💰 价格：学生党也能轻松拥有
🌟 使用感受：服帖不滑片，精华液超多

真的是平价好物天花板！已经回购第三盒了～

你们用过吗？评论区见！
```

### 场景2：抖音教程

**原始内容**：
```
教大家一个整理衣柜的方法，可以节省很多空间。
```

**优化后（幽默风趣风格）**：
```
衣柜爆炸的姐妹看过来！🔥

这个神仙整理法，让我的衣柜瞬间大了一倍！

❌ 以前：衣服堆成山，找件衣服像寻宝
✅ 现在：整整齐齐，一目了然

只需要3个步骤，10分钟搞定！

手残党都能学会，不信你试试～

#衣柜整理 #收纳技巧 #生活小妙招
```

### 场景3：TikTok 国际化内容

**原始内容**：
```
分享一个学英语的小技巧。
```

**优化后（潮流时尚风格）**：
```
POV: You've been learning English the wrong way 😱

This ONE trick changed everything for me! 🚀

No more boring textbooks
No more endless grammar drills

Just pure, fun learning! 🎯

Try it and thank me later 💯

#EnglishLearning #LanguageTips #StudyHacks
```

## 最佳实践

1. **选择合适的平台**：不同平台有不同的用户群体和内容偏好
2. **匹配风格**：根据内容主题选择合适的风格
3. **提供上下文**：填写目标受众和额外要求，获得更精准的优化
4. **多版本对比**：使用 count 参数生成多个版本，选择最佳方案
5. **人工润色**：AI 生成的内容建议人工审核后再发布

## 注意事项

- 原始内容长度限制：5000 字符
- 生成数量限制：1-5 个版本
- 建议使用 GPT-3.5-turbo 或更高版本的模型
- 生成的标签和标题仅供参考，可根据实际情况调整

