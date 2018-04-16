# neo4j-clj
Clojuresque client to Neo4j database, based upon the bolt protocol.

[![Clojars Project](https://img.shields.io/clojars/v/gorillalabs/neo4j-clj.svg)](https://clojars.org/gorillalabs/neo4j-clj)
[![Build Status](https://travis-ci.org/gorillalabs/neo4j-clj.svg)](https://travis-ci.org/gorillalabs/neo4j-clj)
[![Dependencies Status](https://versions.deps.co/gorillalabs/neo4j-clj/status.svg)](https://versions.deps.co/gorillalabs/neo4j-clj)
[![Downloads](https://versions.deps.co/gorillalabs/neo4j-clj/downloads.svg)](https://versions.deps.co/gorillalabs/neo4j-clj)

neo4j-clj is a clojure client library to the [Neo4j graph database](https://neo4j.com/),
relying on the [Bolt protocol](https://boltprotocol.org/).


## Features

This library provides a clojuresque way to deal with connections, sessions, transactions
and [Cypher](https://www.opencypher.org/) queries.


## Status

neo4j-clj is in active use at our own projects,
but it's not a feature complete client library in every possible sense.

You might choose to issue new feature requests,
or clone the repo to add the feature and create a PR.

If you'd like to ask a question, you might want to
join our [Gorillalabs Slack Group](https://gorillalabs.slack.com).
Ask right away in the #neo4j-clj channel.


We appreciate any help on the open source projects we provide.


## Example usage

Throughout the examples, we assume you're having a Neo4j instance up and running.
See our [Neo4j survival guide](docs/neo4j) for help on that.

You can clone our repository and run the [example](example/) for yourself.


```clojure
(ns example.core
  (:require [neo4j-clj.core :as db]))

;; first of all, connect to a Neo4j instance using URL, user and password credentials.
;; Remember not to check in credentials into source code repositories, but use environment variables
;; instead.

(def local-db
  (db/connect "bolt://localhost:7687" "neo4j" "password"))

;; We're big fans of using Strings to represent Cypher queries, and not wrap Cypher into some
;; other data structure to make things more complicated then necessary. So simply defquery your query...

(db/defquery create-user
  "CREATE (u:user $user)")
  
;; ... and you'll get a function `create-user` to call with a session and the parameters. See below.  

;; Define any other queries you'll need. I'd suggest to keep all the Cypher queries in a separate namespace,
;; but hey, that's up to you.

(db/defquery get-all-users
  "MATCH (u:user) RETURN u as user")

(defn -main
  "Example usage of neo4j-clj"
  [& args]

  ;; Using a session
  (with-open [session (db/get-session local-db)]
    (create-user session {:user {:first-name "Luke" :last-name "Skywalker"}}))

  ;; Using a transaction
  (db/with-transaction local-db tx
    (get-all-users tx)) ;; => ({:user {:first-name "Luke", :last-name "Skywalker"}}))
```

## In-depth look

### Connection

First of all, you need to connect to the database.

```clojure
(db/connect "bolt://localhost:7687" "neo4j" "password")
```

### Session

Everything on the Bolt protocol is organized in a session. neo4j-clj uses the
standard `with-open` to handle lexical-scoped sessions.

```clojure
(with-open [session (db/get-session local-db)]
  ;; ... use session here
  )
```

### Transaction

neo4j-clj can also handle transactions (on top of sessions) by utilizing a 
`with-transaction`-macro.

```clojure
(db/with-transaction local-db tx
   ;; run queries within transaction
)
```

`local-db` is the Neo4j instance you're connected to.
`tx` is the symbol the transaction is bound to. You might use
the `tx` transaction for query functions. 

You do not need to have everything inside a `with-transaction` block,
as all statements in a session are wrapped in transactions anyhow
(see [Sessions and Transactions](https://neo4j.com/docs/developer-manual/current/drivers/sessions-transactions/)). 

You fail transactions by throwing an Exception.

### Cypher queries

You create new queries using `defquery`. We strongly belief that using Cypher
as a String directly in your code comes with a couple of advantages.

Both paramaters and return values are transformed from Clojure datastructures
into Neo4j types. That transformation is done transparently, so you won't have
to deal with that.

#### Parameters

For some / most queries you need to supply some parameters. Just include a
variable into your Cypher query using the `$` notation or the `{}` notation,
and provide a Clojure map of variables to the query:


```clojure
(db/defquery users-by-age "MATCH (u:User {age: $age}) RETURN u as user")
(users-by-age tx {:age 42})
```

is equivalent to

```clojure
(db/defquery users-by-age "MATCH (u:User {age: {age}}) RETURN u as user")
(users-by-age tx {:age 42})
```

A query will always return a collection. Here, it will return `({:user {...}})`,
i.e. a collection of maps with the user-key populated.

Think of it as a map per result row.

You can have more than one parameter in your map:

```clojure
(db/defquery create-user "CREATE (u:User {name: $name, age: $age})")
(create-user tx {:name "..." :age 42})
```

And you can also nest paramters:

```clojure
(db/defquery create-user "CREATE (u:User $user)")
(create-user tx {:user {:name "..." :age 42}})
```
and un-nest them as necessary:

```clojure
(db/defquery create-user "CREATE (u:User {name : {user}.name})")
(create-user tx {:user {:name "..." :age 42}})
```



<!--

```clojure
(db/defquery get-users "MATCH (u:User) RETURN u as user")
(get-users tx)
;; => ({:user {...}})

;; Extracted parameters
(db/defquery create-user "CREATE (u:User {name: $name, age: $age})")
(create-user tx {:name "..." :age 42})

(db/defquery get-users "MATCH (u:User) RETURN u.name as name, u.age as age")
(get-users tx)
;; => ({:name "..." :age 42}, ...)
```

#### Return values

-->


## Caveats

One thing you cannot do: Neo4j cannot cope with dashes really well (you need to escape them),
so the Clojure kebab-case style is not really acceptable.


## Testing

The test semantics are the same. Just use

```clojure
(def test-db
  (neo4j/create-in-memory-connection))
;; instead of (neo4j/connect url user password)
```

## Joplin integration

neo4j-clj comes equipped with support for [Joplin](https://github.com/juxt/joplin)
for datastore migration and seeding.

As we do not force our users into Joplin dependencies, you have to add [joplin.core "0.3.10"]
to your projects dependencies yourself.

