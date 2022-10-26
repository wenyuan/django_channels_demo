# Django 集成 Channels 使用案例

该项目出自[官方文档](https://channels.readthedocs.io/en/stable/tutorial/index.html)的案例，增加了一些更详细的代码解读和注释。实现步骤可以通过官方文档进行学习。

## 依赖环境

* Linux
* Python 3.8
* Django 4.1.2（兼容 2.x 和 3.x）
* Channels 3.0.5

## 前置准备

* 本项目配套文档参考 [Django 与 Channels 的集成](https://www.fedbook.cn/backend-knowledge/django/django-integrate-channels/)
* 安装 Redis 参考 [Redis 的安装与卸载](https://www.fedbook.cn/basic-skills/redis/installation-of-redis/)
* 安装 Python 及虚拟环境参考 [Python 多版本虚拟环境共存](https://www.fedbook.cn/backend-knowledge/python/multiple-python-install-on-linux/)

## 运行方式

下载、安装、运行代码：

```bash
git clone https://github.com/wenyuan/django_channels_demo.git
pip install -r requirements.txt
cd django_channels_demo/
python manage.py runserver
```

浏览器访问 `ip:port/chat`，输入任意聊天室的名字，进入聊天室开始聊天（WebSocket 测试）。

## 主要文件

下面对这个小案例涉及到的主要文件/文件夹进行标注，省略跟 Channels 无关的文件。

```
django_channels_demo/
├── chat
│   │── templates/      # 模板文件（即 html）
│   │── urls.py         # 映射到视图的路由，处理 http 请求
│   │── views.py        # 视图文件，接收并处理 http 链接
│   │── routing.py      # 到消费者的路由，专门处理 websocket，类似普通的处理 http 请求的 urls.py
│   │── consumers.py    # 消费者，接收并处理 websocket 链接
│   └── ...（其它省略）
├── django_channels_demo
│   │── settings.py/    # Django 配置文件
│   │── urls.py         # 项目根路由
│   │── asgi.py         # 配置 ASGI 协议，让它替代并兼容原本的 WSGI 协议
│   └── ...（其它省略）
└── ...
```
