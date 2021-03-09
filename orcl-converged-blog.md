## orcl-converged-blog
```
SINGLE-PURPOSE DATABASES
```

```bash
1 # INSTALL NPM MODULES
2 npm i mongo
3 npm i neo4j-driver
4 npm i node-firebird
```

```jsx
// DEMONSTRATION PURPOSES ONLY
// JS
// ...
1   // CONNECT TO MONGODB CLIENT
2   let mongoClient = require('mongodb').MongoClient; // includes Mongodb module
3       MongoClient.connect(mongoURL,mongoOptions, (err, client) => {
4          if (err) {/* ERROR HANDLING */} else {/* CONNECTED TO MONGODB */}
5      });
6   // CONNECT TO NEO4J
7   const neo4j = require('neo4j-driver'); //include Neo4j module
8   const driver = neo4j.driver(uri, neo4j.auth.basic(user, password));
9   const neo4jSession = driver.session();
10      try {
11        const result = await neo4jSession.run(/* CONNECTED TO NEO4J */);
12      } finally {
13        await session.close();
14      }
15  // CONNECT TO FIREBIRD CLIENT
16  var Firebird = require('node-firebird'); // include firebird module
17  Firebird.attach(options, function(err, firebirdDB) {
18    if (err) {/* ERROR HANDLING */} else {/* CONNECTED TO FIREBIRD */}  
19  });
// ...
// DEMONSTRATION PURPOSES ONLY
```

```
ORACLE DATABASE (CONVERGED)
```

```bash
1 # INSTALL NPM MODULES
2 npm i oracledb
3
4
```

```jsx
// DEMONSTRATION PURPOSES ONLY
// JS
// ...
1   // CONNECT TO ORACLE DATABASE CLIENT
2   const oracledb = require('oracledb');
3   function oracleDbConnect(dbOptions) {
4       return new Promise(async (resolve, reject) => {
5           try {
6               await oracledb.getConnection(dbOptions).then(conn => {
7                   resolve(/* CONNECTED TO ORACLE DB */);
8               });
9           } catch (err) {     
10              reject(err);  
11          } 
12      });
13  }
14
15
16
17
18
19
// ...
// DEMONSTRATION PURPOSES ONLY
```

```
SINGLE-PURPOSE DATABASES
```

```jsx
// DEMONSTRATION PURPOSES ONLY
// JS
// Use Case: Host-a-bnb (vacation rental company) wants to build a dashboard that visualizes session data of users who booked bnbs at a
// given date with long-term friends (> 5 year) and needs to pull out the session data to do it. 
// The query below will pull the session data using predefined variables 'VAR_USER_ID', 'VAR_BOOKING_DT', and 'VAR_FRIENDSHIP_LENGTH'.
// ...
// Step 1.a: Let's apporach this problem by querying and pulling all the 'firebirdBookingIds' the user had on the 'VAR_BOOKING_DT' from Firdbird
firebirdBookingIds = [];
Firebird.attach(options, function(err, firebirdDB) {
    if (err) {/* ERROR HANDLING */} else {/* CONNECTED TO FIREBIRD */}  
    firebirdDB.query('SELECT BOOKING_ID FROM BOOKINGS WHERE USER_ID = ? AND BOOKING_DATE = ?', 
    [VAR_USER_ID, VAR_BOOKING_DT], (err, result) => 
    {
        if (err) throw err;
        // Step 1.b: Clean 'firebirdBookingIds' for use in next step (think ETL, what a pain!)
        firebirdBookingIds = transformResult(result) {/* ... */};
        firebirdDB.detach();
    });
})
// Step 2.a: Then, using the 'firebirdBookingIds', query which bookings included friends with 'VAR_FRIENDSHIP_LENGTH' > 5 years from Neo4j
neo4jFilteredBookingIds = [];
session.run(
    'MATCH (u:Person)-[:IS_FRIENDS_WITH]-(friends) WHERE u.id = $user_id AND friends.friendship_length > $fLength' + 
    'RETURN FILTER(x in friends.booking_ids WHERE x IN $userBookingIds)',
    { userId: VAR_USER_ID,
      fLength: VAR_FRIENDSHIP_LENGTH,
      userBookingIds: firebirdBookingIds
    }).then(result => {
        // Step 2.b: Clean 'neo4jFilteredBookingIds' for use in next step (think ETL, what a pain!)
        neo4jFilteredBookingIds = transformResult(result.records) {/* ... */};
    }).catch (error => {
        console.log(error);
    }).then(() => session.close())
// Step 3: Finally, using 'neo4jFilteredBookingIds', return the session history from MongoDB
finalResult = [];   
MongoClient.connect(url,mongoOptions, (err, client) => {
    if(err) console.log(err);
    const db = client.db(/* DB NAME */);
    const sessionHistoryCollection = db.collection(/* COLLECTION_SESSION_HISTORY */);
    sessionHistoryCollection.find({"booking_id": {"$in": [neo4jFilteredBookingIds]}},{"session_data": 1}).toArray((err, items) => {
        if (err) console.log(err);
            finalResult = items;
    })
});
console.log(finalResults); // Print session history
// ...
// DEMONSTRATION PURPOSES ONLY
```



