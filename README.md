# Awkward Coder

一个基于 Astro 搭建的个人技术博客。

模板来源于 [cassidoo/blahg](https://github.com/cassidoo/blahg)。

## 本地开发

```bash
npm install        # 安装依赖
npm run dev        # 启动开发服务器 (localhost:4321)
npm run build      # 构建生产版本到 ./dist/
npm run preview    # 本地预览构建结果
```

## 写文章

在 `posts/` 目录下新建 Markdown 文件即可。参考已有文章格式。

## 配置

- 修改站点域名：`astro.config.mjs`
- 修改博客信息：`src/settings/settings.json`
- 修改导航链接：`src/components/Header.astro`
- 修改关于页面：`src/pages/about.md`
- Twitter 作者标签：`src/components/BaseHead.astro`
