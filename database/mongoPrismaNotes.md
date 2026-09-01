# MongoDB, Mongoose, Prisma & Aggregation — Deep Dive Roadmap

We'll go from fundamentals → internals → modeling → aggregation → performance → testing → production → interview problems.

---

## 1. MongoDB Fundamentals

**Definition:** MongoDB is a **document-oriented NoSQL database** that stores data as flexible, JSON-like documents (BSON) grouped into collections, rather than rows in fixed-schema tables — designed for applications where data shape varies or evolves quickly.

**Databases, collections, documents — Definition:** a MongoDB **database** holds multiple **collections**; a collection is an unordered group of **documents** (analogous to a SQL table holding rows); a document is a single record, represented as a set of field/value pairs (analogous to a row, but nestable and schema-flexible).

**BSON vs JSON — Definition:** BSON ("Binary JSON") is the binary-encoded serialization format MongoDB actually stores and transmits documents in — a superset of JSON's data model that adds types JSON lacks (dates, binary data, `ObjectId`, 64-bit integers) and is faster to parse/traverse than text JSON.

**`_id` and `ObjectId` — Definition:** every document has a unique `_id` field, MongoDB's primary key; if not supplied, MongoDB generates an `ObjectId` — a 12-byte value encoding a timestamp, a random value, and a counter, making it both unique and roughly sortable by creation time.

**CRUD operations:**

```js
db.users.insertOne({ name: 'Ada', age: 30 });
db.users.insertMany([{ name: 'Bob' }, { name: 'Cid' }]);

db.users.find({ age: { $gt: 25 } });          // matches multiple documents → cursor
db.users.findOne({ name: 'Ada' });             // matches one document (or null)

db.users.updateOne({ name: 'Ada' }, { $set: { age: 31 } });
db.users.updateMany({ age: { $lt: 18 } }, { $set: { minor: true } });
db.users.replaceOne({ name: 'Ada' }, { name: 'Ada', age: 31 }); // replaces the WHOLE document

db.users.deleteOne({ name: 'Bob' });
db.users.deleteMany({ age: { $lt: 0 } });
```

**Query operators — Definition:** special `$`-prefixed keys used inside a query filter to express conditions beyond exact equality.

```js
db.users.find({ age: { $gt: 18, $lte: 65 } });      // $gt, $gte, $lt, $lte
db.users.find({ status: { $in: ['active', 'pending'] } });
db.users.find({ name: { $regex: /^A/, $options: 'i' } });
db.users.find({ email: { $exists: true } });
```

**Update operators — Definition:** `$`-prefixed keys inside an update document that specify *how* to modify fields, rather than replacing the whole document.

```js
db.users.updateOne({ _id }, { $set: { name: 'New' } });
db.users.updateOne({ _id }, { $unset: { tempField: '' } });
db.users.updateOne({ _id }, { $inc: { loginCount: 1 } });
db.users.updateOne({ _id }, { $push: { tags: 'vip' } });
db.users.updateOne({ _id }, { $pull: { tags: 'inactive' } });
```

**Projections — Definition:** the second argument to `find()`, specifying which fields to include (`1`) or exclude (`0`) in the returned documents, reducing network/memory overhead for wide documents.

```js
db.users.find({ status: 'active' }, { name: 1, email: 1, _id: 0 });
```

**Cursors & pagination — Definition:** `find()` returns a **cursor** — a pointer to the result set, not the results themselves — iterated lazily as you consume it.

```js
db.users.find().sort({ createdAt: -1 }).skip(20).limit(10); // offset pagination — slow at high skip values
db.users.find({ _id: { $gt: lastSeenId } }).limit(10);       // cursor-based pagination — scales better
```

**`mongosh`** — MongoDB's official interactive shell (Node.js-based) for running the commands above directly against a database.

---

## 2. Data Modeling in MongoDB

**Definition:** data modeling in MongoDB is the deliberate design of document shape and relationships to fit the application's actual read/write patterns — MongoDB's lack of an enforced schema doesn't mean skipping schema design, it means schema design happens at the application layer instead of the database layer.

