---
title: web.py异常统一处理
date: 2017-09-15 19:02:53
tags: python
---

今天在改造系统的异常统一处理过程中遇到一个坑，在web.py的add_processor中捕获不到异常对象，报 “exceptions must be old-style classes or derived from baseexception, not str” 错误，原因是在 application.py 中， handle_with_processors 方法抛出来的所有异常都被转成 web.py _InternalError。

- web.py 的 handle_with_processors

  ```python
  def handle_with_processors(self):
          def process(processors):
              try:
                  if processors:
                      p, processors = processors[0], processors[1:]
                      return p(lambda: process(processors))
                  else:
                      return self.handle()
              except web.HTTPError:
                  raise
              except (KeyboardInterrupt, SystemExit):
                  raise
              except:
                  print >> web.debug, traceback.format_exc()
                  raise self.internalerror()
  ```


- self.internalerror

  ```python
      def internalerror(self):
          """Returns HTTPError with '500 internal error' message"""
          parent = self.get_parent_app()
          if parent:
              return parent.internalerror()
          elif web.config.get('debug'):
              import debugerror
              return debugerror.debugerror()
          else:
              return web._InternalError()
  ```

  返回的是 web._InternalError，再看看这个 _InternalError 是什么？

  ```python
  class _InternalError(HTTPError):
      """500 Internal Server Error`."""
      message = "internal server error"

      def __init__(self, message=None):
          status = '500 Internal Server Error'
          headers = {'Content-Type': 'text/html'}
          HTTPError.__init__(self, status, headers, message or self.message)
  ```

所以需要自己重新写一个 class 继承 web.application，重写一下 handle_with_processors 方法。

```python

class CKRApplication(web.application):
    def handle_with_processors(self):
        def process(processors):
            try:
                if processors:
                    p, processors = processors[0], processors[1:]
                    return p(lambda: process(processors))
                else:
                    return self.handle()
            except (web.HTTPError, KeyboardInterrupt, SystemExit):
                raise
            except ParamError as ex:
                web.ctx.status = "200 OK"
                logger.exception(ex.__str__())
                return response.error(response.status_code.PARAM_ERROR, ex.__str__())
            except Exception as ex:
                logger.exception(ex.__str__())
                web.ctx.status = '200 OK'
                return response.error(response.status_code.UNKNOWN_ERROR, u'未知异常')
        return process(self.processors)
```



另外还有一个问题，在重新抛出另一个异常的时候，捕获的上一个异常的 traceback 信息丢失了。这样的话非常不利于查找问题。可以通过 `sys.exc_info()` 获取当前捕获的异常信息。

```python
raise ParamNotFoundError("test"), None, sys.exc_info()[2]
```

这是 raise 的高级用法

```python
raise exception, value, traceback
```

- `exception`: 异常类实例/异常类
- `value`: 初始化异常类的参数值/异常类实例（使用这个实例作为 raise 的异常实例）/元组/None
- `traceback`: traceback 对象/None

#### 参考资料

- [https://groups.google.com/forum/#!topic/webpy/k2-pLAJf5r4](https://groups.google.com/forum/#!topic/webpy/k2-pLAJf5r4)
- [https://mozillazg.github.io/2016/08/python-the-right-way-to-catch-exception-then-reraise-another-exception.html](https://mozillazg.github.io/2016/08/python-the-right-way-to-catch-exception-then-reraise-another-exception.html)