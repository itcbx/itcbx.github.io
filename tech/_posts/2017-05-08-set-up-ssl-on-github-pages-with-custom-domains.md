---
title:    在使用自定义域名的Github Pages上启用SSL
subtitle: 通过CloudFlare启用SSL功能
author:   itcbx
---

## 开启SSL

1. 先确保你的Github Pages上已经开启了自定义域名。
2. 先注册一个CloudFlare账号，[点我注册](https://www.cloudflare.com/a/sign-up)。
3. 用注册好的账号登陆CloudFlare，根据提示添加你的域名，到最后一步会给出CloudFlare的NameServer，到你原来的域名提供商后台，将域名的NameServer改为CloudFlare提供的NameServer。
4. 到CloudFlare的设置里，将SSL的设置更改为`Flexible SSL`，这个的意思是用户到CloudFlare的访问使用SSL，CloudFlare到源网站的访问不使用SSL。

以上几步操作完成之后，你就可以使用https访问你的自定义域名了。以下一些设置建议也配置一下。

## 优化搜索引擎

在你的`_config.yml`添加以下配置：

```yml
url: https://www.yoursite.com   # with the https protocol
enforce_ssl: www.yoursite.com   # without any protocol
```

确保你网页文件的`<head>`标签里有以下语句：
```html
<link rel="canonical" href=" { { site.url } }{ { page.url } }" />
```

## 强制使用SSL

在你网页文件中的`<head>`标签里添加以下语句，以强制将http的访问跳转到https访问：
```html
{% if site.enforce_ssl %}
<script type="text/javascript">
if (window.location.protocol != "https:")
        window.location.protocol = "https";
</script>
{% endif %}
```

## 问题

如果改为SSL访问之后部分资源找不到，请将所有资源文件做如下修改：

```html
<!-- Change this -->
<link rel="stylesheet" href="http://www.somesite.com/path/to/styles.css">

<!-- to this: -->
<link rel="stylesheet" href="//www.somesite.com/path/to/styles.css">
```
