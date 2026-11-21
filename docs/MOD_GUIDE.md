# Lsky Pro 图床魔改指南：解除上传限制与原生视频支持

本指南详细记录了将 Lsky Pro 兰空图床从“纯图片图床”改造为“支持全格式文件上传、支持前端自动提取视频封面、支持全站原生弹出播放视频”的完整步骤。您可以按照以下指引，在任意未修改的 Lsky Pro 原版源码上复刻这一套功能。

> **原文博客**：[Lsky Pro 图床魔改指南：解除上传限制与原生视频支持 - 一叶梓喵的小站](https://yyzmiao.top/2026/06/10/lskypro/)

> [!WARNING]
> 修改底层源码存在一定风险。请在修改前**务必备份**原始文件。所有的文件路径均以 Lsky Pro 根目录（例如 `/opt/app-stack/lskypro`）为基准。

---

## 目录
1. [后端改造：取消后缀审核与保存前端截图](#1-后端改造取消后缀审核与保存前端截图)
2. [后台面板改造：极简显示“全部允许”](#2-后台面板改造极简显示全部允许)
3. [上传页改造：前端视频截帧与预览](#3-上传页改造前端视频截帧与预览)
4. [用户画廊改造：支持视频弹出播放](#4-用户画廊改造支持视频弹出播放)
5. [公共画廊改造：支持视频弹出播放](#5-公共画廊改造支持视频弹出播放)
6. [后台图片管理改造：直接播放视频](#6-后台图片管理改造直接播放视频)
7. [终末步骤：清理缓存](#7-终末步骤清理缓存)

---

## 1. 后端改造：取消后缀审核与保存前端截图
**文件路径**：`app/Services/ImageService.php`

在此文件中，我们要放开上传类型限制，遇到非图片文件跳过图片处理（如压缩、水印、鉴黄识别），并且增加处理前端发送来的视频封面的方法。

### 1.1 注释或删除严格的后缀名检查
在 `store()` 方法中，找到以下代码并将其**注释掉**或删除：
```php
// if (! in_array($extension, $configs->get(GroupConfigKey::AcceptedFileSuffixes))) {
//     throw new UploadException('不支持的文件类型');
// }
```

### 1.2 增加图片判断并跳过非图片处理
在判断质量和压缩的地方（约 154 行附近），加入 `$isImage` 判断变量。将后续的所有处理逻辑（压缩、加水印、涉黄扫描）全部包裹起来。修改后的大致结构如下：

```php
$isImage = in_array($extension, ['jpg', 'jpeg', 'png', 'gif', 'webp', 'bmp', 'ico', 'tif', 'svg']);

// 图片处理，跳过 ico gif svg
if ($isImage && ! in_array($extension, ['ico', 'gif', 'svg'])) {
    // 图片保存质量与格式
    // ...
}
```
向下寻找违规扫描部分，并同理进行修改：
```php
// 图片检测，跳过 tif、ico、psd、svg 格式
if ($isImage && $configs->get(GroupConfigKey::IsEnableScan) && ! in_array($extension, ['psd', 'ico', 'tif', 'svg'])) {
    // ...
}
```

### 1.3 接收视频封面
继续在 `store()` 方法底部，原本只生成图片的缩略图，现在让它接收我们前端制作的 base64 视频封面：

**修改前**：
```php
$this->makeThumbnail($image, $file);
```

**修改后**：
```php
// 生成缩略图
if ($isImage) {
    $this->makeThumbnail($image, $file);
} else if ($request->has('thumbnail_base64')) {
    // 如果是视频等非图片，且前端传来了 base64 封面
    $this->makeThumbnailFromBase64($image, $request->input('thumbnail_base64'));
}
```

### 1.4 新增处理 Base64 封面的方法
在 `ImageService.php` 文件的底部（`makeThumbnail` 方法之后），新增如下方法：

```php
/**
 * 从前端截图的 Base64 数据生成缩略图
 */
public function makeThumbnailFromBase64(Image $image, string $base64Data, int $max = 400, bool $force = false): void
{
    $pathname = public_path($image->getThumbnailPathname());

    if (! file_exists($pathname) || $force) {
        try {
            if (! is_dir(dirname($pathname))) {
                @mkdir(dirname($pathname), 0755, true);
            }
            @ini_set('memory_limit', '512M');
            
            // 剥离 Base64 头部
            $base64Data = preg_replace('/^data:image\/\w+;base64,/', '', $base64Data);
            $imgData = base64_decode($base64Data);

            $img = InterventionImage::make($imgData);

            $width = $w = $img->width();
            $height = $h = $img->height();

            if ($w > $max && $h > $max) {
                $scale = min($max / $w, $max / $h);
                $width  = (int)($w * $scale);
                $height = (int)($h * $scale);
            }

            $img->fit($width, $height, fn($constraint) => $constraint->upsize())->encode('png', 60)->save($pathname);
            $img->destroy();
        } catch (\Throwable $e) {
            Utils::e($e, '生成视频缩略图时出现异常');
        }
    }
}
```

---

## 2. 后台面板改造：极简显示“全部允许”
**文件路径**：
1. `resources/views/admin/group/add.blade.php`
2. `resources/views/admin/group/edit.blade.php`

为了让控制面板清爽，避免大段无用的多选框。找到两份文件中渲染“允许上传的图片类型”的 `<x-fieldset title="允许上传的图片类型">` 区域，将其替换为：

```html
<x-fieldset title="允许上传的图片类型">
    <div class="mt-2 text-sm text-green-600 font-bold">全部允许</div>
    <div class="hidden">
        @foreach($default->get('accepted_file_suffixes') as $extension)
            <input type="hidden" name="configs[accepted_file_suffixes][]" value="{{ $extension }}">
        @endforeach
    </div>
</x-fieldset>
```

---

## 3. 上传页改造：前端视频截帧与预览
**文件路径**：`resources/views/components/upload.blade.php`

这是最核心的前端部分。

### 3.1 取消前端类型拦截与选择器过滤
1. 移除顶部 `<input type="file" id="picker">` 的 `accept="..."` 属性，使文件选择窗口默认展示所有文件。
2. 找到 Javascript 部分的 `add: (e, data) => {` 方法，将类型拦截**注释掉**：
```javascript
// if (allowSuffixes.indexOf(ext.toLowerCase()) === -1) {
//     toastr.warning(`不支持的图片格式 ${file.name}`);
//     return true;
// }
```

### 3.2 增加前端截帧与表单注入
紧接着在下方，将图片预览生成的逻辑替换成支持视频读取的代码：

```javascript
let fileExt = file.name.substring(file.name.lastIndexOf(".") + 1).toLowerCase();
let isImage = ['jpg', 'jpeg', 'png', 'gif', 'webp', 'bmp', 'ico', 'tif', 'svg'].indexOf(fileExt) !== -1;
let isVideo = ['mp4', 'webm', 'ogg', 'mov', 'avi'].indexOf(fileExt) !== -1;

if (isVideo) {
    let video = document.createElement('video');
    video.src = URL.createObjectURL(file);
    video.muted = true;
    video.onloadeddata = () => { video.currentTime = 1; }; // 截取第一秒
    video.onseeked = () => {
        let canvas = document.createElement('canvas');
        canvas.width = video.videoWidth;
        canvas.height = video.videoHeight;
        canvas.getContext('2d').drawImage(video, 0, 0, canvas.width, canvas.height);
        let base64 = canvas.toDataURL('image/png');
        data.thumbnail_base64 = base64; // 将封面存在上下文中
        push(base64); // 渲染前端缩略图
        URL.revokeObjectURL(video.src);
    };
    video.onerror = () => {
        push('data:image/png;base64,...');
        URL.revokeObjectURL(video.src);
    };
} else if (!isImage || excludes.indexOf(fileExt) !== -1) {
    push('data:image/png;base64,...');
} else {
    loadImage(file, function (img) { ... /* 原来的 loadImage 逻辑 */ ... });
}
```

并且增加 `submit: function (e, data)` 回调，将刚才制作好的封面合并进请求表单：
```javascript
submit: function (e, data) {
    let currentFormData = [{name: 'strategy_id', value: $('#strategy-selected').data('id')}];
    if (data.thumbnail_base64) {
        currentFormData.push({name: 'thumbnail_base64', value: data.thumbnail_base64});
    }
    data.formData = currentFormData;
},
```

### 3.3 点击上传列表查看原生视频预览
在 `method === 'preview'` 中，增加弹窗逻辑：

```javascript
if (method === 'preview') {
    let file = queue[id].files[0];
    let fileExt = file.name.substring(file.name.lastIndexOf(".") + 1).toLowerCase();
    let isVideo = ['mp4', 'webm', 'ogg', 'mov', 'avi'].indexOf(fileExt) !== -1;

    if (isVideo) {
        let url = URL.createObjectURL(file);
        let videoModal = `
            <div id="video-preview-modal" style="z-index: 999999;" class="fixed inset-0 flex items-center justify-center bg-black bg-opacity-80" onclick="$(this).remove(); URL.revokeObjectURL('${url}')">
                <button style="position: fixed; top: 20px; right: 30px; font-size: 50px; line-height: 1; color: white; z-index: 9999999; cursor: pointer; background: transparent; border: none; text-shadow: 0 2px 4px rgba(0,0,0,0.8);" onclick="$('#video-preview-modal').remove(); URL.revokeObjectURL('${url}')">&times;</button>
                <div style="width: 100%; max-width: 1000px; padding: 20px; text-align: center;" onclick="event.stopPropagation()">
                    <video src="${url}" controls autoplay style="max-width: 100%; max-height: 85vh; margin: 0 auto; border-radius: 8px; box-shadow: 0 10px 25px rgba(0,0,0,0.8); background: #000; outline: none;"></video>
                </div>
            </div>
        `;
        $('body').append(videoModal);
    } else if (excludes.indexOf(fileExt) === -1) {
        let reader = new FileReader();
        reader.readAsDataURL(file);
        reader.onloadend = function (e) {
            $('#preview-modal img').attr('src', e.target.result);
            Alpine.store('modal').open('preview-modal')
        }
    }
}
```

---

## 4. 用户画廊改造：支持视频弹出播放
**文件路径**：`resources/views/user/images.blade.php`

在此文件中我们要绕过默认的 `Viewer.js`（它不支持播放 mp4），转而抛出定制的视频播放器。

### 4.1 绕过 Viewer.js 拦截视频
搜索 `new Viewer`，为其加上 `filter` 选项：
```javascript
const viewer = new Viewer(document.getElementById('images-grid'), {
    url: 'data-original',
    filter(image) {
        let src = image.getAttribute('data-original');
        return !src.match(/\.(mp4|webm|mov|avi|ogg)$/i);
    }
});
```

### 4.2 自己监听视频点击并弹出播放器
在文件底部的 JavaScript 区域，添加专门针对视频缩略图被点击的监听器：
```javascript
$photos.on('click', 'img', function (e) {
    let url = $(this).attr('data-original');
    if (url.match(/\.(mp4|webm|mov|avi|ogg)$/i)) {
        e.stopPropagation();
        let videoModal = `
            <div id="video-preview-modal" style="z-index: 999999;" class="fixed inset-0 flex items-center justify-center bg-black bg-opacity-80" onclick="$(this).remove()">
                <button style="position: fixed; top: 20px; right: 30px; font-size: 50px; line-height: 1; color: white; z-index: 9999999; cursor: pointer; background: transparent; border: none; text-shadow: 0 2px 4px rgba(0,0,0,0.8);" onclick="$('#video-preview-modal').remove()">&times;</button>
                <div style="width: 100%; max-width: 1000px; padding: 20px; text-align: center;" onclick="event.stopPropagation()">
                    <video src="${url}" controls autoplay style="max-width: 100%; max-height: 85vh; margin: 0 auto; border-radius: 8px; box-shadow: 0 10px 25px rgba(0,0,0,0.8); background: #000; outline: none;"></video>
                </div>
            </div>
        `;
        $('body').append(videoModal);
    }
});
```

---

## 5. 公共画廊改造：支持视频弹出播放
**文件路径**：`resources/views/common/gallery.blade.php`

这是对外公开的图片画廊页。

### 5.1 修改图片项的 HTML 类名
搜索展示项的 `a` 标签，为它加入一个特有的类名 `gallery-image-link`：
```html
<a target="_blank" href="{{ $image->url }}" class="gallery-image-link">
    <div class="relative overflow-hidden w-full h-32">
        <img class="grow object-cover object-center w-full h-full" src="{{ $image->thumb_url }}"/>
    </div>
</a>
```

### 5.2 监听点击并弹出播放器
在底部的 `@push('scripts')` 中，加入视频拦截代码：
```javascript
$('.images-grid').on('click', '.gallery-image-link', function(e) {
    let url = $(this).attr('href');
    if (url.match(/\.(mp4|webm|mov|avi|ogg)$/i)) {
        e.preventDefault();
        let videoModal = `
            <div id="video-preview-modal" style="z-index: 999999;" class="fixed inset-0 flex items-center justify-center bg-black bg-opacity-80" onclick="$(this).remove()">
                <button style="position: fixed; top: 20px; right: 30px; font-size: 50px; line-height: 1; color: white; z-index: 9999999; cursor: pointer; background: transparent; border: none; text-shadow: 0 2px 4px rgba(0,0,0,0.8);" onclick="$('#video-preview-modal').remove()">&times;</button>
                <div style="width: 100%; max-width: 1000px; padding: 20px; text-align: center;" onclick="event.stopPropagation()">
                    <video src="${url}" controls autoplay style="max-width: 100%; max-height: 85vh; margin: 0 auto; border-radius: 8px; box-shadow: 0 10px 25px rgba(0,0,0,0.8); background: #000; outline: none;"></video>
                </div>
            </div>
        `;
        $('body').append(videoModal);
    }
});
```

---

## 6. 后台图片管理改造：直接播放视频
**文件路径**：`resources/views/admin/image/index.blade.php`

这是管理员的控制台。我们要将原本固定渲染 `<img>` 的 HTML 模板改成动态占位符，由 JS 来判断该渲染视频还是图片。

### 6.1 修改 Alpine Modal 渲染模板
搜索 `id="image-tpl"`，将其中的图片渲染部分改为占位符 `__media__`：
```html
<script type="text/html" id="image-tpl">
    <div class="w-full mt-4">
        <div class="w-full mb-4 rounded-sm overflow-hidden flex items-center justify-center">
            __media__
        </div>
```

### 6.2 修改 JS 注入逻辑
在页面底部找到 `$('.item').click(function () { ...`，增加视频的识别，并替换 `__media__`：

```javascript
$('.item').click(function () {
    let image = $(this).data('json');
    let previewUrl = ['psd', 'tif'].indexOf(image.extension) === -1 ? image.url : image.thumb_url;
    
    // 判断并动态生成标签
    let isVideo = ['mp4', 'webm', 'ogg', 'mov', 'avi'].indexOf(image.extension) !== -1;
    let mediaHtml = isVideo 
        ? '<video controls autoplay class="w-full h-auto max-h-[80vh] shadow-lg rounded bg-black" src="'+previewUrl+'"></video>'
        : '<a class="w-full" href="'+previewUrl+'" target="_blank"><img src="'+previewUrl+'" alt="'+image.name+'" class="w-full object-center object-cover"></a>';

    let html = $('#image-tpl').html()
        .replace(/__media__/g, mediaHtml)  // 替换刚才预留的占位符
        .replace(/__id__/g, image.id)
        // ... 继续保留后面的原始代码
```

---

## 7. 终末步骤：清理缓存
Lsky Pro 拥有严密的视图层缓存系统。当您完成了以上所有 `.blade.php` 模板文件的更改并覆盖回服务器后，您**必须**在服务器终端（或者对应的 Docker 容器中）执行一次：

```bash
php artisan view:clear
```
清理掉旧的缓存视图。然后去浏览器中按下 **Ctrl + F5（强制刷新页面）**。至此，您的极客魔改大业全部完成！
