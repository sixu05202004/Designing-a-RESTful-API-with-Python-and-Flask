.. _second:


使用 Flask-RESTful 设计 RESTful API
==========================================

前面我已经用 Flask 实现了一个 RESTful 服务器。今天我们将会使用 Flask-RESTful 来实现同一个 RESTful 服务器，Flask-RESTful 是一个可以简化 APIs 的构建的 Flask 扩展。


RESTful 服务器
-----------------

作为一个提醒， 这里就是待完成事项列表 web service 所提供的方法的定义::

  ==========  ===============================================  =============================
  HTTP 方法   URL                                              动作
  ==========  ===============================================  ==============================
  GET         http://[hostname]/todo/api/v1.0/tasks            检索任务列表
  GET         http://[hostname]/todo/api/v1.0/tasks/[task_id]  检索某个任务
  POST        http://[hostname]/todo/api/v1.0/tasks            创建新任务
  PUT         http://[hostname]/todo/api/v1.0/tasks/[task_id]  更新任务
  DELETE      http://[hostname]/todo/api/v1.0/tasks/[task_id]  删除任务
  ==========  ================================================ =============================

这个服务唯一的资源叫做“任务”，它有如下一些属性:

* **id**: 任务的唯一标识符。数字类型。
* **title**: 简短的任务描述。字符串类型。
* **description**: 具体的任务描述。文本类型。
* **done**: 任务完成的状态。布尔值。


路由
-------

在上一遍文章中，我使用了 Flask 的视图函数来定义所有的路由。

Flask-RESTful 提供了一个 Resource 基础类，它能够定义一个给定 URL 的一个或者多个 HTTP 方法。例如，定义一个可以使用 HTTP 的 GET, PUT 以及 DELETE 方法的 User 资源，你的代码可以如下::

    from flask import Flask
    from flask.ext.restful import Api, Resource

    app = Flask(__name__)
    api = Api(app)

    class UserAPI(Resource):
        def get(self, id):
            pass

        def put(self, id):
            pass

        def delete(self, id):
            pass

    api.add_resource(UserAPI, '/users/<int:id>', endpoint = 'user')

add_resource 函数使用指定的 endpoint 注册路由到框架上。如果没有指定 endpoint，Flask-RESTful 会根据类名生成一个，但是有时候有些函数比如 url_for 需要 endpoint，因此我会明确给 endpoint 赋值。

我的待办事项 API 定义两个 URLs：/todo/api/v1.0/tasks（获取所有任务列表），以及 /todo/api/v1.0/tasks/<int:id>（获取单个任务）。我们现在需要两个资源::

    class TaskListAPI(Resource):
        def get(self):
            pass

        def post(self):
            pass

    class TaskAPI(Resource):
        def get(self, id):
            pass

        def put(self, id):
            pass

        def delete(self, id):
            pass

    api.add_resource(TaskListAPI, '/todo/api/v1.0/tasks', endpoint = 'tasks')
    api.add_resource(TaskAPI, '/todo/api/v1.0/tasks/<int:id>', endpoint = 'task')


解析以及验证请求
-----------------

当我在以前的文章中实现此服务器的时候，我自己对请求的数据进行验证。例如，在之前版本中如何处理 PUT 的::

    @app.route('/todo/api/v1.0/tasks/<int:task_id>', methods = ['PUT'])
    @auth.login_required
    def update_task(task_id):
        task = filter(lambda t: t['id'] == task_id, tasks)
        if len(task) == 0:
            abort(404)
        if not request.json:
            abort(400)
        if 'title' in request.json and type(request.json['title']) != unicode:
            abort(400)
        if 'description' in request.json and type(request.json['description']) is not unicode:
            abort(400)
        if 'done' in request.json and type(request.json['done']) is not bool:
            abort(400)
        task[0]['title'] = request.json.get('title', task[0]['title'])
        task[0]['description'] = request.json.get('description', task[0]['description'])
        task[0]['done'] = request.json.get('done', task[0]['done'])
        return jsonify( { 'task': make_public_task(task[0]) } )

在这里, 我必须确保请求中给出的数据在使用之前是有效，这样使得函数变得又臭又长。

