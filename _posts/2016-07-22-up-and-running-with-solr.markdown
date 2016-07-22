---
layout: post
title:  "Up and running with SOLR"
date:   2016-07-22 12:30:00
categories: Solr
---


<style>
table{
    border-collapse: collapse;
    border-spacing: 0;
    border:1px solid #333;
}

th{
    border:1px solid #333;
    padding:5px;
}

td{
    border:1px solid #333;
    padding:5px;
}
</style>

Solr
--

Solr is an open source project built on Apache Lucene. Features of Solr include full-text search, hit highlighting, faceted search, real-time indexing, dynamic clustering, database integration, NoSQL features and rich document (e.g., Word, PDF) handling.
Solr powers the search and navigation features of many of the world's largest internet sites.

Java
--

Solr is written in java and hence can be installed on any system where a suitable JRE(Java Runtime Environment) is available. You will need JRE 1.8 or higher. To check java version run this from your command line `java -version` .If you have JRE installed you will get output similar to (although perhaps slightly different from) the following:

    java version "1.8.0_91"
    Java(TM) SE Runtime Environment (build 1.8.0_91-b14)
    Java HotSpot(TM) 64-Bit Server VM (build 25.91-b14, mixed mode)

If you don't have JRE installed or the version is less than 1.8 please download and install the latest from the official [site](http://www.oracle.com/technetwork/java/javase/downloads/index.html).

Installing Solr
--

