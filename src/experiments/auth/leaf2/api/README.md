# API Auth

::: warning Version support
This experiment supports `v2.4-beta` upwards.
:::

## What Are We Building?

Modern web app conventions have led to a lot of web apps relying on AJAX requests to a backend (API) using libraries like [axios](https://github.com/axios/axios). These backend APIs take in json encoded data from the frontend, perform some operations and send back a response.

In this experiment, we'll be looking at how to create logins, signups and user updates with Leaf v2.4 +. We'll have various requests in JSON form, and some with headers which we'll be using for this experiment.

::: details Detailed Explanation: A quick look at APIs
*If you know what an API is, you can skip this part*

***What is an API?***

> An Application Programming Interface (API) allows two systems to communicate with one another. - smartbear.com

For our use case, we can say that our API is the backend of whatever application we are building. Our frontend handles user interactions, however, if the frontend needs to perform an operation that needs data from our database for instance, it sends a request to our backend (API) which then checks and returns a response to our frontend. The frontend then performs an action based on whatever response the API returns.
:::

::: details Detailed Explanation: JWT Auth Concept
*If you know how JWTs work, you can skip this part*

***What is a JWT?***

> JSON Web Token (JWT) is an open standard (RFC 7519) that defines a compact and self-contained way for securely transmitting information between parties as a JSON object. This information can be verified and trusted because it is digitally signed. - jwt.io

Basically, you can see this as hashed token containing some specified information token locked with a secret key. JWTs also have expiration times, which means that you can issue a token for a particular timeframe.

***JWTs in user authentication***

- A user's information will be submited to our API
- We check to the credentials against our database and return the user
- We take the encode some unique user info into the JWT and return that to the frontend

These steps will be explained into details below. You can check the [JWT spec](https://jwt.io/introduction) for more info.
:::

## Prerequisites

To get started with this tutorial, you will first need to set up a leaf 2 application. The easiest way to do this is to use the [Leaf CLI](https://cli.leafphp.dev).

```sh
leaf create test-app --basic
```

This will generate a simple leaf 2 app with an `index.php` file as well.

## Building Our Login

Our authentication flow starts with a user entering their credentials on the frontend of an app. This is parsed and sent to the backend (our API) through an [API endpoint](https://smartbear.com/learn/performance-monitoring/api-endpoints/) or route. This means that our first task is to set up a route to handle the data passed into our API.

We'll use leaf's core router for this.

```php
require __DIR__ . "/vendor/autoload.php";

$app = new Leaf\App;

$app->post("/login", function() use($app) {
  // login stuff here
});

$app->run();
```

Before we move on, let's give our data from the frontend a structure so we know exactly what we are working with. This app will take in a username and a password defined in this way.

```js
{
  "username": "mychi",
  "password": "password"
}
```

Our next task is to get the data passed into our app and cross-check it with the data in our database. If the user credentials are found and match those in our database, we will generate a JWT and attach it to the response along with the user, otherwise, we throw an error.

We can retrieve the `username` and `password` passed into our app using the leaf request object.

```php
$data = $app->request()->get(["username", "password"]);
```

This grabs the username and password from the request and saves them in the `$data` variable.

Now, we need to tackle the part that checks the database for the current credentials and generates a JWT if the user is found. Leaf provides an auth class to simply and quickly tackle this. But before we can work with our database, we need to connect to it:

```php
$auth = new Leaf\Auth;
$auth->connect("host", "user", "password", "dbName");
```

Now that we're done with that, let's check our database and generate our token. As usual, leaf provides a simple way of doing this.

```php
$res = $auth->login("users", $data);
```

This single line of code checks for a user in our database, prepares an error if authentication fails, and finally returns the user with a generated JWT if the authentication is successful.

::: warning Watch out
By default, Leaf v2.4 downwards use `md5` to encode the password before comparing in the database. We recommend using a stronger hashing algorithm like bcrypt or sodium. You can select this using the [Auth Settings](https://archive.leafphp.dev/#/leaf/v/2.4-beta/core/auth?id=auth-config-small-classnew-tag-1new) allow us to customize the way Auth behaves. We can also set a custom password hash.

```php
$auth->config("PASSWORD_ENCODE", function($password) {
  return \CustomPasswordHash::create($password);
});
```

<hr>

v2.5 upwards uses PHP's `PASSWORD_DEFAULT` for encryption and so, you don't need to do anything extra. For added security, Leaf Auth uses the Leaf password helper under the hood which allows you to dynamically salt passwords.

:::

When the authentication is successful, the `$res` variable above will have the user and the generated JWT. But what happens when authentication fails? When authentication fails, the `$res` variable will be `null`. But how do we know what exactly went wrong? Simple! We can use the `errors` method on leaf auth:

```php
$res = $auth->login("users", $data);

if ($res === null) {
  $app->response()->throwErr([
    "status" => "failed",
    "data" => $auth->errors(),
  ], 401);
}
```

We are using `response()->throwErr` to break the app with a specified response. This will make sure that the code below it doesn't run just like adding a `return` statement does.

This handles the error response, but if the authentication is successful, the `$res` variable will contain the user and the generated JWT as mentioned above.

```php
$res = $auth->login("users", $data);

if ($res === null) {
  $app->response()->throwErr([
    "status" => "failed",
    "data" => $auth->errors(),
  ], 401);
}

$app->response()->json([
  "status" => "success",
  "data" => $res,
]);
```

So, putting all of this together, we'll have something like this:

```php
<?php

require __DIR__ . "/vendor/autoload.php";

$app = new Leaf\App;
$auth = new Leaf\Auth;

$auth->connect("host", "user", "password", "dbName");
$auth->config("PASSWORD_ENCODE", function($password) {
  return \CustomPasswordHash::create($password);
});

$app->post("/login", function() use($app, $auth) {
  $data = $app->request()->get(["username", "password"]);
  $res = $auth->login("users", $data);

  if ($res === null) {
    $app->response()->throwErr([
      "status" => "failed",
      "data" => $auth->errors(),
    ]);
  }

  $app->response()->json([
    "status" => "success",
    "data" => $res,
  ]);
});

$app->run();
```

The output will look like this:

```js
{
  "user": {
    "username": "mychi",
    "email": "mickdd22@gmail.com",
    "created_at": "2019-09-20 13:47:48"
  },
  "token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpYXQiOjE1NzYxMzUzMjgsImlzcyI6ImxvY2FsaG9zdCIsImV4cCI6MTU3NjEzNjIyOCwidXNlcklkIjoxfQ.7FODXGGJKioGQVX4ic0DJLoMIQTVUlsd4zFAJA4DAkg"
}
```

## Using our login

After creating our login handler, we just need to plug this into our frontend. We can do this using any library for handling network requests. For this example, we will use [axios](https://github.com/axios/axios)

::: tip Note
For this demonstration, we will be using axios with promises. Take a look at ES6 if you are not familiar with promises.
:::

```js
axios.post('/login', {
  username: 'mcyhi',
  password: 'password',
}).then((res) => {
  // success login instance
  if (res.data.status === "success") {
    localStorage["user"] = res.data.data.user;
    localStorage["token"] = res.data.data.token;
  }
}).catch((err) => {
  if (err?.response && err?.response?.data?.status) {
    // a toast library
    toast.error(err?.response?.data?.auth);
  }
});
```

## Building Our Register

It's this easy to create and use logins, so what about registrations? Even easier! Registration involves saving the user in the database, if the user is returned immedietely, a token needs to be created as well. Since we've already configured Leaf Auth, let's just jump right into the code.

```php{3}
$app->post("/register", function() use($app, $auth) {
  $data = $app->request()->get(["username", "email", "password"]);
  $res = $auth->register("users", $data, ["username", "email"]);

  if ($res === null) {
    $app->response()->throwErr([
      "status" => "failed",
      "data" => $auth->errors(),
    ]);
  }

  $app->response()->json([
    "status" => "success",
    "data" => $res,
  ]);
});
```

The third line calls the `register` function which is responsible for registering the user. Unlike `login` above, we passed in a third parameter: an array of items which need to be unique in the database. In this case, leaf auth will make sure that there is no same `username` or `email` in the database before saving the user's credentials.

## Building Our Update User

To edit a user, we have to find the user we want to edit. This means that the user should be logged in. Since we're using JWT, we'll need to pass the token into the request as a bearer token in the authorization header.

An example of this with axios will look like this:

```js{7}
import axios from "axios";

axios({
  url: `${API_URL}/update`,
  method: "POST",
  headers: {
    Authorization: `Bearer ${token}`,
  },
  ...
})
```

Back on our system, we'll have to detect the token, grab and decode it, find the data encoded into it and retrieve the user id. Sounds complex, but again, one line of code with Leaf.

```php
$userId = $auth->id() ?? $app->response()->throwErr([
  "status" => "failed",
  "data" => $auth->errors(),
]);
```

Based on this id, we can create a condition to find the user we want to update. This condition will be passed together with our params into the `update` method, which will look like this:

```php
$data = $request->get(["username", "email"]);

// credentials to find user by
$where = ["id" => $auth->id() ?? $app->response()->throwErr($auth->errors())];

// unique data
$uniques = ["username", "email"];

$user = $auth->update("users", $data, $where, $uniques);
```

## Putting it all together

```php
<?php

require __DIR__ . "/vendor/autoload.php";

$app = new Leaf\App;
$auth = new Leaf\Auth;

$auth->connect("host", "user", "password", "dbName");
$auth->config("PASSWORD_ENCODE", function($password) {
  return \CustomPasswordHash::create($password);
});

$app->post("/login", function() use($app, $auth) {
  $data = $app->request()->get(["username", "password"]);
  $res = $auth->login("users", $data);

  if (!$res) {
    $app->response()->throwErr($auth->errors());
  }

  $app->response()->json($res);
});

$app->post("/register", function() use($app, $auth) {
  $data = $app->request()->get(["username", "email", "password"]);
  $res = $auth->register("users", $data, ["username", "email"]);

  if (!$res) {
    $app->response()->throwErr($auth->errors());
  }

  $app->response()->json($res);
});

$app->post("/update", function() use($app, $auth) {
  $data = $request->get(["username", "email"]);
  $where = ["id" => $auth->id() ?? $app->response()->throwErr($auth->errors())];

  $user = $auth->update("users", $data, $where, ["username", "email"]);
});

$app->run();
```

<br>

Experiment by <a href="https://twitter.com/mychidarko" style="font-size: 20px; text-decoration: underline; color: #ddd;" target="_blank">Mychi Darko</a>
