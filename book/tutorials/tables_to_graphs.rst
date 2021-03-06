Tables to Graphs
================

For anyone used to `relational databases <http://en.wikipedia.org/wiki/Relational_database_management_system>`_,
the notion of a `graph <http://en.wikipedia.org/wiki/Graph_%28mathematics%29>`_
database may seem mystifying, inappropriate or just downright wrong. This
article attempts to demonstrate how to begin the transition from a relational
database philosophy into the graphic environment provided by
`Neo4j <http://neo4j.org/>`_

Static Structure to Dynamic Structure
-------------------------------------

Tables are the cornerstone of relational databases, dictating a formal
structure over your data model. These structures serve not only to validate
and constrain incoming data but also to provide an abstract model of the
domain in question, whether that be business processes or something more
esoteric. However, challenges can arise when this data structure is required
to evolve. Often new fields will be tacked onto the end of tables and some
fields may be retired or have their purpose changed, as the effort required to
remodel can be disproportionate to the benefit gained from doing so. The same
framework that provides these structural benefits can also act as a barrier to
change.

Having a much looser approach to structure, Neo4j provides an environment in
which a data model can not only evolve but can vary internally as required.
The values used to represent a person, for example, can be as many or as few
as necessary for each person represented. On top of this, the data types used
may vary - there are no constraints of consistency.

However, this is not to suggest that a complete lack of internal consistency
would make a good design. Instead, structural rules and constraints required
by the data can either be deferred to the application layer or kept within the
data layer as required. The major gain being flexibility of design.

Records to Nodes
----------------

Tables are of course comprised of a number of records, each holding a
representation of an entity instance. The closest equivalent to a record in
Neo4j is a *node*. At its simplest level, a node is simply a map of properties
in the form of key:value pairs. Once again taking the example of a person, one
might choose to model such an entity as follows::

    {
        "given_name"    : "Alice",
        "family_name"   : "Allison",
        "date_of_birth" : "1986-12-25"
    }

Just as such a record could be inserted into a table, a node could be created
to store this information. In py2neo, this is as simple as::

    from py2neo import neo4j
    graph_db = neo4j.GraphDatabaseService("http://localhost:7474/db/data/")
    alice = graph_db.create_node({
        "given_name"    : "Alice",
        "family_name"   : "Allison",
        "date_of_birth" : "1986-12-25"
    })

A second person could then be modelled with a slightly varied set of
properties::

    bob = graph_db.create_node({
        "name"          : "Bob Robertson",
        "date_of_birth" : "1981-07-01",
        "occupation"    : "Hacker"
    })

As you may have noticed, nodes in Neo4j are typeless data structures. There is nothing specific which states that the entities we see above are people, cars, items of clothing or anything else; they are simply sets of properties. There is nothing to restrict you from adding a "type" property to every node created but there is nothing forcing you to do so either.

