# Neo4j-core/Neo4j-wrapper 3.0 DRAFT

## Version 3.0 Specification

The neo4j-core version 3.0 uses the java Neo4j 2.0 jars and takes advantage of the new label feature in order to do mappings
  between `Neo4j::Node` (java objects) and your own ruby classes.

The code base for the 3.0 should be smaller and simpler to maintain because there is less work to be done in the
Ruby layer but also by removing features that are too complex or not that useful.

The neo4j-wrapper source code is included in this git repo until the refactoring has stabilized.
The old source code for neo4j-core is also included (lib.old). The old source code might later on be copied into the
 3.0 source code (the lib folder).

The neo4j-core gem will work for both the embedded Neo4j API and the server api.
That means that neo4j.rb will work on any Ruby implementation and not just JRuby. This is under investigation !
It's possible that some features for the Neo4j.rb 2.0 will not be available in the 3.0 version since it has to work
 with both the Neo4j server and Neo4j embedded APIs.

Since neo4j-core provides one unified API to both the server end embedded neo4j database the neo4j-wrapper and neo4j
gems will also work with server and embedded neo4j databases.

New features:

* investigate: neo4j-core should provide one API to both the Embedded database and the Neo4j Server
* investigate: support for both Cypher and REST implementation of Neo4j::Node and Neo4j::Relationship API.
* auto commit is default (neo4j-core)

Removed features:

* auto start of the database (neo4j-core)
* wrapping of Neo4j::Relationship java objects but there will be a work around (neo4j-wrapper)
* traversals (the outgoing/incoming/both methods) moves to a new gem, neo4j-traversal.

Changes:

* `Neo4j::Node.create` now creates a node instead of `Neo4j::Node.new`

### Neo4j-core specs

Example of index using labels and the auto commit.

```ruby
  db = Neo4j::Database.new('hej', auto_commit: true)  # ?
  db.start

  red = Label.new(:red)
  red.index(:name)

  # notice, label argument can be both Label objects or string/symbols.
  node = Node.create({name: 'andreas'}, red, :green)
  puts "Created node #{node[:name]} with labels #{node.labels.map(&:name).join(', ')}"

  # Find nodes using the label
  red.find_nodes(:name, "andreas").each do |node|
    puts "FOUND #{node[:name]} class #{node.class} with labels #{node.labels.map(&:name).join(', ')}"
  end
```

All method prefixed with `_` gives direct access to the java layer/rest layer.


### Neo4j Embedded and Neo4j Server support

Investigate this:

Using the Embedded database:

```ruby
  db = Neo4j::Embedded::Database.new('db/location')
  db.start
  # Notice, auto commit is by default enabled
  node = Neo4j::Node.create(name: 'foo')
```

Using the Server database:

```ruby
  Neo4j::Server::RestDatabase.new('http://localhost:7474/db/dat') # only auto commit is allowed
  node = Neo4j::Node.create(name: 'foo')
```

Using the Server database with the cypher language:


```ruby
  db = Neo4j::Server::CypherDatabase.new('http://localhost:7474/db/dat')
  db.start
  node = Neo4j::Node.create(name: 'foo')

  # With transactions
  db = Neo4j::Server::CypherDatabase.new('http://localhost:7474/db/data', auto_commit: false)
  db.start

  Neo4j::Transaction.run do
    node = Neo4j::Node.create(name: 'foo')
  end
```

The Neo4j::Database contains the reference to the default database used (Neo4j::Database.instance) which is the
first database created. This is used for example when a database is not specified, e.g. `node[:name] = 'me'`
It is also possible to use several databases at the same time, e.g.

```ruby
  db1 = Neo4j::Embedded::Database.new('location', auto_commit: true).start
  node = Neo4j::Node.create(name: 'foo', db1)

  db2 = Neo4j::Server::Database.new('http:://end.point', auto_commit: true).start
  node = Neo4j::Node.create(name: 'foo', db2)
```

### Relationship

How to create a relationship between node 1 and node 2 with one property

```ruby
n1 = Neo4j::Node.create
n2 = Neo4j::Node.create
rel = n1.create_rel(:knows, since: 1994).to(n2)
# or
rel = n1.create_rel(:knows, since: 1994).to(n2)
```

How to create a relationship between node 1 and a new node with one property

```ruby
n1 = Neo4j::Node.create
n2 = Neo4j::Node.create
rel = n1.create_rel(:knows, since: 1994).to(name: 'jimmy')
```

Finding relationships

```ruby
rels = n1.rels(:outgoing, :know).to_a
rel = n1.rel(:outgoing, :best_friend) # expect only one relationship
```


## Implementation:

No state is cached in the neo4j-core (e.g. neo4j properties).

The public `Neo4j::Node` classes is abstract and provides a common API/docs for both the embedded and
  neo4j server.

The Neo4j::Embedded and Neo4j::Server modules contains drivers for classes like the `Neo4j::Node`.
This is implemented something like this:

```ruby
  class Neo4j::Node
    # YARD docs
    def [](key)
      get_property(key) # abstract method - impl using either HTTP or Java API
    end


    def self.create(props, db=Neo4j::Database.instance)
     db.create_node(props)
    end
  end
```

Both implementation use the same E2E specs.

### Neo4j-wrapper specs

Example of mapping a Neo4j::Node java object to your own class.

```ruby
  # will use Neo4j label 'Person'
  class Person
    include Neo4j::NodeMixin
  end
```

Example of mapping the Baaz ruby class to Neo4j labels 'Foo', 'Bar' and 'Baaz'

```ruby
  module Foo
    def self.label_name
       "Foo" # specify the label for this module
    end
  end

  module Bar
    extend Neo4j::Wrapper::LabelIndex # to make it possible to search using this module (?)
    index :stuff # (?)
  end

  class Baaz
    include Foo
    include Bar
    include Neo4j::NodeMixin
  end

  Bar.find_nodes(...) # can find Baaz object but also other objects including the Bar mixin.
```

Example of inheritance.

```ruby
  # will only use the Vehicle label
  class Vehicle
    include Neo4j::NodeMixin
  end

  # will use both Car and Vehicle labels
  class Car < Vehicle
  end
```

### Testing

The testing will be using much more mocking.

* The `unit` rspec folder only contains testing for one Ruby module. All other modules should be mocked.
* The `integration` rspec folder contains testing for two or more modules but mocks the neo4j database access.
* The `e2e` rspec folder for use the real database (or Neo4j's ImpermanentDatabase (todo))
* The `shared_examples` common specs for different types of databases


== The public API

* `Neo4j::Node` The Java Neo4j Node

* {Neo4j::Relationship} The Java Relationship

* {Neo4j::Database} The (default) Database

* {Neo4j::Embedded::Database} - good name ?

* {Neo4j::Server::RestDatabase}

* {Neo4j::Server::CypherDatabase}

* {Neo4j::Cypher} Cypher Query DSL, see {Neo4j Wiki}[https://github.com/andreasronge/neo4j/wiki/Neo4j%3A%3ACore-Cypher]

* {Neo4j::Algo} Included algorithms, like shortest path

### License
* Neo4j.rb - MIT, see the LICENSE file http://github.com/andreasronge/neo4j-core/tree/master/LICENSE.
* Lucene -  Apache, see http://lucene.apache.org/java/docs/features.html
* \Neo4j - Dual free software/commercial license, see http://neo4j.org/

Notice there are different license for the neo4j-community, neo4j-advanced and neo4j-enterprise jar gems.
Only the neo4j-community gem is by default required.
