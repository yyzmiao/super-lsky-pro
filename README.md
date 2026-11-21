<div align="center">

# Super Lsky Pro

**基于 Lsky Pro v2 深度改造的全格式多媒体图床与轻量级私有云盘**

全格式文件放行 · 浏览器端 Canvas 视频自动截帧 · 零 FFmpeg 服务端依赖 · 全站原生视频弹窗播放

<p>
  <a href="https://github.com/yyzmia1/super-lsky-pro/releases"><img alt="Base Version" src="https://img.shields.io/badge/Base-Lsky%20Pro%20v2-0369A1?style=flat-square" /></a>
  <a href="LICENSE"><img alt="License: GPL v3" src="https://img.shields.io/badge/License-GPLv3-blue.svg?style=flat-square" /></a>
  <img alt="PHP" src="https://img.shields.io/badge/PHP-%3E%3D8.0.2-777BB4?style=flat-square&logo=php&logoColor=white" />
  <img alt="Laravel" src="https://img.shields.io/badge/Laravel-Blade%20%2B%20JS-FF2D20?style=flat-square&logo=laravel&logoColor=white" />
</p>

<p>
  <a href="https://yyzmiao.top/2026/06/10/lskypro/">技术博文</a> ·
  <a href="docs/MOD_GUIDE.md">改造文档</a> ·
  <a href="https://docs.lsky.pro">上游文档</a>
</p>

</div>

---

## 项目简介

