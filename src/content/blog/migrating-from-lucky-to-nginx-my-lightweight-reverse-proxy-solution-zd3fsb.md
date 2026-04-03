---
title: 从 Lucky 迁移到 Nginx：我的轻量级反向代理方案
description: ''
pubDate: '2026-01-21'
tags:
  - 折腾
  - nginx
keywords: 折腾,nginx
---



# 从 Lucky 迁移到 Nginx：我的轻量级反向代理方案

Lucky 确实非常方便——证书自动续订、DDNS、Web 服务代理等功能开箱即用，几下点击就能搞定。但作为一个有点“开源洁癖”的用户，我还是决定换回更透明、可控的 Nginx。

起初我尝试过 ​**Nginx Proxy Manager（NPM）** ​，但它配置略显繁琐，尤其在多端口场景下不够灵活。最终，我选择了 **[nginx-ui](https://github.com/uozi/nginx-ui)** ——一个简洁、现代化且功能完整的 Nginx 可视化管理工具。

## 部署 nginx-ui

我使用 Docker Compose 进行部署，配置如下：

```vb
services:
  npm:
    image: uozi/nginx-ui:latest
    restart: unless-stopped
    container_name: nginxui
    network_mode: host
    environment:
      TZ: Asia/Shanghai
    ports:
      - 7080:80
      - 7443:443
    volumes:
      - /opt/dockerfile/nginxui/conf:/etc/nginx
      - /opt/dockerfile/nginxui/nginx-ui:/etc/nginx-ui
      - /var/run/docker.sock:/var/run/docker.sock
```

**说明**：这里使用 `network_mode: host`​ 是为了简化端口映射，其中的7080和7443就直接失效了，无法通过7080访问。我们代理了`/opt/dockerfile/nginxui/conf`可以去修改里面的端口，来达到修改默认访问地址的目的。

### 防火墙配置

在 Lucky 中，服务能直接访问，是因为它默认被加入了防火墙白名单。而自行部署的 nginx-ui 则需要手动放行。

进入你路由器或系统的 ​**防火墙 → 通信规则**，添加允许外部访问你所需端口（如 7080、7443 或自定义 HTTPS 端口）的规则：

![image](https://media.yuxh.cc/blog/20260121145645.png!inyaa)

## 配置 DDNS（以腾讯云 DNSPod 为例）

如果你使用的是内网穿透 + DDNS 方案，记得在代理软件中将检测本机ip的url加入 **直连列表**，避免请求到错误的ip：

![image](https://media.yuxh.cc/blog/20260121145803.png!inyaa)

### 腾讯云子账号权限配置

为了安全起见，建议使用子账号配合最小权限策略。在 **腾讯云控制台 → 访问管理 → 策略** 中创建如下策略：

```vb
{
    "statement": [
        {
            "action": [
                "dnspod:DescribeDomainList",
                "dnspod:DescribeRecordFilterList",
                "dnspod:DescribeRecordList",
                "dnspod:CreateRecord",
                "dnspod:ModifyRecord",
                "dnspod:DeleteRecord"
            ],
            "effect": "allow",
            "resource": [
                "*"
            ]
        }
    ],
    "version": "2.0"
}
```

然后在 nginx-ui 的 **凭证管理** 中填入子账号的 `SecretId`​ 和 `SecretKey`：

![image](https://media.yuxh.cc/blog/20260121145919.png!inyaa)

然后你配置dns域名的时候才好直接关联

![image](https://media.yuxh.cc/blog/20260121150222.png!inyaa)

## 申请 SSL 证书（DNS 验证方式）

在 nginx-ui 中，选择 ​**SSL 证书 → 新建证书**​，填写你的域名，并选择刚才配置好的腾讯云凭证。验证方式选择 ​**DNS**，系统会自动完成记录添加与验证：

![image](https://media.yuxh.cc/blog/20260121150302.png!inyaa)

然后就可以来站点列表配置站点了

![image](https://media.yuxh.cc/blog/20260121150343.png!inyaa)

因为，我们的证书使用的是dns验证，所以nginx本身配置具体如下

```vb
server {
    listen 42221 ssl;
    listen [::]:42221 ssl;
    server_name home.inyaw.com;
    ssl_certificate /etc/nginx/ssl/home.inyaw.com_P256/fullchain.cer;
    ssl_certificate_key /etc/nginx/ssl/home.inyaw.com_P256/private.key;
    location / {
        proxy_pass http://192.168.5.5:5001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

好的，这样就能访问了

## 总结

迁移到 nginx-ui 后，虽然初期配置稍多，但换来的是更高的灵活性、透明度和对数据的完全掌控。对于注重隐私和可维护性的用户来说，这是一次值得的“折腾”。

如果你也厌倦了黑盒工具，不妨试试这个组合：​**Nginx + nginx-ui + 腾讯云 DDNS + ACME 自动证书**，既轻量又强大。

---

希望这篇整理对你和其他读者有所帮助！
