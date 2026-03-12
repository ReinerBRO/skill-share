---
name: notion-api
description: Notion REST API 操作规范。创建数据库/页面、添加内容、查询、更新、删除。包含认证、字段类型、rich text 格式和常见错误处理。
---

# Notion REST API 操作规范

## 核心规则

- API 版本: `2022-06-28`（不要用 `2025-09-03`，MCP 工具不兼容）
- 认证: `Authorization: Bearer ${NOTION_TOKEN}`（环境变量已在 .zshrc 配置）
- Rich Text 必须用对象数组: `[{"type":"text","text":{"content":"xxx"}}]`，不能用纯字符串
- Markdown 语法不会被自动解析，加粗/斜体等必须用 `annotations` 显式标记
- MCP 工具创建页面有 bug（parent 双重序列化），创建操作用 curl

## 通用请求头

```bash
-H "Authorization: Bearer ${NOTION_TOKEN}" \
-H "Notion-Version: 2022-06-28" \
-H "Content-Type: application/json"
```

## 创建数据库

```bash
curl -sS https://api.notion.com/v1/databases \
  -H "Authorization: Bearer ${NOTION_TOKEN}" \
  -H "Notion-Version: 2022-06-28" \
  -H "Content-Type: application/json" \
  --data '{
    "parent": {"type": "page_id", "page_id": "<parent_page_id>"},
    "title": [{"type": "text", "text": {"content": "数据库标题"}}],
    "properties": {
      "Name": {"title": {}},
      "Status": {"select": {"options": [{"name": "Todo"}, {"name": "Done"}]}},
      "Date": {"date": {}},
      "Notes": {"rich_text": {}}
    }
  }'
```

## 创建页面（数据库条目）

```bash
curl -sS https://api.notion.com/v1/pages \
  -H "Authorization: Bearer ${NOTION_TOKEN}" \
  -H "Notion-Version: 2022-06-28" \
  -H "Content-Type: application/json" \
  --data '{
    "parent": {"database_id": "<db_id>"},
    "properties": {
      "Name": {"title": [{"type": "text", "text": {"content": "标题"}}]},
      "Status": {"select": {"name": "Todo"}},
      "Date": {"date": {"start": "2026-03-01"}}
    }
  }'
```

## 添加内容到页面

```bash
curl -sS -X PATCH https://api.notion.com/v1/blocks/<page_id>/children \
  -H "Authorization: Bearer ${NOTION_TOKEN}" \
  -H "Notion-Version: 2022-06-28" \
  -H "Content-Type: application/json" \
  --data '{
    "children": [
      {"type": "paragraph", "paragraph": {"rich_text": [{"type": "text", "text": {"content": "段落"}}]}},
      {"type": "heading_2", "heading_2": {"rich_text": [{"type": "text", "text": {"content": "标题"}}]}},
      {"type": "bulleted_list_item", "bulleted_list_item": {"rich_text": [{"type": "text", "text": {"content": "列表项"}}]}},
      {"type": "code", "code": {"language": "python", "rich_text": [{"type": "text", "text": {"content": "print('hello')"}}]}}
    ]
  }'
```

## 更新页面属性

```bash
curl -sS -X PATCH https://api.notion.com/v1/pages/<page_id> \
  -H "Authorization: Bearer ${NOTION_TOKEN}" \
  -H "Notion-Version: 2022-06-28" \
  -H "Content-Type: application/json" \
  --data '{"properties": {"Status": {"select": {"name": "Done"}}}}'
```

## 查询数据库

```bash
curl -sS -X POST https://api.notion.com/v1/databases/<db_id>/query \
  -H "Authorization: Bearer ${NOTION_TOKEN}" \
  -H "Notion-Version: 2022-06-28" \
  -H "Content-Type: application/json" \
  --data '{"page_size": 10, "filter": {"property": "Status", "select": {"equals": "Todo"}}}'
```

## 删除（移到回收站）

```bash
curl -sS -X PATCH https://api.notion.com/v1/databases/<db_id> \
  -H "Authorization: Bearer ${NOTION_TOKEN}" \
  -H "Notion-Version: 2022-06-28" \
  -H "Content-Type: application/json" \
  --data '{"in_trash": true}'
```

## 字段类型速查

| 类型 | 定义 |
|------|------|
| 标题 | `{"title": {}}` |
| 文本 | `{"rich_text": {}}` |
| 数字 | `{"number": {}}` |
| 单选 | `{"select": {"options": [{"name": "X"}]}}` |
| 多选 | `{"multi_select": {"options": []}}` |
| 日期 | `{"date": {}}` |
| 复选框 | `{"checkbox": {}}` |
| URL | `{"url": {}}` |
| 关联 | `{"relation": {"database_id": "xxx"}}` |

## Rich Text 格式化（annotations）

```json
{"type": "text", "text": {"content": "加粗文本"}, "annotations": {"bold": true}}
```

可用: `bold`, `italic`, `strikethrough`, `underline`, `code`, `color`

## 工作空间已知 ID

```
Student Planner: 308c1df9-c564-8115-a7b0-ecd3d233e8c7
Knowledge Tree Root: 308c1df9-c564-8108-a130-db8916a6b12a
AI Knowledge Base (DB): 308c1df9-c564-8147-ad81-000b5313b990
Code Library (DB): 308c1df9-c564-8168-82d7-000b8f14601e
```

## 常见错误

1. Rich Text: 不要用 `["文本"]`，必须用 `[{"type":"text","text":{"content":"文本"}}]`
2. Markdown 不解析: `**加粗**` 原样显示，必须用 annotations
3. API 版本: 用 `2022-06-28`，不要用 `2025-09-03`
