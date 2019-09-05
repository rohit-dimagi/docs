
# Setting commcarhq in Multi vm Environment

in this doc we will be setting up commcarehq in multiple vm's environment. Idea is to keep different component of the project on different server. we will deploying it on following environment. you can see the different component of the project and their corresponding requirement to run it on different servers.

### What is the Phone responsible for
-   Processing CommCare XML (XForms Spec)
-   Playing the forms specified by the XML
-   Sending the processed XML back to the server
### What is website responsible for 
-   Building the XML to be played on the forms
-   Analyzing and viewing the data from the submissions
-   Other meta things like user management

##### List of servers and their count

+---------------------------------------------------------------------------------+

| VM summary |

+--------------+--------------+------------+--------------------------+-----------+

| | Cores Per VM | RAM Per VM | Data Storage Per VM (GB) | VMs Total |

+--------------+--------------+------------+--------------------------+-----------+

| celery | 8 | 32 | 100 | 1 |

+--------------+--------------+------------+--------------------------+-----------+

| couchdb | 2 | 4 | 92 | 1 |

+--------------+--------------+------------+--------------------------+-----------+

| django | 4 | 8 | 50 | 3 |

+--------------+--------------+------------+--------------------------+-----------+

| es_datanode | 2 | 16 | 400 | 1 |

+--------------+--------------+------------+--------------------------+-----------+

| formplayer | 2 | 4 | 100 | 1 |

+--------------+--------------+------------+--------------------------+-----------+

| kafka | 2 | 4 | 86 | 1 |

+--------------+--------------+------------+--------------------------+-----------+

| nginx | 2 | 4 | 50 | 1 |

+--------------+--------------+------------+--------------------------+-----------+

| pg_main | 2 | 8 | 126 | 1 |

+--------------+--------------+------------+--------------------------+-----------+

| pillowtop | 8 | 32 | 100 | 1 |

+--------------+--------------+------------+--------------------------+-----------+

| rabbitmq | 2 | 4 | 50 | 1 |

+--------------+--------------+------------+--------------------------+-----------+

| redis | 1 | 8 | 0.076 | 1 |

+--------------+--------------+------------+--------------------------+-----------+

| riakcs | 4 | 16 | 1340 | 1 |

+--------------+--------------+------------+--------------------------+-----------+

  
  

