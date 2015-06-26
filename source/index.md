---
title: Emissary 2.0.0 Manual

language_tabs:
  - python
  - shell

toc_footers:
  - <a href='http://github.com/LukeB42/Emissary'>Emissary on Github</a>
  - <a href='http://src.psybernetics.org'>Psybernetics Open Source</a>

includes:
  - errors

search: true
---

# Emissary Manual

```json
{
    "active": true, 
    "article_count": 651, 
    "created": 1434517499.0, 
    "group": "Reuters", 
    "name": "Reuters UK", 
    "running": true, 
    "schedule": "15! * * * *", 
    "uid": "NjEzNTYwMA", 
    "url": "http://uk.reuters.com"
}
```
Welcome to the documentation for Emissary and its API.

With minimal fuss and some code examples this guide can help you build a news dataset
for interrogating at your leisure.

## Installation

If you've already installed Emissary then feel free to jump to the [API reference](#api-reference).

Some libraries Emissary uses require header files to link in during compilation.

Those are the Python headers, LibEvent, LibXML2, LibXSLT and libsnappy.

All of these can be obtained on Debian-based systems with the following:

<code>sudo apt-get install -y zlib1g-dev libxml2-dev libxslt1-dev python-dev libevent-dev libsnappy-dev gcc g++</code>

To install Emissary for all users:

<code>git clone git://src.psybernetics.org.uk/emissary.git</code>

<code>cd emissary</code>

<code>sudo python setup.py install</code>

Congratulations, you're now a one-person intelligence agency.
## SSL

Communicating with the API is done over HTTPS, and for this you'll need a certificate and a key.
These can be quickly generated with the following two commands:

<code>openssl genrsa 1024 > key</code>

<code>openssl req -new -x509 -nodes -sha1 -days 365 -key key > cert</code>

## Database URI

Before you can run Emissary a database URI is required in the environment. The reasoning behind this one is to securely keep
your API keys out of any version-control system you might be using if you're working on Emissary itself.

Quickly export a database URI:

<code>export EMISSARY_DATABASE=sqlite://///home/you/.emissary.db</code>

<aside class="notice">Put the export command  into your shell's rcfile to save you having to perform it too often.</aside>

Currently accepted connection types are <code>sqlite:////</code>, <code>mysql://</code>, <code>oracle://</code> and <code>postgres://</code>.

## Your first API key

<pre>
user@host $ python -m emissary.run --cert cert --key key
14/06/2015 16:31:30 - Emissary - INFO - Starting Emissary 2.0.0.
5a59e0a-b457-45c6-9d30-d983419c43e1
</pre>

After following the prior steps you're ready to run the server component for the first time.
This will generate a key based on the name defined in the config as <code>MASTER_KEY_NAME</code>. 

