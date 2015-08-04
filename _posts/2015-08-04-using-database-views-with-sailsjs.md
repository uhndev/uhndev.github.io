---
-layout: post
-title: Using database views with sails.js
---

## Background
For context, our project at UHN used [sails.js](http://sailsjs.org/) + [mongodb](https://www.mongodb.com/) serving a REST API backend, with [angular.js](angularjs.org) sitting in our frontend.  As we trudged on writing controller actions and adding functionality to our web application, we slowly realized that our data was in fact, relational and we had been trying to shoehorn relational concepts into mongodb.  We had decided to make the switch over to a more mature database, postgresql and this post will outline some of the steps we took to leverage the power of database views through sails' ORM, waterline.

(Disclaimer: mongodb is a fantastic database, and excels in particular use cases but ours was not one of them.  [Here's a good article that explores the advantages and disadvantages more eloquently than I can.](http://www.sarahmei.com/blog/2013/11/11/why-you-should-never-use-mongodb/)

## Prerequisites, Assumptions, and Goals
- sails.js v0.11
- postgresql v9.4.4
- Want to be able to access database view via sails' ORM

## Setup and Steps
As an oversimplified initial use case, consider the models `Product`, `Sale`, and `Customer`.  `Product` has many `Sales`, `Sale` has a `Product` and a `Customer`, and Customer has many `Sales`.  Now, sails' ORM is capable of populating one level deep, so you could get `ProductSales` or `CustomerSales` etc. but any deeper than that you'd have to delve into constructing promise chains and other clutter in your controller code.

So after our base models are in place, we need a couple more steps:

1. Create associated model [file](https://github.com/uhndev/sails-views-example/blob/master/api/models/customersale.js) representing our database view
2. Write SQL [script](https://github.com/uhndev/sails-views-example/blob/master/config/db/customersale.sql) to create the desired database view
3. Setup sails [hook](https://github.com/uhndev/sails-views-example/blob/master/api/hooks/sails-drop-views.js) to drop views before the application lifts
4. Setup sails [hook](https://github.com/uhndev/sails-views-example/blob/master/api/hooks/sails-create-views.js) to recreate views after the ORM has loaded

## Final Product
[Complete Example](https://github.com/uhndev/sails-views-example)
