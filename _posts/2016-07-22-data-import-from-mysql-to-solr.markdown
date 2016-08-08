---
layout: post
title:  "Data Import from MYSQL to SOLR"
date:   2016-07-22 12:45:00
categories: Solr Data-Import
---

In our earlier [post]({% post_url 2016-07-22-up-and-running-with-solr %}) we started working with solr. Most of the time our primary data source is some other database like MySQL or mongodb. We use solr to index data from those database and serve it to client.<!--excerpt--> The benefit of such architecture is that solr is not the right choice as a primary data source. Though the read speed in solr is blazingly fast the write speed isnt a lot. Plus when you use MySQL or mongodb or any other datasource as your primary data source you get the advantages of those specific datasource. So the question now is how do you import data from such primary data source to your solr?

Data Import Handler
--

The Data Import Handler (DIH) provides a mechanism for importing content from a data store and indexing it. You can build Solr documents by aggregating data from multiple columns and tables according to configuration. You can schedule the import based on how frequently you want your primary DS(data source) and solr to be in sync.

There are two types of import

* Full Import
* Delta Import

**Full Import**

As the name suggest a full import reindexes the entire data in your solr from your primary DS. This can be okay if your data is not too much and the entire imports of data do not take a lot of time.

**Delta Import**

We only import those data and index it in solr which have been updated/added after the last import was done. This is usually the preferred way when you have a lot of data to index.

What we are going to build?
--

We will use Sakila database in our mysql, we will then try to import this data to our solr and query to get some results. We assume you already have a running instance of mysql on your machine.

**Set up Sakila data**

