# Benchmark Results: sync vs gthread on Python 3.14t

These benchmarks were run with concurrency `5` and `50` requests per endpoint (`-n 50 -c 5`) on the local Docker setup to compare traditional sync workers against the new Python 3.14t `gthread` true parallel implementation.

## Gunicorn sync
- **Time taken for tests:** 0.026 seconds
- **Requests per second:** 1948.56 [#/sec] (mean)

## uWSGI sync
- **Time taken for tests:** 0.022 seconds
- **Requests per second:** 2309.26 [#/sec] (mean)

## Gunicorn gthread (Python 3.14t)
- **Time taken for tests:** 0.015 seconds
- **Requests per second:** 3251.19 [#/sec] (mean)

## uWSGI threads (Python 3.14t)
- **Time taken for tests:** 0.017 seconds
- **Requests per second:** 2932.55 [#/sec] (mean)
