---
layout: post
title:  Using database views with sails.js
---

## Background
For context, our project at UHN used [sails.js](http://sailsjs.org/) + [mongodb](https://www.mongodb.com/) serving a REST API backend, with [angular.js](angularjs.org) sitting in our frontend.  As we trudged on writing controller actions and adding functionality to our web application, we slowly realized that our data was in fact, relational and we had been trying to shoehorn relational concepts into mongodb.  We had decided to make the switch over to a more mature database, postgresql and this post will outline some of the steps we took to leverage the power of database views through sails' ORM, waterline.

(Disclaimer: mongodb is a fantastic database, and excels in particular use cases but ours was not one of them.  [Here's a good article that explores the advantages and disadvantages more eloquently than I can.](http://www.sarahmei.com/blog/2013/11/11/why-you-should-never-use-mongodb/)

## What is a database view?
Shamelessly stolen from [Wikipedia](https://en.wikipedia.org/wiki/View_(SQL)):

1. **Views can represent a subset of the data contained in a table.** Consequently, a view can limit the degree of exposure of the underlying tables to the outer world: a given user may have permission to query the view, while denied access to the rest of the base table.
2. **Views can join and simplify multiple tables into a single virtual table.**
3. **Views can act as aggregated tables**, where the database engine aggregates data (sum, average, etc.) and presents the calculated results as part of the data.
4. **Views can hide the complexity of data.** For example, a view could appear as Sales2000 or Sales2001, transparently partitioning the actual underlying table.
5. **Views take very little space to store;** the database contains only the definition of a view, not a copy of all the data that it presents.
Depending on the SQL engine used, views can provide extra security.

## Prerequisites, Assumptions, and Goals
- sails.js v0.11
- postgresql v9.4.4
- Want to be able to access database view via sails' ORM

## Setup and Steps
As an oversimplified initial use case, consider the models `Product`, `Sale`, and `Customer`.  `Product` has many `Sales`, `Sale` has a `Product` and a `Customer`, and Customer has many `Sales`.  Now, sails' ORM is capable of populating one level deep, so you could get `ProductSales` or `CustomerSales` etc. but any deeper than that you'd have to delve into constructing promise chains and other clutter in your controller code.

So after our base models are in place, we need a couple more steps:

1. Create associated model [file](https://github.com/uhndev/sails-views-example/blob/master/api/models/customersale.js) with the same columns returned from our database view:
{% highlight js %}
attributes: {
	id:           { type: 'integer' },
	customerName: { type: 'string' },
	productName:  { type: 'string' },
	price:        { type: 'string' },
	saleId:       { type: 'integer' },
	saleNumber:   { type: 'integer' },
	createdAt:    { type: 'datetime' },
	updatedAt:    { type: 'datetime' }
}
{% endhighlight %}
2. Write SQL [script](https://github.com/uhndev/sails-views-example/blob/master/config/db/customersale.sql) which will create the desired database view:
{% highlight sql %}
CREATE OR REPLACE VIEW customersale AS
 SELECT customer.id,
		customer.name AS customerName,
		product.name AS productName,
		product.price,
		sale.id AS saleId,
		sale.saleNumber,
		customer.createdAt,
		customer.updatedAt
	 FROM sale
		 LEFT JOIN product ON sale.product = product.id
		 LEFT JOIN customer ON sale.customer = customer.id;
{% endhighlight %}
3. Setup a sails [hook](https://github.com/uhndev/sails-views-example/blob/master/api/hooks/sails-drop-views.js) in api/hooks to drop views before the application lifts:
{% highlight js %}
(function() {
  var pg = require('pg');
  var _ = require('lodash');

  module.exports = function (sails) {
    var env = sails.config.environment;
    var connection = sails.config.connections[env];

    var connectionStr = [
      'postgres://', connection.user, ':', connection.password,
      '@', connection.host, ':', connection.port, '/', connection.database
    ].join('');

    var views = _.map(require('fs').readdirSync('config/db'), function(file) {
      return file.slice(0, -4);
    });

    return {
      initialize: function (next) {
        sails.after('hook:blueprints:loaded', function () {
          pg.connect(connectionStr, function (err, client, done) {
            if (err) {
              console.log('Error fetching client from pool', err);
              return next(err);
            }
            var dropQuery = _.map(views, function (view) {
              return 'DROP VIEW IF EXISTS ' + view + ';';
            }).join(' ');
            client.query(dropQuery, function (err, result) {
              if (err) {
                console.log('Error running query: ' + err);
              }
              done();
              sails.log('Drop View Query executed successfully');
              next();
            })
          })
        });
      }
    };
  };
})();
{% endhighlight %}
4. Setup a sails [hook](https://github.com/uhndev/sails-views-example/blob/master/api/hooks/sails-create-views.js) in api/hooks to recreate views after the ORM has loaded:
{% highlight js %}
(function() {
  var fs = require('fs');
  var pg = require('pg');
  var _ = require('lodash');

  module.exports = function (sails) {
    var env = sails.config.environment;
    var connection = sails.config.connections[env];

    var connectionStr = [
      'postgres://', connection.user, ':', connection.password,
      '@', connection.host, ':', connection.port, '/', connection.database
    ].join('');

    return {
      initialize: function (next) {
        sails.after('hook:orm:loaded', function () {
          pg.connect(connectionStr, function (err, client, done) {
            if (err) {
              console.log('Error fetching client from pool', err);
              return next(err);
            }

            var createQuery = _.map(fs.readdirSync('config/db'), function (view) {
              return fs.readFileSync('config/db/' + view, 'utf-8');
            }).join(' ');

            client.query(createQuery, function (err, result) {
              if (err) {
                console.log('Error running query: ' + err);
              }
              done();
              sails.log('Create View Query executed successfully');
              next();
            });
          });
        });
      }
    };
  };
})();
{% endhighlight %}
## Final Product
[Complete Example](https://github.com/uhndev/sails-views-example)
