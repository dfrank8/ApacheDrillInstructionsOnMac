---


|    |            |
|---------|:-------------:|
| Title: |  Tableau using Apache Drill on MongoDB |
| Date: |    09/15/2015   |
| Categories: | MongoDB, Apache Drill, Tableau |
| Author: | Douglas Franklin (dfrank8) |

Target
======
This article is intended for an audience that is attempting to attach Apache Drill on top of [MongoDB][mongo] for relational querying of said MongoDB database, and thus allowing the database to be used by Tableau. A basic understanding of command-line programming, [hierarchical data structures][hds], and [SQL][sql] would be beneficial but not required. There are resources amongst the writing to fill any knowledge gaps that may hamper your progress.

Introduction
====== 
Non-relational databases (see [NoSQL][nosql]), such as MongoDB or [Accumulo][accumulo] have become much more popular in the recent decade due to an increase in desire for simplicity and scalability in database systems. For many analysts this has posed many additional challenges. The popular structure of SQL databases has proven a need for tools to allow a top-level SQL system to be attached onto non-relational databases. UI-based Modeling tools such as Tableau are used by analysts who don't require the intricacies of querying. However, these applications require a relational database. Because of this, NoSQL databases would require manual queries in their native language, which would be out of the question for users of Tableau.

This is why [Apache Drill][drill] was made.

SQL hasn't had any major syntax changes in recent years, and the people who enjoy SQL <i>really</i> enjoy SQL. I personally am one of those people. In contrast, however, [Java Script Object Notation (JSON)][json] is new and is rapidly being adapted, especially by the web community looking to replace [XML][xml]. Apache drill is used to tie these two technologies together. To learn more about SQL and some of its basics, I'd suggest checking out [SQLCourse.com][sqlcourse]. 

System Setup and Requirements
=====

The setup which is tested here is based on the following components:

  * Macintosh OS X 10.10.4 (14E46) 64-bit `*`
  * [Java JVM OpenJDK 8][jdk8-downloads] `*`
  * [Apache Drill 1.1.0][drill-download]
  * MongoDB 3.0.6 based on this [MongoDB package][mongodb-package] `*`

The components marked with `*` are presupposed to be already installed
properly, and will not be explained here in further detail. Please note
that Apache Drill requires an AMD64 platform (64bit). For a full
documentation about Apache Drill please have a look at the [Drill
website][drill].

Installing Apache Drill
=====
Apache Drill is not currently available under any standard package managers such as [HomeBrew][homebrew] or [NPM][npm]. Instead, you must download the most recent gzip `tar.gz` file from [Apache Drill Downloads Page][drill-download], or simply [click here][download-drill-now]. 

At the time of this article, the most up-to-date version has the following properties: 

- `File Name: apache-drill-1.1.0.tar.gz`
- `Date: 04-Jul-2015 22:19`
- `Packed - Size: 144MB`
- `Unpacked - Size: 345MB`

Unpack the file into whichever directory you wish. I would suggest putting it somewhere that is easily referenced as we will be accessing this directory quite a bit during the rest of the set up process. I placed mine here:

 `/Users/douglasfranklin/Documents/Dremio`

 Navigate to the parent directory (in this case, `/Users/douglasfranklin/Documents`) and type the following commands:

```
$ mkdir Dremio
$ cd Dremio
$ tar -xzf ../apache-drill-1.1.0.tar.gz
```
The directory `apache-drill-1.1.0` has been created. This directory contains all the data needed to run Apache Drill from your machine. Apache Drill is based off the [Java-Virtual-Machine][jvm]. After the installation, it is suggested that you adjust your heap size and maximum direct memory settings. The defaults will be 

`default maximum direct memory: 8G`

`default value of heap size: 4G`

In the Apache Drill configuration file `conf/drill-env.sh` you may change the settings to match your machine:
```
DRILL_MAX_DIRECT_MEMORY="1G"
DRILL_HEAP="512M"
```

Testing your Apache Drill Installation
=====
Navigate into your `/apache-drill-1.1.0`. To make sure everything is working, test the environment with these commands:

Fire up the shell:

`$ bin/drill-embedded`

If all went well, and I truly hope it has, then you should see this prompt:

`0: jdbc:drill:zk=local>`

This Apache Drill package was so kind as to provide very basic sample data for you to test your connection. 
For example, enter the following SQL query at the Apache Drill prompt. Append your statements with a "`;`" to end your SQL
query properly:

*Note: If you see '...' in your prompt, then you likely forgot to close with a semi-colon, or some other brace. This '...' allows you to append the previous command. (i.e It is giving you another chance)*

