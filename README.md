# Agentic Learning and Note

这是一个面向「读书笔记」的 GitHub Pages 知识库。

## 本地预览

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
mkdocs serve
```

打开终端提示的本地地址即可预览。

## 写一篇新的读书笔记

1. 复制 `docs/books/_template.md`。
2. 放到 `docs/books/<book-name>/index.md`。
3. 在 `mkdocs.yml` 的 `nav` 中加入入口。
4. 提交并推送到 `main` 分支。

推送后，GitHub Actions 会自动发布到 GitHub Pages。