Flask-RESTful 提供了一个更好的方式来处理数据验证，它叫做 RequestParser 类。这个类工作方式类似命令行解析工具 argparse。

首先，对于每一个资源需要定义参数以及怎样验证它们::

    from flask.ext.restful import reqparse

    class TaskListAPI(Resource):
        def __init__(self):
            self.reqparse = reqparse.RequestParser()
            self.reqparse.add_argument('title', type = str, required = True,
                help = 'No task title provided', location = 'json')
            self.reqparse.add_argument('description', type = str, default = "", location = 'json')
            super(TaskListAPI, self).__init__()

        # ...

    class TaskAPI(Resource):
        def __init__(self):
            self.reqparse = reqparse.RequestParser()
            self.reqparse.add_argument('title', type = str, location = 'json')
            self.reqparse.add_argument('description', type = str, location = 'json')
            self.reqparse.add_argument('done', type = bool, location = 'json')
            super(TaskAPI, self).__init__()

        # ...

在 TaskListAPI 资源中，POST 方法是唯一接收参数的。参数“标题”是必须的，因此我定义一个缺少“标题”的错误信息。当客户端缺少这个参数的时候，Flask-RESTful 将会把这个错误信息作为响应发送给客户端。“描述”字段是可选的，当缺少这个字段的时候，默认的空字符串将会被使用。一个有趣的方面就是 RequestParser 类默认情况下在 request.values 中查找参数，因此 location 可选参数必须被设置以表明请求过来的参数是 request.json 格式的。

TaskAPI 资源的参数处理是同样的方式，但是有少许不同。PUT 方法需要解析参数，并且这个方法的所有参数都是可选的。

当请求解析器被初始化，解析和验证一个请求是很容易的。 例如，请注意 TaskAPI.put() 方法变的多么地简单::

    def put(self, id):
        task = filter(lambda t: t['id'] == id, tasks)
        if len(task) == 0:
            abort(404)
        task = task[0]
        args = self.reqparse.parse_args()
        for k, v in args.iteritems():
            if v != None:
                task[k] = v
        return jsonify( { 'task': make_public_task(task) } )

使用 Flask-RESTful 来处理验证的另一个好处就是没有必要单独地处理类似 HTTP 400 错误，Flask-RESTful 会来处理这些。


生成响应
-----------

原来设计的 REST 服务器使用 Flask 的 jsonify 函数来生成响应。Flask-RESTful 会自动地处理转换成 JSON 数据格式，因此下面的代码需要替换:: 
    
    return jsonify( { 'task': make_public_task(task) } )

现在需要写成这样::

    return { 'task': make_public_task(task) }

Flask-RESTful 也支持自定义状态码，如果有必要的话::

    return { 'task': make_public_task(task) }, 201

Flask-RESTful 还有更多的功能。make_public_task 能够把来自原始服务器上的任务从内部形式包装成客户端想要的外部形式。最典型的就是把任务的 id 转成 uri。Flask-RESTful 就提供一个辅助函数能够很优雅地做到这样的转换，不仅仅能够把 id 转成 uri 并且能够转换其他的参数::

    from flask.ext.restful import fields, marshal

    task_fields = {
        'title': fields.String,
        'description': fields.String,
        'done': fields.Boolean,
        'uri': fields.Url('task')
    }

    class TaskAPI(Resource):
        # ...

        def put(self, id):
            # ...
            return { 'task': marshal(task, task_fields) }

task_fields 结构用于作为 marshal 函数的模板。fields.Uri 是一个用于生成一个 URL 的特定的参数。
它需要的参数是 endpoint。


认证
------

在 REST 服务器中的路由都是由 HTTP 基本身份验证保护着。在最初的那个服务器是通过使用 Flask-HTTPAuth 扩展来实现的。

因为 Resouce 类是继承自 Flask 的 MethodView，它能够通过定义 decorators 变量并且把装饰器赋予给它::

    from flask.ext.httpauth import HTTPBasicAuth
    # ...
    auth = HTTPBasicAuth()
    # ...

    class TaskAPI(Resource):
        decorators = [auth.login_required]
        # ...

    class TaskAPI(Resource):
        decorators = [auth.login_required]
        # ...
        