```
0: jdbc:drill:zk=local> SELECT employee_id,full_name,position_title FROM cp.`employee.json` LIMIT 3;
+--------------+------------------+---------------------+
| employee_id  |    full_name     |   position_title    |
+--------------+------------------+---------------------+
| 1            | Sheri Nowmer     | President           |
| 2            | Derrick Whelply  | VP Country Manager  |
| 4            | Michael Spence   | VP Country Manager  |
+--------------+------------------+---------------------+
3 rows selected (0,52 seconds)
```

You see the result of the SQL query. It consists of the three columns
named `employee_id`, `full_name`, and `position_title` as well as the
respective values from the retrieved data set. We see that Apache Drill
runs with success, and we can continue with the next step: enabling the
Apache Drill database connector for MongoDB in the Apache Drill web user
interface.

Adding Data to your MongoDB Database
=====
Finally, we get to the fun part where we can start exploring with some data.

First things first, let's create some data! Shout-out to the people at [Mockaroo.com][mockaroo]. It is a great resource to create all sorts of data for testing/experimenting with databases. You don't need to create the same data-types as I have, but it would make your life a little easier throughout the rest of this article. For an additional challenge, create as complicated of tables as you desire. We want to demostrate the relational capabilities of drill, and the relationality is required for use in Tableau, so we will be creating 2 tables, linked by a common ID.

First, create 100 rows of People. See the picture below for more details:

![Create 100 People][create100people]

Next, create 100 rows of credit card information. These two tables will align via ID, assigning each credit card to one person. 

![Create 100 Cards][create100cards]

<i>A few things to note:
- The card balance is a simple number, so comparators such as > and < can be used. The account balance data type was not used because it is a string.
- Mockaroo creates random data, your results will not be the same as mine.</i>

Once both .JSON files are downloaded, place them in your directory and name them something appropriate. We will next import them into a MongoDB database. *And remember, it is already assumed that the MongoDB database is up and running!*

```
mongod --dbpath /Users/douglasfranklin/Documents/Dremio/data/db // Spin up your local Mongo server
```
Open a new terminal window, and enter these commands sepatately:
```
mongoimport --host 0.0.0.0:27017 --db example --collection MockPeople < /Users/douglasfranklin/Documents/Dremio/MOCK_DATA_People.json // Import your people into the db. 
mongoimport --host 0.0.0.0:27017 --db example --collection MockCards < /Users/douglasfranklin/Documents/Dremio/MOCK_DATA_Cards.json // Import your cards into the db. 
```
For dbs and collections that don't exist, they will be created. 

Now to test the database to make sure the data loaded properly. Use the following Mongo queries to test the data:
`db.MockPeople.find({"last_name": "Webb"})`

 Which return something similar to:
```
{ "_id" : ObjectId("55f5c24e633034ea5bc2566f"), "id" : 7, "first_name" : "George", "last_name" : "Webb", "email" : "gwebb6@ted.com", "country" : "China" }
{ "_id" : ObjectId("55f5c24e633034ea5bc25679"), "id" : 14, "first_name" : "Irene", "last_name" : "Webb", "email" : "iwebbd@goodreads.com", "country" : 			"France" }
{ "_id" : ObjectId("55f5c24e633034ea5bc256c8"), "id" : 99, "first_name" : "Christina", "last_name" : "Webb", "email" : "cwebb2q@dailymotion.com", 			"country" : "Belarus" }
```
This means that there is data in our database, and it is collectable

Attaching Mongo to Drill
=====

Now that is cool and all, but it is not why we are here. Let's set up our Apache Drill shell with this Mongo database, and do some relational Queries. 

Navigate to `.../apache-drill-1.1.0`. Then call `bin/drill-embedded` to spin up instance. If you don't have JDK installed, you will be prompted to install it. If it isn't installed, close and reopen your terminal window after installation. Use the command `SHOW DATABASES` as seen below to view the available databases reachable by jdbc:

```
0: jdbc:drill:zk=local> SHOW DATABASES;
+---------------------+
|     SCHEMA_NAME     |
+---------------------+
| INFORMATION_SCHEMA  |
| cp.default          |
| dfs.default         |
| dfs.root            |
| dfs.tmp             |
| mongo.local         |
| mongo.example       |
| sys                 |
+---------------------+
8 rows selected (0.148 seconds)
```
Pay close attention to the `mongo.<database>` schemas. `mongo.example` is the one we created before. 

Use the following command to navigate into our table:

```
USE mongo.example;

```

Afterwards, you can view the available tables using 

