---
layout: post
title:  flask的路由注册
categories: [web]
tags: [flask]
---

利用装饰器来注册

```python
@app.route('/')
def hello():
        return 'hello flask'

```

利用flask自带的add_url_rule注册

```python
def hello():
    return 'hello flask'

app.add_url_rule('/', view_func=hello)

```

两个方法的实质都是通过调用add_url_rule方法来实现

```python
    def route(self, rule: str, **options: t.Any) -> t.Callable:
        def decorator(f: t.Callable) -> t.Callable:
            endpoint = options.pop("endpoint", None)
            self.add_url_rule(rule, endpoint, f, **options)
            return f

        return decorator

```

查看add_url_rule方法的实现

```python
    # Flask类中
    @setupmethod
    def add_url_rule(
        self,
        rule: str,
        endpoint: t.Optional[str] = None,
        view_func: t.Optional[t.Callable] = None,
        provide_automatic_options: t.Optional[bool] = None,
        **options: t.Any,
    ) -> None:
        # 获取endpoint，如果为空则取函数名
        if endpoint is None:
            endpoint = _endpoint_from_view_func(view_func)  # type: ignore
        # 设置endpoint
        options["endpoint"] = endpoint
        
        # method为HTTP动作的元组或列表，如['GET', 'POST']
        methods = options.pop("methods", None)
        # 如果为空，则寻找这个view_func的methods属性
        # 否则默认是('GET')，即默认只处理GET动作
        if methods is None:
            methods = getattr(view_func, "methods", None) or ("GET",)
        # method不能是字符串
        if isinstance(methods, str):
            raise TypeError(
                "Allowed methods must be a list of strings, for"
                ' example: @app.route(..., methods=["POST"])'
            )
        # 将methods的所有元素转为大写，即能够在methods参数中使用小写，如('get', 'post')，因为这里有转换
        methods = {item.upper() for item in methods}
        
        
        # Methods that should always be added
        # 必须要添加的HTTP动作
        required_methods = set(getattr(view_func, "required_methods", ()))

        # starting with Flask 0.8 the view_func object can disable and
        # force-enable the automatic options handling.
        # 是否自动添加options动作
        if provide_automatic_options is None:
            provide_automatic_options = getattr(
                view_func, "provide_automatic_options", None
            )

        if provide_automatic_options is None:
            if "OPTIONS" not in methods:
                provide_automatic_options = True
                required_methods.add("OPTIONS")
            else:
                provide_automatic_options = False
          
        # Add the required methods now.
        # 将两个集合合并
        methods |= required_methods
        
        # 创建规则
        rule = self.url_rule_class(rule, methods=methods, **options)
        rule.provide_automatic_options = provide_automatic_options  # type: ignore
        
        # 将规则添加到url_map中
        self.url_map.add(rule)
        if view_func is not None:
            # 不同视图必须有不同的endpoint，即endpoint唯一，是不同视图的标识符
            old_func = self.view_functions.get(endpoint)
            if old_func is not None and old_func != view_func:
                raise AssertionError(
                    "View function mapping is overwriting an existing"
                    f" endpoint function: {endpoint}"
                )
            # 将视图存入view_functions
            self.view_functions[endpoint] = view_func

```

self.view_functions[endpoint] = view_func这一步是以endpoint作为字典的值将视图函数绑定给路由。
可以看出，flask 内部是先把URL地址映射到端点(endpoint)上，然后再映射到视图函数(view_func)，endpoint没有, 就调用_endpoint_from_view_func方法返回view_func的名称

endpoint通常用作反向查询URL地址（viewfunction–>endpoint–>URL）
在flask中有个视图，想把它关联到另一个视图上（或从站点的一处连接到另一处），可以直接使用URL_for()

