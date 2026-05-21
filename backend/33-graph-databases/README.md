# Graph Databases: When and Why

Your relational database handles orders, users, and products just fine. But what happens when your data stops looking like neat rows and columns, and starts looking more like a tangled web?

That's your customer bought this, knows that person, visited this page, and those pages connect to these topics. Every query turns into five JOINs. Performance starts hurting. And you're asking yourself — is there a better way?

This is where graph databases come in.

## What Even Is a Graph Database?

Instead of tables with foreign keys, graph databases store **nodes** (entities) and **edges** (relationships). Both can have **properties** — key-value data attached to them.

```
(User: Alice) → [KNOWS] → (User: Bob)
     ↓                        ↓
   [BOUGHT]                [BOUGHT]
     ↓                        ↓
(Product: Headphones)   (Product: Keyboard)
```

In a relational DB, that's four tables with JOINs across `users`, `products`, `orders`, and a junction table for `friends`. In a graph DB, it's just nodes and edges. You traverse relationships directly.

## When Graph Databases Shine

### 1. Social Networks / Friendship Graphs

This is the classic use case. "Show me friends of friends who like this product."

```cypher
// Cypher (Neo4j query language)
MATCH (me:User {id: "123"})
      -[:KNOWS]->()-[:LIKES]->(p:Product)
RETURN distinct p, count(*)
ORDER BY count(*) DESC
```

In SQL, this requires a self-join on the friends table, then another join for likes, then an aggregation. With deep nesting (friends-of-friends-of-friends), it gets exponentially messier. Graph databases handle this at the data structure level — relationships are stored as direct pointers, not computed through JOINs.

### 2. Recommendation Engines

"People who bought X also bought Y" is a graph pattern. The relationship `[PURCHASED]` between users and products creates natural co-occurrence paths. Graph traversal across purchase patterns is significantly faster than computing this with table scans.

### 3. Fraud Detection

Banks and fintechs love graph databases for fraud because fraud rings are **pattern-based**.

```cypher
// Find suspicious clusters: same device, different identities
MATCH (device:Device {id: "ABC123"})
      <-[:USED_DEVICE]-(p:Person)
RETURN p, device
```

A single device used by five different IDs within hours? That's a red flag a graph query can spot in milliseconds. A relational query doing the same thing? Painful join after painful join.

### 4. Knowledge Graphs

Think Wikipedia's category system, or Google's Knowledge Graph. Every entity links to other entities through typed relationships. "Alan Turing invented the Turing Machine, which relates to Computability Theory, and he also has a connection to WWII Codebreaking."

This is nearly impossible to model elegantly in a relational database. With a graph, it's trivial:

```cypher
MATCH (p:Person {name: "Alan Turing"})-[r]-()
RETURN type(r), endNode(r).name
```

### 5. Routing and Logistics

Shortest path, network analysis, supply chain optimization — these are graph problems by nature. Graph databases come with built-in graph algorithms (shortest path, PageRank, community detection) that you'd have to build from scratch on top of a relational DB.

## When NOT to Use Graph Databases

Here's the part that usually gets glossed over. Graph databases are not relational replacements.

### ❌ Heavy Aggregation Queries

> "Show me total revenue by month for the last year, grouped by product category"

Graph databases suck at this. They're optimized for **traversal**, not **aggregation**. A relational database with proper indexing will destroy Neo4j on analytic queries.

### ❌ Simple CRUD / Transactional Workloads

If your app is standard CRUD — create user, update profile, list orders — you want a relational database or a document store. Graph databases add complexity (new query language, new deployment model, different consistency guarantees) with zero benefit.

### ❌ Large-Scale Bulk Operations

Loading 10 million rows into PostgreSQL is straightforward. Doing the same in Neo4j requires batch importers, careful transaction management, and significantly more planning.

### ⚠️ The "Graph of Everything" Trap

I've seen teams model every single entity as a graph node because "graph databases are the future." Their queries got slower, developers struggled with Cypher, and they eventually migrated back to PostgreSQL.

**Graph databases are a tool, not a religion.**

## Quick Comparison

| Aspect | Relational (e.g., PostgreSQL) | Graph (e.g., Neo4j) |
|--------|------|------|
| Deep relationships | Multiple JOINs, slow at depth | Direct pointer traversal, fast |
| Aggregations | Excellent (GROUP BY, window functions) | Poor |
| Schema flexibility | Rigid (need migrations) | Flexible (schema-optional) |
| Query language | SQL (widely known) | Cypher / Gremlin (new to learn) |
| Bulk operations | Battle-tested | Requires special tooling |
| Transaction support | ACID mature | ACID, but less battle-hardened |
| Ecosystem maturity | Decades old | ~15 years |

## The Honest Take

Here's what I've seen work well in practice:

- **Most of your data stays in PostgreSQL.** It handles 90% of workloads perfectly.
- **Add a graph database alongside**, not instead of.
- Use Neo4j (or Amazon Neptune) only for the parts that benefit from graph — recommendation paths, fraud detection, authorization trees, knowledge graphs.
- Sync data you need in both systems using change data capture or event-driven patterns.

Real-world example: An e-commerce company keeps order data, user profiles, and inventory in PostgreSQL. They run Neo4j just for "customers who viewed this" and product recommendation paths. The graph DB holds a fraction of the total data — only what's needed for graph traversal.

```sql
-- PostgreSQL still handles the hard stuff
SELECT sum(amount), date_trunc('month', created_at) as month
FROM orders
WHERE status = 'completed'
GROUP BY month
ORDER BY month;
```

Then for recommendations:

```cypher
// Neo4j handles what it's good at
MATCH (u:User {id: "456"})-[:VIEWED]->(p:Product)
MATCH (other:User)-[:VIEWED]->(p)
MATCH (other)-[:VIEWED]->(rec:Product)
WHERE NOT (u)-[:VIEWED]->(rec)
RETURN rec.name, count(*) as score
ORDER BY score DESC
LIMIT 10
```

Two databases, each doing what they're best at. No unnecessary complexity.

## Takeaways

- Graph databases excel at connected data with deep relationships — social graphs, recommendation, fraud detection, knowledge graphs
- They're **terrible** at aggregations, bulk operations, and simple CRUD — keep using PostgreSQL or MySQL for that
- Add Neo4j as a complementary system, not a replacement
- If you find yourself JOINing five tables for a "friends of friends" query, it's time to evaluate a graph DB
- Don't model everything as a graph just because you can — you'll regret it
- The best graph DB adoption pattern: start with a relational DB, prove you need graph features, then add Neo4j incrementally
