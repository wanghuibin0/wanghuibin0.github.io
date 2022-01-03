# 模拟登录链滴网站


用 python 模拟登录链滴
<!--more-->

### 分析登录流程

通过浏览器抓包发现，登录过程就是向服务器发送一个 POST 请求，用户名密码伴随此请求传递给服务器。

![image.png](/assets/image-20210314175625-n7o21ay.png)

![image.png](/assets/image-20210314175659-r9033a3.png)

很容易看出，密码是明文经过加密后再传出去的，但是加密后的密码是不会变的，跟明文密码是对应的。模拟登录时，只需加密后的字符串复制过来给 POST 请求即可。

### 编写脚本

于是开始撸 python 脚本（下面是脚本的核心部分）：

```python
    login_url = 'https://ld246.com/login'
    login_headers = {
        'user-agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/89.0.4389.82 Safari/537.36 Edg/89.0.774.50',
    }

    data = {"nameOrEmail":"XXXX","userPassword":"XXXX","captcha":""}
    try:
        r = requests.post(login_url, headers=login_headers, json=data)
        r.raise_for_status()   #如果状态码不是200,抛出HTTPError异常
        r.encoding = r.apparent_encoding
        return r.text
    except:
        return "产生异常"
```

### 遇到的小问题

刚开始 post 请求写成了 `r = requests.post(login_url, headers=login_headers, data=data)`，然后就死活登录不成功，服务器总是返回“密码错误”，后来把 `data=data` 改为 `json=data` 才解决问题。
[Python requests.post 方法中 data 与 json 参数区别 - 随风飘-挨刀刀 - 博客园 (cnblogs.com)](https://www.cnblogs.com/yanlin-10/p/9820694.html)

其实在抓包时，是可以看出 json 和 data 在提交表单那里是不一样的。

ps: 本来我以为是网站有反爬措施，还研究了它的 cookie 生成机制，结果发现其实并没有什么反爬措施。

