.. _first:


使用 Python 和 Flask 设计 RESTful API
=============================================

近些年来 REST (REpresentational State Transfer) 已经变成了 web services 和 web APIs 的标配。

在本文中我将向你展示如何简单地使用 Python 和 Flask 框架来创建一个 RESTful 的 web service。

什么是 REST？
--------------

六条设计规范定义了一个 REST 系统的特点:

* **客户端-服务器**: 客户端和服务器之间隔离，服务器提供服务，客户端进行消费。
* **无状态**: 从客户端到服务器的每个请求都必须包含理解请求所必需的信息。换句话说， 服务器不会存储客户端上一次请求的信息用来给下一次使用。
* **可缓存**: 服务器必须明示客户端请求能否缓存。
* **分层系统**: 客户端和服务器之间的通信应该以一种标准的方式，就是中间层代替服务器做出响应的时候，客户端不需要做任何变动。
* **统一的接口**: 服务器和客户端的通信方法必须是统一的。
* **按需编码**: 服务器可以提供可执行代码或脚本，为客户端在它们的环境中执行。这个约束是唯一一个是可选的。


什么是一个 RESTful 的 web service？
------------------------------------

REST 架构的最初目的是适应万维网的 HTTP 协议。

RESTful web services 概念的核心就是“资源”。 资源可以用 `URI <https://en.wikipedia.org/wiki/Uniform_resource_identifier>`_ 来表示。客户端使用 HTTP 协议定义的方法来发送请求到这些 URIs，当然可能会导致这些被访问的”资源“状态的改变。

HTTP 标准的方法有如下::

  ==========  ==============  ==================================
  HTTP 方法   行为            示例
  ==========  ==============  ==================================
  GET         获取资源的信息  http://example.com/api/orders
  GET         获取资源的信息  http://example.com/api/orders/123
  POST        创建新资源      http://example.com/api/orders
  PUT         更新资源        http://example.com/api/orders/123
  DELETE      删除资源        http://example.com/api/orders/123
  ==========  ==============  ==================================

REST 设计不需要特定的数据格式。在请求中数据可以以 `JSON <http://en.wikipedia.org/wiki/JSON>`_ 形式, 或者有时候作为 url 中查询参数项。


设计一个简单的 web service
----------------------------

坚持 REST 的准则设计一个 web service 或者 API 的任务就变成一个标识资源被展示出来以及它们是怎样受不同的请求方法影响的练习。

比如说，我们要编写一个待办事项应用程序而且我们想要为它设计一个 web service。要做的第一件事情就是决定用什么样的根 URL 来访问该服务。例如，我们可以通过这个来访问::

http://[hostname]/todo/api/v1.0/

在这里我已经决定在 URL 中包含应用的名称以及 API 的版本号。在 URL 中包含应用名称有助于提供一个命名空间以便区分同一系统上的其它服务。在 URL 中包含版本号能够帮助以后的更新，如果新版本中存在新的和潜在不兼容的功能，可以不影响依赖于较旧的功能的应用程序。

下一步骤就是选择将由该服务暴露(展示)的资源。这是一个十分简单地应用，我们只有任务，因此在我们待办事项中唯一的资源就是任务。

我们的任务资源将要使用 HTTP 方法如下::

  ==========  ===============================================  =============================
  HTTP 方法   URL                                              动作
  ==========  ===============================================  ==============================
  GET         http://[hostname]/todo/api/v1.0/tasks            检索任务列表
  GET         http://[hostname]/todo/api/v1.0/tasks/[task_id]  检索某个任务
  POST        http://[hostname]/todo/api/v1.0/tasks            创建新任务
  PUT         http://[hostname]/todo/api/v1.0/tasks/[task_id]  更新任务
  DELETE      http://[hostname]/todo/api/v1.0/tasks/[task_id]  删除任务
  ==========  ================================================ =============================

我们定义的任务有如下一些属性:

* **id**: 任务的唯一标识符。数字类型。
* **title**: 简短的任务描述。字符串类型。
* **description**: 具体的任务描述。文本类型。
* **done**: 任务完成的状态。布尔值。

目前为止关于我们的 web service 的设计基本完成。剩下的事情就是实现它！

Flask 框架的简介
----------------------------

如果你读过 `Flask Mega-Tutorial 系列 <http://www.pythondoc.com/flask-mega-tutorial/index.html>`_，就会知道 Flask 是一个简单却十分强大的 Python web 框架。

在我们深入研究 web services 的细节之前，让我们回顾一下一个普通的 Flask Web 应用程序的结构。

我会首先假设你知道 Python 在你的平台上工作的基本知识。 我讲讲解的例子是工作在一个类 Unix 操作系统。简而言之，这意味着它们能工作在 Linux，Mac OS X 和 Windows(如果你使用Cygwin)。
如果你使用 Windows 上原生的 Python 版本的话，命令会有所不同。 

让我们开始在一个虚拟环境上安装 Flask。如果你的系统上没有 virtualenv，你可以从 https://pypi.python.org/pypi/virtualenv 上下载::

  $ mkdir todo-api
  $ cd todo-api
  $ virtualenv flask
  New python executable in flask/bin/python
  Installing setuptools............................done.
  Installing pip...................done.
  $ flask/bin/pip install flask

既然已经安装了 Flask，现在开始创建一个简单地网页应用，我们把它放在一个叫 app.py 的文件中::

  #!flask/bin/python
  from flask import Flask

  app = Flask(__name__)

  @app.route('/')
  def index():
      return "Hello, World!"

  if __name__ == '__main__':
      app.run(debug=True)

为了运行这个程序我们必须执行 app.py::

  $ chmod a+x app.py
  $ ./app.py
   * Running on http://127.0.0.1:5000/
   * Restarting with reloader

现在你可以启动你的网页浏览器，输入 http://localhost:5000 看看这个小应用程序的效果。

简单吧？现在我们将这个应用程序转换成我们的 RESTful service！


使用 Python 和 Flask 实现 RESTful services 
-------------------------------------------

使用 Flask 构建 web services 是十分简单地，比我在 `Mega-Tutorial <http://www.pythondoc.com/flask-mega-tutorial/index.html>`_ 中构建的完整的服务端的应用程序要简单地多。

在 Flask 中有许多扩展来帮助我们构建 RESTful services，但是在我看来这个任务十分简单，没有必要使用 Flask 扩展。

我们 web service 的客户端需要添加、删除以及修改任务的服务，因此显然我们需要一种方式来存储任务。最直接的方式就是建立一个小型的数据库，但是数据库并不是本文的主体。学习在 Flask 中使用合适的数据库，我强烈建议阅读 `Mega-Tutorial <http://www.pythondoc.com/flask-mega-tutorial/index.html>`_。

这里我们直接把任务列表存储在内存中，因此这些任务列表只会在 web 服务器运行中工作，在结束的时候就失效。 这种方式只是适用我们自己开发的 web 服务器，不适用于生产环境的 web 服务器， 这种情况一个合适的数据库的搭建是必须的。

我们现在来实现 web service 的第一个入口::

  #!flask/bin/python
  from flask import Flask, jsonify

  app = Flask(__name__)

  tasks = [
      {
          'id': 1,
          'title': u'Buy groceries',
          'description': u'Milk, Cheese, Pizza, Fruit, Tylenol', 
          'done': False
      },
      {
          'id': 2,
          'title': u'Learn Python',
          'description': u'Need to find a good Python tutorial on the web', 
          'done': False
      }
  ]

  @app.route('/todo/api/v1.0/tasks', methods=['GET'])
  def get_tasks():
      return jsonify({'tasks': tasks})

  if __name__ == '__main__':
      app.run(debug=True)

正如你所见，没有多大的变化。我们创建一个任务的内存数据库，这里无非就是一个字典和数组。数组中的每一个元素都具有上述定义的任务的属性。

