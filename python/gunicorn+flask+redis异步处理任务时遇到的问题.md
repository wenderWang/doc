**问题介绍**
在做一次 flask 项目时，客户有一需求：批量上传一批图片去丢到系统中去处理；
因为任务比较耗时，就采用了redis作为任务队列，然后起一个线程去redis中取任务，并将执行的结果写入数据库中，
因为session的原因，在启动线程的时候需要把app传入到执行的任务函数中去。
否则的话就需要创建threadsession，导致需要修改很多代码。

代码如下：
```python
# coding=utf-8
# @Time: 2020/12/25 10:26
import time
import json
from app.common.log import logger
from app.config.config import RedisConfig
from redis import StrictRedis, ConnectionError


class StartServer(object):

    def __init__(self):
        self._CINKATE_TASK = "cinkate_task"
        self.rcon = StrictRedis(RedisConfig.REDIS_HOST, RedisConfig.REDIS_PORT,
                                RedisConfig.REDIS_DB, RedisConfig.REDIS_PASSWORD)

    def start(self, _app):
        from app.service.extract_task_mgr_service import ExtractTaskMgrService
        while True:
            try:
                task = json.loads(self.rcon.blpop(self._CINKATE_TASK, 0)[1])

                logger.info('get task from {}, task: {}'.format(self._CINKATE_TASK, task))
                with _app.app_context():
                    ExtractTaskMgrService.push_pic_detail(**task)
                del task
            except ConnectionError as e:
                logger.exception(e)
                time.sleep(5)
            except Exception as e:
                logger.exception(e)


def start_server(app):
    server = StartServer()
    server.start(app)

```
因为此线程要同步监听redis，所以在 start_server.py 中启动线程

```python
# -*- coding: utf-8 -*-
# creation date: 2020-05-22
import os
import click
import typing
import threading
from flask.cli import AppGroup

from app.app import create_app
from app.common.extension import session
from app.common.redis_server import start_server


REPORT_SENTRY = os.environ.get('REPORT_SENTRY', "0")

if REPORT_SENTRY == "1":
    import sentry_sdk
    from sentry_sdk.integrations.flask import FlaskIntegration
    
    sentry_sdk.init(
        dsn="http://44836f0c859746e0a7c497bb9cd0d450@starport.datagrand.com/29",
        integrations=[FlaskIntegration()]
    )

app = create_app(os.getenv('ENV', 'development'))
app_cli = AppGroup('app', help='some commands work with app.')


@app.shell_context_processor
def make_shell_context() -> typing.Dict:
    return dict(
        dict(
            session=session,
        ),
        app=app,
    )


@app_cli.command('print')
@click.argument('name')
def print_app_attr_by_name(name) -> None:
    """
    > flask app print config
    <Config {...}>
    """
    print(getattr(app, name, None))

app.cli.add_command(app_cli)

if __name__ == "__main__":
    threading.Thread(target=start_server, args=[app]).start()
    app.run(host='0.0.0.0', port=10001, debug=True)
```

因为在debug的时候，运行的是这个文件，所以线程可以正常启动。
但是为了提高并发，在实际的镜像中是使用 gunicorn 启动的服务。如下
```shell
#!/usr/bin/env bash
SCRIPTS_DIR=$(cd `dirname $0`; pwd)
PROJECT_DIR=$(cd $(dirname ${SCRIPTS_DIR}); pwd)
cd "${PROJECT_DIR}"
pwd
ls

export PYTHONPATH=$(pwd)
export FLASK_APP=start_server.py
export FLASK_ENV=production
export FLASK_DEBUG=0
export UNRAR_LIB_PATH=/usr/lib/libunrar.so

python ready.py && flask db upgrade -d /ocr_platform_api/app/migrations

if [ "$?" != "0" ]; then
    echo "An error occurred when flask db migrate or upgrade"
    exit 1
fi

gunicorn start_server:app -c gunicorn_conf.py --reload
```
这就导致了，线程在初始化服务的时候没有被启动，导致上传的任务只能存在redis中，并没有真正的被处理


**解决办法**
在启动服务的时候，启动一个单独的服务去监听redis
复制一份start_server.py 为start_redis_server.py, 此脚本只用来起一个任务去监听redis任务队列
```python
# -*- coding: utf-8 -*-
# email: xunuo@datagrand.com
# creation date: 2020-05-22
import os
import click
import typing
from flask.cli import AppGroup

from app.app import create_app
from app.common.extension import session
from app.common.redis_server import start_server


REPORT_SENTRY = os.environ.get('REPORT_SENTRY', "0")

if REPORT_SENTRY == "1":
    import sentry_sdk
    from sentry_sdk.integrations.flask import FlaskIntegration
    
    sentry_sdk.init(
        dsn="http://44836f0c859746e0a7c497bb9cd0d450@starport.datagrand.com/29",
        integrations=[FlaskIntegration()]
    )

app = create_app(os.getenv('ENV', 'development'))
app_cli = AppGroup('app', help='some commands work with app.')


@app.shell_context_processor
def make_shell_context() -> typing.Dict:
    return dict(
        dict(
            session=session,
        ),
        app=app,
    )


@app_cli.command('print')
@click.argument('name')
def print_app_attr_by_name(name) -> None:
    """
    > flask app print config
    <Config {...}>
    """
    print(getattr(app, name, None))

app.cli.add_command(app_cli)

if __name__ == "__main__":
    start_server(app)
```
修改服务启动脚本为：
```shell
#!/usr/bin/env bash
SCRIPTS_DIR=$(cd `dirname $0`; pwd)
PROJECT_DIR=$(cd $(dirname ${SCRIPTS_DIR}); pwd)
cd "${PROJECT_DIR}"
pwd
ls

export PYTHONPATH=$(pwd)
export FLASK_APP=start_server.py
export FLASK_ENV=production
export FLASK_DEBUG=0
export UNRAR_LIB_PATH=/usr/lib/libunrar.so

python ready.py && flask db upgrade -d /ocr_platform_api/app/migrations

if [ "$?" != "0" ]; then
    echo "An error occurred when flask db migrate or upgrade"
    exit 1
fi

nohup python start_redis_server.py >> log/root.log 2>&1 &

gunicorn start_server:app -c gunicorn_conf.py --reload
```

问题解决！！！