```
0: jdbc:drill:zk=local> SHOW TABLES;
+---------------+-----------------+
| TABLE_SCHEMA  |   TABLE_NAME    |
+---------------+-----------------+
| mongo.example | MockCards       |
| mongo.example | MockPeople      |
| mongo.example | system.indexes  |
+---------------+-----------------+
3 rows selected (0.154 seconds)
```

And *voila!* You should now be able to run SQL queries using basic SQL syntax. 
Let's start with a simple one:

```
0: jdbc:drill:zk=local> SELECT * FROM MockPeople WHERE country = 'China'; 
// Show all info about all the people from China
+-------+-------------+------------+------------------------------+----------+
|  id   | first_name  | last_name  |            email             | country  |
+-------+-------------+------------+------------------------------+----------+
| 7.0   | George      | Webb       | gwebb6@ted.com               | China    |
| 16.0  | Theresa     | Morales    | tmoralesf@unc.edu            | China    |
| 21.0  | Lawrence    | Peters     | lpetersk@amazon.com          | China    |
| 25.0  | Clarence    | Fisher     | cfishero@photobucket.com     | China    |
| 32.0  | Andrea      | Fowler     | afowlerv@cnet.com            | China    |
| 43.0  | Doris       | Green      | dgreen16@telegraph.co.uk     | China    |
| 45.0  | Adam        | Bishop     | abishop18@usnews.com         | China    |
| 50.0  | Anne        | Alexander  | aalexander1d@ameblo.jp       | China    |
| 52.0  | Lillian     | Morales    | lmorales1f@ted.com           | China    |
| 58.0  | Teresa      | Boyd       | tboyd1l@merriam-webster.com  | China    |
| 64.0  | Kathryn     | Johnston   | kjohnston1r@usatoday.com     | China    |
| 68.0  | Lisa        | Wells      | lwells1v@washington.edu      | China    |
| 75.0  | Rachel      | Matthews   | rmatthews22@flavors.me       | China    |
| 83.0  | Frank       | Andrews    | fandrews2a@apache.org        | China    |
| 93.0  | Victor      | Johnston   | vjohnston2k@bloomberg.com    | China    |
+-------+-------------+------------+------------------------------+----------+
15 rows selected (0.276 seconds)
```
These are some pretty American-sounding names for a bunch of Chinese people, but that's what you get with totally randomized data. Cultural ambiguity aside, we have some working data here. Let's try something a little more complex and test the more powerful aspects of relational data with JOINS:

```
0: jdbc:drill:zk=local> 
SELECT p.first_name AS First_Name, c.cardBalance AS Card_Balance// Show first name and card balance and name columns
	FROM mongo.people.MockPeople p // View initial reading from MockPeople table
	JOIN mongo.people.MockCards c 
		ON p.id = c.id // Join with MockCards table by ID
	WHERE c.cardBalance > 250 // Gather the rich folk with a balance of more than $250
	ORDER BY p.first_Name ASC // Put them in alphabetical order for good measure. 
	LIMIT 15; // Return first 15 results
+-------------+--------------+
| First_Name  | Card_Balance |
+-------------+--------------+
| Adam        | 453.98       |
| Alan        | 290.47       |
| Amy         | 267.48       |
| Andrew      | 467.84       |
| Andrew      | 265.6        |
| Ann         | 259.58       |
| Ann         | 328.31       |
| Annie       | 264.43       |
| Anthony     | 269.29       |
| Barbara     | 387.24       |
| Benjamin    | 398.63       |
| Billy       | 282.8        |
| Bobby       | 412.69       |
| Bruce       | 372.69       |
| Charles     | 285.69       |
+-------------+--------------+
15 rows selected (0.282 seconds)
``` 
There you have it! Relational modeling on a non-relational database. Thanks to the hard work of many engineers, you can now utilize these features quickly and easily. It is truly remarkable. 

Tableau (Unavailable on Macs)
=====
Now we have the last step, and that is connecting this database to Tableau for further analysis. Download it for your desktop from [Tableau.com](tableau-download), and unpack the download file and install the application. Once installed, definitely consider the 14-day free trial, because free is the best. Fill in your personal information and proceed into the application. 

