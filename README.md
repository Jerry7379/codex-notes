# codex-notes

用于沉淀 Codex 任务产出的 Markdown 文档。

## 目录

- `docs/`：文档正文
- `INDEX.md`：文档索引（手工维护）

## 最简流程

1. 本地写好文档，放到 `docs/`
2. 在 `INDEX.md` 增加一条记录
3. 提交并推送：

```bash
git add docs INDEX.md
git commit -m "docs: add <topic>"
git push
```

## 检索

```bash
rg -n "关键词" docs INDEX.md
```