The master key can be used to modify other keys on the system and remotely determine whether new keys can be anonymously created. More on that in the [API Keys](#keys) section.

You don't have to copy your key in any way or memorise it. Your Primary API key is available from the [REPL](#repl) providing it's executed
with the database URI in the environment.

The [Authenticating](#authenticating) section explains how these keys are used to obtain and transmit data.

## Crontabs
<pre>
apikey: your-api-key-here

# url                                                 name         group     minute  hour    day     month   weekday
http://news.ycombinator.com/rss                       "HN"         "HN"      20!     *       *       *       *
http://mf.feeds.reuters.com/reuters/UKdomesticNews    "Reuters UK" "Reuters" 0       3!      *       *       *
</pre>

Cron table files can operate on multiple API keys and will create groups and feeds if they don't already exist.

To use a crontab, fire Emissary up with:

<code>python -m emissary.run -c crontab</code>

For the most part you will probably be creating feeds over the network. The schedule syntax is the same whether you're
updating a local instance via file or a remote instance in a different country.

The [Feeds](#feeds) section details how to create and modify feeds with the REST API.


### Timing syntax

<pre>
 minute  hour    day month   weekday
 10!     6,12    *   0-11    mon-sun
 30      9       20  feb     mon
 1       6!      2!  *       *
</pre>
Examine the example provided to see how schedules can be formed.

The first example would create a feed that is fetched every ten minutes at six in the morning and midday every day of all months.

The second one schedules a feed for half nine on the 20th of Febuary if that day is a Monday.

<aside class="success">Read the source for <code>emissary.controllers.cron.parse_timings</code> to see exactly what's supported.</aside>

## Command line options

<pre>
Usage: python -m emissary.run [options]

  --version             show program's version number and exit
  -h, --help            show this help message and exit
  -c, --crontab         Crontab to parse
  --config              (defaults to emissary.config)
  -a, --address         (defaults to 0.0.0.0)
  -p PORT, --port       (defaults to 6362)
  --key=KEY             SSL key file
  --cert=CERT           SSL certificate
  --pidfile             (defaults to ./emissary.pid)
  --logfile             (defaults to ./emissary.log)
  --stop                
  --debug               Log to stdout
  -d                    Run in the background
  --run-as              (defaults to the invoking user)
  --scripts-dir         (defaults to ./scripts/)
</pre>

<code>--config</code> should point to a python file that defines how variables are obtained, from the local environment or a remote server.
Use the environment to store variables by default.

<code>--run-as</code> will be required if you're running as root, like in an init script.

<code>--debug</code> produces extra logging messages  and recompiles script files every time they're used.
This facilitates rapid development of post-receive components.

# API Reference

You are reading the manual for Emissary 2.0.0. The current API version is 1.0.

All requests to this API are prefixed with <code>/v1/</code>.

For example <code>/v1/articles/search/string:terms</code>.

#### Quick Reference

Endpoint  | Description
--------- | -----------
/keys | Collection of API keys
/keys/<code>string:name</code> | A specific API key
/feeds | A paginated resource for accessing Feed Groups
/feeds/<code>string:groupname</code> | A specific Feed Group 
/feeds/<code>string:groupname</code>/stop | Halt the execution of all Feeds in a specific Feed Group
/feeds/<code>string:groupname</code>/start | Load all feeds in a specific Feed Group
/feeds/<code>string:groupname</code>/articles | A paginated resource for access all articles in a Feed Group
/feeds/<code>string:groupname</code>/search/<code>string:terms</code> | A paginated resource for searching by title within a Feed Group
/feeds/<code>string:groupname</code>/count</code> | A resource for querying the number of articles within a Feed Group
/feeds/<code>string:groupname</code>/<code>string:name</code> | A specific Feed
/feeds/<code>string:groupname</code>/<code>string:name</code>/articles | A paginated resource for accessing the articles pertaining to a specific Feed
/feeds/<code>string:groupname</code>/<code>string:name</code>/stop | Halt the execution of a specific Feed
/feeds/<code>string:groupname</code>/<code>string:name</code>/start | Load a specific feed for fetching
/articles | A paginated resource for accessing all articles pertaining to the requesting API key
/articles/<code>string:uid</code> | A specific article including body content and summary
/articles/search/<code>string:terms</code> | A paginated resource for searching all articles associated with the requesting API key by title
/articles/count | A resource for querying the number of articles associated with the requesting API key

# Authenticating

```shell
# Header control with curl
curl "https://localhost:6362/v1/feeds" \
  -kH "Authorization: Basic your-api-key-here"
```
```python
from emissary.client import Client
from emissary.models import APIKey
# Grab the primary API key
key = APIKey.query.first()
# and construct a client with it
c = Client(key.key, "https://localhost:6362/v1/", verify=False)
```

Emissary expects an API key to be included in all requests in a header like the following:
`Authorization: Basic your-api-key-here`

# Keys

## GET
Sending a <code>GET</code> to <code>/v1/keys</code> will present you with a view of your own key, including its feed groups.

If you send a request to this endpoint with a key whose name corresponds with <code>MASTER_KEY_NAME</code>, by default the first key generated,
the response will include an attribute named system which includes other registered keys and the current state of <code>PERMIT_NEW</code>, which
determines whether new keys can be anonymously created.

## PUT

```python
body = {'name':'some user or department'}
response = c.put('keys', body)
# Check for 201 ("CREATED")
response[1]
201
# Check the contents
response[0]
{u'active': True, u'apikey': u'12848459-1b59-4ae6-8e63-8718d9a5d399', u'name': u'some user or department'}
```

Create new keys by telling the service what name you'd like.

Parameter  | Description
--------- | -----------
name | The name of the new key to create

The name must be unique and if <code>PERMIT_NEW</code> is set to <code>False</code> then new keys will only be created
through whichever key is named <code>MASTER_KEY_NAME</code>.

## POST

Update any key but the master key, from the master key.

Parameter  | Description
--------- | -----------
key | The key string
name | The name of the key to modify
permit_new | Toggle <code>PERMIT_NEW</code>
active | Toggle whether a key is active

The values for <code>key</code> and <code>name</code> determine which key to operate on when used on their own

If values for both <code>key</code> and <code>name</code> are supplied and no other key exists with the supplied name, then the key will be renamed.


## DELETE
<pre>
(3,300) > get keys/some user or department
200
{
    "active": true,
    "apikey": "12848459-1b59-4ae6-8e63-8718d9a5d399",
    "name": "some user or department",
    "feedgroups": []
}
(3,302) > delete keys key=12848459-1b59-4ae6-8e63-8718d9a5d399
204
{}
</pre>

Delete a key and all associated objects.

Parameter  | Description
--------- | -----------
key | The key string


# Feed Groups

Each feed must exist in a group.

A given url will only permit one feed to be created for it, so think about what groups you'll place your feeds in.

## GET

```shell
(3,731) > get feeds?per_page=1&page=2
200
[
    {
        "active": true, 
        "feeds": [
            {
                "group": "MIT Technology Review", 
                "name": "Technology Review", 
                "created": 1434286454.0, 
                "url": "http://www.technologyreview.com/stream/rss", 
                "schedule": "30 * * * *", 
                "article_count": 57, 
                "running": true, 
                "active": true, 
                "uid": "NjExNjM5MQ"
            }
        ], 
        "uid": "MzExMzIzNg", 
        "name": "MIT Technology Review", 
        "created": 1434286396.0
    }
]
```
```python
>>> response = c.get("feeds?per_page=1&page=2")
>>> p(response[0])
[{u'active': True,
  u'created': 1434286396.0,
  u'feeds': [{u'active': True,
              u'article_count': 57,
              u'created': 1434286454.0,
              u'group': u'MIT Technology Review',
              u'name': u'Technology Review',
              u'running': True,
              u'schedule': u'30 * * * *',
              u'uid': u'NjExNjM5MQ',
              u'url': u'http://www.technologyreview.com/stream/rss'}],
  u'name': u'MIT Technology Review',
  u'uid': u'MzExMzIzNg'}]
```
<code>/v1/feeds</code>

A paginated resource for reviewing groups of feeds.

Parameter | Description
--------- | -----------
per_page | The amount of feed groups to return (defaults to 10).
page | The page number to use (defaults to 1).

<code>/v1/feeds/string:name</code>

An individual feed group.

## PUT

<pre>
(3,584) > put feeds name="A group"
201
{
    "active": true, 
    "feeds": [], 
    "uid": "MjE2MTg3Mg", 
    "name": "A group", 
    "created": 1435120617.0
}
</pre>

<code>/v1/feeds</code>

Create new groups by telling the service what name you'd like.

Parameter  | Description
--------- | -----------
name | The name of the new group to create

<code>/v1/feeds/string:name</code>

Rename or change active status of a feed.

Parameter  | Description
--------- | -----------
name | The name of the new group to create
active | Toggle whether a group is active

## POST

<pre>
(3,863) > post feeds/The%20Grauniad name="The Guardian"
200
{
    "active": true, 
    "feeds": [
        {
            "group": "The Guardian", 
            "name": "The Guardian", 
            "created": 1434282329.0, 
            "url": "http://guardian.co.uk/", 
            "schedule": "10! * * * *", 
            "article_count": 1540, 
            "running": true,
            "active": true, 
            "uid": "MDQ0OTczMQ"
        }
    ], 
    "uid": "NjQ0NTg4Mw", 
    "name": "The Guardian", 
    "created": 1434282292.0
}
</pre>

Rename a group or toggle whether it's active.

<code>/v1/feeds/string:name</code>

Parameter | Description
--------- | -----------
name | A unique name to rename the group to
status | A boolean flag that determines whether the scheduler ignores feeds in the group

## DELETE

Delete a feed group and all child objects.



## Start and Stop

<pre>
(4,171) > post feeds/Reuters/stop 1=1
200
{}

(4,171) > get feeds/Reuters
200
{
    "active": true, 
    "feeds": [
        {
            "group": "Reuters", 
            "name": "Reuters UK", 
            "created": 1434517499.0, 
            "url": "http://uk.reuters.com", 
            "schedule": "15! * * * *", 
            "article_count": 1485, 
            "running": false, 
            "active": true, 
            "uid": "NjEzNTYwMA"
        }
    ], 
    "uid": "NjU4NTc0Mw", 
    "name": "Reuters", 
    "created": 1434220040.0
}
</pre>

Start or stop all feeds in a group.

A payload is required as it's a POST request. It doesn't matter what the payload is.

### POST

<code>/v1/feeds/string:name/start</code>

<code>/v1/feeds/string:name/stop</code>

## Articles

A paginated view of all articles within a feed group.

Parameter | Description
--------- | -----------
per_page | The amount of feed groups to return (defaults to 10).
page | The page number to use (defaults to 1).

## Count

<pre>
(3,302) > get feeds/HN/count
200
844
</pre>
Returns an integer count of all articles within a given feed group.

## Search
<pre>
(3,362) > get feeds/HN/search/quant?per_page=2
200
[
    {
        "feed": "HN", 
        "uid": "MDU0MDI2Mw", 
        "title": "The Quantum Thermodynamic Revolution", 
        "url": "http://fqxi.org/community/articles/display/202", 
        "created": 1434897307.0, 
        "content_available": true
    }, 
    {
        "feed": "HN", 
        "uid": "NjM2NTk0NQ", 
        "title": "Quantum Python: Animating the Schrodinger Equation", 
        "url": "https://jakevdp.github.io/blog/2012/09/05/quantum-python/", 
        "created": 1434834124.0, 
        "content_available": true
    }
]
</pre>
A paginated search resource for feed groups.

Parameter | Description
--------- | -----------
per_page | The amount of feed groups to return (defaults to 10).
page | The page number to use (defaults to 1).

# Feeds

## GET

> From the REPL

<pre>
(3,413) > get feeds?per_page=1
200
[
    {
        "active": true, 
        "feeds": [
            {
                "group": "MIT Technology Review", 
                "name": "Technology Review", 
                "created": 1434286454.0, 
                "url": "http://www.technologyreview.com/stream/rss", 
                "schedule": "30 * * * *", 
                "article_count": 57, 
                "running": true, 
                "active": true, 
                "uid": "NjExNjM5MQ"
            }
        ], 
        "uid": "MzExMzIzNg", 
        "name": "MIT Technology Review", 
        "created": 1434286396.0
    }
]
</pre>

Parameter | Description
--------- | -----------
per_page | The amount of feed groups to return (defaults to 10).
page | The page number to use (defaults to 1).

## PUT

<pre>
(3,819) > put feeds/BBC%20News name="BBC News" schedule="50 * * * *" url=http://news.bbc.co.uk/
201
{
    "group": "BBC News", 
    "name": "BBC News", 
    "created": 1435122353.0, 
    "url": "http://news.bbc.co.uk", 
    "schedule": "50 * * * *", 
    "article_count": 0, 
    "running": null, 
    "active": true, 
    "uid": "NjgyNjE2MQ"
}
</pre>

Feeds are created with a <code>PUT</code> request to an existing feed group.

URLs don't need to be RSS feeds. They can point to any regularly updated collection of links, like the index page of a news site.

Parameter | Description
--------- | -----------
name | The name of the new feed
url | The url for the new feed
schedule | The schedule for fetching this new feed on
active | Boolean flag that determines whether the scheduler ignores this feed (default: True)

<aside class="warning">URLs must be unique to your API key. You can't schedule the same URL for multiple feeds.</aside>


## DELETE

<code>/v1/feeds/string:name</code>

Delete a feed and all associated articles.

## Start and Stop

<pre>
(4,741) > post feeds/HN/HN/stop 1=1
200
{
    "group": "HN", 
    "name": "HN", 
    "created": 1434214772.0, 
    "url": "http://news.ycombinator.com/rss", 
    "schedule": "2! * * * *", 
    "article_count": 1018, 
    "running": false, 
    "active": true, 
    "uid": "NDIyNDM0Mg"
}
</pre>

<code>/v1/feeds/string:groupname/string:name/start</code>

<code>/v1/feeds/string:groupname/string:name/stop</code>

Start or stop a feed.

A payload is required as it's a POST request. It doesn't matter what the payload is.

## Articles

Parameter | Description
--------- | -----------
per_page | The amount of article items to return (defaults to 10).
page | The page number to use (defaults to 1).

# Articles
## GET
```python
from emissary.client import Client
from emissary.models import APIKey
# Grab the primary API key
key = APIKey.query.first()
# and construct a client object with it
c = Client(key.key, "https://localhost:6362/v1/", verify=False)

# Get the last article with content
articles = c.get('articles?content=1&per_page=1')

# articles is now a tuple containing a dictionary of results and the response status

# HTTP status code
articles[1]
200
# Construct a pretty printing function
import pprint
p = pprint.PrettyPrinter()
p=p.pprint

# Review what we got
p(articles[0])
[{u'content_available': True,
  u'created': 1434230465.0,
  u'feed': u'HN',
  u'title': u'Interprocess Communication in the Ninth Edition Unix System (1990)',
  u'uid': u'MjQzMTcxMw',
  u'url': u'http://cm.bell-labs.co/who/dmr/ipcpaper.html'}]

# Get article by UID
a = c.get("articles/MjQzMTcxMw")

```
```shell
curl "https://localhost:6362/v1/articles?page=50&per_page=3" \
 -kH "Authorization: Basic your-api-key-here"
[
    {
        "content_available": true, 
        "created": 1434991810.0, 
        "feed": "The Guardian", 
        "title": "Nina Simone: 'Are you ready to burn buildings?'", 
        "uid": "MDQ3MDMxNg", 
        "url": "http://www.theguardian.com/music/2015/jun/22/nina-simone-documentary-what-happened-miss-simone"
    }, 
    {
        "content_available": true, 
        "created": 1434991809.0, 
        "feed": "The Guardian", 
        "title": "Airbnb and Uber\u2019s sharing economy is one route to dotcommunism", 
        "uid": "NTQxNTEwOQ", 
        "url": "http://www.theguardian.com/commentisfree/2015/jun/21/airbnb-uber-sharing-economy-dotcommunism-economy"
    }, 
    {
        "content_available": true, 
        "created": 1434991808.0, 
        "feed": "The Guardian", 
        "title": "Accademia della Crusca: saving Italian one bastardised word at a time", 
        "uid": "MTIxMDUyOA", 
        "url": "http://www.theguardian.com/world/shortcuts/2015/jun/22/accademia-della-crusca-rescuing-beauty-italian"
    }
]

```


### URL Parameters

Parameter | Description
--------- | -----------
per_page | The amount of article items to return (defaults to 10).
page | The page number to use (defaults to 1).
content | A an optional boolean denoting whether to return only items with or without body content.

## DELETE

<code>/v1/articles/string:uid</code>

Delete an article by ID.
## Search

<pre>
(3,408) > get articles/search/electricity?per_page=1
200
[
    {
        "feed": "HN", 
        "uid": "MTg3MDU3Mw", 
        "title": "The Way Humans Get Electricity Is About to Change Forever", 
        "url": "http://www.bloomberg.com/news/articles/2015-06-23/the-way-humans-get-electricity-is-about-to-change-forever", 
        "created": 1435075690.0, 
        "content_available": true
    }
]
</pre>

<code>/v1/articles/search/string:terms</code>

A paginated resource for searching within the titles of all articles associated with the requesting API key.

Parameter | Description
--------- | -----------
per_page | The amount of article items to return (defaults to 10).
page | The page number to use (defaults to 1).


## Retrieving content

<pre>
(3,408) > get articles/MTg3MDU3Mw
200
{
    "feed": "HN", 
    "uid": "MTg3MDU3Mw", 
    "title": "The Way Humans Get Electricity Is About to Change Forever", 
    "url": "http://www.bloomberg.com/news/articles/2015-06-23/the-way-humans-get-electricity-is-about-to-change-forever", 
    "created": 1435075690.0, 
    "summary": "The renewable-energy boom is here. Trillions of dollars will be invested over the next 25 years, driving some of the most profound changes yet in how humans get their electricity. That's according [redacted for this documentation]",
    "content": "The renewable-energy\u00a0boom is here. Trillions of dollars will be invested over\u00a0the next 25 years, driving some of the most profound changes yet in how humans get their electricity. That's[redacted for this documentation]",
    "compressed": true
}

</pre>

<code>/v1/articles/string:uid</code>

Retrieve a JSON object containing body content and summary when available and whether a compressed copy is being stored.

## Count

<pre>
(3,408) > get articles/count
200
3411

(3,411) > 
</pre>

<code>/v1/articles/count</code>

A count of all articles associated with the requesting API key.

# Scripts

> Send SIGHUP to the master process to reload your scripts

<pre>
user@host $ ps faux|grep emissary
user     26303 24.3  2.1 200000 84908 pts/2  Rl+  20:04   0:01  \_ emissary
user     26308  0.0  0.8 150488 33844 pts/2  Sl+  20:04   0:00     \_ emissary
user@host $ kill -1 26303
22/06/2015 20:04:48 - Emissary - INFO - Reloading scripts.
</pre>

Emissary can compile Python files and then run its own instance of the Python runtime over these objects.

This is done for each loadable script in the directory passed to the <code>--scripts-dir</code> parameter for every article
extracted, just before it's committed to the backend database. 

Your scripts can always expect the presence of three objects: <code>feed</code>, <code>article</code> and <code>cache</code>.

<code>feed</code>is the model of the feed that the <code>article</code> will be stored within.

<code>article</code> is the model of the article that has just been retrieved.
Modifying its attributes changes what's passed to subsequent scripts and ultimately what's committed to the database.
This is a good place to perform language detection and translation.

<code>cache</code> is a dictionary that will be passed unmodified to subsequent runs of your script.
If you insert a new key/value pair into this dictionary they'll be available again the next time your script is called.
This is useful for storing database connections, implementing sampling algorithms and growing algebraic data structures.

By default the <code>cache</code> dictionary contains a reference to the internal <code>app</code> object, where the logging interface can be accessed via
<code>app.log</code> and the feed manager can be spoken to via <code>app.inbox</code>.

You're invited to read the source code for Emissary to gain a brighter insight into how the feed manager works.

## Composing extensible systems

```python
# Write new article data to a named pipe
pipe = "/tmp/emissary.pipe"
import os, stat
if not os.path.exists(pipe):
    os.mkfifo(pipe)

fd = os.open(pipe, os.O_CREAT | os.O_WRONLY | os.O_NONBLOCK)
os.write(fd, "%s: %s\n%s\n" % (feed.name, article.title, article.url))
os.close(fd)
del fd, os, stat, pipe
```
To achieve ideal performance, perform analysis with a different process.

Simply use scripts to get the data into the separate process.
<aside class="warning">Scripts that never finish will hold the whole thing up. Use non-blocking IO.</aside>

# REPL

### GET
### PUT
### POST
### DELETE
### SEARCH
### READ
