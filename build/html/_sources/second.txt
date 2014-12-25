.. _second:


使用 Flask-RESTful 设计 RESTful API
==========================================

This is the third article in which I explore different aspects of writing RESTful APIs using the Flask microframework.

The example RESTful server I wrote before used only Flask as a dependency. Today I will show you how to write the same server using Flask-RESTful, a Flask extension that simplifies the creation of APIs.

The RESTful server
As a reminder, here is the definition of the ToDo List web service that has been serving as an example in my RESTful articles:

HTTP Method	URI	Action
GET	http://[hostname]/todo/api/v1.0/tasks	Retrieve list of tasks
GET	http://[hostname]/todo/api/v1.0/tasks/[task_id]	Retrieve a task
POST	http://[hostname]/todo/api/v1.0/tasks	Create a new task
PUT	http://[hostname]/todo/api/v1.0/tasks/[task_id]	Update an existing task
DELETE	http://[hostname]/todo/api/v1.0/tasks/[task_id]	Delete a task
The only resource exposed by this service is a "task", which has the following data fields:

uri: unique URI for the task. String type.
title: short task description. String type.
description: long task description. Text type.
done: task completion state. Boolean type.
Routing
In my first RESTful server example (source code here) I have used regular Flask view functions to define all the routes.

Flask-RESTful provides a Resource base class that can define the routing for one or more HTTP methods for a given URL. For example, to define a User resource with GET, PUT and DELETE methods you would write:

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
The add_resource function registers the routes with the framework using the given endpoint. If an endpoint isn't given then Flask-RESTful generates one for you from the class name, but since sometimes the endpoint is needed for functions such as url_for I prefer to make it explicit.

My ToDo API defines two URLs: /todo/api/v1.0/tasks for the list of tasks, and /todo/api/v1.0/tasks/<int:id> for an individual task. Since Flask-RESTful's Resource class can wrap a single URL this server will need two resources:

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
Note that while the method views of TaskListAPI receive no arguments the ones in TaskAPI all receive the id, as specified in the URL under which the resource is registered.

Request Parsing and Validation
When I implemented this server in the previous article I did my own validation of the request data. For example, look at how long the PUT handler is in that version:

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
Here I have to make sure the data given with the request is valid before using it, and that makes the function pretty long.

Flask-RESTful provides a much better way to handle this with the RequestParser class. This class works in a similar way as argparse for command line arguments.

First, for each resource I define the arguments and how to validate them:

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
In the TaskListAPI resource the POST method is the only one the receives arguments. The title argument is required here, so I included an error message that Flask-RESTful will send as a response to the client when the field is missing. The description field is optional, and when it is missing a default value of an empty string will be used. One interesting aspect of the RequestParser class is that by default it looks for fields in request.values, so the location optional argument must be set to indicate that the fields are coming in request.json.

The request parser for the TaskAPI is constructed in a similar way, but has a few differences. In this case it is the PUT method that will need to parse arguments, and for this method all the arguments are optional, including the done field that was not part of the request in the other resource.

Now that the request parsers are initialized, parsing and validating a request is pretty easy. For example, note how much simpler the TaskAPI.put() method becomes:

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
A side benefit of letting Flask-RESTful do the validation is that now there is no need to have a handler for the bad request code 400 error, this is all taken care of by the extension.

Generating Responses
My original REST server generates the responses using Flask's jsonify helper function. Flask-RESTful automatically handles the conversion to JSON, so instead of this:

        return jsonify( { 'task': make_public_task(task) } )
I can do this:

        return { 'task': make_public_task(task) }
Flask-RESTful also supports passing a custom status code back when necessary:

        return { 'task': make_public_task(task) }, 201
But there is more. The make_public_task wrapper from the original server converted a task from its internal representation to the external representation that clients expected. The conversion included removing the id field and adding a uri field in its place. Flask-RESTful provides a helper function to do this in a much more elegant way that not only generates the uri but also does type conversion on the remaining fields:

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
The task_fields structure serves as a template for the marshal function. The fields.Uri type is a special type that generates a URL. The argument it takes is the endpoint (recall that I have used explicit endpoints when I registered the resources specifically so that I can refer to them when needed).

Authentication
The routes in the REST server are all protected with HTTP basic authentication. In the original server the protection was added using the decorator provided by the Flask-HTTPAuth extension.

Since the Resouce class inherits from Flask's MethodView, it is possible to attach decorators to the methods by defining a decorators class variable:

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
    