Solr can be downloaded from their [official website](http://lucene.apache.org/solr/) . At the time of writing the latest version of solr is 6.1 . For linux/unix/osx system download the .tgz file . For windows download the .zip file. We just need to extract the downloaded file to get started with solr. We will cover production based solr configuration later. Go ahead and extract the downloaded file at a convenient location.

Once extracted you are now ready to run solr.

Running Solr
--

From the root folder where you extracted solr run the following to start a solr server

    $ bin/solr start

To start solr on windows machine run

    bin\solr.cmd start

**solr script options**

There are various options provided by bin/solr script.

**Help**

The following command helps you too see how you can execute bin/solr

    $ bin/solr -help

To get help on a specific command such as `start` use :

    $ bin/solr start -help

We recommend you to go through the help documentation. For now lets start working with solr

If you have not started your solr server you can do that by running `bin/solr start`. You can see the UI of solr admin at `http://localhost:8983/solr/#/` .

Create a Core
--

**Core** : Core in solr is used to refer to a single index and associated transaction log and configuration files (including the solrconfig.xml and Schema files, among others). Your Solr installation can have multiple cores if needed, which allows you to index data with different structures in the same server, and maintain more control over how your data is presented to different audiences.

To create a core run : 

    $ bin/solr create -c <name of your core>

Add Documents
--

Alright so you created a new core, but its empty right now. To start indexing and querying on your code you need to fill your core with some documents. Assuming you named your core `books` you can make a http request to put some documents into it.

    $ curl http://localhost:8983/solr/books/update?commitWithin=5000 -H 'Content-type:text/csv' -d '
    id,cat_s,pubyear_i,title_t,author_s,series_s,sequence_i,publisher_s
    book1,fantasy,2010,The Way of Kings,Brandon Sanderson,The Stormlight Archive,1,Tor
    book2,fantasy,1996,A Game of Thrones,George R.R. Martin,A Song of Ice and Fire,1,Bantam
    book3,fantasy,1999,A Clash of Kings,George R.R. Martin,A Song of Ice and Fire,2,Bantam
    book4,sci-fi,1951,Foundation,Isaac Asimov,Foundation Series,1,Bantam
    book5,sci-fi,1952,Foundation and Empire,Isaac Asimov,Foundation Series,2,Bantam
    book6,sci-fi,1992,Snow Crash,Neal Stephenson,Snow Crash,,Bantam
    book7,sci-fi,1984,Neuromancer,William Gibson,Sprawl trilogy,1,Ace
    book8,fantasy,1985,The Black Company,Glen Cook,The Black Company,1,Tor
    book9,fantasy,1965,The Black Cauldron,Lloyd Alexander,The Chronicles of Prydain,2,Square Fish
    book10,fantasy,2001,American Gods,Neil Gaiman,,,Harper'

What this command does is it hits our solr server running on localhost, targets books core and says it to update with the data we are providing. The `commitWithin=5000` says that we would like to see our updates within 5000ms(5 secs) . The data section is fairly trivial we are sending a csv format data to store information related to books. After you have added the documents its time to start querying.

Start Asking Questions
--

The simplest way to query is by building a URL that includes the query. For example :

    http://localhost:8983/solr/books/query?q=*:*&wt=json
 
 Will return you all the books that are stored in solr. To get books of a specific author the query would be

    http://localhost:8983/solr/books/query?q=author_s:Neil Gaiman&wt=json

This will return you all the books written by Neil Gaiman. There is so much more that you can do. If you want books that had been published in a range of years, heres the query that would do it

    http://localhost:8983/solr/books/select?q=pubyear_i:[1992 TO 2010]&wt=json

**fl**

fl stands for field lists and specifies what stored fields should be returned from documents matching the query. The following query will return books with title containing "black" and return only author_s and title_t fields

    http://localhost:8983/solr/books/select?q=title_t:black&fl=author_s,title_t&wt=json

the above query would return the following result 

    {
    "responseHeader":{
        "status":0,
        "QTime":1,
        "params":{
        "q":"title_t:black",
        "indent":"on",
        "fl":"author_s,title_t",
        "wt":"json"}},
    "response":{"numFound":2,"start":0,"docs":[
        {
            "title_t":["The Black Company"],
            "author_s":"Glen Cook"},
        {
            "title_t":["The Black Cauldron"],
            "author_s":"Lloyd Alexander"}]
    }}

**Sorting Results**

The following query would sort the result based on pubyear_i in descending order

    http://localhost:8983/solr/books/select?q=*:*&sort=pubyear_i desc&wt=json

**Paginating Results**

 To paginate a query include the start and rows. For example the following query returns only 3 results starting from 0

    http://localhost:8983/solr/books/select?q=*:*&sort=pubyear_i desc&wt=json&start=0&rows=3

Digging Deep
--

We added a core and some documents into this. We also queried some data but we never said solr what to index nor did we say which fields should support full-text searches. So did the solr index our documents? Yes it did ! How you ask? 

**Dynamic Fields**

The "id" field is already predefined in every schema and hence our "id" field was already indexed. Solr also needs to know the type of your fields in order to correctly index it. Look at the name of the fields that we used to create our books document. "cat_s, pubyear_i, title_t, author_s, series_s, sequence_i, publisher_s " instead of manually defining the type of our fields we used dynamic fields to tell solr the proper types of our field. There are a number of ways for defining new fields :

* Edit the managed-schema file to define the new fields and there types
* Use dynamicFields, a form of **convention-over-configuration** that maps field names to field types based on patterns in the field name. For example, every field ending in “_i” is taken to be an integer.

Some common dynamicFields pattern defined for use are :

| Field Suffix  |Multivalued Suffix     | Type  | Description |
| :-------------: |:-------------:| :-----:| :-----------:|
| _t      | _txt | text_general | Indexed for full-text search so individual words or phrases may be matched.|
| _s      | _ss      |   string | A string value is indexed as a single unit. This is good for sorting, faceting, and analytics. It’s not good for full-text search. |
| _i | _is    |    int | a 32 bit signed integer |
| _l | _ls | long | a 64 bit signed long |
| _f | _fs | float | IEEE 32 bit floating point number (single precision) |
| _d | _ds | double | IEEE 64 bit floating point number (double precision) |
| _b | _bs | boolean | true or false |
| _dt| _dts| date | A date |


Directory Structures
--

Lets explore further, put on your fedora (Big Indiana Jones fan!!). A typical directory structure for solr looks like this

    <solr home dir>/
        solr.xml
        core_name1/
            core.properties
            conf/
                solrconfig.xml
                managed-schema
            data/
        core_name2/
            core.properties
            conf/
                solrconfig.xml
                managed-schema
            data/

Mostly solr home dir is in your root folder/server/solr.

There are some other files but we mentioned only the main ones here.

* solr.xml - Specifies configuration options for your Solr server instance.
* Each Solr core
    * core.properties - defines specific properties for each core such as its name, the collection the core belongs to, the location of the schema, and other parameters.
    * solrconfig.xml controls high-level behavior. You can, for example, specify an alternate location for the data directory.
    * managed-schema - describes the documents you will ask Solr to index. The Schema define a document as a collection of fields. You get to define both the field types and the fields themselves.
    * data/ The directory containing the low level index files.


There is so much more to learn about solr, we will explore them further  in our other posts. For now you can play with what you have learnt.

Issues or Doubts
--

Please mail me at [amitrai48@gmail.com](mailto:amitrai48@gmail.com) for any doubts or issues.

Content Contribution
--

If you find any error in this tutorial or you want to further contribute to the text, please raise a PR. Any contributions you make to this effort are of course greatly appreciated.
