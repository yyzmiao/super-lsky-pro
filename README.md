<img align="right" width="100" src="https://avatars.githubusercontent.com/u/100565733?s=200" alt="Super Lsky Pro Logo"/>

<h1 align="left">☁️ 兰空网盘 (Super Lsky Pro)</h1>

**Your media & files on the cloud.** —— 基于 [Lsky Pro v2](https://github.com/lsky-org/lsky-pro) 深度魔改的全格式多媒体图床与轻量级私有云网盘。

[![PHP](https://img.shields.io/badge/PHP->=8.0.2-orange.svg)](https://www.php.net)
[![Mod Guide](https://img.shields.io/badge/Blog-魔改技术指南-blueviolet.svg)](https://yyzmiao.top/2026/06/10/lskypro/)
[![Base](https://img.shields.io/badge/Based%20on-Lsky%20Pro%20v2-00b4d8.svg)](https://github.com/lsky-org/lsky-pro)
[![License](https://img.shields.io/badge/license-GPL_v3.0-yellowgreen.svg)](./LICENSE)

[📖 魔改技术博文](https://yyzmiao.top/2026/06/10/lskypro/) &middot;
[🛠️ 本地魔改手册](./docs/MOD_GUIDE.md) &middot;
[📘 Lsky Pro 原版文档](https://docs.lsky.pro) &middot;
[🏠 作者博客 (一叶梓喵)](https://yyzmiao.top)

---

## ✨ 为什么需要 Super Lsky Pro？

原版 **Lsky Pro (兰空图床)** 是一款极其优雅的开源云相册与图床系统，但受限于其“纯图片图床”的设计定位：
1. **严格限制上传后缀**：无法上传视频（`.mp4` / `.webm` / `.mov` 等）及其他通用网盘文件；
2. **非图片处理崩溃**：强行放开后缀会导致 `Intervention/Image` 图像压缩、水印叠加与鉴黄扫描在处理视频或普通文件时直接报错中断；
3. **无视频封面与播放能力**：即使通过底层绕过上传，画廊中也只会显示裂图，且 `Viewer.js` 灯箱无法播放视频。

**Super Lsky Pro（兰空网盘）** 在不引入任何臃肿后端组件（**零 FFmpeg 依赖**、不额外消耗低配服务器 CPU/内存）的前提下，通过前后端协同改造，让兰空图床平滑进化为**支持全格式文件托管、视频自动提取封面、全站原生弹窗播放**的全能多媒体云盘！

---

## 🚀 核心魔改特性

### 1. 🔓 解除全格式文件上传限制（智能分流管线）
- **前后端双重解禁**：移除上传组件 `<input type="file">` 的 `accept` 格式过滤与 JS 后缀拦截，后端 `ImageService::store()` 放开 `AcceptedFileSuffixes` 校验。
- **非图片安全跳过**：在后端引入 `$isImage` 智能分流，仅对标准图片格式执行格式转换、质量压缩、水印叠加与违规扫描；遇到视频或任意其他格式文件自动原样直通存储，彻底杜绝 `InterventionImage` 解析非图片异常。

### 2. 🎬 纯前端 Canvas 视频自动截帧封面（零 FFmpeg 依赖）
- **浏览器端算力卸载**：当用户在上传页拖入或选择视频（`mp4`、`webm`、`ogg`、`mov`、`avi`）时，前端自动通过内存 `<video>` 定位至第 `1` 秒，使用 `<canvas>` 提取高清视频帧并转为 Base64 缩略图。
- **后端缩略图落盘**：新增 `ImageService::makeThumbnailFromBase64()` 方法，随上传请求接收前端生成的视频封面并自动裁剪生成 `400x400` 标准缩略图，使视频在瀑布流画廊中拥有媲美原生视频网站的精美封面。

### 3. ▶️ 全站四端原生视频弹窗播放
全面改造站内所有预览入口，智能识别视频流并弹出高优先级沉浸式 `<video>` 播放器模态框：
- **上传队列实时预览**（`resources/views/components/upload.blade.php`）：上传前点击列表缩略图即可本地流式预览视频。
- **用户个人画廊**（`resources/views/user/images.blade.php`）：通过 `Viewer.js` 的 `filter` 钩子自动放行视频，点击视频封面直接弹出全屏播放器。
- **公共画廊广场**（`resources/views/common/gallery.blade.php`）：拦截视频直链跳转，原地弹出播放器流畅播放。
- **后台图片管理控制台**（`resources/views/admin/image/index.blade.php`）：将 `#image-tpl` 改造为动态 `__media__` 容器，管理员审查时可直接在详情弹窗内播放视频。

### 4. 🎛️ 后台角色组极简“全部允许”面板
- 精简 `resources/views/admin/group/add.blade.php` 与 `edit.blade.php` 中繁琐的后缀复选框矩阵，直观显示绿色高亮 **“全部允许”**，同时内置隐藏域完美兼容原版后端表单验证。

---

## 📂 魔改涉及文件清单

如果你已经部署了原版 Lsky Pro，只需将本仓库中的以下 **7 个文件** 覆盖至你的服务器对应路径即可完成升级（详见 [docs/MOD_GUIDE.md](./docs/MOD_GUIDE.md)）：

| 模块 | 文件路径 | 改造说明 |
| :--- | :--- | :--- |
| **后端存储服务** | [`app/Services/ImageService.php`](./app/Services/ImageService.php) | 取消后缀限制、非图片跳过压缩/水印/鉴黄、新增 `makeThumbnailFromBase64()` |
| **角色组创建页** | [`resources/views/admin/group/add.blade.php`](./resources/views/admin/group/add.blade.php) | 上传类型面板简化为“全部允许”并维持隐藏域校验 |
| **角色组编辑页** | [`resources/views/admin/group/edit.blade.php`](./resources/views/admin/group/edit.blade.php) | 上传类型面板简化为“全部允许”并维持隐藏域校验 |
| **前端上传组件** | [`resources/views/components/upload.blade.php`](./resources/views/components/upload.blade.php) | 移除文件选择器过滤、Canvas 视频第 1 秒截帧、队列内视频弹窗预览 |
| **用户个人画廊** | [`resources/views/user/images.blade.php`](./resources/views/user/images.blade.php) | 绕过 `Viewer.js` 视频拦截、绑定画廊视频弹窗播放器 |
| **公共画廊广场** | [`resources/views/common/gallery.blade.php`](./resources/views/common/gallery.blade.php) | 拦截视频卡片点击、弹出原生视频播放器 |
| **后台图片管理** | [`resources/views/admin/image/index.blade.php`](./resources/views/admin/image/index.blade.php) | 动态 `__media__` 模板替换，支持后台弹窗直接播放视频 |

---

## 🛠️ 安装与存量覆盖升级指南

### 方式一：存量 Lsky Pro 覆盖升级（推荐 Docker / 宝塔 / LNMP 已部署用户）

1. **备份原文件**：建议先备份站点目录下的 `app/Services/ImageService.php` 和 `resources/views/` 目录。
2. **覆盖魔改文件**：将上表列出的 7 个文件覆盖到你的 Lsky Pro 根目录（例如 `/opt/app-stack/lskypro` 或容器挂载目录）。
3. **清理 Laravel 视图缓存（必做）**：
   ```bash
   # 宿主机直接部署：
   php artisan view:clear

   # Docker 容器化部署（假设容器名为 lskypro）：
   docker exec -it lskypro php artisan view:clear
   ```
4. **强制刷新浏览器**：在浏览器按下 `Ctrl + F5` 刷新缓存，即可体验视频与全格式文件上传！
5. **调整上传大小限制**：如需上传较大视频，请同步调大后台 **角色组 -> 最大文件大小**，以及 Nginx 的 `client_max_body_size` 和 PHP 的 `upload_max_filesize` / `post_max_size`。

### 方式二：从源码全新部署

#### 环境要求
- PHP >= 8.0.2（需启用 `BCMath`, `Ctype`, `DOM`, `Fileinfo`, `JSON`, `Mbstring`, `OpenSSL`, `PDO`, `Tokenizer`, `XML`, `Imagick` 扩展）
-启用 `exec`, `shell_exec`, `readlink`, `symlink`, `putenv`, `getenv`, `chmod`, `chown`, `fileperms` 函数
- MySQL 5.7+ / PostgreSQL 9.6+ / SQLite 3.8.8+ / SQL Server 2017+

#### 部署步骤
```bash
git clone https://github.com/yyzmia1/super-lsky-pro.git
cd super-lsky-pro
composer install --no-dev -o
chmod -R 755 .
chown -R www-data:www-data .
```
将 Web 服务器（Nginx / Apache）的运行根目录指向 `public/`，配置伪静态规则后访问域名即可进入图形化安装向导。

---

## 📦 继承自 Lsky Pro 的原版核心特性

* [x] 支持 `本地` 及多种第三方云储存：`AWS S3`、`阿里云 OSS`、`腾讯云 COS`、`七牛云`、`又拍云`、`SFTP`、`FTP`、`WebDav`、`Minio`
* [x] 多种数据库驱动支持：`MySQL 5.7+`、`PostgreSQL 9.6+`、`SQLite 3.8.8+`、`SQL Server 2017+`
* [x] 支持多种缓存驱动：`Memcached`、`Redis`、`DynamoDB` 等，默认文件缓存
* [x] 多文件上传、拖拽上传、粘贴上传、动态切换存储策略、一键复制多格式链接（URL / HTML / BBCode / Markdown）
* [x] 瀑布流画廊展示，支持鼠标框选、右键菜单、单选多选、相册分类、重命名等操作
* [x] 自由度极高的角色组与储存策略多对多配置，支持路径/文件命名规则、频率限制、图片水印与文字水印
* [x] 完整的 RESTful API 接口支持

---

## 🙏 致谢与上游项目

- 上游开源项目：[lsky-org/lsky-pro](https://github.com/lsky-org/lsky-pro)（感谢原作者 [Wisp X](https://github.com/Wisp-X) 打造了如此优秀的图床系统）
- 魔改与维护：[yyzmia1 (一叶梓喵)](https://github.com/yyzmia1) · [技术博客详细解析](https://yyzmiao.top/2026/06/10/lskypro/)

---

## 📃 开源许可 (License)

本项目基于 [GNU General Public License v3.0 (GPL-3.0)](./LICENSE) 协议开源。

- Original Work Copyright (c) 2018-present [Lsky Pro (Wisp X)](https://github.com/lsky-org/lsky-pro).
- Modified Work Copyright (c) 2026-present [yyzmia1 (一叶梓喵)](https://github.com/yyzmia1).
