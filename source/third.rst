.. _third:


使用 Flask 设计 RESTful 的认证
======================================

This article is the fourth in my series on RESTful APIs. Today I will be showing you a simple, yet secure way to protect a Flask based API with password or token based authentication.

This article stands on its own, but if you feel you need to catch up here are the links to the previous articles:

Designing a RESTful API with Python and Flask
Writing a Javascript REST client
Designing a RESTful API using Flask-RESTful
Example Code
The code discussed in the following sections is available for you to try and hack. You can find it on github: REST-auth.

The User Database
To give this example some resemblance to a real life project I'm going to use a Flask-SQLAlchemy database to store users.

The user model will be very simple. For each user a username and a password_hash will be stored.

class User(db.Model):
    __tablename__ = 'users'
    id = db.Column(db.Integer, primary_key = True)
    username = db.Column(db.String(32), index = True)
    password_hash = db.Column(db.String(128))
For security reasons the original password will not be stored, after the hash is calculated during registration it will be discarded. If this user database were to fall in malicious hands it would be extremely hard for the attacker to decode the real passwords from the hashes.

Passwords should never be stored in the clear in a user database.

Password Hashing
To create the password hashes I'm going to use PassLib, a package dedicated to password hashing.

PassLib provides several hashing algorithms to choose from. The custom_app_context object is an easy to use option based on the sha256_crypt hashing algorithm.

To add password hashing and verification two new methods are added to the User model:

from passlib.apps import custom_app_context as pwd_context

class User(db.Model):
    # ...

    def hash_password(self, password):
        self.password_hash = pwd_context.encrypt(password)

    def verify_password(self, password):
        return pwd_context.verify(password, self.password_hash)
The hash_password() method takes a plain password as argument and stores a hash of it with the user. This method is called when a new user is registering with the server, or when the user changes the password.

The verify_password() method takes a plain password as argument and returns True if the password is correct or False if not. This method is called whenever the user provides credentials and they need to be validated.

You may ask how can the password be verified if the original password was thrown away and lost forever after it was hashed.

Hashing algorithms are one-way functions, meaning that they can be used to generate a hash from a password, but they cannot be used in the reverse direction. But these algorithms are deterministic, given the same inputs they will always generate the same output. All PassLib needs to do to verify a password is to hash it with the same function that was used during registration, and then compare the resulting hash against the one stored in the database.

User Registration
In this example, a client can register a new user with a POST request to /api/users. The body of the request needs to be a JSON object that has username and password fields.

The implementation of the Flask route is shown below:

@app.route('/api/users', methods = ['POST'])
def new_user():
    username = request.json.get('username')
    password = request.json.get('password')
    if username is None or password is None:
        abort(400) # missing arguments
    if User.query.filter_by(username = username).first() is not None:
        abort(400) # existing user
    user = User(username = username)
    user.hash_password(password)
    db.session.add(user)
    db.session.commit()
    return jsonify({ 'username': user.username }), 201, {'Location': url_for('get_user', id = user.id, _external = True)}
This function is extremely simple. The username and password arguments are obtained from the JSON input coming with the request and then validated.

If the arguments are valid then a new User instance is created. The username is assigned to it, and the password is hashed using the hash_password() method. The user is finally written to the database.

The body of the response shows the user representation as a JSON object, with a status code of 201 and a Location header pointing to the URI of the newly created user.

Note: the implementation of the get_user endpoint is now shown here, you can find it in the full example on github.

Here is an example user registration request sent from curl:

$ curl -i -X POST -H "Content-Type: application/json" -d '{"username":"miguel","password":"python"}' http://127.0.0.1:5000/api/users
HTTP/1.0 201 CREATED
Content-Type: application/json
Content-Length: 27
Location: http://127.0.0.1:5000/api/users/1
Server: Werkzeug/0.9.4 Python/2.7.3
Date: Thu, 28 Nov 2013 19:56:39 GMT

{
  "username": "miguel"
}
Note that in a real application this would be done over secure HTTP. There is no point in going through the effort of protecting the API if the login credentials are going to travel through the network in clear text.

Password Based Authentication
Now let's assume there is a resource exposed by this API that needs to be available only to registered users. This resource is accessed at the /api/resource endpoint.