A Basic Architecture will look like this.
![enter image description here](http://img.rohit.one/arch.jpg)
  
  ##### A Brief overview of the systems involved in above and their role in supporting the commcarehq.
  ##### Key Services
  Database
  * Postgresql
  * CouchDB
  * Elasticsearch
  * Redis
  * RiakCS
Commcare Processes
  * Django
  * Celery
  * Pillows
  * Formplayer
  Othes
  * Nginx(Proxy)
  * RabbitMQ
  * Shared Directory
  
 **Nginx:-** Entry point for the stack involved in the commcarehq , This will act as a load balancer/reverse proxy and forward the user request to the webworkers.
  **Django Webworker:-** This is where the python code runs and interact with different subsystem depending on the user requests. It handles almost all the logic of commcarehq
 **Redis:-** Used for caching the frequent needed information to reduce round-trips and increase the response time. It is also used for locking and User sessions.
**Formplayer:-** Formplayer is a Java service that allows us to use applications on the web instead of on a mobile device. Formplayer cache session instance in redis and stores session instances via Postgres. For more information on building and running it . see it here [Formplayer](https://github.com/dimagi/formplayer)
5. Rabbitmq :- it runs RabbitMQ which is an open source message broker software. It accepts messages from producers, and delivers them to consumers. It acts like a middleman which can be used to reduce loads and delivery times taken by web application servers.
6. Celery:- This machines runs the celery process. [Celery](http://www.celeryproject.org/) consumes from rabbitmq queue and process them. Anything which usually takes time like sending an email/sms would be run through celery systems.
7. pg_main :- This is the Primary Database for commcarehq . All the forms,user,application etc info are stored in this DB. You can have different dbs for report,sync,warehouse stuff either in the same machine or in the different machines. you just have to provide the connection info in settings of django webworker.
8. CouchDB:-  It's running couchdb which is a no sql databse and also our legacy Primary datasource. Mostly it contains user related info.
9. Kafka :- Kafka is a distributed streaming platform that is used for  publish and subscribe to streams of records. kafka  replicates topic log partitions to multiple servers. Any Changes that happens on couchdb is feed to kafka topics systems from there it's consumed by the pillowtop servers.
10. Pillowtop:- it Takes changes to primary data(couchdDB) and updates secondary data sources(Postgresql,Elasticsearch).Pillow detects changes in couchdb with change_feed feature available in couchdb. it pushes the changes in kafka topics, from there some other pilow service read the changes and applies necessaries changes to postgresql or elasticsearch.
11. Elasticsearch :- It's our secondary analytics data. It's great for searching and querying.It's used for reports,Users,forms etc.
12. Riakcs :- It's Key value store of binary data and used for saving forms. it's massive scalable. The forms submitted through user is saved here and then process by Django Webworkers.
13. Shared Directory :-  it's a Network File System which is shared across Django Webworkers . it's Mostly useful for Caching a file and lookup for File.

### Life of a Form submission
-   In web worker process
-   Received as XML document
-   Parsed for form metadata and case updates
-   Form metadata and case changes are saved to primary DB (postgres models)
-   Form metadata is a structured model
-   Case metadata is a structured model, and case properties are stored as a JSON column
-   Form XML is saved to RiakCS
-   Record of form/case update event is pushed to Kafka where it is picked up by asynchronous processors (pillows)
-   Asynchronous processors pick up events from Kafka handle ETL into ElasticSearch, Postgres report tables, and any other events.

15. 11. ##### Assumption:-

- Domain Name:- testcommcarehq.org A record points to nginx_prox_ip.

- All the required port is open and related server is able to ping themselves.

- Debian based Linux systems.

- You know how to add package repo if not available in systems-repo.

#### Let's Begin with easy setup first

**Nginx**

Installation

  

$ apt install nginx

  

configuration

Edit /etc/nginx/sites-available/default

  

upstream backend {

server backend1.example.com; # ip/domain name of django web server1

server backend2.example.com; # ip/domain name of django web server2

server backend3.example.com; # ip/domain name of django web server3

}

server {

listen 127.0.0.1:80;

listen [::1]:80;

location / {

proxy_pass http://backend;

}

}

After that do a reload

  

$ nginx reload

  

**Rabbitmq**

Installation

$ apt install rabbitmq-server

Configuration

  
  

**Couchdb** >= 1.0 (1.2 recommended)

Installation

  

$ apt install couchdb

  

Configuration:

Start couchdb, and then open [http://localhost:5984/_utils/](http://localhost:5984/_utils/) and create a new database named `commcarehq` and add a user named `commcarehq` with password `commcarehq`.

  

To set up CouchDB from the command line, create the database:

```

$ curl -X PUT http://localhost:5984/commcarehq

```

And add an admin user:

```

$ curl -X PUT http://localhost:5984/_config/admins/commcarehq -d '"commcarehq"'

```

**Elasticsearch**

Installation

Elasticsearch 1.7.4. In Ubuntu and other Debian derivatives, download the deb package, install, and then **hold** the version to prevent automatic upgrades:

  

$ sudo dpkg -i elasticsearch-1.7.4.deb

$ sudo apt-mark hold elasticsearch

**redis >**= 3.0.3

Installation

`sudo apt-get install redis-server`

Kafka

Installation

  

$ apt install kafka

$ /etc/init.d/kafka start

$ /etc/init.d/kafka start

  

Configuration

We are running it with default configuration and we will create topic through django-webworkers

**PostgreSQ**L >= 9.4

Installation

  

$ apt install postgresql

  

Configuration

Log in as the postgres user, and create a `commcarehq` user with password `commcarehq`, and `commcarehq` and `commcarehq_reporting` databases:

```

$ sudo su - postgres

postgres$ createuser -P commcarehq # When prompted, enter password "commcarehq"

postgres$ createdb commcarehq

postgres$ createdb commcarehq_reporting

```

#### Formplayer

Formplayer is a Java service that allows us to use applications on the web instead of on a mobile device.

Prerequisites:

- Install Java

- Initialize formplayer database

log in to postgresql machine and run

``` $ createdb formplayer -U commcarehq -h localhost # Update connection info as necessary```

  

To get set up, download the settings file and `formplayer.jar`. You may run this in the commcare-hq repo root.

  

$ curl https://raw.githubusercontent.com/dimagi/formplayer/master/config/application.example.properties -o formplayer.properties

$ curl https://s3.amazonaws.com/dimagi-formplayer-jars/latest-successful/formplayer.jar -o formplayer.jar

$ java -jar formplayer.jar --spring.config.name=formplayer
