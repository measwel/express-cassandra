express-cassandra
===================

This is one of the steps needed to transform expressjs into a complete MVC framework, by adding Models-support. No more hassling with code in your models. express-cassandra automatically loads your models and provides you with object oriented mapping with your cassandra tables like a standard ORM.

This module uses datastax <a href="https://github.com/datastax/nodejs-driver">cassandra-driver</a> for node and most of the orm features are wrapper over a modified version of <a href="https://github.com/3logic/apollo-cassandra">apollo-cassandra</a> module. The documentation here is largely copied from apollo-cassandra orm with appropriate changes to fit the criteria of this module.


## Installation

    $ npm install express-cassandra

## Usage
	

```js
var models = require('express-cassandra');

//Tell express-cassandra to use the models-directory, and 
//use bind() to load the models using cassandra configurations.

//If your keyspace doesn't exist it will be created automatically
//using the default replication strategy provided here.

//If dropTableOnSchemaChange=true, then if your model schema changes,
//the corresponding cassandra table will be dropped and recreated with
//the new schema. Setting this to false will send an error message
//in callback instead for any model attribute changes.
models.setDirectory( __dirname + '/models').bind(
    {
        clientOptions: {
            contactPoints: ['127.0.0.1'],
            keyspace: 'mykeyspace',
            queryOptions: {consistency: models.consistencies.one}
        },
        ormOptions: {
            defaultReplicationStrategy : {
                class: 'SimpleStrategy',
                replication_factor: 1
            },
            dropTableOnSchemaChange: false
        }
    },
    function(err) {
        if(err) console.log(err.message);
        else console.log(models.timeuuid());
    }
);

```

## Write a Model named `PersonModel.js` inside models directory

```js

module.exports = { 
    fields:{
        name    : "text",
        surname : "text",
        age     : "int"
    }, 
    key:["name"] 
}

```

Note that a model class name should contain the word `Model` in it,
otherwise it won't be treated as a model class.

## Let's insert some data into PersonModel

```js

var alex = new models.instance.Person({name: "Alex", surname: "Rubiks", age: 32});
alex.save(function(err){
    if(err) console.log(err);
    else console.log('Yuppiie!');
});

```

## Now let's find it

```js

models.instance.Person.find({name: 'jhon'}, function(err, people){
    if(err) throw err;
    console.log('Found ', people);
});

```

## Model Schema in detail

```js

module.exports = {
    "fields": {
        "id"     : { "type": "uuid", "default": {"$db_function": "uuid()"} },
        "name"   : { "type": "varchar", "default": "no name provided"},
        "surname"   : { "type": "varchar", "default": "no surname provided"},
        "complete_name" : { "type": "varchar", "default": function(){ return this.name + ' ' + this.surname;}},
        "age"    :  { "type": "int" },
        "created"     : {"type": "timestamp", "default" : {"$db_function": "now()"} }
    },
    "key" : [["id"],"created"],
    "indexes": ["name"]
}

```

What does the above code means?
- `fields` are the columns of your table. For each column name the value can be a string representing the type or an object containing more specific informations. i.e.
    + ` "id"     : { "type": "uuid", "default": {"$db_function": "uuid()"} },` in this example id type is `uuid` and the default value is a cassandra function (so it will be executed from the database). 
    + `"name"   : { "type": "varchar", "default": "no name provided"},` in this case name is a varchar and, if no value will be provided, it will have a default value of `no name provided`. The same goes for `surname`.
    + `complete_name` the default values is calculated from others field. When apollo processes you model instances, the `complete_name` will be the result of the function you defined. In the function `this` is bound to the current model instance.
    + `age` no default is provided and we could write it just as `"age": "int"`.
    + `created`, like uuid(), will be evaluated from cassandra using the `now()` function.
- `key`: here is where you define the key of your table. As you can imagine, the first value of the array is the `partition key` and the others are the `clustering keys`. The `partition key` can be an array defining a `compound key`. Read more about keys on the <a href="http://www.datastax.com/documentation/cql/3.1/cql/cql_using/use_compound_keys_t.html" target="_blank">documentation</a>
- `indexes` are the index of your table. It's always an array of field names. You can read more on the <a href="http://www.datastax.com/documentation/cql/3.1/cql/ddl/ddl_primary_index_c.html" target="_blank">documentation</a>


When you instantiate a model, every field you defined in schema is automatically a property of your instances. So, you can write:

```js
john.age = 25;
console.log(john.name); //John
console.log(john.complete_name); // undefined.
```
__note__: `john.complete_name` is undefined in the newly created instance but will be populated when the instance is saved because it has a default value in schema definition

