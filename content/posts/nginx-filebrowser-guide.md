---
title: "Nginx"
date: "2026-10-10"
updated: "2026-10-10"
draft: false
sticky: null
tags: ["Nginx", "HTTPS"]
categories: ["分享"]
description: "Nginx。"
image: "/images/r.jpg"
---
# Nginx + File Browser 配置实战笔记

> 场景：Windows 下使用 Nginx 搭建 HTTPS 文件下载服务（fancyindex + Basic Auth），并按用户隔离目录；同时把本机 File Browser（127.0.0.1:6060）通过反向代理挂到 `/file/` 虚拟目录。
> 域名示例：`alienx.ggff.net:88`（外网 443→88 映射）

---

## 目录

1. [基础配置：HTTP 强转 HTTPS + fancyindex](#一基础配置http-强转-https--fancyindex)
2. [美化列表：CSS / header / footer](#二美化列表-css--header--footer)
3. [Basic Auth 认证与密码文件](#三basic-auth-认证与密码文件)
4. [按用户隔离根目录（多用户）](#四按用户隔离根目录多用户)
5. [反向代理：把 File Browser 挂到 /file/](#五反向代理把-file-browser-挂到-file)
6. [完整可用配置（整合版）](#六完整可用配置整合版)
7. [排错速查表](#七排错速查表)

---

## 一、基础配置：HTTP 强转 HTTPS + fancyindex

### 1.1 最小结构

```nginx
worker_processes  1;

events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile      on;
    keepalive_timeout 65;

    # HTTP → HTTPS 强制跳转
    server {
        listen       80;
        server_name  alienx.ggff.net;
        return 301 https://$host:88$request_uri;
    }

    # HTTPS 主服务
    server {
        listen       443 ssl;
        server_name  alienx.ggff.net;

        ssl_certificate      D:/nginx/key/alienx.pem;
        ssl_certificate_key  D:/nginx/key/alienx.key;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers HIGH:!aNULL:!MD5;

        root E:/实用软件/;
        autoindex off;

        location / {
            fancyindex on;
            fancyindex_exact_size off;   # 显示友好大小(KB/MB)
            fancyindex_localtime on;     # 使用本地时间
            try_files $uri $uri/ =404;
        }
    }
}
```

### 1.2 关键指令说明

| 指令 | 作用 |
|------|------|
| `fancyindex on` | 开启美化目录列表 |
| `fancyindex_exact_size off` | 显示 KB/MB 而非字节 |
| `fancyindex_localtime on` | 时间按服务器本地时区 |
| `fancyindex_ignore "xxx"` | 列表中隐藏指定文件/目录 |
| `fancyindex_css_href` | 引入自定义 CSS |
| `fancyindex_header` / `footer` | 插入页头/页脚 HTML 片段 |

> ⚠️ `fancyindex_theme builtin` 需要较新版本的 ngx-fancyindex 模块。老模块不支持该指令，启动会报 `unknown directive "fancyindex_theme"`。老版本请用 `fancyindex_css_href` 自定义样式。

---

## 二、美化列表：CSS / header / footer

### 2.1 目录结构约定

```
D:/nginx/Nginx-Fancyindex/
    styles.css       # 样式
    header.html      # 页头片段
    footer.html      # 页脚片段
```

### 2.2 配置引用

```nginx
location / {
    fancyindex on;
    fancyindex_exact_size off;
    fancyindex_localtime on;

    fancyindex_css_href   /Nginx-Fancyindex/styles.css;
    fancyindex_header     /Nginx-Fancyindex/header.html;
    fancyindex_footer     /Nginx-Fancyindex/footer.html;

    fancyindex_ignore "dufs.exe" "run.bat" "sync.ffs_db" "_gsdata_";
    try_files $uri $uri/ =404;
}

# 静态资源免认证（放 CSS/图片等）
location /Nginx-Fancyindex {
    alias D:/nginx/Nginx-Fancyindex;
    auth_basic off;
    access_log off;
}
```

### 2.3 修改 CSS 是否需要重启？

- **不需要重启 Nginx**。CSS 是静态文件，Nginx 直接读盘返回。
- **但要清浏览器缓存**：`Ctrl+F5` / 无痕模式，或在引用上加版本号：
  ```nginx
  fancyindex_css_href /Nginx-Fancyindex/styles.css?v=2;
  ```
- 若列表宽度太宽：调整 `body { max-width: ... }`，用开发者工具确认样式是否被应用。

### 2.4 header.html 示例

```html
<div style="display:flex;align-items:center;justify-content:space-between;padding:16px 0;border-bottom:1px solid #334155;margin-bottom:12px;">
  <div>
    <h1 style="margin:0;font-size:22px;color:#f8fafc;">📁 AlienX 文件服务器</h1>
    <p style="margin:4px 0 0;font-size:14px;color:#94a3b8;">欢迎下载，请先登录</p>
  </div>
</div>
```

### 2.5 footer.html 示例

```html
<div style="margin-top:20px;padding:12px 0;border-top:1px solid #334155;text-align:center;font-size:13px;color:#64748b;">
  © 2026 AlienX · <a href="https://alienx.ggff.net" style="color:#60a5fa;text-decoration:none;">alienx.ggff.net</a>
</div>
```

---

## 三、Basic Auth 认证与密码文件

### 3.1 开启认证

```nginx
auth_basic "Restricted";
auth_basic_user_file D:/nginx/pass/.htpasswd;
```

### 3.2 密码文件说明

- **文件名不必叫 `.htpasswd`**，任意名均可（如 `alienx.users`），只要 `auth_basic_user_file` 指向正确。
- 文件内容格式：`用户名:哈希`
- 建议放在 **web 根之外**的目录（如 `D:/nginx/pass/`），避免被直接下载。

### 3.3 创建用户

```cmd
:: 新建（首个用户，用 -c）
htpasswd -c D:/nginx/pass/alienx.users alice
:: 追加用户（不要 -c，否则覆盖）
htpasswd D:/nginx/pass/alienx.users bob
```

无 `htpasswd` 时用 OpenSSL：

```cmd
openssl passwd -apr1
```

### 3.4 自定义 401 页面

```nginx
error_page 401 /401.html;
location = /401.html {
    root D:/nginx/html;
    internal;          # 仅内部跳转可访问
    auth_basic off;    # 必须关闭，否则死循环
}
```

### 3.5 HTTPS 下密码是否加密传输？

**是**。HTTPS（TLS）加密整个通信，包括 Basic Auth 的凭证。Basic Auth 本身是 Base64（非加密），但被 TLS 层保护，传输过程无法被窃听。**务必配合 HTTPS 使用**。

---

## 四、按用户隔离根目录（多用户）

### 4.1 原理

Basic Auth 成功后，Nginx 把用户名存入 `$remote_user` 变量。用 `map` 把它映射成不同的根目录。

```nginx
http {
    map $remote_user $user_root {
        default     E:/实用软件/no_access/;   # 未匹配/未登录
        alice       E:/实用软件/alice/;
        bob         E:/实用软件/bob/;
    }

    server {
        # ...
        auth_basic "Restricted";
        auth_basic_user_file D:/nginx/pass/alienx.users;

        root $user_root;     # 动态根目录
        autoindex off;

        # 防路径穿越
        if ($uri ~ '\.\.') { return 403; }

        location / {
            fancyindex on;
            try_files $uri $uri/ =404;
        }
    }
}
```

### 4.2 安全性

- alice 登录后只能看到 `E:/实用软件/alice/` 下的内容，无法跨到 bob 目录。
- Nginx 默认会规范化 `..` 路径，不会泄露上级目录。
- `default` 指向一个空/无权限目录，防止未登录用户看到默认内容。

---

## 五、反向代理：把 File Browser 挂到 /file/

### 5.1 目标

- File Browser 运行在本机 `127.0.0.1:6060`
- 通过 `https://alienx.ggff.net:88/file/` 访问，复用 HTTPS 证书与域名
- File Browser 自己管理登录，不走 Nginx 的 Basic Auth

### 5.2 Nginx 配置（关键）

```nginx
# /file 不带斜杠 → 重定向（也可直接代理）
location = /file {
    auth_basic off;
    proxy_pass http://127.0.0.1:6060/;
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_buffering off;
    client_max_body_size 0;
}

# /file/ 带斜杠：正常代理
location ^~ /file/ {
    auth_basic off;                       # 关键：关掉下载服务的密码
    proxy_pass http://127.0.0.1:6060/;   # 末尾斜杠 = 去掉 /file 前缀
    proxy_http_version 1.1;

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    proxy_buffering off;
    client_max_body_size 0;               # 上传不限大小，按需调整

    # WebSocket（如需）：
    # proxy_set_header Upgrade $http_upgrade;
    # proxy_set_header Connection "upgrade";
}
```

### 5.3 必须设置 File Browser 的 baseurl

让 File Browser 知道自己挂在 `/file` 下，否则它生成的 CSS/JS/API 链接会指向根路径，导致样式丢失或跳回 Nginx 认证页。

```cmd
filebrowser config set --address 127.0.0.1 --port 6060 --baseurl /file
```

或在启动命令中指定：

```cmd
filebrowser -r E:\实用软件 -a 127.0.0.1 -p 6060 --baseurl /file
```

> 设置 baseurl 后，本地测试也应访问 `http://127.0.0.1:6060/file/`。

### 5.4 路径对应规则

| 浏览器访问 | Nginx 转发到 FB | FB 看到的路径 |
|-----------|----------------|--------------|
| `/file/login` | `proxy_pass .../` | `/login` ✅ |
| `/file/api/...` | `proxy_pass .../` | `/api/...` ✅ |

若 `proxy_pass` **不带**末尾斜杠，则 `/file/xxx` 会原样转发为 `/file/xxx`，此时 FB **不能**设 baseurl（两者只能选一种对齐方式）。推荐：带斜杠 + baseurl。

---

## 六、完整可用配置（整合版）

```nginx
worker_processes  1;

events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile      on;
    keepalive_timeout 65;

    # ===== 用户 → 根目录映射（多用户隔离，可选）=====
    map $remote_user $user_root {
        default     E:/实用软件/no_access/;
        alice       E:/实用软件/alice/;
        bob         E:/实用软件/bob/;
    }

    # HTTP → HTTPS 强制跳转
    server {
        listen       80;
        server_name  alienx.ggff.net;
        return 301 https://$host:88$request_uri;
    }

    # HTTPS 主服务
    server {
        listen       443 ssl;
        server_name  alienx.ggff.net;

        ssl_certificate      D:/nginx/key/alienx.pem;
        ssl_certificate_key  D:/nginx/key/alienx.key;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers HIGH:!aNULL:!MD5;

        root E:/实用软件/;
        autoindex off;

        # 全局 Basic Auth
        auth_basic "Restricted";
        auth_basic_user_file D:/nginx/pass/.htpasswd;

        # 防路径穿越
        if ($uri ~ '\.\.') { return 403; }

        # 自定义 401
        error_page 401 /401.html;
        location = /401.html {
            root D:/nginx/html;
            internal;
            auth_basic off;
        }

        # 主目录：fancyindex
        location / {
            fancyindex on;
            fancyindex_exact_size off;
            fancyindex_localtime on;
            fancyindex_css_href   /Nginx-Fancyindex/styles.css;
            fancyindex_header     /Nginx-Fancyindex/header.html;
            fancyindex_footer     /Nginx-Fancyindex/footer.html;
            fancyindex_ignore "dufs.exe" "run.bat" "sync.ffs_db" "_gsdata_";
            try_files $uri $uri/ =404;
        }

        # ===== File Browser 反向代理 =====
        location = /file {
            auth_basic off;
            proxy_pass http://127.0.0.1:6060/;
            proxy_http_version 1.1;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_buffering off;
            client_max_body_size 0;
        }

        location ^~ /file/ {
            auth_basic off;
            proxy_pass http://127.0.0.1:6060/;
            proxy_http_version 1.1;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_buffering off;
            client_max_body_size 0;
        }

        # 禁止直接访问隐藏文件
        location = /dufs.exe    { return 404; }
        location = /run.bat     { return 404; }
        location = /sync.ffs_db { return 404; }
        location ~* /(dufs\.exe|run\.bat|sync\.ffs_db)$ { return 404; }
        location ~ /_gsdata_(/|$) { return 404; }

        # 静态资源（CSS 等）免认证
        location /Nginx-Fancyindex {
            alias D:/nginx/Nginx-Fancyindex;
            auth_basic off;
            access_log off;
        }

        # 禁止访问 .ht 文件
        location ~ /\.ht {
            deny all;
        }
    }
}
```

---

## 七、排错速查表

| 现象 | 原因 | 解决 |
|------|------|------|
| `unknown directive "fancyindex_theme"` | 老模块不支持 | 删掉该指令，用 `fancyindex_css_href` |
| CSS 改了没变化 | 浏览器缓存 / 路径错 | 无痕模式；确认 `/Nginx-Fancyindex/styles.css` 可访问；加 `?v=2` |
| header/footer 不显示 | 路径错 / 被认证拦截 | 确认文件存在 + `location` 下 `auth_basic off` |
| 访问 `/aaa` 也弹认证 | Basic Auth 在认证前生效，正常 | 想免认证：`location ^~ /aaa/ { auth_basic off; ... }` |
| `/file` 跳到下载登录页 | `/file` location 没关 auth_basic | 在 `location = /file` 也加 `auth_basic off` |
| File Browser 空白/样式丢 | baseurl 未设置 / proxy 路径错 | 设 `--baseurl /file` + `proxy_pass` 末尾带 `/` |
| 502 Bad Gateway | FB 没运行 / 端口错 | `netstat -ano | findstr :6060` 检查 |
| 404 on /file/ | baseurl 与 proxy 不匹配 | 统一为「带斜杠 + baseurl /file」 |
| 未登录用户能看到默认目录 | map default 没设好 | 指向空目录或加 `if ($remote_user = "") { return 403; }` |

### 常用命令

```cmd
D:\nginx\nginx.exe -t            :: 测试配置语法
D:\nginx\nginx.exe -s reload     :: 重载配置（不中断服务）
D:\nginx\nginx.exe -T            :: 显示最终生效的全部配置
netstat -ano | findstr :6060     :: 检查 File Browser 是否在监听
```

### 最佳实践小结

1. 密码文件放在 web 根之外 + 配合 HTTPS。
2. fancyindex 的静态资源（CSS/header/footer）所在 location 必须 `auth_basic off`。
3. 反向代理到 File Browser 的两个 location（`= /file` 和 `^~ /file/`）都要 `auth_basic off`。
4. **File Browser 务必设 `--baseurl /file`**，与 `proxy_pass http://.../;` 末尾斜杠成对出现。
5. 每次改配置：`-t` 测试 → `-s reload` 重载 → 无痕窗口验证。

---

*整理自 Nginx + File Browser 配置实战全过程，供后续查阅。*
