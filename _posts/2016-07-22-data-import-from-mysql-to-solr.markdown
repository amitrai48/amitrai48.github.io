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

    <!--Request Handler for data import -->
    <requestHandler name="/dataimport" class="org.apache.solr.handler.dataimport.DataImportHandler">
        <lst name="defaults">
        <str name="config">db-data-config.xml</str>
        </lst>
    </requestHandler>

You can add this just above other requestHandlers for us it was above `<requestHandler name="/select" class="solr.SearchHandler">
`. The important thing to notice in the above requestHandler is the config parameter, which specifies the location of the DIH configuration file that contains specifications for the data source, how to fetch data, what data to fetch, and how to process it to generate the Solr documents to be posted to the index. We will work on `db-data-config.xml` later first we need to add some jars which will help us to connect to mysql.

**Adding jars to connect to MySQL**

Download JDBC driver for MySQL from http://dev.mysql.com/downloads/connector/j/.

Copy file from the downloaded archive 'mysql-connector-java-*.jar' to the folder 'contrib/dataimporthandler/lib' in the folder where Solr was installed. Create 'lib' folder if needed. Now add the following lines in `solrconfig.xml` which you just edited

    <!-- Lib for data import-->
    <lib dir="${solr.install.dir:../../../..}/contrib/dataimporthandler/lib" regex=".*\.jar" />
    <lib dir="${solr.install.dir:../../../..}/dist/" regex="solr-dataimporthandler-.*\.jar" />
    

**create db-data-config.xml**

Now lets create the configuration file in the same folder where `solrconfig.xml` exists that is in `solr-home-dir/server/solr/scala-actor/conf/` with the following content .

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

We specify the datasource url,username,password in the dataSource tag, make sure you change it to proper values.
Entity elements may contain one or more 'field' elements, which map the data source field names to Solr fields, and optionally specify per-field transformations. We say that take `actor_id` column of our table from MySQL and store it as `id` in our solr core.As id fields are automatically indexed in solr, this field will be indexed.

**Edit managed-schema**

Now we need to edit managed-schema and let solr know about our other fields and its types. Add the following lines in `managed-schema` 

    <field name="first_name" type="string" indexed="false" stored="false" required="true" />
    <field name="last_name" type="string" indexed="false" stored="false" required="false" />
    <field name="last_update" type="date" indexed="false" stored="false" required="true" />


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

Issues or Doubts
--

Please mail me at [amitrai48@gmail.com](mailto:amitrai48@gmail.com) for any doubts or issues.

Content Contribution
--

If you find any error in this tutorial or you want to further contribute to the text, please raise a PR. Any contributions you make to this effort are of course greatly appreciated.
