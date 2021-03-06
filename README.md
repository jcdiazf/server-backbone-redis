# Why?

I created this module because in a socket.io app I'm working on, all data is created, processed and persisted on the server. By having the models on the server-side with a Redis data store, it is easy to persist the data. With the export/import to JSON from JSON functionality, I can easily push data to clients connected through socket.io. Then, on the client-side, I continue to use the backbone models within views to update the UI.

# Documentation

* For up-to-date documentation, see the [Wiki](https://github.com/JeromeParadis/server-backbone-redis/wiki)
* I sometimes update the doc at the bottom of this README

# Dependencies

	npm install underscore
	npm install backbone
	npm install redis

# Install Latest version

	npm install server-backbone-redis
	
# Howto

## Documentation

This module does two things:

* sharing of models through server and client for a single code base
* using Redis on the server-side as a data store to Backbone

## Backbone Models
You can share your Backbone models for a single definition on the server and the client. Example of a ./models/models.js file. It uses CommonJS to include the right stuff when included in a node application. On the server side and client side, you will always use models.Backbone instead of Backbone.

Create the name property in your models to define the object name. This name will be reused to generate Redis keys. Example:

* Model name = "user", ID=2 -> Redis key = "user:2"
	
	    (function () {
	    var server = false, models;
	    if (typeof exports !== 'undefined') {
		Backbone = require('../../../server-backbone-redis');

		models = exports;
		server = true;

	    } else {
		models = this.models = {};
	    }

		models.Backbone = Backbone;

	    //
	    //models
	    //
		models.User = Backbone.Model.extend({
			defaults: {
				"id":		null,
				"name":		null,
				"entry":        null
			},
			initialize: function() {
				var time = new Date();
				if (!this.get("entry"))
					this.set({entry: time});
			},
			name: "testuser"
		});

		models.UsersCollection = Backbone.Collection.extend({
		    model: models.User
		});

		models.Wrapper = Backbone.Model.extend({});



	    })()

## New JSON export/import model methods (available on server and client)
### xport()
Exports object to JSON.
### mport()
Import data into object from JSON.
EJS example:

    var user1 = new models.User();
    user1.mport('<%- user1.xport() %>');
    

## New server-side Backbone properties/methods
### search(model_class,pattern,cb,err_cb)

Example:

    models.Backbone.search(models.User,"*",function(results) {
        // follow the fetch collections results where results are an array of Backbone model attributes and not Backbone model objects
        // NOTE: if (new models.User()).name === "user", then Redis is searched with "keys user:*" command
        // So, we can load the collections ourselves
        var users = new models.UsersCollection(results);
        ... do something with users Backbone collection
    },function(err) { console.log(err); });

### search_delete(model_class,keypattern,cb,cb_err)
Same as search() but will delete the objects of model_class who's IDs match the keypattern
### setClient(rc)
Setup Redis client to avoid creating a second Redis connection.
### initServer(app)
Setup a connect/express app with a handler to add a new server route the xport/mport script to the client. Script can be then include on the client:
    <script type="text/javascript" src="/sbr/backbone-exp-imp.js"></script> 
### debug
Set this property to true to show debugging information.

## Typical usage on the server:
Once your models.js file is defined like above to include the server-side redis stuff. You simply include your models:

    var models = require('./models/models');
You can access Backbone through models.Backbone.

## Typical includes on the client:

    <script type="text/javascript" src="http://ajax.googleapis.com/ajax/libs/jquery/1.5.1/jquery.min.js"></script> 
    <script type="text/javascript" src="http://cdnjs.cloudflare.com/ajax/libs/underscore.js/1.1.7/underscore-min.js"></script> 
    <script type="text/javascript" src="http://cdnjs.cloudflare.com/ajax/libs/backbone.js/0.5.1/backbone-min.js"></script> 
    <script type="text/javascript" src="/models/models.js"></script> 
    <script type="text/javascript" src="/sbr/backbone-exp-imp.js"></script> 


## Redis data store
Server-side, you can use standard Backbone methods to save/persist to redis. The store uses the xport() and mport() methods to save and fetch the objects content in Redis. You can use the usual Backbone model and collections methods such as:

* model.fetch()
* model.save()
* collection.fetch()

You can supply your own Backbone IDs through the id property/attributes in Backbone models. If not supplied, the record will be incremented through a Redis incr counter. For example, the User model with it's name="user" will internally use these Redis keys:

    Record keys -> user:[ID]  (i.e.: "user:1", "user:2", "user:3", etc.
    Counter key  -> "next.user.id"

You can specify your own keys to manually create relationship records. For example:
	models.User = Backbone.Model.extend({
		defaults: {
			"id":		null,
			"name":		null
		},
		name: "user"
	});

		
	models.Group = Backbone.Model.extend({
		defaults: {
			"id":		null
			"name":		null
		},
		name: "group"
	});

	models.UserGroup = Backbone.Model.extend({
		defaults: {
			"id":		null
			"specs":	null
		},
		name: "usergroup"
	});
	...
	// user is loaded or fetched and has an id = 1 (Redis key: "user:1")
	//
	// group1 and group2 are fetched/saved and have ids 1 and 2 (Redis keys "group:1" & "group:2"
	...
	var usergroup1 = new models.UserGroup({userid:user.id,groupid:group1.id});
	usergroup1.id = "group:" + group1.id + ":user:" + user.id;
	usergroup1.save({},{success: function(saved) {
		console.log("Saved usergroup1: " + saved.xport());
	}});
	var usergroup2 = new models.UserGroup({userid:user.id,groupid:group2.id});
	usergroup2.id = "group:" + group2.id + ":user:" + user.id;
	usergroup2.save({},{success: function(saved) {
		console.log("Saved usergroup2: " + saved.xport());

	}});
	// Will create the user group records with Redis keys= "usergroup:group:1:user:1" and "usergroup:group:2:user:1"


# Acknowledgments
 
* xport()/mport() code taken from this fine article: http://andyet.net/blog/2011/feb/15/re-using-backbonejs-models-on-the-server-with-node/
* This article inspired me to add Node.js server-side Redis support to backbone

# Disclaimer / To Do

* Use at your own risk
* Some code is redundant. Some cleanup to do.
* Haven't tested for Redis injection vulnerability. Sanitize/check IDs on server before doing any fetch/create, etc.
* It was created to do everything on the server and push some updates to the client through socket.io. Could add some express routes and backbone.js code on client-side to make calls from the client.
* No Redis pub/sub features

# MIT License

Copyright (c) 2011 Jerome Paradis

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in
all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
THE SOFTWARE.