Working with such free-form data can be quite unnerving at first, especially if one is used to modelling within a more rigid framework. The difference can also be very liberating, however, making use instead of notions such as [[http://en.wikipedia.org/wiki/Duck_typing|duck typing]]. The same contrast can be seen when switching between different programming languages such as Java and Python.

Tables to Relationships
-----------------------

Clearly, individual records are little use on their own and, within a relational database, tables also serve to group similar items. When grouped, these may be queried and retrieved according to common properties. Neo4j can also model such groupings by using //relationships//.

Alongside nodes, relationships are the other fundamental entity within Neo4j. Mathematically known as //edges//, one or more relationships can be laid between any two nodes within a graph to model a particular connection. Unlike nodes, relationships within Neo4j have a mandatory type and direction. This means that connecting A to B is different from connecting B to A. Doing so in py2neo is illustrated below:

    # create a relationship from Alice to Bob
    # (Alice)-[:KNOWS]->(Bob)
    alice_bob = alice.create_relationship_to(bob, "KNOWS")

    # create a relationship from Bob to Alice
    # (Alice)<-[:KNOWS]-(Bob)
    bob_alice = bob.create_relationship_to(alice, "KNOWS")

Although nodes do not have a specific type attached to them, relationships can be used to achieve a kind of typing. To achieve this, a separate node (sometimes called a //category node//) is used as a common base point from which relationships extend to all its member nodes. To show that Alice and Bob are both people, we can link them both to a third node using relationships of type "PERSON". The code below illustrates this:

    # create a category node for "People" and
    # PERSON relationships to Alice and Bob
    #
    #    ,--[:PERSON]->(Alice)
    #    |
    # (People)
    #    |
    #    `--[:PERSON]->(Bob)
    #
    people = graph_db.create_node()
    people.create_relationship_to(alice, "PERSON")
    people.create_relationship_to(bob, "PERSON")

The number of relationships attached to category nodes such as this can clearly grow very large but Neo4j can happily cope and this is indeed a commonly used pattern.

Using SQL, we could select all //people// with a statement like "{{{SELECT * FROM people}}}". The equivalent, using Neo4j's query language [[http://docs.neo4j.org/chunked/milestone/cypher-query-lang.html|Cypher]] (and assuming the "People" category node has an ID of 100), would be "{{{START z=node(100) MATCH (z)-[:PERSON]->(p) RETURN p}}}". This starts the query at the category node and locates all nodes connected by an outgoing "PERSON" relationship, returning them as query results.

Taking it a step further, this pattern can also allow nodes to be simultaneously assigned multiple "types". This is to some extent akin to how an object within an object oriented environment may implement multiple interfaces and can be easily achieved by linking from a second category node such as in the code below:

    # create category nodes for "People" and "Mothers" and
    # PERSON and MOTHER relationships to Alice and Bob
    #
    #    ,--[:PERSON]->(Alice)<-[:MOTHER]--,
    #    |                                 |
    # (People)                         (Mothers)
    #    |
    #    `--[:PERSON]->(Bob)
    #
    people, mothers = graph_db.create({}, {})
    people.create_relationship_to(alice, "PERSON")
    people.create_relationship_to(bob, "PERSON")
    mothers.create_relationship_to(alice, "MOTHER")

Alice is now described as both a PERSON and a MOTHER whereas Bob is only a PERSON. Cypher allows us to select all nodes which are members of both sets as follows (once again assuming nodes 100 and 101 are the category nodes in question):

    START z1=node(100), z2=node(101)
    MATCH (z1)-[:PERSON]->(p)<-[:MOTHER]-(z2)
    RETURN p

Here, Cypher's MATCH clause is used to identify all nodes which have incoming relationships from the "People" and "Mothers" category nodes. The Cypher language is incredibly powerful and allows much more complex queries to be built up to work with more intricate graphs.

Foreign Keys to More Relationships
----------------------------------

There are many further reasons to build relationships between nodes besides those highlighted above. Very commonly, relationships are used in the way that foreign keys are used in a relational database. Associating people with the companies in which they work would traditionally be achieved by storing the the ID of the required company record within the person record. With Neo4j, that foreign key instead becomes a relationship which we may choose to implement as either {{{(company)-[:EMPLOYS]->(person)}}} or {{{(person)-[:IS_EMPLOYED_BY]->(company)}}}; we may even choose to model both relationships. Such relationships are in fact more analogous to a [[http://en.wikipedia.org/wiki/Junction_table|junction table]] in the relational model, removing the need for either record to directly store information about the other.

The code below shows another example of such a relationship:

    # create category node for "Companies" and Megacorp
    # attach Alice and Bob to their employer
    #
    #                     (Companies)
    #                          |
    #                     [:COMPANY]
    #                          |
    #                          v
    # (Alice)<-[:EMPLOYS]-(Megacorp)-[:EMPLOYS]->(Bob)
    #
    companies, megacorp = graph_db.create({}, {"name": "Megacorp"})
    companies.create_relationship_to(megacorp, "COMPANY")
    megacorp.create_relationship_to(alice, "EMPLOYS")
    megacorp.create_relationship_to(bob, "EMPLOYS")

Go Forth and Nodify...
----------------------

Putting aside an established methodology is not always easy but the transition
to graphical data modelling is well worth the effort. A graph allows a more
multi-dimensional approach to data representation and can eventually end up
feeling like a much more natural approach.

