# neo4cl - a CL library for interacting with Neo4J

## DEPRECATED VERSION

This branch is no longer actively maintained, but may be useful to somebody using an older version of Neo4j.

As always, pull-requests are welcome.


## Short description:

From the [Neo4J website](http://neo4j.com/): "Neo4j is a highly scalable native graph database."

Graph databases emphasise the relationships between things, and the information to be found there, in contrast to the relational focus on the things themselves. They are typically much more flexible in their definitions of things than RDBMSes, which fits well with Lisp's fluid approach to functionality.

Neo4J is a very popular graph database that scales well. It implements a property graph, which means you can assign attributes to relationships as well as to nodes, and its query language (Cypher) is very expressive. It's an excellent transactional database which satisfies the ACID model, in contrast to something like RDF, which is better suited to data warehousing where responsiveness is traded off for more semantic richness.

Neo4CL is very simple library that sends Cypher queries to a Neo4J server, and decodes the responses into something useful for processing in CL. The queries and their responses take the form of alists. This library aims at compliance with Neo4J 3.0 via the HTTP API, based on the documentation in the [Neo4J developer manual](http://neo4j.com/docs/developer-manual/current/#http-api-index), and is intended as something to build applications on, more than for interactive use.


### What it does

- transforms alists into queries, sends them to a Neo4J server, and returns its responses as alists
- very basic error reporting


### What it doesn't do

- anything involving Cypher itself. It forwards the query string, and assumes the server will know what to do with it.
- the BOLT protocol. This may be added in the future.
- any kind of object-graph mapping.


## Compatibility

### Neo4j versions

3.5.3


### What it runs on

It's developed and tested on SBCL.

The maintainer hasn't had the time to test it on other implementations, but well-formatted pull-requests are always welcome.


### Dependencies

All available via Quicklisp:

- drakma
- cl-ppcre
- cl-json
- flexi-streams
- cl-base64


### How it works

It's organised around a `neo4j-rest-server` object, which holds the details needed for connecting to a Neo4J server. Its initargs and defaults are:

- :protocol - default = "http", but can be whatever Drakma will accept
- :hostname - default = "localhost"
- :port - default = 7474
- :dbuser - default = "neo4j"
- :dbpasswd - default = "neo4j"

It comes with a basic test-suite, in the package `neo4cl-test`, which requires the FiveAM framework.


## Example usage:

We'll assume it's a default installation of Neo4J, so it's listening on `http://localhost:7474`, and the username and password are both 'neo4j'.
```
(defvar *server*
  (make-instance 'neo4cl:neo4j-rest-server))

;; First, we change the password.
;; It changes the password stored in *server* for you, so it continues to "just work."
(neo4cl:change-password *server* "foobar")

;; Create some bloke called Andre
(neo4cl:neo4j-transaction
 *server*
 `((:STATEMENTS
     ((:STATEMENT . "CREATE (n:Person { properties })")
      (:PARAMETERS .
       ((:properties . ((:name . "Andre")))))))))

;; This should return the following:
((:RESULTS ((:COLUMNS) (:DATA))) (:ERRORS))
200
"OK"

;; Is he there?
(neo4cl:neo4j-transaction
  *server*
  `((:STATEMENTS
      ((:STATEMENT . "MATCH (x:Person {name: 'Andre'}) RETURN x.name")))))

;; The result should look so:
((:RESULTS ((:COLUMNS "x.name") (:DATA ((:ROW "Andre") (:META NIL)))))
 (:ERRORS))
200
"OK"

;; We're bored with Andre; get rid of him
(neo4cl:neo4j-transaction
  *server*
  `((:STATEMENTS
      ((:STATEMENT . "MATCH (x:Person {name: 'Andre'}) DELETE x")))))

;; Finally, we should see this in response:
((:RESULTS ((:COLUMNS) (:DATA))) (:ERRORS))
200
"OK"
```

## Error/condition handling

Three errors and one non-error condition are defined, following the [Neo4J classifications](http://neo4j.com/docs/developer-manual/current/reference/#status-codes):

- client-error
- transient-error
- database-error
- client-notification

They're further broken down into Category and Title, by simply splitting on the dot, and the Message is also included.

E.g, `Neo.ClientError.Schema.ConstraintValidationFailed` would be raised as
```
(error neo4cl:client-error
    :category "Schema"
    :title "ConstraintValidationFailed"
    :message "Node 126 already exists with label routers and property \"uid\"=[whitesands]")
```

The symbols `category`, `title` and `message` are exported so these fields can be read.