[Sakila](https://dev.mysql.com/doc/sakila/en/sakila-introduction.html) is a sample database provided by mysql. Follow the below steps to get this on your local mysql : 

    cd ~/Downloads

    wget http://downloads.mysql.com/docs/sakila-db.tar.gz

    tar -xzf sakila-db.tar.gz

    cd sakila-db

    mysql -u root -p < sakila-schema.sql

    mysql -u root -p < sakila-data.sql

You now have the database Sakila, we recommmend you to go through its database design [here](https://dev.mysql.com/doc/sakila/en/sakila-structure.html) . We start with a simple full import of table `actor` from sakila database. We then show you how to use a delta import on the same table and then finally show you how to join multiple tables and index such joins in your solr.

Configuration
--

Start by creating a new core named sakila-actor. Run the following code from the root folder of your solr installation

    bin/solr create -c sakila-actor

Now move to the created directories conf folder

    cd server/solr/scala-actor/conf

we need to add the data import handler in our `solrconfig.xml` file.

{% highlight xml %}
<!--Request Handler for data import -->
<requestHandler name="/dataimport" 
                class="org.apache.solr.handler.dataimport.DataImportHandler">
    <lst name="defaults">
        <str name="config">db-data-config.xml</str>
    </lst>
</requestHandler>
{% endhighlight %}

You can add this just above other requestHandlers for us it was above `<requestHandler name="/select" class="solr.SearchHandler">
`. The important thing to notice in the above requestHandler is the config parameter, which specifies the location of the DIH configuration file that contains specifications for the data source, how to fetch data, what data to fetch, and how to process it to generate the Solr documents to be posted to the index. We will work on `db-data-config.xml` later first we need to add some jars which will help us to connect to mysql.

**Adding jars to connect to MySQL**

Download JDBC driver for MySQL from http://dev.mysql.com/downloads/connector/j/.

Copy file from the downloaded archive 'mysql-connector-java-*.jar' to the folder 'contrib/dataimporthandler/lib' in the folder where Solr was installed. Create 'lib' folder if needed. Now add the following lines in `solrconfig.xml` which you just edited

{% highlight xml %}
<!-- Lib for data import-->
<lib dir="${solr.install.dir:../../../..}/contrib/dataimporthandler/lib" regex=".*\.jar" />
<lib dir="${solr.install.dir:../../../..}/dist/" regex="solr-dataimporthandler-.*\.jar" />
{% endhighlight %}    

**create db-data-config.xml**

Now lets create the configuration file in the same folder where `solrconfig.xml` exists that is in `solr-home-dir/server/solr/scala-actor/conf/` with the following content .

{% highlight xml %}
<dataConfig>
    <dataSource type="JdbcDataSource"
                driver="com.mysql.jdbc.Driver"
                url="jdbc:mysql://localhost:3306/sakila"
                user="username here"
                password="password here" />
    <document>
        <entity name="actor" query="select * from actor;">
            <field column="actor_id" name="id"/>
        </entity>
    </document>
</dataConfig>
{% endhighlight %}

We specify the datasource url,username,password in the dataSource tag, make sure you change it to proper values.
Entity elements may contain one or more 'field' elements, which map the data source field names to Solr fields, and optionally specify per-field transformations. We say that take `actor_id` column of our table from MySQL and store it as `id` in our solr core.As id fields are automatically indexed in solr, this field will be indexed.

**Edit managed-schema**

Now we need to edit managed-schema and let solr know about our other fields and its types. Add the following lines in `managed-schema` 

{% highlight xml %}
<field name="first_name" type="string" indexed="false" stored="false" required="true" />
<field name="last_name" type="string" indexed="false" stored="false" required="false" />
<field name="last_update" type="date" indexed="false" stored="false" required="true" />
{% endhighlight %}

Please restart your solr server after these changes. From your root installation folder run :

    bin/solr restart

Solr Admin UI
--

Move to `http://localhost:8983/solr/` to get the UI of solr admin. We can run our full import through http calls as well but lets use the UI for now.

![solr ui]({{ site.url }}/img/solr-ui.png)

Select the core `scala-actor` from the core selector dropdown. Then click on `DataImport` tab. On this screen select the auto-refresh status check box then click execute.

![data import ui]({{ site.url }}/img/data-import.png)

Since the number of rows of actors is 200(which is really less) the indexing will be done in a second or two. Now go to the query tab and press the button execute, you will get the result.
There you go! You just imported the data from your MySQL to solr. 

Delta Import
--

We just did a full-import of our data from MySQL to SOLR and it hardly took some few seconds. So why invest time in a delta-import? Well we only had 200 rows in our `actor` table but in real world scenario this number is gonna be huge. Every time you do a `full-import` you are wasting a lot of resource. A delta-import only imports those rows that have changed since the last time you indexed. For most scenarios we check the `last_updated`(any such column name which stores the time this row was last updated) columns value with our `last_index_time` and import only those values which are greater.

Edit your `db-data-config.xml` file with the following content : 

{% highlight xml%}
<dataConfig>
<dataSource type="JdbcDataSource"
            driver="com.mysql.jdbc.Driver"
            url="jdbc:mysql://localhost:3306/sakila"
            user="username here"
            password="password here" />
<document>
    <entity name="actor" query="select * from actor;" 
     deltaQuery="select actor_id from actor where convert_tz(last_update,@@session.time_zone,'UTC') > '${dataimporter.last_index_time}'"
     deltaImportQuery="select * from actor where actor_id='${dataimporter.delta.actor_id}'">
        <field column="actor_id" name="id"/>
    </entity>
</document>
</dataConfig>

{% endhighlight %}

* The 'query' gives the data needed to populate fields of the Solr document in full-import
* The 'deltaImportQuery' gives the data needed to populate fields when running a delta-import
* The 'deltaQuery' gives the primary keys of the current entity which have changes since the last index time 

Have a look at the below query and lets understand what it does : 

    deltaQuery="select actor_id from actor where convert_tz(last_update,@@session.time_zone,'UTC') > '${dataimporter.last_index_time}'"

We select primary key of those rows which was updated after we had last indexed. And how do we know when was my data last indexed? Simple we use `dataimporter.last_index_time` and this values comes from `dataimport.properties` in your `conf` folder. Please go through this file and notice the timezone used by solr. Did you see `UTC` ? Solr uses `UTC` time zone and the time zone used by MySQL would be based on your system's time zone. So how do you delta-import properly with the diffference in timezones?
Well you convert your MySQL timezone to UTC and then Solr does the proper comparison and hence the statement : 

    convert_tz(last_update,@@session.time_zone,'UTC')

We are converting the timezome from our system's timezone to UTC. But wait for `conver_tz` to work you need to have your `mysql.time_zone_name` table populated. Please run the following command from terminal to make sure the conversion works without any hussle : 

    mysql_tzinfo_to_sql /usr/share/zoneinfo | mysql -u root mysql -p

If you are on some other OS other than Linux/Unix you could follow the steps in the [official documentation](http://dev.mysql.com/doc/refman/5.7/en/time-zone-support.html).
Verify everything is ok by : 

    mysql> SELECT CONVERT_TZ('2010-01-22 12:00:00','CET','UTC');
    +-----------------------------------------------+
    | CONVERT_TZ('2010-01-22 12:00:00','CET','UTC') |
    +-----------------------------------------------+
    | 2010-01-22 11:00:00                           |
    +-----------------------------------------------+
    1 row in set (0.00 sec)

And finally our `deltaImportQuery` populates the required fields based on the primary key selected earlier by `deltaQuery` command. Its important that you run your delta-import command with clean=false else even delta-import would first clean all your index and then you will basically do a full-import again.. DUH!!!! Go ahead update some rows in your `actors` table and make sure to update the `last_update` column. Now from your solr admin ui data-import section when you select delta-import from dropdown(make sure you untick clean checkbox) and execute your query only those rows will be updated which were modified.

**Imports through URLS**

If you want to import data through urls you can do the following 

* full-import - use URL: `http://localhost:8983/solr/<core name>/dataimport?command=full-import&wt=json`
* delta-import - use URL: `http://localhost:8983/solr/<core name>/dataimport?command=delta-import&clean=false&wt=json`

Data From Multiple Table
--

The best way to store data in solr is in a flat(denormalized) structure. This helps in fast searches as join is a costly expression. So how do we select data from multiple datas and store them as a flat structure  in solr? Before that lets write a SQL query to fetch actors and the films they have worked in. We would probably do something like this : 

{% highlight sql%}
SELECT actor.actor_id, film.film_id, actor.first_name, actor.last_name, film.title 
FROM film_actor 
INNER JOIN actor ON film_actor.actor_id = actor.actor_id 
INNER JOIN film ON film_actor.film_id = film.film_id;
{% endhighlight %}

Have a look at the result, each actors might have worked in multiple films. The primary key is therefore a composite ones containing `actor_id` and `film_id` (Keep this is in mind, more on this soon). Now lets try to get the same result in solr. We start with changing our `db-data-config.xml` file to the following : 

{% highlight xml %}
<dataConfig>
<dataSource type="JdbcDataSource"
            driver="com.mysql.jdbc.Driver"
            url="jdbc:mysql://localhost:3306/sakila"
            user="username here"
            password="password here" />
<document>
    <entity name="actor" query="SELECT actor.actor_id, film.film_id, actor.first_name, actor.last_name, film.title FROM film_actor INNER JOIN 
    actor ON film_actor.actor_id = actor.actor_id INNER JOIN film ON film_actor.film_id = film.film_id;">
        <field column="actor_id" name="id"/>
    </entity>
</document>
</dataConfig>
{% endhighlight %}

Since our schema has changed we will also have to modify our `managed-schema` and let solr know of the new fields. Modify your `managed-schema` to : 

{% highlight xml %}
<field name="id" type="string" indexed="true" stored="true" required="true" multiValued="false" />
<field name="film_id" type="string" indexed="true" stored="true" required="true" multiValued="false" />
<field name="first_name" type="string" indexed="false" stored="false" required="true" />
<field name="last_name" type="string" indexed="false" stored="false" required="false" />
<field name="title" type="string" indexed="false" stored="false" required="true" />
{% endhighlight %}

Restart solr and now lets do a full-import. When you query now, and get data back, do you get the expected result? Wait what? This isn't right is it? Every actor only has acted in one movie? We messed up something didn't we? Well you remember the primary key in the above join was a composite one. Solr still thinks the primary key is `actor_id` and hence just one film per actor. How do we tell solr about composite primary key? Should we combine these fields and make a new field? Also keep in mind that during delta-import we would require a primary key to select our potential candidates to index. So how do we go ahead and solve this? Have a look at the modified `db-data-config.xml` file.

{% highlight xml %}
...

<entity name="actor" query="SELECT actor.actor_id, film.film_id, actor.first_name, actor.last_name, film.title FROM film_actor INNER JOIN 
    actor ON film_actor.actor_id = actor.actor_id INNER JOIN film ON film_actor.film_id = film.film_id;" pk="primaryKey" transformer="TemplateTransformer">
        <field column="primaryKey" name="id" template="${actor.actor_id}_${actor.film_id}"/>
</entity>

...
{% endhighlight %}

The first thing you need to notice is this line 

    pk="primaryKey" transformer="TemplateTransformer"

We say that the primary key for the entity is a field called `primaryKey` and we add a `TemplateTransformer` to it.
Next we map our `primaryKey` column to `id` in the following line.

{% highlight xml %}
<field column="primaryKey" name="id" template="${actor.actor_id}_${actor.film_id}"/>
{% endhighlight %}
 
 We combine `actor_id` and `film_id` to create a new unique field. 

 **Note** : Make sure to change your `managed-schema` to include `actor_id` as a field as now the `id` field is mapped to the combination of `actor_id` and `film_id`.

{% highlight xml %}
...
<field name="actor_id" type="string" indexed="true" stored="true" required="true" multiValued="false" />
...
{% endhighlight %}

Restart solr and do a full-import now. You would see that the data would now be in sync with your primary database. Take a good look at the `id` field it would be something like this `1_23` because we combined two fields two produce it. What about delta-import? Have a look at the modified `db-data-config.xml` file. 

{% highlight xml %}
...
    <entity name="actor" query="SELECT actor.actor_id, film.film_id, actor.first_name, actor.last_name, film.title FROM film_actor INNER JOIN 
    actor ON film_actor.actor_id = actor.actor_id INNER JOIN film ON film_actor.film_id = film.film_id;" pk="primaryKey" transformer="TemplateTransformer,DateFormatTransformer" 
deltaQuery="select actor.actor_id from actor INNER JOIN film_actor ON film_actor.actor_id = actor.actor_id INNER JOIN film ON film_actor.film_id=film.film_id where convert_tz(actor.last_update,@@session.time_zone,'UTC') > '${dataimporter.last_index_time}' OR convert_tz(film_actor.last_update,@@session.time_zone,'UTC') > '${dataimporter.last_index_time}' OR convert_tz(film.last_update,@@session.time_zone,'UTC') > '${dataimporter.last_index_time}'"
     deltaImportQuery="SELECT actor.actor_id, film.film_id, actor.first_name, actor.last_name, film.title FROM film_actor INNER JOIN 
    actor ON film_actor.actor_id = actor.actor_id INNER JOIN film ON film_actor.film_id = film.film_id where actor.actor_id='${dataimporter.delta.actor_id}'">
        <field column="primaryKey" name="id" template="${actor.actor_id}_${actor.film_id}"/>
    </entity>
...
{% endhighlight %}

The `deltaQuery` is really important. What we do is select `actor_id` from the join of three tables where the `last_update` of any of the three tables is greater than `last_index_time` .
Then we use this selected `actor_id` to populate all the data required by us in our `deltaImportQuery`.Restart your solr server and update some rows in your database and use delta-import to check if you get proper data.

There you go, girls and guys we just used solr's Data Import Handler to import data from our primary database to solr.

Issues or Doubts
--

Please mail me at [amitrai48@gmail.com](mailto:amitrai48@gmail.com) for any doubts or issues.

Content Contribution
--

If you find any error in this tutorial or you want to further contribute to the text, please raise a PR. Any contributions you make to this effort are of course greatly appreciated.