To protect this resource I'm going to use HTTP Basic Authentication, but instead of implementing this protocol by hand I'm going to let the Flask-HTTPAuth extension do it for me.

Using Flask-HTTPAuth an endpoint is protected by adding the login_required decorator to it:

from flask.ext.httpauth import HTTPBasicAuth
auth = HTTPBasicAuth()

@app.route('/api/resource')
@auth.login_required
def get_resource():
    return jsonify({ 'data': 'Hello, %s!' % g.user.username })
But of course Flask-HTTPAuth needs to be given some more information to know how to validate user credentials, and for this there are several options depending on the level of security implemented by the application.

The option that gives the maximum flexibility (and the only that can accomodate PassLib hashes) is implemented through the verify_password callback, which is given the username and password and is supposed to return True if the combination is valid or False if not. Flask-HTTPAuth invokes this callback function whenever it needs to validate a username and password pair.

An implementation of the verify_password callback for the example API is shown below:

@auth.verify_password
def verify_password(username, password):
    user = User.query.filter_by(username = username).first()
    if not user or not user.verify_password(password):
        return False
    g.user = user
    return True
This function finds the user by the username, then verifies the password using the verify_password() method. If the credentials are valid then the user is stored in Flask's g object so that the view function can use it.

Here is an example curl request that gets the protected resource for the user registered above:

$ curl -u miguel:python -i -X GET http://127.0.0.1:5000/api/resource
HTTP/1.0 200 OK
Content-Type: application/json
Content-Length: 30
Server: Werkzeug/0.9.4 Python/2.7.3
Date: Thu, 28 Nov 2013 20:02:25 GMT

{
  "data": "Hello, miguel!"
}
If an incorrect login is used, then this is what happens:

$ curl -u miguel:ruby -i -X GET http://127.0.0.1:5000/api/resource
HTTP/1.0 401 UNAUTHORIZED
Content-Type: text/html; charset=utf-8
Content-Length: 19
WWW-Authenticate: Basic realm="Authentication Required"
Server: Werkzeug/0.9.4 Python/2.7.3
Date: Thu, 28 Nov 2013 20:03:18 GMT

Unauthorized Access
Once again I feel the need to reiterate that in a real application the API should be available on secure HTTP only.

Token Based Authentication
Having to send the username and the password with every request is inconvenient and can be seen as a security risk even if the transport is secure HTTP, since the client application must have those credentials stored without encryption to be able to send them with the requests.

An improvement over the previous solution is to use a token to authenticate requests.

The idea is that the client application exchanges authentication credentials for an authentication token, and in subsequent requests just sends this token.

Tokens are usually given out with an expiration time, after which they become invalid and a new token needs to be obtained. The potential damage that can be caused if a token is leaked is much smaller due to their short life span.

There are many ways to implement tokens. A straightforward implementation is to generate a random sequence of characters of certain length that is stored with the user and the password in the database, possibly with an expiration date as well. The token then becomes sort of a plain text password, in that can be easily verified with a string comparison, plus a check of its expiration date.

A more elaborated implementation that requires no server side storage is to use a cryptographically signed message as a token. This has the advantage that the information related to the token, namely the user for which the token was generated, is encoded in the token itself and protected against tampering with a strong cryptographic signature.

Flask uses a similar approach to write secure cookies. This implementation is based on a package called itsdangerous, which I will also use here.

The token generation and verification can be implemented as additional methods in the User model:

from itsdangerous import TimedJSONWebSignatureSerializer as Serializer

class User(db.Model):
    # ...

    def generate_auth_token(self, expiration = 600):
        s = Serializer(app.config['SECRET_KEY'], expires_in = expiration)
        return s.dumps({ 'id': self.id })

    @staticmethod
    def verify_auth_token(token):
        s = Serializer(app.config['SECRET_KEY'])
        try:
            data = s.loads(token)
        except SignatureExpired:
            return None # valid token, but expired
        except BadSignature:
            return None # invalid token
        user = User.query.get(data['id'])
        return user
In the generate_auth_token() method the token is an encrypted version of a dictionary that has the id of the user. The token will also have an expiration time embedded in it, which by default will be of ten minutes (600 seconds).

