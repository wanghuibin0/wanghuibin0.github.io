# 利用Nginx实现服务请求按名称分发——再也不用担心记不住端口号啦

### 缘起

在 vps 上部署了几个服务：nextcloud、ttrss、siyuan。这些服务监听着不同的端口，从浏览器访问时，必须输入特定的端口才能连上对应的服务。于是我突发奇想，何不效仿 dns 做法，用直观的名字去映射难记的数字呢？说干就干，决定用反向代理软件 Nginx，实现我这个想法。
<!--more-->

### 实现方法

在 Nginx 中配置一个虚拟服务器端口，做为分发前的总入口，然后利用 location 字段，针对不同路径做分发，重定向至对应服务的端口，即可。具体如下：

```bash
  server {
    listen 127.0.0.1:8080;

    location ~ /ttrss {
      #access_log /home/wanghb/nginx-log/ttrss-access.log;
      #error_log /home/wanghb/nginx-log/ttrss-error.log debug;

      return 301 http://$server_name:portA;
    }
    location ~ /nextcloud {
      return 301 http://$server_name:portB/apps/dashboard/;
    }
    location ~ /siyuan {

      return 301 http://$server_name:portC/stage/build/desktop/;
    }
    location / {
      proxy_pass https://wanghuibin0.github.io;
    }
  }
```

### 效果

效果就是，在浏览器输入 `https://server-name:8080/ttrss`，能自动重定向到 ttrss 服务的端口去，其他两个服务亦然。最后一个 `location / {}` 块的含义是，如果与前面都不匹配，就默认反向代理自己的博客网站。

ps:

最开始，想用 `proxy_pass` 反向代理的方式实现路径分流，但是对 Nginx 配置不熟，捣鼓半天也没成功，后来改为 `return 301` 重定向的方法，很容易达到了自己想要的效果。

pps:

中间曾觉得 Nginx 配置太复杂，一度想换 caddy 做。但是在了解 caddy 的过程中，发现 caddy 也不是完美的，简单是简单一些，但灵活性不如 Nginx。后来遂换回 Nginx。不过现在想想，如果只是简单的重定向，caddy 应该也是能够胜任的。