取代了首页，我们现在拥有一个 get_tasks 的函数，访问的 URI 为 /todo/api/v1.0/tasks，并且只允许 GET 的 HTTP 方法。

这个函数的响应不是文本，我们使用 JSON 数据格式来响应，Flask 的 jsonify 函数从我们的数据结构中生成。

使用网页浏览器来测试我们的 web service 不是一个最好的注意，因为网页浏览器上不能轻易地模拟所有的 HTTP 请求的方法。相反，我们会使用 curl。如果你还没有安装 curl 的话，请立即安装它。

通过执行 app.py，启动 web service。接着打开一个新的控制台窗口，运行以下命令::

  $ curl -i http://localhost:5000/todo/api/v1.0/tasks
  HTTP/1.0 200 OK
  Content-Type: application/json
  Content-Length: 294
  Server: Werkzeug/0.8.3 Python/2.7.3
  Date: Mon, 20 May 2013 04:53:53 GMT

  {
    "tasks": [
      {
        "description": "Milk, Cheese, Pizza, Fruit, Tylenol",
        "done": false,
        "id": 1,
        "title": "Buy groceries"
      },
      {
        "description": "Need to find a good Python tutorial on the web",
        "done": false,
        "id": 2,
        "title": "Learn Python"
      }
    ]
  }

我们已经成功地调用我们的 RESTful service 的一个函数！

现在我们开始编写 GET 方法请求我们的任务资源的第二个版本。这是一个用来返回单独一个任务的函数::

  from flask import abort

  @app.route('/todo/api/v1.0/tasks/<int:task_id>', methods=['GET'])
  def get_task(task_id):
      task = filter(lambda t: t['id'] == task_id, tasks)
      if len(task) == 0:
          abort(404)
      return jsonify({'task': task[0]})

第二个函数有些意思。这里我们得到了 URL 中任务的 id，接着 Flask 把它转换成 函数中的 task_id 的参数。

我们用这个参数来搜索我们的任务数组。如果我们的数据库中不存在搜索的 id，我们将会返回一个类似 404 的错误，根据 HTTP 规范的意思是 “资源未找到”。

如果我们找到相应的任务，那么我们只需将它用 jsonify 打包成 JSON 格式并将其发送作为响应，就像我们以前那样处理整个任务集合。

调用 curl 请求的结果如下::

  $ curl -i http://localhost:5000/todo/api/v1.0/tasks/2
  HTTP/1.0 200 OK
  Content-Type: application/json
  Content-Length: 151
  Server: Werkzeug/0.8.3 Python/2.7.3
  Date: Mon, 20 May 2013 05:21:50 GMT

  {
    "task": {
      "description": "Need to find a good Python tutorial on the web",
      "done": false,
      "id": 2,
      "title": "Learn Python"
    }
  }
  $ curl -i http://localhost:5000/todo/api/v1.0/tasks/3
  HTTP/1.0 404 NOT FOUND
  Content-Type: text/html
  Content-Length: 238
  Server: Werkzeug/0.8.3 Python/2.7.3
  Date: Mon, 20 May 2013 05:21:52 GMT

  <!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
  <title>404 Not Found</title>
  <h1>Not Found</h1>
  <p>The requested URL was not found on the server.</p><p>If you     entered the URL manually please check your spelling and try again.</p>


当我们请求 id #2 的资源时候，我们获取到了，但是当我们请求 #3 的时候返回了 404 错误。有关错误奇怪的是返回的是 HTML 信息而不是 JSON，这是因为 Flask 按照默认方式生成 404 响应。由于这是一个 Web service 客户端希望我们总是 以 JSON 格式回应，所以我们需要改善我们的 404 错误处理程序::

  from flask import make_response

  @app.errorhandler(404)
  def not_found(error):
      return make_response(jsonify({'error': 'Not found'}), 404)
  And we get a much more API friendly error response:

  $ curl -i http://localhost:5000/todo/api/v1.0/tasks/3
  HTTP/1.0 404 NOT FOUND
  Content-Type: application/json
  Content-Length: 26
  Server: Werkzeug/0.8.3 Python/2.7.3
  Date: Mon, 20 May 2013 05:36:54 GMT

  {
    "error": "Not found"
  }

