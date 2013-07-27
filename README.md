dblite
======
a zero hassle wrapper for sqlite


### The What And The Why
I've created `dblite` module because there's still not a simple and straight forward or standard way to have [sqlite](http://www.sqlite.org) in [node.js](http://nodejs.org) without requiring to re-compile, re-build, download sources a part or install dependencies instead of simply `apt-get install sqlite3` or `pacman -S sqlite` in your \*nix system.

`dblite` has been created with portability, simplicity, and reasonable performance for **embedded Hardware** such [Raspberry Pi](http://www.raspberrypi.org) and [Cubieboard](http://cubieboard.org) in mind.

Generally speaking all linux based distributions like [Arch Linux](https://www.archlinux.org), where is not always that easy to `node-gyp` a module and add dependencies that work, can now use this battle tested wrap and perform basic to advanced sqlite operations.


### API
Right now a created `db` has 3 methods: `.query()`, `.lastRowID()`, and `.close()`.

The `.lastRowID(table, callback(rowid))` helper simplifies a common operation with SQL tables after inserts, handful as shortcut for the following query:
`SELECT ROWID FROM `table` ORDER BY ROWID DESC LIMIT 1`.

The method `.close()` does exactly what it suggests: it closes the database connection.
Please note that it is **not possible to perform other operations once it has been closed**.

Being an `EventEmitter` instance, the database variable will be notified with the `close` listener, if any.


### Understanding The .query() Method
The main role in this module is played by the `dblite.query()` method, a method rich in overloads all with perfect and natural meaning.

The amount of parameters goes from one to four, left to right, where left is the input going through the right which is the eventual output.

All parameters are optionals except the SQL one, where if non specified, `console.log(arguments)` will be used as implicit callback if none has been specified as last parameters used when `.query()` was invoked.


### dblite.query() Possible Combinations
```javascript
dblite.query(SQL)
dblite.query(SQL, callback:Function)
dblite.query(SQL, params:Array|Object)
dblite.query(SQL, fields:Array|Object)
dblite.query(SQL, params:Array|Object, callback:Function)
dblite.query(SQL, fields:Array|Object, callback:Function)
dblite.query(SQL, params:Array|Object, fields:Array|Object)
dblite.query(SQL, params:Array|Object, fields:Array|Object, callback:Function)
```
All above combinations are [tested properly in this file](test/dblite.js) together with many other tests able to make `dblite` robust enough and ready to be used.

Please note how `params` is always before `fields` and/or `callback` if `fields` is missing, just as reminder that order is left to right accordingly with what are trying to do.

Following detailed explanation per each parameter.

#### The SQL:string
This string [accepts any query understood by SQLite](http://www.sqlite.org/lang.html) plus it accepts all commands that regular SQLite shell would accept such `.databases`, `.tables`, `.show` and all others passing through the specified `info` listener, if any.
```javascript
var dblite = require('dblite'),
    db = dblite('./db.sqlite');

db.on('info', function (data) {
  // generic info, not a SELECT result
  // neither a PRAGMA one - just commands
  console.log(String(data));
});

db.query('.show');
/* will console.log something like:

     echo: off
  explain: off
  headers: off
     mode: csv
nullvalue: ""
   output: stdout
separator: ","
    stats: off
    width:
*/

// normal query
db.query('CREATE TABLE IF NOT EXISTS test (id INTEGER PRIMARY KEY, value TEXT)');
db.query('INSERT INTO test VALUES(null, ?)', ['some text']);
db.query('SELECT * FROM test');
// will implicitly log the following
// [ [ '1', 'some text' ] ]
```

#### The params:Array|Object
If the SQL string is **not a command** and **contains special chars** such `?`, `:key`, `$key`, or `@key` properties, these will be replaced accordingly with the `params` `Array` or `Object` that, in this case, MUST be present.
```javascript
// params as Array
db.query('SELECT * FROM test WHERE id = ?', [1]);

// params as Object
db.query('SELECT * FROM test WHERE id = :id', {id:1});
// same as
db.query('SELECT * FROM test WHERE id = $id', {id:1});
// same as
db.query('SELECT * FROM test WHERE id = @id', {id:1});
```

#### The fields:Array|Object
By default, results are returned as an `Array` where all rows are the outer `Array` and each single row is another `Array`.
```javascript
db.query('SELECT * FROM test');
// will log something like:
[
  [ '1', 'some text' ],     // row1
  [ '2', 'something else' ] // rowN
]
```
If we specify a fields parameter we can have each row represented by an object, instead of an array.
```javascript
// same query using fields as Array
db.query('SELECT * FROM test', ['key', 'value']);
// will log something like:
[
  {key: '1', value: 'some text'},     // row1
  {key: '2', value: 'something else'} // rowN
]
```

#### Parsing Through The fields:Object
[SQLite Datatypes](http://www.sqlite.org/datatype3.html) are different from JavaScript plus SQLite works via affinity.
This module also parses sqlite3 output which is **always a string** and as string every result will always be returned **unless** we specify `fields` parameter as object, suggesting validation per each field.
```javascript
// same query using fields as Object
db.query('SELECT * FROM test', {
  key: Number,
  value: String
});
// note the key as integer!
[
  {key: 1, value: 'some text'},     // row1
  {key: 2, value: 'something else'} // rowN
]
```
More complex validators/transformers can be passed without problems:
```javascript
// same query using fields as Object
db.query('SELECT * FROM `table.users`', {
  id: Number,
  name: String,
  adult: Boolean,
  skills: JSON.parse,
  birthday: Date,
  cube: function (fieldValue) {
    return fieldValue * 3;
  }
});
```

#### The params:Array|Object AND The fields:Array|Object
Not a surprise we can combine both params, using the left to right order input to output so **params first**!
```javascript
// same query using params AND fields
db.query('SELECT * FROM test WHERE id = :id', {
  id: 1
},{
  key: Number,
  value: String
});

// same as...
db.query('SELECT * FROM test WHERE id = ?', [1], ['key', 'value']);
// same as...
db.query('SELECT * FROM test WHERE id = ?', [1], {
  key: Number,
  value: String
});
// same as...
db.query('SELECT * FROM test WHERE id = :id', {
  id: 1
}, [
  'key', 'value'
]);
```

#### The callback:Function
When a `SELECT` or a `PRAGMA` `SQL` is executed the module puts itself in a *waiting for results* state.
As soon as results are fully pushed to the output the module parses this result and send it to the specified callback.

The callback is **always the last specified parameter**, if any, or the implicit equivalent of `console.log.bind(console)`.
Latter case is simply helpful to operate directly via `node` **console** and see results without bothering writing a callback each `.query()` call.


### Raspberry Pi Performance
This is the output generated after a `make test` call in this repo folder within Arch Linux for RPi.
```
npm test

> dblite@0.1.2 test /home/dblite
> node test/.test.js

/home/dblite/dblite.test.sqlite
------------------------------
main
passes: 1, fails: 0, errors: 0
------------------------------
create table if not exists
passes: 1, fails: 0, errors: 0
------------------------------
100 sequential inserts
100 records in 3.067 seconds
passes: 1, fails: 0, errors: 0
------------------------------
1 transaction with 100 inserts
200 records in 0.178 seconds
passes: 1, fails: 0, errors: 0
------------------------------
auto escape
passes: 1, fails: 0, errors: 0
------------------------------
auto field
fetched 201 rows as objects in 0.029 seconds
passes: 1, fails: 0, errors: 0
------------------------------
auto parsing field
fetched 201 rows as normalized objects in 0.038 seconds
passes: 1, fails: 0, errors: 0
------------------------------
many selects at once
different selects in 0.608 seconds
passes: 1, fails: 0, errors: 0
------------------------------
dblite.query() arguments
[ [ '1' ] ]
[ [ '2' ] ]
[ { id: 1 } ]
[ { id: 2 } ]
passes: 5, fails: 0, errors: 0
------------------------------
utf-8
¥ · £ · € · $ · ¢ · ₡ · ₢ · ₣ · ₤ · ₥ · ₦ · ₧ · ₨ · ₩ · ₪ · ₫ · ₭ · ₮ · ₯ · ₹
passes: 1, fails: 0, errors: 0
------------------------------
erease file
passes: 1, fails: 0, errors: 0

------------------------------
          15 Passes
------------------------------
```
If an SD card can do this good, I guess any other environment should not have problems here ;-)