原版 [Lsky Pro](https://github.com/lsky-org/lsky-pro) 定位为纯图片图床，在服务端校验、图像处理管线与前端预览组件中均强耦合了图片格式假设：不仅拦截视频及常规文件上传，若直接放开后缀限制，还会触发 `Intervention/Image` 解码异常、`Viewer.js` 灯箱无法播放以及画廊缩略图缺失等问题。

**Super Lsky Pro** 在保留原版完整存储策略与权限体系的基础上，通过前后端协同改造打通了全格式文件托管与原生视频支持链路。整个方案**不依赖 FFmpeg 等外部二进制工具**，将视频首帧提取算力卸载至客户端浏览器，服务端仅负责安全分流与缩略图落盘，适用于轻量级 VPS 与容器化部署环境。

---

## 核心改造

- **全格式上传放行与安全分流**
  - 移除前端文件选择器 `<input type="file">` 的 `accept` 格式过滤及上传队列后缀校验，注释服务端 `ImageService::store()` 中的 `AcceptedFileSuffixes` 白名单拦截。
  - 在上传管线中引入 `$isImage` 媒体类型判定：仅对标准图片格式执行质量压缩、格式转换、水印叠加与第三方违规检测；视频及非图片文件直接透传至底层存储适配器（Local / S3 / OSS / COS / WebDAV 等），避免图像处理引擎报错中断。
- **浏览器端 Canvas 视频自动截帧（零 FFmpeg 依赖）**
  - 上传队列识别到 `mp4`、`webm`、`ogg`、`mov`、`avi` 视频文件时，在浏览器内存创建临时 `<video>` 实例并定位至第 `1` 秒，通过 `<canvas>` 绘制帧图像并导出为 Base64 缩略图。
  - 随表单提交字段 `thumbnail_base64` 回传至服务端，由新增的 `ImageService::makeThumbnailFromBase64()` 统一缩放裁剪为标准 `400×400` PNG 缩略图并落盘至 `public/thumbnails/`，确保画廊瀑布流正常渲染视频封面；非图片且非视频的常规文件则自动应用内置默认文件图标。
- **全站四端原生视频弹窗播放**
  - **上传队列预览**（`components/upload.blade.php`）：点击队列缩略图即可通过 `URL.createObjectURL` 本地流式播放待传视频。
  - **用户个人画廊**（`user/images.blade.php`）：通过 `Viewer.js` 的 `filter` 钩子排除视频条目，绑定自定义点击事件弹出全屏原生 `<video>` 播放器。
  - **公共画廊广场**（`common/gallery.blade.php`）：拦截 `.gallery-image-link` 视频直链跳转，原地弹出模态播放器。
  - **后台图片管理**（`admin/image/index.blade.php`）：将 `#image-tpl` 重构为 `__media__` 动态占位容器，后台审查详情时直接内嵌视频播放器。
- **后台角色组面板精简**
  - 将角色组创建与编辑页的“允许上传的图片类型”复选框矩阵简化为 **全部允许** 状态展示，同时保留隐藏域输出以完整兼容原有后端表单验证逻辑。

---

## 改动文件清单

全部改造集中于以下 **7 个核心文件**（逐行操作说明参见 [docs/MOD_GUIDE.md](docs/MOD_GUIDE.md)）：

| 模块 | 文件路径 | 核心变更 |
| :--- | :--- | :--- |
| 后端上传服务 | [`app/Services/ImageService.php`](app/Services/ImageService.php) | 解除后缀限制、`$isImage` 分流、新增 `makeThumbnailFromBase64()` |
| 角色组创建页 | [`resources/views/admin/group/add.blade.php`](resources/views/admin/group/add.blade.php) | 简化上传格式面板为“全部允许”并保留隐藏域 |
| 角色组编辑页 | [`resources/views/admin/group/edit.blade.php`](resources/views/admin/group/edit.blade.php) | 简化上传格式面板为“全部允许”并统一 `$default->get()` 调用 |
| 上传核心组件 | [`resources/views/components/upload.blade.php`](resources/views/components/upload.blade.php) | 移除 `accept` 过滤、Canvas 视频截帧、队列本地视频预览 |
| 用户个人画廊 | [`resources/views/user/images.blade.php`](resources/views/user/images.blade.php) | `Viewer.js` 视频过滤、画廊原生视频弹窗播放 |
| 公共画廊广场 | [`resources/views/common/gallery.blade.php`](resources/views/common/gallery.blade.php) | 视频卡片点击拦截与原生模态框播放 |
| 后台图片管理 | [`resources/views/admin/image/index.blade.php`](resources/views/admin/image/index.blade.php) | `__media__` 动态模板替换，支持后台详情直接播放视频 |

---

## 部署与升级

### 方式一：存量 Lsky Pro 实例覆盖升级

适用于已运行原版 Lsky Pro（Docker / 宝塔 / LNMP）的站点：

1. **备份原文件**：备份站点目录下的 `app/Services/ImageService.php` 与 `resources/views/`。
2. **覆盖改造文件**：将上述 7 个文件覆盖至站点根目录（如 `/opt/app-stack/lskypro`）。
3. **清理视图缓存**：
   ```bash
   # 宿主机部署
   php artisan view:clear

   # Docker 容器部署（以容器名 lskypro 为例）
   docker exec -it lskypro php artisan view:clear
   ```
4. **调整上传体积上限**：如需上传大体积视频，请同步调大后台 **角色组 → 最大文件大小**，以及 Nginx `client_max_body_size` 与 PHP `upload_max_filesize`、`post_max_size`。

### 方式二：全新源码部署

#### 运行环境要求

- PHP >= 8.0.2（需启用 `BCMath`、`Ctype`、`DOM`、`Fileinfo`、`JSON`、`Mbstring`、`OpenSSL`、`PDO`、`Tokenizer`、`XML`、`Imagick` 扩展）
- 需放行 `exec`、`shell_exec`、`readlink`、`symlink`、`putenv`、`getenv`、`chmod`、`chown`、`fileperms` 函数
- 数据库支持：`MySQL 5.7+` / `PostgreSQL 9.6+` / `SQLite 3.8.8+` / `SQL Server 2017+`

#### 安装命令

```bash
git clone https://github.com/yyzmia1/super-lsky-pro.git
cd super-lsky-pro
composer install --no-dev -o
chmod -R 755 .
chown -R www-data:www-data .
```

将 Web 服务根目录指向 `public/` 并配置 Laravel 伪静态规则，访问站点域名即可进入安装向导。

---

<details>
<summary>继承自上游 Lsky Pro 的原版基础能力</summary>

- **多存储后端**：支持本地及 `AWS S3`、`阿里云 OSS`、`腾讯云 COS`、`七牛云`、`又拍云`、`SFTP`、`FTP`、`WebDav`、`Minio`。
- **多缓存驱动**：支持 `Memcached`、`Redis`、`DynamoDB` 及文件缓存。
- **文件管理**：支持拖拽、粘贴、多文件并发上传，一键复制 URL / HTML / BBCode / Markdown 链接；瀑布流相册、鼠标框选、右键菜单与批量操作。
- **精细化权限与策略**：支持多角色组与多存储策略交叉绑定、路径与文件命名规则、上传频率限制、水印及第三方内容审核。
- **开放接口**：提供完整的 RESTful API 支持上传与媒体管理。

</details>

---

## 致谢

- 上游项目：[lsky-org/lsky-pro](https://github.com/lsky-org/lsky-pro)（原作者 [Wisp X](https://github.com/Wisp-X)）
- 魔改维护：[yyzmia1](https://github.com/yyzmia1) · [详细改造过程记录](https://yyzmiao.top/2026/06/10/lskypro/)

---

## 开源协议

本项目基于 **[GNU General Public License v3.0 (GPLv3)](LICENSE)** 协议开源。

- Original Work Copyright © 2018-present [Lsky Pro (Wisp X)](https://github.com/lsky-org/lsky-pro)
- Modified Work Copyright © 2026-present [yyzmia1](https://github.com/yyzmia1)
- 允许自由使用、修改与分发，但任何修改、衍生或重新发布的版本**必须同样以 GPLv3 开源并保留原作者版权声明**。
