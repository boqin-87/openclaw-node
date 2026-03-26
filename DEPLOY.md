# 部署指南（Vercel / Cloudflare Pages）

## 0) 本地检查

```bash
npm install
npm run build
```

---

## 1) 接入评论（Giscus）

1. 你的博客仓库开启 **GitHub Discussions**
2. 访问 <https://giscus.app/zh-CN> 生成配置
3. 复制 `.env.example` 为 `.env`，填入：

```env
PUBLIC_GISCUS_REPO=owner/repo
PUBLIC_GISCUS_REPO_ID=...
PUBLIC_GISCUS_CATEGORY=Announcements
PUBLIC_GISCUS_CATEGORY_ID=...
PUBLIC_GISCUS_MAPPING=pathname
PUBLIC_GISCUS_THEME=dark_dimmed
PUBLIC_GISCUS_LANG=zh-CN
```

> Astro 只会暴露 `PUBLIC_` 前缀变量到前端。

---

## 2) 部署到 Vercel

### 方式 A：网页导入（推荐）

1. 把项目推到 GitHub
2. 登录 Vercel，Import Project
3. 选择仓库后，保持默认：
   - Build Command: `npm run build`
   - Output Directory: `dist`
4. 在 Vercel 项目设置里添加上面的 `PUBLIC_GISCUS_*` 环境变量
5. 重新部署

### 方式 B：CLI

```bash
npm i -g vercel
vercel
```

---

## 3) 部署到 Cloudflare Pages

1. 登录 Cloudflare Pages，Create Project
2. 连接 GitHub 仓库
3. 构建设置：
   - Framework preset: `Astro`
   - Build command: `npm run build`
   - Build output directory: `dist`
4. 添加环境变量 `PUBLIC_GISCUS_*`
5. 部署

---

## 4) 常见问题

- **页面正常但评论不显示**：检查 repoId/categoryId 是否填错
- **修改环境变量不生效**：重新触发部署
- **本地可用线上不可用**：确认域名是否开启 HTTPS（Giscus 建议 HTTPS）