接下来就是 POST 方法，我们用来在我们的任务数据库中插入一个新的任务::

  from flask import request

  @app.route('/todo/api/v1.0/tasks', methods=['POST'])
  def create_task():
      if not request.json or not 'title' in request.json:
          abort(400)
      task = {
          'id': tasks[-1]['id'] + 1,
          'title': request.json['title'],
          'description': request.json.get('description', ""),
          'done': False
      }
      tasks.append(task)
      return jsonify({'task': task}), 201

添加一个新的任务也是相当容易地。只有当请求以 JSON 格式形式，request.json 才会有请求的数据。如果没有数据，或者存在数据但是缺少 title 项，我们将会返回 400，这是表示请求无效。

接着我们会创建一个新的任务字典，使用最后一个任务的 id + 1 作为该任务的 id。我们允许 description 字段缺失，并且假设 done 字段设置成 False。

我们把新的任务添加到我们的任务数组中，并且把新添加的任务和状态 201 响应给客户端。

使用如下的 curl 命令来测试这个新的函数::

  $ curl -i -H "Content-Type: application/json" -X POST -d '{"title":"Read a book"}' http://localhost:5000/todo/api/v1.0/tasks
  HTTP/1.0 201 Created
  Content-Type: application/json
  Content-Length: 104
  Server: Werkzeug/0.8.3 Python/2.7.3
  Date: Mon, 20 May 2013 05:56:21 GMT

  {
    "task": {
      "description": "",
      "done": false,
      "id": 3,
      "title": "Read a book"
    }
  }

