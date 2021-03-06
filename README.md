# Connecting SQream with nodejs

## Requirements

- NodeJS 12.x or newer

## Installation

`npm install @sqream/sqreamdb`

## Create a sample connection for testing

Below is a sample nodejs file you can use to test your connection. <br />
Make sure to edit the required details, such as:
 * your server address
 * your server port
 * your username
 * your password
 * cluster: true/false
 * is_ssl: true/false
 * service: service name

```javascript
// filename: test_sqream.js
const Connection = require('@sqream/sqreamdb');

const config = {
  host: '<your server address>',
  port: 5000,
  username: '<your username>',
  password: '<your password>',
  connectDatabase: '<your database>',
};

const query1 = "SELECT 1 as test, 2 as other_test";
const sqream = new Connection(config);
sqream.execute(query1).then((data) => {
  console.log(data);
}, (err) => {
  console.error(err);
});
```

Run your file with node:

```bash
node test_sqream.js
```

On a successful connection, you should see:

```javascript
[ { test: 1, other_test: 2 } ]
```

## Config with cluster

```javascript
const config = {
  host: '<your server address>',
  port: 3108,
  username: '<your username>',
  password: '<your password>',
  connectDatabase: '<your database>',
  cluster: true,
};
```

## Secure connection

```javascript
const config = {
  host: '<your server address>',
  port: 5100,
  username: '<your username>',
  password: '<your password>',
  connectDatabase: '<your database>',
  is_ssl: true
};
```

## Network Timeout

If you run in into a timeout error message, you may need to increase the the connection's timeout. The default timeout is 180000 (3 minutes).

To increase it you need to change a property in the config like so:

```javascript
config.networkTimeout = 1800000 // 30 minutes
const sqream = new Connection(config);
```

## Network Insert

It's possible to perform insert through a prepared statement:

```javascript
const ddl = "CREATE OR REPLACE TABLE temp_table (col1 INT, col2 TEXT)";
const insertQuery = "INSERT INTO temp_table VALUES (?, ?)";
const sqream = new Connection(config);


(async () => {
  await sqream.execute(ddl);
  const inserter = await sqream.executeInsert(insertQuery);
  let insertedRows = 0;
  try {
    for (let i = 0; i < 10; i++) {
      await inserter.putRow([i, "test"]);
      insertedRows++;
    }
    await inserter.flush();
    await inserter.close();

    return insertedRows;
  } catch (e) {
    inserter.close();
    throw e;
  }
})().then((total) => {
  console.log(total, "rows inserted");
}, (err) => {
  console.error(err);
})
```

## Lazyloading

If you want to process rows without keeping them in memory, you can lazyload the rows:

```javascript
const query2 = "SELECT * FROM public.large_table";
const sqream = new Connection(config);

(async () => {
  const cursor = await sqream.executeCursor(query2);
  let count = 0;
  try {
    for await (let rows of cursor.fetchIterator(100)) { // fetch rows in chunks of 100
      count += rows.length;
    }
    await cursor.close();
    
    return count;
  } catch (e) {
    cursor.close();
    throw e;
  }
})().then((total) => {
  console.log('Total rows', total);
}, (err) => {
  console.error(err);
});
```

## Reusing connection

It is possible to execeute multiple queries with the same connection, although only one query can be executed at a time.

```javascript
const sqream = new Connection(config);

(async () => {

  const conn = await sqream.connect();
  try {
    const res1 = await conn.execute("SELECT 1");
    const res2 = await conn.execute("SELECT 2");
    const res3 = await conn.execute("SELECT 3");
    conn.disconnect();
    return {res1, res2, res3};
  } catch (err) {
    conn.disconnect();
    throw err;
  }

})().then((res) => {
  console.log('Results', res)
}, (err) => {
  console.error(err);
});
```

## Input Placeholders

Input placeholders allows to have queries that include dynamic values, like user input, by placing placeholders in the query which get replaced with escaped values that were supplied to the function.

```javascript
const sqream = new Connection(config);
const sql = "SELECT %i FROM public.%i WHERE name = %s AND num > %d AND active = %b";
sqream.execute(sql, "col1", "table2", "john's", 50, true);
```

The above sql gets converted internally to the following:


```sql
SELECT "col1" FROM public."table2" WHERE name = 'john''s' AND num > 50 AND active = TRUE
```

The different types of placeholders:

- `%i` - identifier
- `%s` - string
- `%d` - number
- `%b` - boolean


## Bigint Support

The connector supports fetching bigint values from sqream, however web applicatons encounter the following error when sending JSON with bigint values:

```
TypeError: Do not know how to serialize a BigInt
```

JSON specification does not support bitint values even though they are supported by javascript engines, and you will run into this error when using `JSON.stringify`.

To resolve this issue, objects with bigint values must be converted to string when serializing, and converted back after deserializing. The following will alter the BitInt prototype to convert the number to string when JSON stringified:

```javascript
BigInt.prototype.toJSON = function () { return this.toString(); }

const rows = [{test: 1n}];
console.log(JSON.stringify(rows)); // [{"test": "1"}]
```

## Numeric (BigDecimal) Suppport

Node does not yet support BigDecimal primitives, therfore the connector will provide Numeric types as an object comprising of a BigInt number and a scale number. This object will be converted to a decimal string when it is JSON stringified.

For performing math operations on Numeric types, consider using BigDecimal/BigInt centric libraries on npm.

## Limitation

The node application which utilizes the sqream connector should consider the heap size node configuration.

When processing large datasets, it is recommended to increase the application heap size with the `--max_old_space_size` node run flag:

```
node --max_old_space_size={heapSize} my-application.js
```

In addition, the following error mentions that the heap size configured is not sufficient:

`FATAL ERROR: CALL_AND_RETRY_LAST Allocation failed - JavaScript heap out of memory`
