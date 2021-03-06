Pagination
==========

Pagination is used to step through a set of items in blocks of a specific size. This is often used for data such as search results and can be seen in most search engines and online shopping sites. The concept is well supported in many relational databases, such as the {{{LIMIT}}} and {{{OFFSET}}} keywords in PostgreSQL and MySQL. Within Neo4j, the details differ slightly but the core principle remains the same. Fundamentally, the method described here exploits the order by, skip and limit features of Cypher in order to return only a segment of the total results from the overall result set.

The first script (shown below) is simply used to write some nodes into the database which are the ones we will be paging through. For this example, I have chosen something easy to generate without a great deal of thought: numbers. The data within each node consists of only a number and its English name (e.g. {"number": 12, "name": "twelve"}) something which is simple to create using the fantastic [[http://pypi.python.org/pypi/inflect|inflect]] library.

<<code lang="python">>
{{{
#!/usr/bin/env python

import sys
import inflect

from py2neo import neo4j

if len(sys.argv) < 2:
    sys.exit("Usage: {0} <number_of_records>".format(sys.argv[0]))

# Hook into the database and create the inflection engine
graph_db = neo4j.GraphDatabaseService("http://localhost:7474/db/data/")
root = graph_db.get_subreference_node("NUMBERS")
eng = inflect.engine()

# In case we've run this before, purge any existing number records
nodes = root.get_related_nodes(neo4j.Direction.OUTGOING, "NUMBER")
rels = root.get_relationships(neo4j.Direction.OUTGOING, "NUMBER")
graph_db.delete(*rels)
graph_db.delete(*nodes)

# Create the nodes
count = int(sys.argv[1])
nodes = graph_db.create(*[
    {
        "number": i,
        "name": eng.number_to_words(i)
    }
    for i in range(count)
])

# Connect the nodes to the subreference node
graph_db.create(*[
    (root, "NUMBER", node)
    for node in nodes
])
}}}
<</code>>

Once the data has been inserted, we can get down to business with the pagination proper. The theory is simple: in order to view any particular page from a set of data, we firstly order the data appropriately, secondly skip all lower-numbered pages and thirdly return the items belonging to the page in question. More mathematically, assuming page numbering starts at 1 and we define each page to consist of ten items, we need to skip over the first 10(n - 1) items in order to find the first item for page n.

Our second script returns the contents of a page by building a Cypher query with the appropriate clauses:

<<code lang="python">>
{{{
#!/usr/bin/env python

import sys

from py2neo import neo4j, cypher

if len(sys.argv) < 2:
    sys.exit("Usage: {0} <page_number>".format(sys.argv[0]))

# Hook into the database
graph_db = neo4j.GraphDatabaseService("http://localhost:7474/db/data/")
root = graph_db.get_subreference_node("NUMBERS")

# Set up pagination variables
page_number = int(sys.argv[1])
page_size = 10

# Build the cypher query
query = """\
start x={0}
match (x)-[:NUMBER]->(n)
return n
order by n.name
skip {1}
limit {2}
""".format(
    str(root),
    page_size * (page_number - 1),
    page_size
)

# Grab the results, iterate and print
data, metadata = cypher.execute(graph_db, query)
for row in data:
    print row[0]["name"]
}}}
<</code>>

The whole thing can then simply be run from the command line. In the example below, we are populating the database with 150 numbers (0 to 149) and then looking for page 4:

{{{
$ python populate.py 150
$ python get_page.py 4
forty-one
forty-seven
forty-six
forty-three
forty-two
four
fourteen
nine
nineteen
ninety
}}}

And that's all there is to it. The scripts in this article are all available from the [[https://github.com/nigelsmall/py2neo|py2neo repository on GitHub]], specifically within the examples directory, and should be quite straightforward to pick apart for your own purposes.

