# Smilex 的技术博客

个人中文技术博客:Hugo + GitHub Pages,方向为 AI 与前端(Web)开发。

## 结构

- `content/posts/` — 文章(Markdown,frontmatter 含 title/date/tags/author)
- `themes/PaperMod` — 主题(submodule,固定 v8.0)
- `.github/workflows/hugo-pages.yml` — push main 自动构建并部署到 `gh-pages` 分支

## 发布流程

1. 新文章放入 `content/posts/`(Markdown,带 frontmatter)
2. 提 PR 合并到 `main`
3. Actions 自动构建,推送到 `gh-pages` 分支 → GitHub Pages 自动上线

线上地址:<https://wjp-110.github.io/hermes/>

## 本地预览

```bash
git clone --recurse-submodules https://github.com/wjp-110/hermes.git
cd hermes
hugo server -D
```