The verification is implemented in a verify_auth_token() static method. A static method is used because the user will only be known once the token is decoded. If the token can be decoded then the id encoded in it is used to load the user, and that user is returned.

The API needs a new endpoint that the client can use to request a token:

@app.route('/api/token')
@auth.login_required
def get_auth_token():
    token = g.user.generate_auth_token()
    return jsonify({ 'token': token.decode('ascii') })
Note that this endpoint is protected with the auth.login_required decorator from Flask-HTTPAuth, which requires that username and password are provided.

What remains is to decide how the client is to include this token in a request.

The HTTP Basic Authentication protocol does not specifically require that usernames and passwords are used for authentication, these two fields in the HTTP header can be used to transport any kind of authentication information. For token based authentication the token can be sent as a username, and the password field can be ignored.

This means that now the server can get some requests authenticated with username and password, while others authenticated with an authentication token. The verify_password callback needs to support both authentication styles:

@auth.verify_password
def verify_password(username_or_token, password):
    # first try to authenticate by token
    user = User.verify_auth_token(username_or_token)
    if not user:
        # try to authenticate with username/password
        user = User.query.filter_by(username = username_or_token).first()
        if not user or not user.verify_password(password):
            return False
    g.user = user
    return True
This new version of the verify_password callback attempts authentication twice. First it tries to use the username argument as a token. If that doesn't work, then username and password are verified as before.

The following curl request gets an authentication token:

$ curl -u miguel:python -i -X GET http://127.0.0.1:5000/api/token
HTTP/1.0 200 OK
Content-Type: application/json
Content-Length: 139
Server: Werkzeug/0.9.4 Python/2.7.3
Date: Thu, 28 Nov 2013 20:04:15 GMT

{
  "token": "eyJhbGciOiJIUzI1NiIsImV4cCI6MTM4NTY2OTY1NSwiaWF0IjoxMzg1NjY5MDU1fQ.eyJpZCI6MX0.XbOEFJkhjHJ5uRINh2JA1BPzXjSohKYDRT472wGOvjc"
}
Now the protected resource can be obtained authenticating with the token:

$ curl -u eyJhbGciOiJIUzI1NiIsImV4cCI6MTM4NTY2OTY1NSwiaWF0IjoxMzg1NjY5MDU1fQ.eyJpZCI6MX0.XbOEFJkhjHJ5uRINh2JA1BPzXjSohKYDRT472wGOvjc:unused -i -X GET http://127.0.0.1:5000/api/resource
HTTP/1.0 200 OK
Content-Type: application/json
Content-Length: 30
Server: Werkzeug/0.9.4 Python/2.7.3
Date: Thu, 28 Nov 2013 20:05:08 GMT

{
  "data": "Hello, miguel!"
}
Note that in this last request the password is written as the word unused. The password in this request can be anything, since it isn't used.

OAuth Authentication
When talking about RESTful authentication the OAuth protocol is usually mentioned.

So what is OAuth?

OAuth can be many things. It is most commonly used to allow an application (the consumer) to access data or services that the user (the resource owner) has with another service (the provider), and this is done in a way that prevents the consumer from knowing the login credentials that the user has with the provider.

For example, consider a website or application that asks you for permission to access your Facebook account and post something to your timeline. In this example you are the resource holder (you own your Facebook timeline), the third party application is the consumer and Facebook is the provider. Even if you grant access and the consumer application writes to your timeline, it never sees your Facebook login information.

This usage of OAuth does not apply to a client/server RESTful API. Something like this would only make sense if your RESTful API can be accessed by third party applications (consumers).

In the case of a direct client/server communication there is no need to hide login credentials, the client (curl in the examples above) receives the credentials from the user and uses them to authenticate requests with the server directly.

OAuth can do this as well, and then it becomes a more elaborated version of the example described in this article. This is commonly referred to as the "two-legged OAuth", to contrast it to the more common "three-legged OAuth".

If you decide to support OAuth there are a few implementations available for Python listed in the OAuth website.

Conclusion
I hope this article helped you understand how to implement user authentication for your API.

Once again, you can download and play with a fully working implementation of the server described above. You can find the software on my github site: REST-auth.

If you have any questions or found any flaws in the solution I presented please let me know below in the comments.