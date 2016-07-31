---
layout: post
title:  "Playing with Play"
date:   2016-07-31
categories: Play-2.5 Scala
---

Play is a web application framework written on scala and java, which aims on Developers Experience with features like hot code reloading and display of errors in the browser(ofcourse in development mode).Play 1 was written on java, with Play 2 came the power of scala. This means that not only do you get to write your web applications in Scala but you also benefit from increased type safety provided by the language.<!--excerpt-->

Play is a web framework whose HTTP interface is simple, convenient, flexible, and powerful.Play improves on the most popular non-Java web development languages and frameworks—PHP and Ruby on Rails—by introducing the advantages of the Java Virtual Machine (JVM).

Lets Play!
--

**Java**

Play Requires java 1.8 or higher please verify you have proper JDK installed by running 

    java - version

If java is not installed or you have an older version please download java from their [official site](http://www.oracle.com/technetwork/java/javase/downloads/index.html).

**Installing Play**

You can download Play from their [official website](https://www.playframework.com/download). Unzip the download at a proper location where you have write access and add it to your path.

**Linux**

Make sure that the `activator` script is executable. If not run

    chmod u+x /path/to/activator-x.x.x/activator

Then add `path/to/activator-x.x.x` to your path variable

**Windows**

Please set your environment variable to point to proper activator location such as `C:\path\to\activator-x.x.x\bin`.

**Create a new project**

Move to a location where you want to create the project and run `activator new <project name>` .When asked to select a template, please select play-scala. The first time you run this Play will download half of the internet(LOL!!) and will take some time. Do not worry this is only the one time thing.Once done move to the root of the `<project-name>`.

Directory Structure
--

The following directory structure is created by Play. The following is taken from the excellent play [documentation](https://www.playframework.com/documentation/2.5.x/Anatomy). We recommend you to go through this documentation and know play's anatomy inside out. 

    app                      → Application sources
    └ assets                 → Compiled asset sources
        └ stylesheets        → Typically LESS CSS sources
        └ javascripts        → Typically CoffeeScript sources
    └ controllers            → Application controller
    └ views                  → Templates
    
    build.sbt                → Application build script
    
    conf                     → Configurations files and other non-compiled resources (on classpath)
    └ application.conf       → Main configuration file
    └ routes                 → Routes definition
    
    dist                     → Arbitrary files to be included in your projects distribution
    
    public                   → Public assets
    └ stylesheets            → CSS files
    └ javascripts            → Javascript files
    └ images                 → Image files
    
    project                  → sbt configuration files
    └ build.properties       → Marker for sbt project
    └ plugins.sbt            → sbt plugins including the declaration for Play itself
    
    lib                      → Unmanaged libraries dependencies
    
    logs                     → Logs folder
    └ application.log        → Default log file
    
    target                   → Generated stuff
    └ resolution-cache       → Info about dependencies
    └ scala-2.11
        └ api                → Generated API docs
        └ classes            → Compiled class files
        └ routes             → Sources generated from routes
        └ twirl              → Sources generated from templates
    └ universal              → Application packaging
    └ web                    → Compiled web assets
    
    test                     → source folder for unit or functional tests

What we are going to build?
--

We will build a todo application which will save the list of todos in an embedded DB and learn Play's approach to build RESTful api.

**IDE**

Although you don't need an IDE to do wonders with Play we recommend using one as it increases your productivity and saves you a lot of time. There are many IDEs with support for scala we recommend using [IntelliJ](https://www.jetbrains.com/idea/) . When you are done downloading and start installation remember to install scala module.

**create a todo project**

Move to a convenient location and run `activator new todo-play` from your console. Select play-scala as your template. Open this project from your InteliJ IDE. Click on `File -> Open` and select your project. To start your project from the root of your terminal run `activator run` . If all goes well you would finally see something like this :

    --- (Running the application, auto-reloading is enabled) ---

    [info] p.c.s.NettyServer - Listening for HTTP on /0:0:0:0:0:0:0:0:9000

    (Server started, use Ctrl+D to stop and go back to the console...)

Open a browser and open `http://localhost:9000` you would see a default webpage explaining some basics about play.

**Note** The first time that you open your webpage it's gonnna take some time because thats when play compiles all your code. Do not let this fool you that Play is slow. This actually helps to see all your changes right away with just a refresh button. No need to build or deploy anything i.e hot reload.

**Routes**

Why do you see a default web page when you hit `http://localhost:9000` ? Because there is a mapping of `/` route  to this webpage in your routes file(this file is in conf/). Lets create a new route, open routes file and following

    GET     /hello                      controllers.HomeController.hello

Its very trivial and self-explanatory, you just map a route, in this case a `GET` call to /hello path, to a controller's function in this case `hello`. Now lets define this function in HomeController. Open `app/controllers/Homecontroller.scala` and following lines of code 

{% highlight scala %}
    def hello = Action {
        Ok("Hello world")
    }
{% endhighlight %}

Now when you open `http://localhost:9000/hello` you would get your "hello world" back. But wait you didnt deploy this changes anywhere nor did you restart your server. Ladies and Gentlemen thats hot reload for you. Lets do a mistake in our code and see one more feature of play. Remove a closing brace from your hello function and refresh your browser. You see an error page with a proper explanation of your error. Don't worry this will only be in development mode. You can go ahead and fix it.

Actors
--

Actors handle the request received by a Play application and generates a result to be sent to the client. An action returns a `play.api.mvc.Result` value, representing the HTTP response to send to the web client. In the above example Ok constructs a 200 OK response containing a text/plain response body.

There are several ways to construct an Actor the simplest one was the one you created above. If you want to get a reference to the incoming request the following could help : 

{% highlight scala %}
    def hello = Action { request =>
        Ok("Hello world")
    }
{% endhighlight %}

You can also specify an additional `BodyParser` argument : 

{% highlight scala %}
    def hello = Action(parse.json) { request =>
        Ok("Hello world")
    }
{% endhighlight %}

**Other Results**

`Ok` creates a 200 response what if you want a 404 response or some other response? Here is how you do it : 

{% highlight scala %}
val Ok = Ok("A 200 Response")
val notFound = NotFound // a 404 response with no content
val notFoundWithContent = NotFound(<h1> Page Not Found</h1>)
val badRequest = BadRequest("Bad Request")
val oops = InternalServerError("Oops")
val anyStatus = Status(488)("Strange response type")
{% endhighlight %}

H2 Database
--

The H2 in memory database is very convenient for development, you can also let it mimic your planned production database. To tell h2 that you want to mimic a particular database you add a parameter to the database url in your application.conf file, for example:

{% highlight scala %}
db.default.url="jdbc:h2:mem:play;MODE=MYSQL"
{% endhighlight %}

Slick
--

Slick is a modern database query and access library for Scala. It allows you to work with stored data almost as if you were using Scala collections while at the same time giving you full control over when a database access happens and which data is transferred. You can write your database queries in Scala instead of SQL, thus profiting from the static checking, compile-time safety and compositionality of Scala.

To start using Slick you will have to add it as a libray dependancy in your `build.sbt` file. Add/modify the following lines in your build.sbt

{% highlight scala %}
libraryDependencies ++= Seq(
  cache,
  ws,
  "org.scalatestplus.play" %% "scalatestplus-play" % "1.5.1" % Test,
  "com.typesafe.play"   %%   "play-slick"              %   "2.0.0",
  "com.typesafe.play"   %%   "play-slick-evolutions"   %   "2.0.0",
  "com.h2database" % "h2" % "1.4.192"
)
{% endhighlight %}

We start by saying we need `play-slick` which will also bring along the Slick library as a transitive dependency. This implies you don’t need to add an explicit dependency on Slick.

We then use `play-slick-evolutions` to take care of our evolutions script. Learn about evolutions [here](https://www.playframework.com/documentation/2.5.x/Evolutions) .

Since Play Slick module does not bundle any JDBC driver, hence we explicitly add dependency for h2 drivers.

Database Configuartion
--

It is always a good idea to let Play Slick module handle database lifecycles instead of creating database instances explicitly in your code. You can provide Slick database configurations in your `application.conf` file : 

{% highlight scala %}
#Default database configuration

# Database configuration
# ~~~~~
# You can declare as many datasources as you want.
# By convention, the default datasource is named default

slick.dbs.default.driver="slick.driver.H2Driver$"
slick.dbs.default.db.driver="org.h2.Driver"
slick.dbs.default.db.url="jdbc:h2:mem:play"
{% endhighlight %}

Model
--

We will soon see how to use Slick to do database queries. Lets start working with our todo example. We will follow the MVC architecture. Lets create a models package in our app folder and then create a file `Todo.scala` in models package with the following content : 

{% highlight scala %}
package models

import com.google.inject.Inject
import play.api.db.slick.DatabaseConfigProvider
import slick.driver.JdbcProfile
import slick.lifted.{Rep, Tag}
import scala.concurrent.Future


/**
  * Created by amit on 31/7/16.
  */
case class Todo(id: Option[Int], name: String, completed: Boolean)

class TodoRepo @Inject()(dbConfigProvider: DatabaseConfigProvider){
  val dbConfig = dbConfigProvider.get[JdbcProfile]

  import dbConfig.driver.api._

  lazy protected val todoTableQuery = TableQuery[TodoTable]

  def getAllTodo(): Future[List[Todo]] = dbConfig.db.run {
    todoTableQuery.to[List].result
  }

  class TodoTable(tag: Tag)extends Table[Todo](tag,"TODOS"){
    val id = column[Int]("ID", O.AutoInc, O.PrimaryKey)
    val name = column[String]("NAME", O.SqlType("VARCHAR(200)"))
    val completed = column[Boolean]("COMPLETED",O.Default(false), O.SqlType("BIT"))
    // Every table needs a * projection with the same type as the table's type parameter
    def * = (id.?, name, completed) <>(Todo.tupled, Todo.unapply)
  }
}

{% endhighlight %}

We create a `case class Todo` which makes it easy to map our table to our model. We then use dependency Injection to obtain a `DatabaseConfig` (which is a Slick type bundling a database and driver).

{% highlight scala %}
class TodoRepo @Inject()(dbConfigProvider: DatabaseConfigProvider){
  val dbConfig = dbConfigProvider.get[JdbcProfile]
{% endhighlight %}

**Running a database query**

To run a database query, you will need both a Slick database and driver. From the above we already have obtained a Slick DatabaseConfig, hence we have what we need to run a database query.You will need to import some types and implicits from the driver:

{% highlight scala %}
import dbConfig.driver.api._
{% endhighlight %}

**Schema**

There is a inner class called `TodoTable` which describes our schema. Before we can write Slick queries, we need to describe a database schema with `Table` row classes and `TableQuery` values for our tables.The TableQuery‘s ddl method creates DDL (data definition language) objects with the database-specific code for creating and dropping tables and other database entities.

We finally define a `getAllTodo()` which returns a `Future` of list of todos.

**routes**

Lets add a route which returns all our todos. Add this to your `routes` file:

    GET     /api/todos                  controllers.TodoController.getAllTodos

Controller
--

We now need a controller to access our todos and return it back to our client. Create a file `TodoController.scala` in package controllers with the following content : 

{% highlight scala linenos%}
package controllers

import javax.inject.{Inject, Singleton}

import models.TodoRepo
import play.api.libs.json.Json
import play.api.mvc.{Action, Controller}
import scala.concurrent.ExecutionContext.Implicits.global

/**
  * Created by amit on 31/7/16.
  */

@Singleton
class TodoController @Inject()(todoRepo: TodoRepo) extends Controller{

  def getAllTodos = Action.async{
    todoRepo.getAllTodo().map{ todos =>
      Ok(Json.toJson(todos))
    }
  }
}

{% endhighlight %}

The code is pretty self-explanatory, although the import : `import scala.concurrent.ExecutionContext.Implicits.global` needs special mention. Play Framework is, from the bottom up, an asynchronous web framework.All actions in Play Framework use the default thread pool. Now when you work with futures and do some operations on them like `map` in our above example(line number 18), you may need to provide an implicit execution context to execute the given functions in. An execution context is basically another name for a ThreadPool.
In most situations, the appropriate execution context to use will be the Play **default thread pool** and hence the import. 

At this point if you save all your files and reload your browser, you would see a compilation error.

    No Json serializer found for type List[models.Todo]. Try to implement an implicit Writes or Format for this type.

Now what is that? Its time to start working with JSON.

Play with JSON
--

There is a brilliant documentation on [JSON with Play](https://www.playframework.com/documentation/2.5.x/ScalaJson). Please go through the documentation to save yourself from hair pulling sessions later. For us we need a writer which converts from our case class Todo to a JSON value that we can pass to our client. Create a new package called `utils` in `app` folder. Add a class `JsonFormat.scala` with the following content : 

{% highlight scala linenos%}
package utils

/**
  * Created by amit on 31/7/16.
  */
import models._
import play.api.libs.json.Json

object JsonFormat {
  implicit val todoFormat = Json.format[Todo]
}
{% endhighlight %}

Now make sure you add an import to your `TodoController.scala` as 

{% highlight scala %}
import utils.JsonFormat._
{% endhighlight %}

Save your work and refresh your browser and be calm so you dont break your screen because you are gonna see another error.

    [JdbcSQLException: Table "TODOS" not found; SQL statement:
    select "ID", "NAME", "COMPLETED" from "TODOS" [42102-192]]

We never wrote a script to create our table `TODOS` and hence the error.

Lets Evolve
--

We talked  about play-evolutions earlier, now we use it. There is this brillliant documentation on [Evolutions](https://www.playframework.com/documentation/2.5.x/Evolutions). Again we recommend you go through it to save yourself from losing your head.If you did read the documentation you now know that we have to create a file `conf/evolutions/default/1.sql` with the following content : 

{% highlight sql %}
# Todos schema

# --- !Ups

CREATE TABLE todos (
    id bigint(20) NOT NULL AUTO_INCREMENT,
    name varchar(50) NOT NULL,
    completed BIT NOT NULL DEFAULT 0,
    PRIMARY KEY (id)
);

# --- !Downs

DROP TABLE todos;
{% endhighlight %}

Now when you refresh you would be asked to apply the script! Go ahead and click on apply. And you see that default welcome webpage again. Navigate to `http://localhost:9000/api/todos` and you would see an empty array. Thats okay we havent added any todos yet. Lets add some todos then.

Create Todo
--

We begin by defining a function called `createTodo` in our `app/models/Todo.scala` add the following content inside `TodoRepo` class: 

{% highlight scala %}

...

lazy protected val todoTableQueryInc = todoTableQuery returning todoTableQuery.map(_.id)

...

/*
    Inserting the tuples of data is done with the += and ++= methods, similar to how you add data to mutable Scala collections.
*/

def createTodo(todo: Todo): Future[Int] = dbConfig.db.run {
    todoTableQueryInc += todo  
  }
{% endhighlight %}

The `create`, `+=` and `++=` methods return an Action which can be executed on a database at a later time to produce a result.

**Controller**

Define a `createTodo` function which calls the `createTodo` of our `TodoRepo` class we just added. Add following to your `TodoController.scala` : 

{% highlight scala %}

...

  def createTodo = Action.async(parse.json) {request =>
    request.body.validate[Todo].fold(error => Future.successful(BadRequest(JsError.toJson(error))),{todo =>
      todoRepo.createTodo(todo).map { id =>
        Ok(Json.toJson(Map("id" -> id)))
      }
    })
  }

...

{% endhighlight %}

make sure you import `Todo` from your `models` package.

**Routes**

Finally define a route in your `routes` file :

    POST    /api/todos                  controllers.TodoController.createTodo

We are all done. Refresh your browser to compile the latest code. Use `curl` to post and create a new todo.

    curl -H "Content-Type: application/json" -X POST -d '{"name":"playing with play","completed":false}' http://localhost:9000/api/todos

and now when you do a GET on `http://localhost:9000/api/todos` you should probably get following result : 

{% highlight json %}
[{"id":1,"name":"playing with play","completed":false}]
{% endhighlight %}

Edit and Delete Todos
--

We start by defining routes for these 2 operations.

**Routes**

    PUT    /api/todos                   controllers.TodoController.editTodo

    DELETE  /api/todos/:id              controllers.TodoController.deleteTodo(id: Int)

We define a path param `id` in our `DELETE` route which we pass to our controller. Lets check out our controller now.

**Controller**

Add following content to your `TodoController.scala` file :

{% highlight scala %}
...

def editTodo = Action.async(parse.json) {request =>
    request.body.validate[Todo].fold(error => Future.successful(BadRequest(JsError.toJson(error))),{todo =>
      todoRepo.editTodo(todo).map{rowsEffected =>
        Ok(Json.toJson(rowsEffected))
      }
    })
  }

  def deleteTodo(id: Int) = Action.async {request =>
      todoRepo.deleteTodo(id).map{rowsEffected =>
        Ok(Json.toJson(rowsEffected))
    }
  }

  ...
{% endhighlight %}

The code above is trivial and just calls appropriate function of our `TodoRepo` class.  Add the following in your `Todo.scala` file inside `TodoRepo` class : 

{% highlight scala %}
...

 def editTodo(todo: Todo): Future[Int] = dbConfig.db.run {
    todoTableQuery.filter(_.id === todo.id).update(todo)
  }

  def deleteTodo(id: Int): Future[Int] = dbConfig.db.run {
    todoTableQuery.filter(_.id === id).delete
  }

...
{% endhighlight %}

There you go, since Slick helps you treat database entities as scala collections we apply filter to select appropriate todo to delete or edit. It returns the number of rows affected by the given operation. If the number of rows returned is 0, it means it could not find a row matching your condition.

Play in Production
--

Now lets call `http://localhost:9000/some/url` . What do you see in your browser? You get an action not found page with all the proper routes defined. While hot-reload, error reporting on browser is an awesome feature in development mode we do not want such features in our production mode. There are various ways to run Play in production mode. And heres the best part you just need java installed on your target machine where you wanna deploy your Play app. Run following command from your root folder :

    activator dist

This produces a ZIP file containing all JAR files needed to run your application in the `target/universal` folder of your application.
To run the application, unzip the file on the target server, and then run the script in the bin directory. The name of the script is your application name, and it comes in two versions, a bash shell script, and a windows .bat script.

Copy this .zip file to a convenient location and run following command : 

    $ unzip todo-play-1.0-SNAPSHOT.zip
    $ todo-play-1.0-SNAPSHOT/bin/todo-play -Dplay.crypto.secret=abcdefghijk -Dplay.evolutions.db.default.autoApply=true

If the script file is not set in executable mode run 

    $ chmod +x /path/to/bin/<project-name>

`-Dplay.evolutions.db.default.autoApply=true` property tells that we want evolution scripts to be run(a bad idea in production) if you dont give that Play will throw an error.

Girls and Guys we are all done. We hope you keep playing with play and have fun. GG (Good Game) .

Issues or Doubts
--

Please mail me at [amitrai48@gmail.com](mailto:amitrai48@gmail.com) for any doubts or issues.

Content Contribution
--

If you find any error in this tutorial or you want to further contribute to the text, please raise a PR. Any contributions you make to this effort are of course greatly appreciated.