注意：如果你在 Windows 上并且运行 Cygwin 版本的 curl，上面的命令不会有任何问题。然而，如果你使用原生的 curl，命令会有些不同::

  curl -i -H "Content-Type: application/json" -X POST -d "{"""title""":"""Read a book"""}" http://localhost:5000/todo/api/v1.0/tasks

当然在完成这个请求后，我们可以得到任务的更新列表::

  $ curl -i http://localhost:5000/todo/api/v1.0/tasks
  HTTP/1.0 200 OK
  Content-Type: application/json
  Content-Length: 423
  Server: Werkzeug/0.8.3 Python/2.7.3
  Date: Mon, 20 May 2013 05:57:44 GMT

  {
    "tasks": [
      {
        "description": "Milk, Cheese, Pizza, Fruit, Tylenol",
        "done": false,
        "id": 1,
        "title": "Buy groceries"
      },
      {
        "description": "Need to find a good Python tutorial on the web",
        "done": false,
        "id": 2,
        "title": "Learn Python"
      },
      {
        "description": "",
        "done": false,
        "id": 3,
        "title": "Read a book"
      }
    ]
  }

剩下的两个函数如下所示::

  @app.route('/todo/api/v1.0/tasks/<int:task_id>', methods=['PUT'])
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
      return jsonify({'task': task[0]})

  @app.route('/todo/api/v1.0/tasks/<int:task_id>', methods=['DELETE'])
  def delete_task(task_id):
      task = filter(lambda t: t['id'] == task_id, tasks)
      if len(task) == 0:
          abort(404)
      tasks.remove(task[0])
      return jsonify({'result': True})

delete_task 函数没有什么特别的。对于 update_task 函数，我们需要严格地检查输入的参数以防止可能的问题。我们需要确保在我们把它更新到数据库之前，任何客户端提供我们的是预期的格式
The delete_task function should have no surprises. For the update_task function we are trying to prevent bugs by doing exhaustive checking of the input arguments. We need to make sure that anything that the client provided us is in the expected format before we incorporate it into our database.

A function call that updates task #2 as being done would be done as follows:

$ curl -i -H "Content-Type: application/json" -X PUT -d '{"done":true}' http://localhost:5000/todo/api/v1.0/tasks/2
HTTP/1.0 200 OK
Content-Type: application/json
Content-Length: 170
Server: Werkzeug/0.8.3 Python/2.7.3
Date: Mon, 20 May 2013 07:10:16 GMT

{
  "task": [
    {
      "description": "Need to find a good Python tutorial on the web",
      "done": true,
      "id": 2,
      "title": "Learn Python"
    }
  ]
}
Improving the web service interface
The problem with the current design of the API is that clients are forced to construct URIs from the task identifiers that are returned. This is pretty easy in itself, but it indirectly forces clients to know how these URIs need to be built, and this will prevent us from making changes to URIs in the future.

Instead of returning task ids we can return the full URI that controls the task, so that clients get the URIs ready to be used. For this we can write a small helper function that generates a "public" version of a task to send to the client:

from flask import url_for

def make_public_task(task):
    new_task = {}
    for field in task:
        if field == 'id':
            new_task['uri'] = url_for('get_task', task_id=task['id'], _external=True)
        else:
            new_task[field] = task[field]
    return new_task
All we are doing here is taking a task from our database and creating a new task that has all the fields except id, which gets replaced with another field called uri, generated with Flask's url_for.

When we return the list of tasks we pass them through this function before sending them to the client:

@app.route('/todo/api/v1.0/tasks', methods=['GET'])
def get_tasks():
    return jsonify({'tasks': map(make_public_task, tasks)})
So now this is what the client gets when it retrieves the list of tasks:

$ curl -i http://localhost:5000/todo/api/v1.0/tasks
HTTP/1.0 200 OK
Content-Type: application/json
Content-Length: 406
Server: Werkzeug/0.8.3 Python/2.7.3
Date: Mon, 20 May 2013 18:16:28 GMT

{
  "tasks": [
    {
      "title": "Buy groceries",
      "done": false,
      "description": "Milk, Cheese, Pizza, Fruit, Tylenol",
      "uri": "http://localhost:5000/todo/api/v1.0/tasks/1"
    },
    {
      "title": "Learn Python",
      "done": false,
      "description": "Need to find a good Python tutorial on the web",
      "uri": "http://localhost:5000/todo/api/v1.0/tasks/2"
    }
  ]
}
We apply this technique to all the other functions and with this we ensure that the client always sees URIs instead of ids.

Securing a RESTful web service
Can you believe we are done? Well, we are done with the functionality of our service, but we still have a problem. Our service is open to anybody, and that is a bad thing.

We have a complete web service that can manage our to do list, but the service in its current state is open to any clients. If a stranger figures out how our API works he or she can write a new client that can access our service and mess with our data.

Most entry level tutorials ignore security and stop here. In my opinion this is a serious problem that should always be addressed.

The easiest way to secure our web service is to require clients to provide a username and a password. In a regular web application you would have a login form that posts the credentials, and at that point the server would create a session for the logged in user to continue working, with the session id stored in a cookie in the client browser. Unfortunately doing that here would violate the stateless requirement of REST, so instead we have to ask clients to send their authentication information with every request they send to us.

With REST we always try to adhere to the HTTP protocol as much as we can. Now that we need to implement authentication we should do so in the context of HTTP, which provides two forms of authentication called Basic and Digest.

There is a small Flask extension that can help with this, written by no other than yours truly. So let's go ahead and install Flask-HTTPAuth:

$ flask/bin/pip install flask-httpauth
Let's say we want our web service to only be accessible to username miguel and password python. We can setup a Basic HTTP authentication as follows:

from flask.ext.httpauth import HTTPBasicAuth
auth = HTTPBasicAuth()

@auth.get_password
def get_password(username):
    if username == 'miguel':
        return 'python'
    return None

@auth.error_handler
def unauthorized():
    return make_response(jsonify({'error': 'Unauthorized access'}), 401)
The get_password function is a callback function that the extension will use to obtain the password for a given user. In a more complex system this function could check a user database, but in this case we just have a single user so there is no need for that.

The error_handler callback will be used by the extension when it needs to send the unauthorized error code back to the client. Like we did with other error codes, here we customize the response so that is contains JSON instead of HTML.

With the authentication system setup, all that is left is to indicate which functions need to be protected, by adding the @auth.login_required decorator. For example:

@app.route('/todo/api/v1.0/tasks', methods=['GET'])
@auth.login_required
def get_tasks():
    return jsonify({'tasks': tasks})
If we now try to invoke this function with curl this is what we get:

$ curl -i http://localhost:5000/todo/api/v1.0/tasks
HTTP/1.0 401 UNAUTHORIZED
Content-Type: application/json
Content-Length: 36
WWW-Authenticate: Basic realm="Authentication Required"
Server: Werkzeug/0.8.3 Python/2.7.3
Date: Mon, 20 May 2013 06:41:14 GMT

{
  "error": "Unauthorized access"
}
To be able to invoke this function we have to send our credentials:

$ curl -u miguel:python -i http://localhost:5000/todo/api/v1.0/tasks
HTTP/1.0 200 OK
Content-Type: application/json
Content-Length: 316
Server: Werkzeug/0.8.3 Python/2.7.3
Date: Mon, 20 May 2013 06:46:45 GMT

{
  "tasks": [
    {
      "title": "Buy groceries",
      "done": false,
      "description": "Milk, Cheese, Pizza, Fruit, Tylenol",
      "uri": "http://localhost:5000/todo/api/v1.0/tasks/1"
    },
    {
      "title": "Learn Python",
      "done": false,
      "description": "Need to find a good Python tutorial on the web",
      "uri": "http://localhost:5000/todo/api/v1.0/tasks/2"
    }
  ]
}
The authentication extension gives us the freedom to choose which functions in the service are open and which are protected.

To ensure the login information is secure the web service should be exposed in a HTTP Secure server (i.e. https://...) as this encrypts all the communications between client and server and prevents a third party from seeing the authentication credentials in transit.

Unfortunately web browsers have the nasty habit of showing an ugly login dialog box when a request comes back with a 401 error code. This happens even for background requests, so if we were to implement a web browser client with our current web server we would need to jump through hoops to prevent browsers from showing their authentication dialogs and let our client application handle the login.

A simple trick to distract web browsers is to return an error code other than 401. An alternative error code favored by many is 403, which is the "Forbidden" error. While this is a close enough error, it sort of violates the HTTP standard, so it is not the proper thing to do if full compliance is necessary. In particular this would be a bad idea if the client application is not a web browser. But for cases where server and client are developed together it saves a lot of trouble. The simple change that we can make to implement this trick is to replace the 401 with a 403:

@auth.error_handler
def unauthorized():
    return make_response(jsonify({'error': 'Unauthorized access'}), 403)
Of course if we do this we will need the client application to look for 403 errors as well.

Possible improvements
There are a number of ways in which this little web service we have built today can be improved.

For starters, a real web service should be backed by a real database. The memory data structure that we are using is very limited in functionality and should not be used for a real application.

Another area in which an improvement could be made is in handling multiple users. If the system supports multiple users the authentication credentials sent by the client could be used to obtain user specific to do lists. In such a system we would have a second resource, which would be the users. A POST request on the users resource would represent a new user registering for the service. A GET request would return user information back to the client. A PUT request would update the user information, maybe updating an email address. A DELETE request would delete the user account.

The GET request that retrieves the task list could be expanded in a couple of ways. First, this request could take optional pagination arguments, so that a client can request a portion of the list. Another way to make this function more useful would be to allow filtering by certain criteria. For example, a client might want to see only completed tasks, or only tasks with a title that begins with the letter A. All these elements can be added to the URL as arguments.