Ok, we are done with John, let's delete it:

```js
john.delete(function(err){
    //...
});
```

### A few handy tools for your model

Express cassandra exposes some node driver methods for convenience. To generate uuids e.g. in field defaults:

*   `models.uuid()`  
    returns a type 3 (random) uuid, suitable for Cassandra `uuid` fields, as a string
*   `models.timeuuid()`  
    returns a type 1 (time-based) uuid, suitable for Cassandra `timeuuid` fields, as a string


## Virtual fields

Your model could have some fields which are not saved on database. You can define them as `virtual`

```js
module.exports = {
    "fields": {
        "id"     : { "type": "uuid", "default": {"$db_function": "uuid()"} },
        "name"   : { "type": "varchar", "default": "no name provided"},
        "surname"   : { "type": "varchar", "default": "no surname provided"},
        "complete_name" : {
            "type": "varchar",
            "virtual" : {
                get: function(){return this.name + ' ' +this.surname;},
                set: function(value){
                    value = value.split(' ');
                    this.name = value[0];
                    this.surname = value[1];
                }
            }
        }
    }
}
```

A virtual field is simply defined adding a `virtual` key in field description. Virtuals can have a `get` and a `set` function, both optional (you should define at least one of them!).
`this` inside get and set functions is bound to current instance of your model.

## Validators

Every time you set a property for an instance of your model, an internal type validator checks that the value is valid. If not an error is thrown. But how to add a custom validator? You need to provide your custom validator in the schema definition. For example, if you want to check age to be a number greater than zero:

```js
module.exports = {
    //... other properties hidden for clarity
    age: {
        type : "int",
        rule : function(value){ return value > 0; }
    }
}
```

your validator must return a boolean. If someone will try to assign `john.age = -15;` an error will be thrown.
You can also provide a message for validation error in this way

```js
module.exports = {
    //... other properties hidden for clarity
    age: {
        type : "int",
        rule : {
            validator : function(value){ return value > 0; },
            message   : 'Age must be greater than 0'
        }
    }
}
```

then the error will have your message. Message can also be a function; in that case it must return a string:

```js
module.exports = {
    //... other properties hidden for clarity
    age: {
        type : "int",
        rule : {
            validator : function(value){ return value > 0; },
            message   : function(value){ return 'Age must be greater than 0. You provided '+ value; }
        }
    }
}
```

The error message will be `Age must be greater than 0. You provided -15`

Note that default values _are_ validated if defined either by value or as a javascript function. Defaults defined as DB functions, on the other hand, are never validated in the model as they are retrieved _after_ the corresponding data has entered the DB.
If you need to exclude defaults from being checked you can pass an extra flag:

```js
module.exports = {
    //... other properties hidden for clarity
    email: {
        type : "text",
        default : "<enter your email here>",
        rule : {
            validator : function(value){ /* code to check that value matches an email pattern*/ },
            ignore_default: true
        }
    }
}
```

## Querying your data

Ok, now you have a bunch of people on db. How do I retrieve them?

### Find

```js
models.instance.Person.find({name: 'John'}, function(err, people){
    if(err) throw err;
    console.log('Found ', people);
});
```

In the above example it will perform the query `SELECT * FROM person WHERE name='john'` but `find()` allows you to perform even more complex queries on cassandra.  You should be aware of how to query cassandra. Every error will be reported to you in the `err` argument, while in `people` you'll find instances of `Person`. If you don't want apollo to cast results to instances of your model you can use the `raw` option as in the following example:

```js
models.instance.Person.find({name: 'John'}, { raw: true }, function(err, people){
    //people is an array of plain objects
});
```

Let's see a complex query

```js
var query = {
    name: 'John', // stays for name='john' 
    age : { '$gt':10 }, // stays for age>10 You can also use $gte, $lt, $lte
    surname : { '$in': ['Doe','Smith'] }, //This is an IN clause
    $orderby:{'$asc' :'age'} }, //Order results by age in ascending order. Also allowed $desc and complex order like $orderby:{'$asc' : ['k1','k2'] } }
    $limit: 10 //limit result set

}
```

Note that all query clauses must be Cassandra compliant. You cannot, for example, use $in operator for a key which is not the partition key. Querying in Cassandra is very basic but could be confusing at first. Take a look at this <a href="http://mechanics.flite.com/blog/2013/11/05/breaking-down-the-cql-where-clause/" target="_blank">post</a> and, obvsiouly, at the <a href="http://www.datastax.com/documentation/cql/3.1/cql/cql_using/about_cql_c.html" target="_blank">documentation</a>

