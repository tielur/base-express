# Best practices for Express app structure

Node and Express don’t come with a strict file and folder structure. Instead, you
can build your web app any way you like. This is great, especially for small apps.
It is easy to start, learn and experiment.

However, as your application grows in size and complexity, things might get confusing.
Your code becomes messy. As your team grows, it becomes harder to work on the same
code base. You are fighting with conflicts each time the code is merged. Adding
new features and handling new situations constantly requires changes in your application
structure. Moreover, there are so many different ways to organise your files and
your code, and it is hard to choose which one is the right for you.

You would like to have a file structure where different files and folders are
responsible for different tasks. You want your project to be easy for multiple
people to work on at the same time, and then their work to be merged with as
little conflicts as possible. You want to keep your code clean. You want your
file structure to allow you to easily add new features.

This can be achieved. We’ve had the same problems and there is a way to structure
your app which will improve the situation and fix many of the problems described
above.

Our structure will be based on the [Model-View-Controller](http://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93controller)
(MVC) design pattern. This pattern is great for separating the responsibility of
the different parts of app and makes your code easier to maintain. Let’s see how
you can effectively implement it with an Express web application. We won’t discuss
the merits of MVC, but instead we will focus on how to properly set it up with
Express and we will also look at some other best practices. We’ve used this on
variety of apps and sizes with both large and small teams and the results have
always been great.

### Example

Let’s look at the following example. An application where a user can login, register
and leave a comment for everyone to admire. Below is the files and folders structure.

	   project/
      controllers/
        comments.js
        index.js
        users.js
      helpers/
        dates.js
      middlewares/
        auth.js
        users.js
      models/
        comment.js
        user.js
      public/
        libs/
        css/
        img/
      views/
        comments/
          comment.jade
        users/
        index.jade
      tests/
        controllers/
        models/
          comment.js
        middlewares/
        integration/
        ui/
      .gitignore
      app.js
      package.json

It might seems complex and large, but don’t worry. By the time you have finished
reading this essay, you will fully understand every part of it. It is actually
very simple.

Let’s see what files and folders there are at the root of our project with a brief
explanation of what each of them is about:

*   **controllers/** – defines your app routes and their logic
*   **helpers/** – code and functionality to be shared by different parts of the project
*   **middlewares/** – Express middlewares which process the incoming requests before handling them down to the routes
*   **models/** – represents data, implements business logic and handles storage
*   **public/** – contains all static files like images, styles and javascript
*   **views/** – provides templates which are rendered and served by your routes
*   **tests/** – tests everything which is in the other folders
*   **app.js** – initializes the app and glues everything together
*   **package.json** – remembers all packages that your app depends on and their versions

A thing to bear in mind here: it is important not only how you structure your files
and folders, but also what each file is responsible for and what it should know
about the outside world.

###Models

Models are the files where you interact with your database. They contain all the
methods and functions which will handle your data. This includes not only the
methods for creating, reading, updating and deleting items, but also any additional
business logic. For example, if you had a car model, you could have a mountTyres
method.

You should create at least one file for each type of data in your database. In
our example, we have users and comments, therefore we have a user model and a
comment model. Sometimes, when a model file becomes too large, it might be better
to break it into several files, based on some internal logic.

You should also try to make your models independent from the outside world. They
don’t need to know about other models and they should never include them. They
don’t need to know about controllers or who uses them. They should never receive
request or response objects. They should never return http errros, but they should
return model specific errors.

All this will make your models much more maintainable. They will be tested easily
because they will have very few and clear dependencies. Models can be moved around
if needed and they can be used by anyone. Changing something in one model, doesn’t affect
any other.

Let’s check the models from our example and see how are they implement according
to the points we’ve talked above. Below is the comment model.

```
var db = require('../db')

// Create new comment in your database and return its id
exports.create = function(user, text, cb) {
  var comment = {
    user: user,
    text: text,
    date: new Date().toString()
  }

  db.save(comment, cb)
}

// Get a particular comment
exports.get = function(id, cb) {
  db.fetch({id:id}, function(err, docs) {
    if (err) return cb(err)
    cb(null, docs[0])
  })
}

// Get all comments
exports.all = function(cb) {
  db.fetch({}, cb)
}

// Get all comments by a particular user
exports.allByUser = function(user, cb) {
  db.fetch({user: user}, cb)
}
```
The user model is not included. The only thing we require is that whoever uses
this module provides us with something to identify the user. It might be a user
id or a user name or maybe something else entirely. The comment model doesn’t
care what it is, it only cares that it can be stored.
```
var db = require('../db')
  , crypto = require('crypto')

hash = function(password) {
  return crypto.createHash('sha1').update(password).digest('base64')
}

exports.create = function(name, email, password, cb) {
  var user = {
    name: name,
    email: email,
    password: hash(password),
  }

  db.save(user, cb)
}

exports.get = function(id, cb) {
  db.fetch({id:id}, function(err, docs) {
    if (err) return cb(err)
    cb(null, docs[0])
  })
}

exports.authenticate = function(email, password) {
  db.fetch({email:email}, function(err, docs) {
    if (err) return cb(err)
    if (docs.length === 0) return cb()

    user = docs[0]

    if (user.password === hash(password)) {
      cb(null, docs[0])
    } else {
      cb()
    }
  })
}

exports.changePassword = function(id, password, cb) {
  db.update({id:id}, {password: hash(password)}, function(err, affected) {
    if (err) return cb(err)
    cb(null, affected > 0)
  })
}
```
In addition to the functionality needed for creating and managing users, there
are also methods used for user authentication and password management. Again,
this model doesn’t know about the existence of any other model, controller or
other parts of the application.

### Views

This folder contains all the templates which are rendered by your application.
This is the place where usually the designers in your team will work.

You would like to have one sub-folder for templates corresponding to each of your
controllers. This way, you will group the templates for the same tasks together.

Choosing a template language can be confusing because there are so many choices.
Our favorite template languages, and the ones we use all the time, are [Jade](http://jade-lang.com)
and [Mustache](http://mustache.github.io/). Jade is great for generating html pages. It makes writing
html tags much shorter and more readable. It also uses JavaScript for conditions
and iteration. Mustache on the other hand, is focused on rendering any kind of
template and it provides the minimum logical operators as possible with very little
way of processing your data. This makes it excellent for writing very clean templates,
which are focused on presenting your data instead of processing it.

A best practice for writing good templates is to avoid doing any processing in
the templates. If your data needs to be processed before it is presented, do it
in your controller. Also, avoid adding too much logic, especially if this logi
 can be moved to the controller.

```
doctype html
html
  head
    title Your comment web app
  body
    h1 Welcome and leave your comment
    each comment in comments
      article.Comment
        .Comment-date= comment.date
        .Comment-text= comment.text
```

As you see, it expects that the data is already processed in the controller
rendering this template.

### Controllers

This is the folder where you will be defining all the routes that your app will
serve. Your controllers will handle web requests, serve your templates to the user
and interact with your models to process and retrieve data. It’s the glue which
connects and controls your web app.

Usually you will have at least one file for each logical part of your application.
For example, one file to handle comments action, another file to handle requests
about users and so on. It’s a good practice that all routes from the same controller
begin with the same prefix. For example **/comments/all** and **/comments/new**.

It’s sometimes hard to decide what should go into a controller and what should
go into the model. A best practice is that a controller should never directly
access the database. It should never call methods like “write”, “update”, “fetch”
which most database drivers provide. Instead it should rely on model methods.
For example if you have a **car** model, and the you want to mount 4 wheels to
the car, the controller will not call **db.update(id, {wheels: 4})** but
instead it will call something like **car.mountWheels(id, 4)**.

Below is the controller responsible for the comments.

```
var express = require('express')
  , router = express.Router()
  , Comment = require('../models/comment')
  , auth = require('../middlewares/auth')

router.post('/', auth, function(req, res) {
  user = req.user.id
  text = req.body.text

  Comment.create(user, text, function (err, comment) {
    res.redirect('/')
  })
})

router.get('/:id', function(req, res) {
  Comment.get(req.params.id, function (err, comment) {
    res.render('comments/comment', {comment: comment})
  })
})

module.exports = router
```

There is also an **index.js** file in the controller folder. It’s purpose is
to load all other controllers and maybe define some paths which don’t have a common
prefix like a home page route for example.

```
var express = require('express')
  , router = express.Router()
  , Comment = require('../models/comment')

router.use('/comments', require('./comments'))
router.use('/users', require('./users'))

router.get('/', function(req, res) {
  Comments.all(function(err, comments) {
    res.render('index', {comments: comments})
  })
})

module.exports = router
```

This file’s router holds all your routes. This is the only router that your
application has to load at startup.

### Middlewares

In this folder, you will store all your Express middlewares. The purpose of a
middleware is to extract a common controller code, which should be executed on
multiple requests and usually modifies the request and/or the response objects.

Just like a controller, a middleware should never directly access the database.
Instead it should use your models for such tasks.

Below is the users middleware, from the **middlewares/users.js** file. Its
purpose is to load the user making the request.

```
User = require('../models/user')

module.exports = function(req, res, next) {
  if (req.session && req.session.user) {
    User.get(req.session.user, function(err, user) {
      if (user) {
        req.user = user
      } else {
        delete req.user
        delete req.session.user
      }

      next()
    })
  } else {
    next()
  }
}
```

This middleware uses the user model and it never directly access the database.

Next, the authorization middleware, which is used when you want to prevent
unauthorized access on some routes.

```
module.exports = function(req, res, next) {
  if (req.user) {
    next()
  } else {
    res.status(401).end()
  }
}
```

It doesn’t have any external dependencies. If you look at the controllers files
above, you can see how it is applied.

### Helpers

This folder contains utility code, which is used at multiple models, middlewares
or controllers, but does not fall under the category they cover. Usually you will
have different files for different common tasks.

An example is a helper file which provides methods for managing dates and times.

### Public

This folder is for static files only. Usually, it has-sub folders like **css**,
**libs**, and **img** for CSS styles, images and JavaScript libraries like
jQuery. It’s a best practice this folder to be served not by your application
but by an Nginx or Apache server as they are much better than Node in serving
static files.

### Tests

Every project needs tests, and you need all tests to be together. To help with
managing them, you will separate them in several sub-folders.

*   controllers
*   helpers
*   models
*   middlewares
*   integration
*   ui

**controllers**, **helpers**, **models** and **middlewares** are pretty clear.
They test the code from their respective counterparts at the root of your project.
In each folder you should have one file for each file in the original folders and
it usually has the exact same name. This makes it easier to find and maintain
your tests.

In the 4 folders above, most of your tests will be unit tests, which means that
you are testing your code isolated from the rest of the application. However, the
**integration** folder contains test which will ensure that all your application
parts work correctly together. For example, it will have tests which will check
that the right middlewares are loaded at the right time. These tests are usually
slower than your unit tests.

The UI tests in the **ui** folder are similar to the integration tests because
they also ensure that everything works well together. However, they are usually
executed in the browser and they simulate the behaviour of a real person working
with your application. Usually, they are even slower than the integration tests.

It’s good practice that your unit tests in the **controllers**, **helpers**,
**models** and **middlewares** folders cover as much code as possible. Try to
test every edge case. Your **integration** tests don’t need to be this extensive.
You’ve already covered the functionality. You only need to ensure that all different
parts of your application work together. The UI tests don’t need to test every
edge case either, but they need to ensure that every UI component is working.

### Other files

The last files from our structure are **app.js**, **package.json**.

**app.js** is the starting point of your application. It loads everything and
it begins serving user requests.

```
var express = require('express')
  , app = express()

app.engine('jade', require('jade').__express)
app.set('view engine', 'jade')

app.use(express.static(__dirname + '/public'))
app.use(require('./middlewares/users'))
app.use(require('./controllers'))

app.listen(3000, function() {
  console.log('Listening on port 3000...')
})
```

You need just one line to load all routes from your controllers. It’s
just after loading the middleware responsible for loading the active user.

The **package.json** file main purpose is to remember your application
dependencies and their respective versions.

```
{
  "name": "Comments App",
  "version": "1.0.0",
  "description": "Comments for everyone.",
  "main": "app.js",
  "scripts": {
    "start": "node app.js",
    "test": "node_modules/.bin/mocha tests/**"
  },
  "keywords": [
    "comments",
    "users",
    "node",
    "express",
    "structure"
  ],
  "author": "Terlici Ltd.",
  "license": "MIT",
  "dependencies": {
    "express": "^4.9.5",
    "jade": "^1.7.0"
  },
  "devDependencies": {
    "mocha": "^1.21.4",
    "should": "^4.0.4"
  }
}
```

It can also do a few other handy tricks which will allow you to run your app
with **npm start** and test it with **npm test**. A great interactive guide
explaining it can be found here: [http://browsenpm.org/package.json](http://browsenpm.org/package.json)

### What’s next?

All of the points mentioned here might be best practices, but setting
them for every new project is a tedious task and it is easy to forget
something. To help you, we’ve created a GitHub repository which contains
all of them. You can fork it or clone it and start your new project right
away. What’s more, we are keeping for you all project dependencies up to
date, so that your new app uses always the latest and the best modules.
Check it out:

[https://github.com/terlici/base-express](https://github.com/terlici/base-express)