You will need an ODBC Driver for Drill to allow functionality with Tableau as well as a Tableau Data-Connection Customization file. [Install the ODBC driver from here][mapr-odbc-drill-install] and configure it using [these instructions][odbc-config]. Be sure to [test the driver when you've finished][odbc-tester].
*Note: If you have issues finding the application, so did I. Search in /opt/mapr/drillodbc/lib/universal for it.*

Unfortunately, there is currently no way to connect to Other Databases on Mac. 
See [this cummunity link here][tableau-noodbc], and hope that Tableau updates the functionality soon!


Links and References
=====
- MongoDB Project, https://www.mongodb.org/ 
- NoSQL | Wikipedia, https://en.wikipedia.org/wiki/NoSQL 
- Accumulo | Wikipedia, https://en.wikipedia.org/wiki/Apache_Accumulo
- Structured Query Language (SQL), Wikipedia"https://en.wikipedia.org/wiki/SQL
- Apache Drill Project, http://drill.apache.org/
- SQL Course.com, http://www.sqlcourse.com/intro.html
- Java Script Object Notation (JSON) | Wikipedia, https://en.wikipedia.org/wiki/JSON
- Extensible Markup Language (XML) | Wikipedia, https://en.wikipedia.org/wiki/XML
- Hierarchical Database Model | Wikipedia, https://en.wikipedia.org/wiki/Hierarchical_database_model
- Downloadables for JDK 8, http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html
- Apache Drill Project, download area, http://getdrill.org/drill/download/apache-drill-1.1.0.tar.gz
- Package for MongoDB, https://www.mongodb.org/downloads
- Apache Drill Project, http://drill.apache.org/
- HomeBrew Website, http://brew.sh/
- NPM Website, https://www.npmjs.com/
- Apache Drill Project | Downloads, http://getdrill.org/drill/download/
- Apache Drill Project | Download Now, http://getdrill.org/drill/download/apache-drill-1.1.0.tar.gz
- Java Virtual Machine (JVM) | Wikipedia, https://en.wikipedia.org/wiki/Java_virtual_machine
- Mockaroo | Realistic Data Generator, https://www.mockaroo.com/ 
- Tableau | Business analytics anyone can use, http://www.tableau.com/products/desktop


[mongo]: https://www.mongodb.org/ "MongoDB Project"
[nosql]: https://en.wikipedia.org/wiki/NoSQL "NoSQL | Wikipedia"
[accumulo]: https://en.wikipedia.org/wiki/Apache_Accumulo "Accumulo | Wikipedia"
[sql]: https://en.wikipedia.org/wiki/SQL "Structured Query Language (SQL) | Wikipedia"
[drill]: http://drill.apache.org/ "Apache Drill Project"
[sqlcourse]: http://www.sqlcourse.com/intro.html "SQL Course.com"
[json]: https://en.wikipedia.org/wiki/JSON "Java Script Object Notation (JSON) | Wikipedia"
[xml]: https://en.wikipedia.org/wiki/XML "Extensible Markup Language (XML) | Wikipedia"
[hds]: https://en.wikipedia.org/wiki/Hierarchical_database_model "Hierarchical Database Model | Wikipedia"
[jdk8-downloads]: http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html "Downloadables for JDK 8"
[drill-download]: http://getdrill.org/drill/download/apache-drill-1.1.0.tar.gz "Apache Drill Project | download area"
[mongodb-package]: https://www.mongodb.org/downloads "package for MongoDB"
[drill]: http://drill.apache.org/ "Apache Drill Project"
[homebrew]: http://brew.sh/ "HomeBrew Website"
[npm]: https://www.npmjs.com/ "NPM Website"
[drill-download]: http://getdrill.org/drill/download/ "Apache Drill Project | Downloads"
[download-drill-now]: http://getdrill.org/drill/download/apache-drill-1.1.0.tar.gz "Apache Drill Project | Download Now"
[jvm]: https://en.wikipedia.org/wiki/Java_virtual_machine "Java Virtual Machine (JVM) | Wikipedia"
[mockaroo]: https://www.mockaroo.com/ "Mockaroo | Realistic Data Generator"
[tableau-download]: http://www.tableau.com/products/desktop "Tableau | Business analytics anyone can use"
[mapr-odbc-drill-install]: http://package.mapr.com/tools/MapR-ODBC/MapR_Drill/ "Install ODBC Driver for Drill via MapR"
[odbc-config]: https://drill.apache.org/docs/configuring-odbc/ "Configure ODBC Driver"
[odbc-tester]: https://drill.apache.org/docs/testing-the-odbc-connection/ "ODBC Tester"
[tableau-noodbc]: http://community.tableau.com/message/383764#383764 "Tableau Commutity Forum | No ODBC on Mac"


[create100people]: file://localhost/Users/douglasfranklin/Documents/Dremio/pics/Create100People.png
[create100cards]: file://localhost/Users/douglasfranklin/Documents/Dremio/pics/Create100Cards.png


