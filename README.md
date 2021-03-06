reed
====

A Markdown-based blogging and website core backed by Redis and the
filesystem.

Features:

* Asynchronously turn all markdown (.md) files in a directory into a blog
  stored in the hyper-fast Redis database.
* Turn all markdown files in a (separate) directory into static pages.
* Files are watched for changes and the Redis store is automagically updated.
* Transparently access Redis or the filesystem to find a blog post.
* Markdown metadata to describe your blog posts.
* Fully event-based programming paradigm.

What reed does not do:

* Comments. Use a system like Disqus or roll your own. Comments might be added
  as a separate library later.

What is reed?
-------------
Reed is a (very) lightweight blogging **core** that turns markdown files in a
directory into a blog. It is **not** a fully featured blogging system. If you
are looking for that, check out Wheat or another blog engine.

Reed is intended for developers who want to integrate simple blogging
functionality into their node.js website or application. It makes as little
assumptions as possible about your environment in order to give maximum
flexibility.

How to use reed
----------------
First, install it:

`npm install reed`

Make sure Redis 2.2 or greater is also installed and running. After
Redis and reed are installed, use it thus:

```js
var reed = require("reed");
reed.on("ready", function() {
	//ready.
	reed.get("a post", function(err, metadata, html) {
		//you have a post.
	});
});

reed.open("."); //looks for .md files in current directory.
```

In the above example, .md files will be pulled out of the current directory and
indexed into the Redis database by the `index` function. After having indexed
them, we can `list` the titles in order of post/updated date (that is, last
modified date).

Configuration
-------------
Reed can connect to Redis running on separate hosts, non-standard
ports or using authentication. This requires the use of the
`configure` function before calling `open`.

```js
var reed = require("reed");

//configure reed to connect to another redis
//must be done *before* reed.open()
reed.configure({
    host: 'some.other.host.org',
    port: 1337,
    password: '15qe93rktkf39i4'
});

reed.on("ready", function() {
	//ready.
	reed.get("a post", function(err, metadata, html) {
		//you have a post.
	});
});

reed.open("."); //looks for .md files in current directory.
```

Any property not overridden in the configuration object will use the
Redis defaults. For example, it is possible to override just the port.

Retrieving Posts
----------------
To retrieve an individual post and its associated metadata, use the `get`
function:

```js
reed.get("First Post", function(err, metadata, htmlContent) {
	console.log(JSON.stringify(metadata);
	console.log(htmlContent);
});
```

If retrieval of the post was successful, `err` will be null. `metadata` will be
an object containing a `markdown` property that stores the original markdown
text, a `lastModified` property that stores the last modified date as UNIX
epoch time, plus any user-defined information (see below). `htmlContent` will be
the post content, converted from markdown to HTML.

If the post could not be retrieved, `err` will be an object containing error
information (exactly what depends on the error thrown), and other two objects
will be `undefined`.

Note that the `get` function will hit the Redis database first, and then look
on the filesystem for a title. So, if you have a new post that has not yet
been indexed, it will get automagically added to the index via `get`.

### Article Naming and Metadata ###
Every article in the blog is a markdown file in the specified directory. The
filename is considered the "id" or "slug" of the article, and must be named
accordingly. Reed article ids must have no spaces. Instead, spaces are mapped
from `-`s:

> "the first post" -> the-first-post.md

These ids are case sensitive, so The-First-Post.md is different than
the-first-post.md.

#### Metadata ####
Similar to  Wheat, articles support user-defined metadata at the top of the
article. These take the form of simple headers. They are transferred into the
metadata object as properties.

the-first-post.md:

```
Title: The First Post
Author: me
SomeOtherField: 123skidoo
```

The headers will be accessible thus:

* metadata.title
* metadata.author
* metadata.someOtherField

Field names can only alphabetical characters. So, "Some-Other-Field" is not a
valid article header.

Note: starting in 0.9.3, metadata fields are camelCase, rather than all lower
case.

Blog API
--------
Reed exposes the following functions:

* `configure(options)`: Configures reed. The options object can be
  used to specify connection settings for Redis. Supported settings
  are `host`, `port` and `password`. Any such configuration must be done
  *before* calling `open`.
* `open(dir)`: Opens the given path for reed. When first opened, reed will scan
  the directory for .md files and add them to redis.
* `close()`: Closes reed, shuts down the Redis connection, stops watching all
  .md files, and clears up state.
* `get(id, callback)`: Retrieves a blog post. The callback receives `error`,
  `metadata`, and `htmlContent`.
* `all(callback)`: Retrieves all blog posts. The callback receives `error` and
  `posts`, which is a list of post objects, each containing `metadata` and
   `htmlContent` properties.
* `getMetadata(id, callback)`: Retrieves only the metadata for a blog post. The
  callback receives `error` (if there was an error), and `metadata`, an object
  containing the metadata from the blog post.
* `list(callback)`: Retrieves all post IDs, sorted by last modified date. The
  callback receives `error` if there was an error, and `titles`, which is a
  list of post IDs.
* `remove(id, callback)`: Removes a blog post. The callback receives `error`, if
  an error occurred.
* `removeAll(callback)`: Removes all blog posts. The callback is called after
  all posts have been deleted, and receives `error` if there was an error during
  deletion. **This deletion is not transactional!**
* `index(callback)`: Forces a full refresh of the opened directory. This should
  usually not be necessary, as reed should automatically take care of posts
  being added and updated. The callback receives `error` if indexing was
  prematurely interrupted by an error.
* `refresh()`: Forces a refresh of the Redis index, removing any entries that
  are no longer present on the filesystem. This should usually not be necessary,
  as reed should handle this internally.
  
**Note**: `get`, `list`, `index`, `remove`, and `removeAll` asynchronously
block until reed is in a ready state. This means they can be called before
`open`, and they will run after opening has completed.

Reed exposes the following events:

* `error`: Fired when there is an error in certain internal procedures. Usually,
  inspecting the error object of callbacks is sufficient.
* `ready`: Fired when reed has loaded.
* `add`: Fired when a post is added to the blog. Note: posts updated while reed
  is not running are currently considered `add` events.
* `update`: Fired when a blog post is updated while reed is running. Note: posts
  updated while reed is not running are currently considered `add` events.
* `remove`: Fired when a blog post is removed (from the filesystem, through an
  API call, etc). The callback receives the full path of the file that was
  removed.

Pages API
---------
Reed 0.9 introduces pages functionality. This operates similarly to the blog
functionality. Each page is a markdown file in a specified directory. The main
difference is that the pages API is not indexed like blog posts are. There
are no events exposed by the pages API, and there is no way to get a list of
all pages in the system.

This functionality is useful for static pages on a website. A simple example,
using [Express](http://www.expressjs.com) to send the HTML of a reed page to
a user:

```javascript
app.get('/pages/:page', function(req, res) {
	reed.pages.get(req.params.page, function(err, metadata, htmlContent) {
		//In a real scenario, you should use a view
		//and make use of the metadata object.
		if (err) {
			res.send('There was an error: ' + JSON.stringify(err));
		}
		else {
			res.send(htmlContent);
		}
	});
});
```

The pages API is contained within the `pages` namespace:

* `pages.open(dir, callback)`: Opens the given path for reed pages. This
  directory should be separate from the blog directory. Calling open()
  more than once will cause it to throw an error. The callback is called once
  the page system is running.
* `pages.get(title, callback)`: Attempts to find the page with the given title.
  The  callback receives `error`, `metadata`, and `htmlContent`, as in the
  regular `get` method.
* `pages.remove(title, callback)`: Removes the specified page, deleting it from
  Redis and the filesystem.
* `pages.close()`: Closes the pages portion of reed.

Contributors
============
These people have contributed to the development of reed in some way or another:

* [ProjectMoon](https://github.com/ProjectMoon): primary author.
* [algesten](https://github.com/algesten): bug fixes and redis conf.

License
=======
MIT License. Detailed in the LICENSE file.