**Embedding vs referencing — Definition:**
- **Embedding** — nesting related data directly inside the parent document, so a single read retrieves everything.
- **Referencing** — storing just the related document's `_id` and fetching it separately (or via `$lookup`/`populate`), the way a SQL foreign key works.

```js
// embedded — good for data that's always read together, bounded in size
{ _id: 1, name: 'Ada', address: { street: '1 Main St', city: 'NYC' } }

// referenced — good for large, independently-queried, or unbounded-growth related data
{ _id: 1, name: 'Ada', addressId: ObjectId('...') }
```

**One-to-one / one-to-many / many-to-many patterns:**
- One-to-one, small — embed.
- One-to-many, "many" side bounded (e.g. a blog post's comments, capped) — embed an array.
- One-to-many, "many" side unbounded (e.g. a user's millions of log events) — reference, with the "many" side pointing back to the "one."
- Many-to-many — reference in both directions, or a join collection, depending on query patterns.

**Denormalization tradeoffs — Definition:** duplicating data across documents (e.g. storing an author's name on every post, not just their ID) to avoid an extra lookup on read, at the cost of needing to update every copy when the source data changes — a deliberate MongoDB-idiomatic tradeoff, since joins (`$lookup`) are comparatively more expensive than in SQL.

**Document size limit** — a single BSON document cannot exceed **16MB**; unbounded embedded arrays (e.g. "all comments ever" inside a post document) risk hitting this limit, which is itself a signal to switch that relationship from embedding to referencing.

**Modeling for read patterns vs write patterns** — the guiding MongoDB modeling principle (unlike SQL's "normalize first, optimize later") is: design the document shape around how the application will *query* the data, since that access pattern — not abstract relational purity — determines whether embedding or referencing performs better.

**Polymorphic schemas — Definition:** storing documents with different shapes in the same collection (e.g. a `payments` collection where a `card` document and a `bank_transfer` document have different fields), discriminated by a `type` field — natural in MongoDB's flexible-schema model, awkward in a fixed-schema SQL table.

**Schema versioning — Definition:** including a `schemaVersion` field on documents so application code can detect and migrate/handle documents written under an older shape, since there's no database-enforced migration the way SQL `ALTER TABLE` provides.

---

## 3. Indexing

**Definition:** an index is a separate, ordered data structure (a B-tree, by default) that MongoDB maintains alongside a collection to make queries on indexed fields fast — without one, a query must scan every document in the collection (a "collection scan").

**Single-field index:**

```js
db.users.createIndex({ email: 1 }); // 1 = ascending, -1 = descending
```

**Compound index — Definition:** an index on multiple fields together, whose field *order* determines what query/sort patterns it can serve efficiently.

```js
db.orders.createIndex({ userId: 1, createdAt: -1 });
```

**The ESR rule — Definition:** a guideline for ordering fields in a compound index: **E**quality fields first, then **S**ort fields, then **R**ange fields — this ordering lets MongoDB narrow down candidates with exact matches first, apply the sort using the index's existing order, and finally filter the range, minimizing work.

**Multikey indexes — Definition:** an index automatically created as "multikey" when the indexed field holds an array — MongoDB indexes each array element individually, letting queries match any element without a full scan.

**Text indexes — Definition:** a specialized index supporting full-text search (`$text` queries) across string fields — a basic search capability, not a substitute for a dedicated search engine (Elasticsearch/Atlas Search) at scale.

**Geospatial indexes** (`2dsphere`) — support location-based queries (points within a radius, intersecting shapes).

**Unique indexes — Definition:** an index that rejects any insert/update that would create a duplicate value for the indexed field(s) — the mechanism behind uniqueness constraints (e.g. unique `email`).

**TTL indexes — Definition:** a special index on a date field that automatically deletes documents a set number of seconds after that date passes — useful for session data, verification tokens, or logs that should self-expire.

```js
db.sessions.createIndex({ createdAt: 1 }, { expireAfterSeconds: 3600 });
```

**Covered queries — Definition:** a query fully satisfied by the index itself, without touching the actual documents (all requested fields, including the filter and projection, are present in the index) — the fastest possible query.

**`explain()` — Definition:** runs a query and returns its execution plan instead of its results — showing whether an index was used (`IXSCAN`) or a full collection scan happened (`COLLSCAN`), how many documents were examined vs returned, and execution time — the primary tool for diagnosing slow queries.

```js
db.users.find({ email: 'a@b.com' }).explain('executionStats');
```

**Index selectivity & when NOT to index** — every index speeds up matching reads but slows down writes (each write must also update every index on that collection) and consumes memory; indexing a low-cardinality field (e.g. a boolean) rarely helps, since it doesn't narrow the candidate set much.

---

## 4. The Aggregation Framework

A major deep-dive topic.

**Definition:** the aggregation pipeline is MongoDB's framework for processing documents through a sequence of **stages**, each transforming the data and passing its output to the next stage — analogous to a Unix pipeline, and to SQL's `GROUP BY`/`JOIN`/`HAVING` combined, but expressed as a declarative array of stage objects.

```js
db.orders.aggregate([
  { $match: { status: 'completed' } },
  { $group: { _id: '$customerId', total: { $sum: '$amount' } } },
  { $sort: { total: -1 } },
  { $limit: 10 },
]);
```

### Pipeline stages

**`$match` — Definition:** filters documents, using the same query syntax as `find()` — should generally be placed as early as possible in the pipeline so later stages process fewer documents.

**`$project` — Definition:** reshapes each document — include/exclude fields, rename them, or compute new fields from expressions.

```js
{ $project: { name: 1, yearJoined: { $year: '$createdAt' } } }
```

**`$group` — Definition:** groups documents by a specified key (`_id` in the stage) and computes an aggregate value per group using accumulator operators — the aggregation equivalent of SQL's `GROUP BY`.

```js
{ $group: { _id: '$status', count: { $sum: 1 }, avgAmount: { $avg: '$amount' } } }
```

**`$sort` / `$limit` / `$skip`** — order, cap, and offset the document stream, same semantics as in `find()`.

**`$unwind` — Definition:** deconstructs an array field, outputting one document per array element (all other fields duplicated) — turns "one document with an array of 3" into "three documents," commonly used before grouping by an array's contents.

```js
{ $unwind: '$tags' }
```

**`$lookup` — Definition:** performs a left outer join against another collection in the same database, matching a local field to a foreign field and adding the matched documents as an array field — MongoDB's answer to a SQL `JOIN`.

```js
{
  $lookup: {
    from: 'users',
    localField: 'userId',
    foreignField: '_id',
    as: 'user',
  }
}
```

**`$addFields` / `$set`** — add or overwrite fields on each document without dropping the rest (unlike `$project`, which requires explicitly listing every field you want to keep).

**`$count`** — outputs a single document with the count of documents at that point in the pipeline.

**`$facet` — Definition:** runs several independent sub-pipelines in parallel over the *same* input documents, combining their separate results into one output document — used to compute, e.g., paginated results and a total count in a single round trip.

```js
{
  $facet: {
    data: [{ $skip: 20 }, { $limit: 10 }],
    totalCount: [{ $count: 'count' }],
  }
}
```

**`$bucket` / `$bucketAuto` — Definition:** groups documents into ranges ("buckets") based on a specified expression — `$bucket` with explicit boundaries, `$bucketAuto` letting MongoDB choose roughly equal-sized groups — useful for histograms.

**`$merge` / `$out` — Definition:** write the pipeline's final output to a collection instead of returning it to the client — `$out` replaces the target collection entirely; `$merge` can insert, update, or merge into an existing collection, enabling incremental materialized views.

**Accumulator operators** (used inside `$group`): `$sum`, `$avg`, `$min`, `$max`, `$push` (collect values into an array), `$first`/`$last` (require a preceding `$sort` to be meaningful), `$addToSet` (like `$push` but deduplicated).

**Expression operators** (used inside `$project`/`$addFields`/etc.): `$cond` (ternary), `$switch` (multi-branch), date operators (`$dateToString`, `$year`, `$dayOfWeek`), arithmetic (`$add`, `$multiply`), string operators (`$concat`, `$toUpper`).

**Pipeline optimization & stage ordering** — put `$match`/`$limit`/`$sort` as early as possible (MongoDB's optimizer can sometimes reorder them automatically, but writing it early is both a hint and clearer intent); avoid `$unwind` on huge arrays before filtering if you can filter first.

**`$lookup` performance considerations** — a `$lookup` is effectively a query per input document (or a hash join in newer versions, more efficient) — ensure the `foreignField` is indexed, and avoid `$lookup`ing large collections without a preceding `$match` to shrink the input first.

**Aggregation vs application-level processing** — pushing grouping/filtering/joining logic into the aggregation pipeline (executed server-side, close to the data, over an index) is almost always faster than pulling raw documents into the application and processing them in JS — use the pipeline whenever the transformation can be expressed in it.

---

## 5. Transactions & Consistency

**Single-document atomicity — Definition:** every write to a *single* document (even one touching multiple nested fields) is always atomic in MongoDB — a partial update to one document is never visible.

**Multi-document transactions — Definition:** MongoDB (since v4.0, replica sets; v4.2, sharded clusters) supports ACID transactions spanning multiple documents/collections — all writes within the transaction commit together or roll back together, via `session.startTransaction()`/`commitTransaction()`/`abortTransaction()`.

```js
const session = client.startSession();
try {
  session.startTransaction();
  await accounts.updateOne({ _id: a }, { $inc: { balance: -100 } }, { session });
  await accounts.updateOne({ _id: b }, { $inc: { balance: 100 } }, { session });
  await session.commitTransaction();
} catch (err) {
  await session.abortTransaction();
  throw err;
} finally {
  session.endSession();
}
```

**Read/write concerns — Definition:** **write concern** specifies how many replica set members must acknowledge a write before it's considered successful (`w: 1` = primary only, `w: 'majority'` = a majority of the set — safer against data loss on failover); **read concern** specifies the consistency/isolation guarantee for a read (e.g. `'majority'` only returns data acknowledged by a majority, avoiding reads that could later be rolled back).

**Read preference — Definition:** which replica set member(s) a read is allowed to target — `primary` (default, always consistent with the latest write), `secondary`/`secondaryPreferred` (spreads read load, but may return slightly stale data due to replication lag).

**Causal consistency — Definition:** a guarantee that, within a single client session, operations are observed in an order consistent with cause-and-effect (e.g. you'll never read a document *before* a write you yourself just made) — provided automatically within a MongoDB session.

---

## 6. Replication & Sharding

**Replica set — Definition:** a group of MongoDB servers holding the same data set, providing redundancy and high availability — one **primary** accepts all writes, while **secondaries** replicate the primary's oplog and can serve reads.

**Primary/secondary election — Definition:** if the primary becomes unreachable, the remaining members of the replica set automatically hold an election (using the Raft-like consensus protocol) to promote a secondary to primary, restoring write availability without manual intervention.

**Read from secondaries** — see read preference above; trades strict consistency for read scalability, appropriate for read-heavy, staleness-tolerant workloads (analytics dashboards, not a bank balance check right after a transfer).

**Sharding — Definition:** horizontally partitioning a collection's data across multiple servers ("shards"), each holding a subset of the data, so the collection's total size and throughput can exceed what a single server could handle.

**Shard key — Definition:** the field (or fields) used to determine which shard a given document lives on — chosen once per collection and effectively permanent, so shard key selection is one of the most consequential MongoDB design decisions; a poorly-chosen key (low cardinality, monotonically increasing) causes uneven data/traffic distribution ("hot shard").

**Chunking & balancing — Definition:** MongoDB automatically splits a sharded collection's data into ranges ("chunks") by shard key value, and the **balancer** migrates chunks between shards in the background to keep data distribution roughly even as the collection grows.

**When to shard** — only once a single replica set can no longer handle the data volume or write throughput; sharding adds significant operational complexity, so it's a scaling tool of last resort, not a default starting architecture.

---

## 7. Mongoose — Fundamentals

**Definition:** Mongoose is an **Object Data Modeling (ODM)** library for MongoDB in Node.js — it layers schemas, validation, middleware, and a model-based API on top of the flexible, schema-less native MongoDB driver.

```js
import mongoose from 'mongoose';
await mongoose.connect('mongodb://localhost:27017/myapp');
```

**Schema — Definition:** a Mongoose construct defining a document's expected shape — field names, types, defaults, and validation rules — enforced at the application layer (MongoDB itself remains schema-flexible underneath).

```js
const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  age: { type: Number, default: 18 },
}, { timestamps: true }); // adds createdAt/updatedAt automatically
```

**Model — Definition:** a compiled constructor function derived from a schema, providing the actual CRUD API (`.find()`, `.create()`, `.save()`) bound to a specific MongoDB collection.

```js
const User = mongoose.model('User', userSchema); // collection: "users" (pluralized, lowercased)
const user = await User.create({ name: 'Ada', email: 'ada@x.com' });
```

**SchemaTypes** — the set of types Mongoose recognizes for fields: `String`, `Number`, `Date`, `Boolean`, `ObjectId`, `Array`, `Map`, nested sub-schemas, etc., each with type-specific validators/options (e.g. `minlength`/`maxlength` for `String`).

---

## 8. Mongoose — Deep Dive

**Validation — Definition:** rules Mongoose checks before saving a document, rejecting the write with a `ValidationError` if they fail.

```js
const userSchema = new mongoose.Schema({
  email: {
    type: String,
    required: [true, 'Email is required'],
    match: [/^\S+@\S+\.\S+$/, 'Invalid email'],
  },
  age: { type: Number, min: 0, max: 120 },
});

userSchema.path('email').validate(async function (value) {   // custom async validator
  const count = await mongoose.models.User.countDocuments({ email: value });
  return count === 0;
}, 'Email already in use');
```

**Middleware (hooks) — Definition:** functions Mongoose runs automatically before (`pre`) or after (`post`) a given operation — document middleware (`save`, `validate`, `remove`) runs on individual document instances; query middleware (`find`, `updateOne`) runs on query objects, which have a different `this` context.

```js
userSchema.pre('save', async function (next) {
  if (this.isModified('password')) this.password = await bcrypt.hash(this.password, 10);
  next();
});

userSchema.post('save', function (doc) { console.log(`User ${doc._id} saved`); });
```

**Virtuals — Definition:** computed properties that are part of a document's API but never persisted to MongoDB — derived from other fields on the fly.

```js
userSchema.virtual('fullName').get(function () { return `${this.firstName} ${this.lastName}`; });
```

**Instance methods & statics — Definition:** **instance methods** are custom functions available on individual documents (`user.comparePassword(...)`); **statics** are custom functions available on the Model itself (`User.findByEmail(...)`).

```js
userSchema.methods.comparePassword = function (candidate) { return bcrypt.compare(candidate, this.password); };
userSchema.statics.findByEmail = function (email) { return this.findOne({ email }); };
```

**Population (`populate`) — Definition:** Mongoose's mechanism for resolving a referenced `ObjectId` field into the actual referenced document(s), performed as a separate query joined in application code (not a native MongoDB join) — the Mongoose-level equivalent of a SQL foreign-key join.

```js
const postSchema = new mongoose.Schema({ author: { type: mongoose.Schema.Types.ObjectId, ref: 'User' } });

const post = await Post.findById(id).populate('author'); // author is now the full User document
```

**Discriminators — Definition:** Mongoose's support for schema inheritance — multiple related schemas sharing a base schema's fields but stored in the same underlying collection, discriminated by a hidden `__t` field — maps onto MongoDB's polymorphic-schema pattern (section 2).

**Plugins — Definition:** reusable functions that add shared schema behavior (fields, methods, middleware) across multiple schemas — e.g. a pagination plugin or a soft-delete plugin applied once and reused.

**Lean queries (`.lean()`) — Definition:** returns plain JavaScript objects instead of full Mongoose Document instances — skips the overhead of change-tracking, virtuals, and instance methods, meaningfully faster for read-only queries that don't need to be saved back.

```js
const users = await User.find({ status: 'active' }).lean();
```

---

## 9. Prisma — Fundamentals

**Definition:** Prisma is a modern, type-safe ORM/query builder that generates a fully-typed database client from a single declarative schema file, giving compile-time-checked queries instead of relying on runtime-only schema validation.

**Prisma schema (`schema.prisma`) — Definition:** the single source of truth describing the database connection, the generator (what client to produce), and the data model — Prisma derives both the SQL migrations and the TypeScript client's types from this one file.

```prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

model User {
  id    Int    @id @default(autoincrement())
  email String @unique
  posts Post[]
}

model Post {
  id       Int  @id @default(autoincrement())
  title    String
  author   User @relation(fields: [authorId], references: [id])
  authorId Int
}
```

**Relations (`@relation`)** — declares how models reference each other; Prisma generates the corresponding foreign key column(s) and lets you query related data via nested `include`/`select` (see section 10) instead of hand-writing joins.

**Migrations (`prisma migrate`) — Definition:** commands that generate and apply SQL migration files reflecting changes to `schema.prisma`, keeping the actual database schema, the migration history, and the generated client type definitions all in sync.

```bash
npx prisma migrate dev --name add_user_table    # dev: generate + apply + regenerate client
npx prisma migrate deploy                        # production: apply pending migrations only
```

**Prisma Studio** — a bundled GUI (`npx prisma studio`) for browsing and editing data in the connected database through a web interface, generated automatically from the schema.

---

## 10. Prisma — Deep Dive

**Query API:**

```ts
await prisma.user.findUnique({ where: { id: 1 } });
await prisma.user.findFirst({ where: { email: { contains: '@company.com' } } });
await prisma.user.findMany({ where: { active: true }, orderBy: { createdAt: 'desc' } });

await prisma.user.create({ data: { email: 'a@b.com' } });
await prisma.user.update({ where: { id: 1 }, data: { email: 'new@b.com' } });
await prisma.user.upsert({ where: { email: 'a@b.com' }, update: {}, create: { email: 'a@b.com' } });
await prisma.user.delete({ where: { id: 1 } });

await prisma.user.createMany({ data: [{ email: 'a@b.com' }, { email: 'c@d.com' }] });
await prisma.user.updateMany({ where: { active: false }, data: { archived: true } });
```

**Filtering & operators:**

```ts
await prisma.user.findMany({
  where: {
    age: { gte: 18, lt: 65 },
    OR: [{ email: { endsWith: '@a.com' } }, { role: 'admin' }],
  },
});
```

**Relation queries — `include` vs `select` — Definition:**
- `include` — fetch a model's own fields **plus** related records.
- `select` — fetch **only** the explicitly listed fields (own or related), excluding everything else — more efficient when you don't need the whole row.

```ts
await prisma.user.findMany({ include: { posts: true } });                       // full user + related posts
await prisma.user.findMany({ select: { email: true, posts: { select: { title: true } } } }); // narrow shape
```

Nested writes let you create/connect related records in one call:

```ts
await prisma.user.create({
  data: { email: 'a@b.com', posts: { create: [{ title: 'First post' }] } },
});
```

**Transactions (`$transaction`) — Definition:** Prisma's API for running multiple operations atomically — either a sequential array of operations, or an interactive callback (needed when later operations depend on earlier results):

```ts
await prisma.$transaction([
  prisma.account.update({ where: { id: 1 }, data: { balance: { decrement: 100 } } }),
  prisma.account.update({ where: { id: 2 }, data: { balance: { increment: 100 } } }),
]);
```

**Raw queries — Definition:** `$queryRaw`/`$executeRaw` drop down to raw SQL (tagged-template, parameterized — safe from injection) for cases the query builder can't express efficiently.

```ts
const result = await prisma.$queryRaw`SELECT * FROM "User" WHERE age > ${18}`;
```

**Middleware — Definition:** functions Prisma runs around every query (`prisma.$use`), able to inspect/modify params or the result — used for cross-cutting concerns like soft deletes or audit logging. (Being superseded by the newer **Client Extensions** API in recent Prisma versions.)

**Prisma with MongoDB — Definition:** Prisma also supports MongoDB as a datasource; the schema/query API stays the same shape, but relations are implemented via embedded documents or reference fields (MongoDB `ObjectId`) rather than SQL foreign keys, and there's no native SQL migration step — MongoDB's schema-flexibility means `db push` is used instead of full migration files.

**Connection pooling & Prisma Accelerate/Data Proxy — Definition:** each Prisma Client instance manages its own connection pool to the database; in serverless environments (where many short-lived function instances can each open their own pool and exhaust the database's connection limit), **Prisma Accelerate**/the **Data Proxy** provides an external connection-pooling layer (plus optional caching) that serverless functions connect through instead of directly.

**Error handling — Definition:** Prisma throws typed error classes — `PrismaClientKnownRequestError` (e.g. unique constraint violation, code `P2002`) is the one most commonly caught and branched on to return a meaningful API error instead of a generic 500.

```ts
try {
  await prisma.user.create({ data: { email } });
} catch (err) {
  if (err instanceof Prisma.PrismaClientKnownRequestError && err.code === 'P2002') {
    throw new ApiError(409, 'Email already in use');
  }
  throw err;
}
```

---

## 11. Mongoose vs Prisma vs Native Driver

| | Native driver | Mongoose | Prisma |
|---|---|---|---|
| Abstraction level | Lowest — raw commands | Schema + ODM features (hooks, virtuals) | Type-safe query builder |
| Type safety | Manual | Partial (schema-derived, less strict) | Strong, auto-generated from schema |
| Best for | Full control, performance-critical paths, aggregation-heavy code | MongoDB-specific apps wanting validation/hooks/population | Multi-database projects, teams wanting compile-time safety |
| SQL database support | N/A | N/A (Mongo only) | Yes (Postgres, MySQL, SQLite, SQL Server, MongoDB) |
| Aggregation pipeline access | Full, native | Via `.aggregate()`, same syntax | More limited — often still drop to `$queryRaw` or Mongo's driver for complex pipelines |

**When to use the native driver directly** — performance-critical hot paths, or complex aggregation pipelines where an ODM/ORM's abstraction adds friction without benefit.

**When to use Mongoose** — MongoDB-only Node projects that want schema validation, middleware hooks, and population without adopting a heavier, multi-database-oriented tool.

**When to use Prisma** — projects that value compile-time type safety, may need to support (or migrate to/from) a SQL database, or want a single consistent query API across the team regardless of underlying database.

---

## 12. Performance & Query Optimization

**Query profiling — Definition:** using `explain()` (section 3) or MongoDB's built-in **Database Profiler** (logs slow operations above a configurable threshold) to identify which queries are slow and why, rather than guessing.

```js
db.setProfilingLevel(1, { slowms: 100 }); // log queries slower than 100ms
```

**Avoiding N+1 queries — Definition:** the anti-pattern of running one query to fetch a list, then one additional query *per item* to fetch related data (`N` extra queries for `N` items) — instead, batch the related fetch into a single query (`$lookup`, Mongoose `.populate()`, or Prisma `include`) or use `$in` to fetch all related documents at once.

```js
// ❌ N+1
const posts = await Post.find();
for (const post of posts) { post.author = await User.findById(post.authorId); }

// ✅ one extra query total
const posts = await Post.find();
const authorIds = posts.map(p => p.authorId);
const authors = await User.find({ _id: { $in: authorIds } });
```

**Batch operations (`bulkWrite`) — Definition:** sends multiple insert/update/delete operations to MongoDB in a single round trip, instead of awaiting each one individually — significantly reduces network overhead for bulk data changes.

```js
await db.users.bulkWrite([
  { updateOne: { filter: { _id: id1 }, update: { $set: { active: true } } } },
  { deleteOne: { filter: { _id: id2 } } },
]);
```

**Connection pooling** — both the native driver and Mongoose maintain a connection pool by default (configurable size); ensure serverless/short-lived function environments reuse a single client instance across invocations rather than reconnecting on every request.

**Avoiding large `$lookup` joins** — see section 4; filter with `$match` before joining, and ensure the joined field is indexed.

**Pagination at scale** — offset-based pagination (`skip`/`limit`) degrades as the offset grows, since MongoDB must still walk past all skipped documents; **cursor-based pagination** (filtering on `_id`/a sort key greater than the last-seen value) stays fast regardless of how deep you page.

---

## 13. Testing

**Testing with an in-memory MongoDB — Definition:** `mongodb-memory-server` spins up a real, temporary MongoDB instance in-process for tests, giving realistic database behavior without needing a persistent test database or Docker container.

```js
import { MongoMemoryServer } from 'mongodb-memory-server';

let mongod;
beforeAll(async () => {
  mongod = await MongoMemoryServer.create();
  await mongoose.connect(mongod.getUri());
});
afterAll(async () => { await mongoose.disconnect(); await mongod.stop(); });
```

**Mocking Mongoose models / Prisma Client** — for pure unit tests (no real database involved), a model/client method is replaced with a test double (`jest.mock`, or a library like `jest-mock-extended` for Prisma), so business logic can be tested in isolation from the database layer.

**Integration testing against a real test database** — a middle ground between full mocking and hitting production data: run tests against a dedicated test database (or a container via `testcontainers`), exercising real queries/indexes/constraints.

**Seeding test data — Definition:** populating the test database with known fixture data before tests run, so assertions have a predictable starting state — Prisma supports a dedicated `prisma/seed.ts` script (`npx prisma db seed`).

---

## 14. Data Integrity & Validation Strategy

**Database-level validation (MongoDB JSON Schema validator) — Definition:** MongoDB collections can optionally be configured with a `$jsonSchema` validator, enforced by the database itself on every write regardless of which client/language wrote it — a safety net beneath application-level validation.

```js
db.createCollection('users', {
  validator: {
    $jsonSchema: {
      required: ['email'],
      properties: { email: { bsonType: 'string', pattern: '^\\S+@\\S+$' } },
    },
  },
});
```

**Application-level validation (Mongoose/Prisma/Zod)** — the first and most flexible line of defense — runs before a write is even attempted, can produce rich, field-specific error messages for API responses, and (Zod especially) can validate data that never touches the database at all (e.g. incoming API request bodies).

**Where validation should live** — application-level validation for user-facing error messages and business rules; database-level `$jsonSchema` as a last-resort safety net against any write path (a script, a different service) that bypasses the application layer.

**Handling schema evolution safely** — since MongoDB won't reject documents that don't match a new shape automatically (unless a validator is added), evolving a schema typically means: write new code that can read both old and new document shapes, backfill/migrate existing documents in the background, then (optionally) tighten validation once the migration is complete.

---

## 15. Production Engineering

**Connection string configuration & secrets** — the MongoDB connection URI (containing credentials) belongs in an environment variable, never committed to source control; MongoDB Atlas (the managed cloud offering) additionally requires IP-allowlisting or VPC peering for network-level access control.

**Connection pooling in production** — size the pool (`maxPoolSize`) based on expected concurrent load and the database's connection limits; in serverless deployments, be deliberate about connection reuse across invocations (see section 12) to avoid exhausting the database's max connections.

**Monitoring (Atlas metrics, slow query logs)** — MongoDB Atlas provides built-in dashboards for query performance, connection counts, and replication lag; self-hosted deployments typically pair the database profiler (section 12) with an external metrics/log aggregation tool.

**Backups & point-in-time recovery — Definition:** regular automated snapshots of the database, plus (in Atlas / enterprise setups) continuous oplog capture that allows restoring to any specific moment in time, not just the most recent snapshot — essential for recovering from accidental deletes/bad migrations, not just hardware failure.

**Index management in production — Definition:** building a new index on a large, live collection is a heavy operation; MongoDB supports building indexes in the background (or, in recent versions, indexes are built without blocking other operations by default) so the collection remains available for reads/writes during the build.

**Scaling reads/writes** — read scaling: add secondaries and route eligible reads to them (section 6); write scaling: sharding (section 6) once a single primary's write throughput is the bottleneck.

**Migration strategy in production (`prisma migrate deploy`)** — in CI/CD, `prisma migrate deploy` applies already-generated, already-reviewed migration files (unlike `migrate dev`, which also generates new ones) — migrations should run as an explicit deploy step, before the new application code that depends on the new schema goes live.

**Common pitfalls & anti-patterns:**
- Unbounded array growth inside embedded documents (hits the 16MB document limit).
- Missing indexes on fields used in frequent queries or `$lookup`s.
- Offset-based pagination at large page depths.
- Choosing a low-cardinality or monotonically-increasing shard key.
- Storing large binary blobs (images, files) directly in documents instead of object storage (S3-equivalent) with just a reference stored in MongoDB.
