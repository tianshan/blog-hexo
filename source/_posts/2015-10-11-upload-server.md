---
title: "搭建简易的文件服务器"
date: 2015-10-11
tags: nginx
---

需求很简单，就是一个能够存取文件的服务器，其他的事情都由客户端自己完成。

<!--more-->

选定nginx服务器后，折腾怎么上传，最多的选择是编译添加nginx-upload-module。
不过看了下，官方版本v2.2很久没更新了，和最新的nginx编译会出现编译出错，使用了一些不再支持的函数。

然后发现了国外13年的一片博文，[Nginx direct file upload without passing them through backend](https://coderwall.com/p/swgfvw/nginx-direct-file-upload-without-passing-them-through-backend), 尝试了 `clientbodyinfileonly` 感觉不太好使又去搜寻。

最后发现nginx-upload-module的github仓库的2.2分支最后14年有更新，而且解决了编译问题，所以还是回归这个，网上的教程也比较多。
可以通过 `git clone -b 2.2 https://github.com/vkholodkov/nginx-upload-module` 直接clone相应代码。
编译过程就不列出来了，demo里都有。

nginx搞定后加个后端稍微处理下文件。默认的nginx接收文件后，会以hash文件名保存到`upload_store`定义的文件夹。 用django移动到指定目录下，并重命名。（django有点大了，后面学习下flask重搞下）。

```bash
#搭建django
django-admin.py startproject uploadmodule
#启动工程
python manager.py runserver 0.0.0.0:9999
```

然后修改下`uploadmodule/urls.py`, 改成

```python
urlpatterns = [
    url(r'^upload/', 'uploadmodule.view.upload'),
    #第二个参数指定到view中的处理函数
]
```

在`uploadmodule/`中添加`view.py`，相比原来的文章，加了LOG和一点小处理，还有把文件的移动改调用 `os.rename`，这样在同一个块设备上不会涉及数据复制。

```python

# -*- coding: utf-8 -*-
import os
import json
import time
import logging
 
from django.http import HttpResponse
from django.views.decorators.csrf import csrf_exempt
 

# init logger
UPLOAD_FILE_PATH = os.path.join(os.path.dirname(__file__), '..', '..', 'files')
os.environ["TZ"] = "Asia/Shanghai"
log_path = 'log/upload.log'

if not os.path.exists(os.path.dirname(log_path)):
    os.makedirs(os.path.dirname(log_path))

LOG = logging.getLogger('view')
LOG.setLevel(logging.DEBUG)
filehandler = logging.handlers.TimedRotatingFileHandler(
    log_path, 'midnight')
formatter = logging.Formatter(
    '%(asctime)s %(name)-8s %(levelname)-5s: %(message)s')
filehandler.setFormatter(formatter)
filehandler.suffix = "%Y%m%d"
LOG.addHandler(filehandler)


@csrf_exempt
def upload(request):
    request_params = request.POST
    file_name = request_params['file_name']
    file_content_type = request_params['file_content_type']
    file_md5 = request_params['file_md5']
    file_path = request_params['file_path']
    file_size = request_params['file_size']
    file_user = request_params.get('user', None)

    ip_address = request.META.get('HTTP_X_REAL_IP') or request.META.get('HTTP_REMOTE_ADD')

    content = {
        'name': file_name,
        'content_type': file_content_type,
        'path': file_path,
        'size': file_size,
        'ip': ip_address,
        'state': 0,
        'error': ''
    }

    if not file_user:
        content['state'] = -1
        content['error'] = 'user field not set'
        LOG.error('no user request: %s %s' % (file_name, ip_address) )
        return http_response(content)
 
    # move to new name 'date+some_tag+name'
    now = time.strftime("%Y%m%d_%H%M", time.localtime())
    new_file_name = '%s_%s' % (now, file_name)
    new_file_dir = os.path.join(UPLOAD_FILE_PATH, file_user)
    new_file_path = os.path.join(new_file_dir, new_file_name)

    LOG.info('result: %s %s %s' % (file_user, file_name, ip_address))

    if not os.path.exists(new_file_dir):
        os.makedirs(new_file_dir)

    # 使用重命名修改文件属性，相比shutil.move是复制数据
    os.rename(file_path, new_file_path)
 
    response = http_response(content)

    return response

def http_response(content):
    content = json.dumps(content)
    response = HttpResponse(content, content_type='application/json; charset=utf-8')
    return response
```

最后，使用uwsgi来启动服务，添加配置文件`uwsgi.ini`

```
[uwsgi]
socket = 127.0.0.1:9999
master = true
pidfile = /tmp/uwsgi-upload-server.pid
daemonize = /tmp/uwsgi-upload-server.log
process = 1 
module = uploadmodule.wsgi:application
max-requests = 100
vacuum = true
limit-as = 1024
logdate = true
env = DJANGO_SETTINGS_MODULE=uploadmodule.settings
enable-threads = true
```


完整的demo放在[github](https://github.com/tianshan/upload-server)上了。

参考:
[使用Nginx Upload Module实现上传文件功能](http://xianglong.me/article/use-nginx-upload-module-to-implement-uploading-file-feature/)