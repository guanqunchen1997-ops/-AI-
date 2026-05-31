
# 微信聊天每日 AI 总结 + 自动同步到滴答清单（手机提醒）

> 用 Claude Code 自动读取你昨日微信聊天 → 生成结构化日报 → 写入 Obsidian → 把待办和会议推送到滴答清单 → iPhone 自动提醒。
> 全程本地运行，零 API 费用，每天 23:00 自动跑。

---

## 一、能实现什么

**早上打开手机 → 滴答清单已经有了一组**「今日待办 + 今日会议」**，全部来自昨晚 Claude 对你微信的智能提炼**：

- **📅 有明确时间的事件**（会议、约访、电话会、面见、活动）→ 自动加日历事件，**提前 1 小时 + 10 分钟双提醒**
- **✅ 没明确时间的待办**（"给某人发资料"、"研究某公司"等）→ 加待办，**次日 09:00 提醒**
- **📝 Obsidian 每日笔记** → 结构化的当日聊天摘要 + 待办 + 日程 + 需跟进事项

闲聊、表情包、新闻链接、行情评论 **不会被做成任务**。

---

## 二、整体架构

```
微信数据库 (SQLCipher 加密)
    ↓ wechat-decrypt 从内存提取密钥后解密
解密后的 SQLite 数据库
    ↓ export_all_chats.py (按日期范围)
当日 JSON 文件 (每个会话一份)
    ↓ Python 脚本合并 + 过滤排除聊天
聚合 Markdown 消息文件
    ↓ Claude Code CLI (headless 模式)
    ↓ 输出两份结构化产出
总结 .md  +  tasks.json
    ↓                ↓
Obsidian Vault    滴答清单 API
    每日笔记          ↓
                  iPhone 推送提醒
```

---

## 三、技术栈

| 组件 | 作用 | 链接 |
|------|------|------|
| **wechat-decrypt** | 解密 Windows 微信 4.x 数据库 | [GitHub](https://github.com/ylytdeng/wechat-decrypt) |
| **Claude Code CLI** | 本地大模型总结（无 API 费用） | [官网](https://claude.com/claude-code) |
| **滴答清单 OpenAPI** | 创建带提醒的任务 | [开发者中心](https://developer.dida365.com/manage) |
| **Obsidian** | 笔记存储（任意 markdown 编辑器都可）| [obsidian.md](https://obsidian.md) |
| **Windows 任务计划程序** | 每日 23:00 自动触发 | 系统内置 |

需要的环境：
- Windows 10/11 + Python 3.10+
- 微信 4.x（电脑版，保持登录状态）
- 任意 Claude Code 订阅（Pro / Max / Team 都可）
- 滴答清单账号（国际版叫 TickTick）
