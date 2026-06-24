# How to use Flask with gthread (uWSGI and Gunicorn editions)

## Create simple Flask application

First, we need to emulate a slow 3rd party API:

```python
# slow_api/api.py
import os

import asyncio
from aiohttp import web

async def handle(request):
    delay = float(request.query.get('delay') or 1)
    await asyncio.sleep(delay)
    return web.Response(text='slow api response')

app = web.Application()
app.add_routes([web.get('/', handle)])

if __name__ == '__main__':
    web.run_app(app, port=os.environ['PORT'])
```

Then, we create a simple flask application with a dependency on the slow 3rd party API:

```python
# flask_app/app.py
import os

import requests
from flask import Flask, request

api_port = os.environ['PORT_API']
api_url = f'http://slow_api:{api_port}/'

app = Flask(__name__)

@app.route('/')
def index():
    delay = float(request.args.get('delay') or 1)
    resp = requests.get(f'{api_url}?delay={delay}')
    return 'Hi there! ' + resp.text
```

## Deploy Flask application using Flask dev server

```bash
# Build and start app served by Flask dev server
$ docker-compose -f sync-devserver.yml build
$ docker-compose -f sync-devserver.yml up

# Test single-threaded deployment
$ ab -r -n 10 -c 5 http://127.0.0.1:3000/?delay=1
> Concurrency Level:      5
> Time taken for tests:   10.139 seconds
> Complete requests:      10
> Failed requests:        0
> Requests per second:    0.99 [#/sec] (mean)

# Test multi-threaded deployment
$ ab -r -n 10 -c 5 http://127.0.0.1:3001/?delay=1
> Concurrency Level:      5
> Time taken for tests:   3.069 seconds
> Complete requests:      10
> Failed requests:        0
> Requests per second:    3.26 [#/sec] (mean)
```

## Deploy Flask application using uWSGI (4 worker processes x 50 threads each)

```bash
# Build and start app served by uWSGI
$ docker-compose -f sync-uwsgi.yml build
$ docker-compose -f sync-uwsgi.yml up

$ ab -r -n 2000 -c 200 http://127.0.0.1:3000/?delay=1
> Concurrency Level:      200
> Time taken for tests:   12.685 seconds
> Complete requests:      2000
> Failed requests:        0
> Requests per second:    157.67 [#/sec] (mean)
```

## Deploy Flask application using Gunicorn (4 worker processes x 50 threads each)

```bash
# Build and start app served by Gunicorn
$ docker-compose -f sync-gunicorn.yml build
$ docker-compose -f sync-gunicorn.yml up

$ ab -r -n 2000 -c 200 http://127.0.0.1:3000/?delay=1
> Concurrency Level:      200
> Time taken for tests:   13.427 seconds
> Complete requests:      2000
> Failed requests:        0
> Requests per second:    148.95 [#/sec] (mean)
```

## Deploy Flask application using uWSGI with Python 3.14t

Under Python 3.14t, threading natively allows parallelism without monkey patching.

```bash
# Build and start app served by uWSGI + threads
$ docker-compose -f async-gthread-uwsgi.yml build
$ docker-compose -f async-gthread-uwsgi.yml up

$ ab -r -n 2000 -c 200 http://127.0.0.1:3000/?delay=1
> Time taken for tests:   13.164 seconds
> Complete requests:      2000
> Failed requests:        0
> Requests per second:    151.93 [#/sec] (mean)
```

## Deploy Flask application using Gunicorn \+ gthread

This setup runs Gunicorn with the `gthread` worker class directly against `app:app`.

```bash
# Build and start app served by Gunicorn + gthread
$ docker-compose -f async-gthread-gunicorn.yml build
$ docker-compose -f async-gthread-gunicorn.yml up

$ ab -r -n 2000 -c 200 http://127.0.0.1:3000/?delay=1
> Concurrency Level:      200
> Time taken for tests:   17.839 seconds
> Complete requests:      2000
> Failed requests:        0
> Requests per second:    112.11 [#/sec] (mean)
```

## Use Nginx reverse proxy in front of application server

See `nginx-gunicorn.yml` and `nginx-uwsgi.yml`:

```bash
$ docker-compose -f nginx-gunicorn.yml build
$ docker-compose -f nginx-gunicorn.yml up

# or

$ docker-compose -f nginx-uwsgi.yml build
$ docker-compose -f nginx-uwsgi.yml up

# and then:

$ ab -r -n 2000 -c 200 http://127.0.0.1:8080/?delay=1
> ...
```

## Bonus: psycopg2 natively unblocked with gthread on Python 3.14t

Under old gevent setups, 3rd party modules like psycopg2 blocked the event loop because they did not use standard library IO.

With Python 3.14t and `gthread`, the OS schedules real parallel threads, so psycopg2 no longer blocks the entire application process when querying the database. No `psycogreen` or `monkey.patch_all()` is needed.

```python
# psycopg2/app.py

import os

import psycopg2
import requests
from flask import Flask, request

api_port = os.environ['PORT_API']
api_url = f'http://slow_api:{api_port}/'

app = Flask(__name__)

@app.route('/')
def index():
    conn = psycopg2.connect(user="example", password="example", host="postgres")
    delay = float(request.args.get('delay') or 1)
    resp = requests.get(f'{api_url}?delay={delay/2}')

    cur = conn.cursor()
    cur.execute("SELECT NOW(), pg_sleep(%s)", (delay/2,))
    
    return 'Hi there! {} {}'.format(resp.text, cur.fetchall()[0])
```

```bash
$ docker-compose -f bonus-psycopg2-gthread.yml build
$ docker-compose -f bonus-psycopg2-gthread.yml up

$ ab -r -n 10 -c 5 http://127.0.0.1:3000/?delay=1
> Concurrency Level:      5
> Time taken for tests:   3.148 seconds
> Complete requests:      10
> Failed requests:        0
> Requests per second:    3.18 [#/sec] (mean)
```