```
ORACLE DATABASE (CONVERGED)
```

```jsx
// DEMONSTRATION PURPOSES ONLY
// JS
// Use Case: Host-a-bnb (vacation rental company) wants to build a dashboard that visualizes session data of users who booked bnbs at a
// given date with long-term friends (> 5 year) and needs to pull out the session data to do it.
// The query below will pull the session data using predefined variables 'VAR_USER_ID', 'VAR_BOOKING_DT', and 'VAR_FRIENDSHIP_LENGTH'.
// ...
finalResult = [];
oracledb.execute(
    `with booking_cte as ( -- Create temporary result set of needed information from *relational data* on the date 'VAR_BOOKING_DT'
    SELECT BOOKING_ID from BOOKINGS WHERE USER_ID = :1 AND BOOKING_DATE = to_date( :2 , 'MM/DD/YY')
), relationship_cte as ( -- Create temporary result set of needed information from *graph data* where frienships with 'VAR_USER_ID' exist
    SELECT USER_ID, FRIEND_ID, RELATIONSHIP_TYPE, FRIENDSHIP_LENGTH, BOOKING_ID FROM RELATIONSHIPS
    START WITH USER_ID = :3
    CONNECT BY NOCYCLE PRIOR
        USER_ID = FRIEND_ID
), session_history_cte as ( -- Create temporary result set of needed information from *key/value data*
SELECT jt.BOOKING_ID, jt.SESSION_DATA from SESSION_HISTORY,
    json_table( json_doc, '$'
        columns (
            nested path '$.history_of_bookings[*]' columns (
                booking_id number path '$.booking_id',
                session_data FORMAT JSON path '$.session_data'
        ))
    ) jt
)SELECT s.SESSION_DATA FROM booking_cte b 
    JOIN relationship_cte r ON r.BOOKING_ID = b.BOOKING_ID
    JOIN session_history_cte s ON s.BOOKING_ID = r.BOOKING_ID
WHERE r.FRIENDSHIP_LENGTH > :4
-- Finally, query to pull 'SESSION_DATA' from the 'BOOKING_ID's where users booked bnbs on the date 'VAR_BOOKING_DT' with 
-- long term friends ('r.FRIENDSHIP_LENGTH' > VAR_FRIENDSHIP_LENGTH [5 years])`,
    [VAR_USER_ID, VAR_BOOKING_DATE, VAR_USER_ID, VAR_FRIENDSHIP_LENGTH], {outFormat: oracledb.ARRAY}
    ).then(result => {
        finalResult = result;
    }).catch(err => {
        console.log(err);
   });
console.log(finalResult); // Print session history
// ...
// DEMONSTRATION PURPOSES ONLY
```

