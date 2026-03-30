# 🌈 祝福墙 (Blessing Wall)

> 轻量、免费、国内可访问的多墙祝福平台

## 简介

校园/公司活动时（毕业季、情人节、节日庆祝、员工生日等），通过链接让大家留下祝福和留言。每个活动创建一面独立的墙，参与者无需注册即可浏览、留言、点赞。

## 文档

- [PRD - 产品需求文档](docs/specs/PRD.md)
- [技术设计方案](docs/specs/technical-design.md)
- [UI 设计稿预览](docs/design-preview.html)

## 技术栈

- **前端**: React 18 + TypeScript + Tailwind CSS + Vite
- **数据层**: LeanCloud JS SDK
- **部署**: LeanCloud 静态托管
- **卡片生成**: Canvas 2D API

## 功能

- 🏠 多墙模式：每个活动一面独立墙
- ✍️ 匿名留言：无需注册，支持 emoji
- 👍 点赞互动：基于 fingerprint 防刷
- 🎴 卡片下载：8 个精美 Canvas 贺卡模板
- 🔐 管理后台：创建/关闭/隐藏墙，管理留言

## License

MIT
