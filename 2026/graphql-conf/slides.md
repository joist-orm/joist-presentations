---
theme: default
title: Making GraphQL Fun for the Backend Too
info: |
  Joist ORM — putting the domain model on the wire
  A 10-minute lightning talk by Stephen Haberman.
author: Stephen Haberman
colorSchema: dark
transition: fade
highlighter: shiki
lineNumbers: false
mdc: true
fonts:
  sans: Inter
  mono: JetBrains Mono
  provider: google

---

<div class="flex flex-col items-center gap-8">
  <img src="/logo.svg" class="w-56" alt="Joist" />
  <div class="title-line font-bold">
    Making GraphQL <span class="text-joist">fun</span> for the backend too
  </div>
  <div class="flex flex-col items-center gap-2">
    <div class="text-2xl opacity-85">
      <span v-mark="{ at: 1, type: 'strike-through', color: '#fc8a22' }">Putting the domain model on the wire</span>
    </div>
    <div v-click="1" class="text-2xl text-joist font-semibold italic">
      Cloning Ent in TypeScript
    </div>
  </div>
  <div class="text-lg opacity-70 mt-5">
    Stephen Haberman · GraphQL Conf 2026
  </div>
</div>

<!--
Hi, I'm Stephen Haberman, an engineer at Homebound, a VC-backed construction startup,
where we don't sell SaaS, we actually build the homes; we have our GC license in several
states, and on the tech side have built a GraphQL/TypeScript monolith to power our
construction platform.

We use GraphQL heavily in our stack, and actually like it, which leads me to the talk
today -- Making...

..aka...aka...
-->

---
layout: default
---

# Clients love GraphQL ♥️

- **Relay** + **Apollo** set the bar for client-side DX
- GraphQL "fat shapes" / adhoc queries are super-easy to render

<!--
First, I think it's pretty obvious that frontends love GraphQL...

Relay & Apollo set the bar for client-side DX, with caching normalization, etc

GraphQL's fat shapes / adhoc queries are super-easy for FEs to render deep trees of data.

GraphQL's client-side DX is really great, in my opinion.

-->

---

# ...but the backend 😰

- **N+1** s by default
- **DataLoader** boilerplate
- **Validation** scattered across every mutation
- "Resolver spaghetti" business logic
- **Auth** 🙈

<div v-click class="mt-12 text-joist text-2xl">
Frontends drove GraphQL's peak hype, the backend DX killed it
</div>

<!--

...but the backend, maybe not so much.

We have N+1s by default.

You can solve with DataLoader, typically a lot of boilerplate.

Validation is potentially scattered around, business logic is scattered around.

Auth we don't really talk about auth.

Which leads me to assert, that imo FEs drove GQL's peak hype a few years, but the BE DX killed it.

Which made me curious, what about Facebook?
-->

---
layout: two-cols-header
---

# How did Facebook do this? 🤔

::left::

<div class="mt-15">

- **Ent** — a rich entity/domain model in Hack
- **GraphQL** — a *wire format* for querying Ent
- Probably lightweight resolvers
- Graph-based traversals, graph-based auth, etc.
- The graph came first

</div>

::right::

<div v-click="1">

...coindentally, at Homebound, we built

- A rich entity/domain model in TypeScript
- Using **GraphQL** as a *wire format*
- Lightweight resolvers
- Graph-based traversals, graph-based auth, etc.
- The graph comes first

😅

</div>

<div v-click="2">

**Joist** &mdash; a TypeScript/Postgres ORM for Majestic Monoliths

</div>

<!--

How did FB do this? They invented GQL, I assume their DX is fairly good.

I haven't worked at Facebook, but piecing together things from the outside in.

My understanding is that they already had this Ent rich entity/domain model/ORM in Hack.

And GraphQL seems like was mostly a wire format for querying their existing Ent graph.

...so probably

Coincidentally, at Homebound we...

And our DX has been pretty great.

We've been open-sourcing our work as Joist, a TypeScript ORM for Majestic Monoliths.

And I'm going to give a whirlwind tour of our killer features.
-->

---

# Workflow: Codegen & Scaffolding 🚀

<div class="grid grid-cols-3 gap-6 text-sm mt-4">

<div>

**1. `authors` table**

```sql
CREATE TABLE authors (
  id    serial primary key,
  name  varchar not null,
  bio   text
);
```

<div class="text-xs opacity-60 mt-1">
Update schema using migrations
</div>


</div>

<div>

**2. `Author` entity**

```ts
// src/entities/Author.ts
export class Author
  extends AuthorCodegen {
  // getters/setters in base class
    
  // add your own fields/logic here
}

// add custom rules and reactions
```

<div class="text-xs opacity-60 mt-1">
<code>yarn joist-codegen</code>
</div>

</div>

<div>

**3. GraphQL Schema**

```graphql
# author.graphql, auto-generated
# scaffold, change as needed
type Author {
  id: ID!
  name: String!
  bio: String
  books: [Book!]!
  # Delete internal fields 
  # password
}

extend type Mutation {
  saveAuthor(input: ...): ...
}
```

<div class="text-xs opacity-60 mt-1">
Evergreen scaffolding
</div>

</div>

</div>

<div class="text-center mt-8 text-xl text-joist font-semibold">
DB &rarr; entities &rarr; GraphQL
</div>

<!--

From the workflow side of things, we really lean into Codegen & Scaffolding to move quickly.

We treat the database as source of truth, so if you have an `authors table`, we codegen an Author entity,
with the getter/setter boilerplate in the base class so you don't see it, and scaffold out the GraphQL schema.

The scheme is a scaffolding, so you can delete fields you don't want exposed, or add new ones
that aren't in the database, and we won't stomp your changes.

And the scaffolding is evergreen, we keep running it after every change to the database, so it's
always on/always helping us move fast.

-->

---
layout: two-cols-header
---

# 1. Safe Graph Traversal 👷

<div class="text-lg -mt-1">No N+1s &mdash; dataloaders for free</div>

::left::

**User code** — `Promise.all` with `.load()`

```ts
// load 100 authors
const authors = await em.loadAll(Author, ids);

// per-author book load, in parallel
const allBooks = await Promise.all(
  // Risks an N+1 of per-Author
  // SELECT * FROM books WHERE author_id = ?
  authors.map(a => a.books.load())
);

// But really 1 SQL query for all 100 authors:
// SELECT * FROM books WHERE author_id IN (...)

// All graph traversal (m2o, o2m, o2o, m2m) is N+1
// safe, including em.find-in-a-loop
```

::right::

**Emergent batching** — no restructuring needed

```ts
// Some big gnarly function
async function someComplicatedLogic(authors: Author[]) {
  // note: _no up-front populate hint_
  await authors.asyncForEach(async (a) => {
    await helerMethod(a);
  });
}


// Still N+1 safe; no Rails-style "pull the populate hint up
// until it is 'outside the loop'" refactorings
async function someHelperMethod(a: Author) {
  const books = await a.books.load();
  return books.map(b => b.title).join(", ");
}
```

<!--

After workflow, our first killer feature is safe graph traversal, and by safe I mean no N+1s, aka dataloaders for free.

On the right is an example of loading one hundred authors, then in a loop / the Promise.all is a loop,
loading each individual author's books -- which is a canonical N+1 scenario.

But with Joist, this won't N+1, we have dataloader baked into all of our operations, and you basically
can't get Joist to N+1.

The other huge win of leveraging dataloader is that the batching is emergent -- you can decompose your
code into smaller functions, and if those functions happen to load data -- that's fine, they're still batched.

This ability to decompose logic, without N+1s, is foundational to everything else we do.

-->

---
layout: two-cols-header
---

# 2. Succinct Graph Traversal

<div class="text-lg -mt-1">"Load in a loop" is now safe but ugly &mdash; instead populate shapes</div>

::left::

**Before** — `Promise.all` soup

```ts
// By default, relations force `await` usage
const author = await em.load(Author, id);

// ...fine but boilerplately
const books = await author.books.load();

const reviews = (await Promise.all(
  books.map(b => b.reviews.load())
)).flat();

const comments = (await Promise.all(
  reviews.map(r => r.comments.load())
)).flat();

const fourStar = reviews
  .filter(r => r.rating >= 4).length;
```

::right::

**After** — populate hint of subgraph

```ts
// Typed as `Loaded<Author, { books: "reviews" }>`
const author = await em.load(Author, id, {
  // Any db-based m2o/o2m/m2m relations or
  // user-defined relations in the Author entity
  books: "reviews",
});

// No awaits!
const fourStar = author.books.get
  .flatMap(b => b.reviews.get)
  .filter(r => r.rating.get >= 4)
  .length;
```

<!--

The next killer feature is succinct graph traversal --

So we've made "Load in a loop" is now safe, but it's still pretty ugly.

On the left, as we're lazy-loading the object graph into memory, we have to await each individual load,
and then "glue together" all the Promises -- leads to lots of ugly `Promise.all` soup.

In Joist, instead we can use a single up-front `await` to populate the subgraph we want, and then Joist
uses TypeScript magic to decorate our relations with synchronous getters -- now our business logic is very clean,
just like synchronous collections, but the key point is that we only get these synchornous getters when
the type system knows its safe.

-->

---
layout: two-cols-header
---

# 3. Graph Validation ✅

<div class="text-lg -mt-1">Invariants belong in entities &mdash; not mutations or endpoints</div>

::left::

**Before** — copy/pasted across mutations

```ts
// createAuthor mutation
if (!input.name)
  throw new UserError("name required");
if (input.name.length > 200)
  throw new UserError("too long");

// updateAuthor mutation
if (input.name?.length > 200)
  throw new UserError("too long");

// bulk import/csv upload is a separate endpoint
// (forgotten — rule drifts)
```

::right::

**After** — declared once, runs automatically

```ts
// Simple field-level rule -- zod can do this
authorConfig.addRule("name", (a) => {
  if (a.name.length > 200) return "name too long";
});

// Cross-entity rules/invariants -- zod cannot do this
authorConfig.addRule(
  // Watch this subgraph
  { books: { reviews: "rating" } },
  // Rerun this lambda whenever it changes
  (a) => {
    const ratings = a.books.get
      .flatMap(b => b.reviews.get)
      .map(br => br.rating);
    if (ratings.sum() < 0) {
      return "aggregate rating must be positive";
    }
  }
);
```

<!--

Our next killer feature is putting validation into the Graph -- we believe invariants
belong in the graph, not mutations or endpoints.

On the left, we have standard "input-based validation" that is onsy-twosy
looking at each input field, typical zod-style per-field validation.

In Joist, instead we push validation into the graph, and want the Author entity to own
validation.

We do this both for simple field-level rules like `name`, this 1st validation rule, fires whenever
the `name` changes, to validate it.

But since we're in the graph, we can also write validation rules across a subgraph of data.

Here, for the author's 2nd validation rule, we first declare the subgraph of that author's
books & reviews that we care about as the 1st param,

And the 2nd parameter is the lambda that uses that data to evaluate the invariant.

So now for any writes that go through Joist, Joist recognizes when these dependencies change,
and will invoke our lambda.

This makes cross-entity business invariants really trivial to write.

-->

---
layout: two-cols-header
---

# 4. Graph Projections

<div class="text-lg -mt-1">

Declarative `ReactiveField`s for derived values

</div>

::left::

**Before** — call `update` from every code path

```ts
// Calculated value, we "just remember" to call
async function updateFavoriteTitles(p: Publisher) {
  const authors = await p.authors.load();
  p.titlesOfFavoriteBooks = authors
    .map(a => a.favoriteBook?.title)
    .filter(Boolean).join(", ");
}

// Then call `updateFavoriteTitles` from saveAuthor,
// deleteBook, updateBook, importAuthors...

// If we miss one → stale data. 😬
```

::right::

**After** — `hasReactiveField`

```ts
class Publisher {
  titlesOfFavoriteBooks = hasReactiveField(
    // Watch this subgraph for changes
    { authors: { favoriteBook: "title" } },
    // Call lambda to recalc whenever it changes
    (p) => p.authors.get
      .map(a => a.favoriteBook.get?.title)
      .compact().join(", ") || undefined,
  );
}

// All em.flush() calls recalc any dirtied fields
//
// I.e. when book1.title changes, Joist walks the "reverse
// path" of book -> author (favoriteBook) -> publisher
// and recalcs titlesOfFavoriteBooks.
```

<!--

Our 4th killer feature is Graph Projections, which are derived values.

I.e. columns in your database that are calculated from other values.

On the left, when doing this by hand, you might have business logic like this `updateFavoriteTitles` function,
that is kinda opaque about what data it loads, and you just have to remember to call it.

In Joist, instead we leverage that we're in the graph, and so declaratively setup our field in the Publisher
entity, and again the 1st param here is our subgraph that we depend on, of a publisher's authors/favoriteBook/title,

And the 2nd parameter is the lambda to calculate the value.

And just like the last slide, Joist will automatically track when these dependencies change, and invoke our lambda.

We really feel spoiled having this built in--it makes derived values very easy to add.

-->

---
layout: two-cols-header
---

# 5. Graph-Based Auth 🔑

<div class="text-lg -mt-1">Bring your own query AST plugin</div>


::left::

**Before** — manually add `tenant_id` to every query

```ts
// Every query adds `WHERE tenant_id = ?`
await db.authors.where("tenant_id", tenantId);
await db.books.where("tenant_id", tenantId);
await db.reviews.where("tenant_id", tenantId);

// Every find, every join, every report...

// Miss one => cross-tenant data leak 😬
```

::right::

**After** — one plugin, applied to the `em`

```ts
class TenantPlugin extends Plugin {
  constructor(private tenantId: string) { super(); }

  // All of the request's graph traversals
  // (finds, joins, etc.) pass through this hook
  beforeFind(meta, query): void {
    for (const table of query.tables) {
      if (meta.fields.tenant) {
        query.conditions.push({
          alias: table.alias,
          column: "tenant_id",
          cond: { kind: "eq", value: this.tenantId },
        }); } } }
}

// In your request handler
em.addPlugin(new TenantPlugin(tenantId));
```

<div class="text-xs opacity-60 mt-1">
Implementing <code>RbacPlugin</code> is an exercise for the reader
</div>

<!--

Finally our last killer feature is Graph-Based Auth -- although it's not really "auth" per se,
it's plugin-base query rewriting.

Auth is a good example use case for the feature -- so if you're in a tenant-based application,
you have to add "where tenant_id = ?" clauses to every single query, and if you miss one, you have
a cross-tenant data leak.

With Joist, instead you can write a plugin with a `beforeFind` hook, and it will be given every
query that's used for graph traversal, and you can examine it, and rewrite it, i.e. inject 
where clauses.

Then in your request, you set up the TenantPlugin once, and get "auth for free".

There's a lot more to do for real Rbac-based auth, which disclaimer we're still in the
prototyping stages for that, but the capabilities & infra are there.

-->

---
layout: two-cols-header
---

# 6. Query Resolvers

<div class="text-lg -mt-1">Every field + relation, N+1 safe</div>

::left::

**Before** — a `DataLoader` per relation

```ts
const authorLoader = new DataLoader(async (ids) => {
  const rows = await db.authors
    .whereIn("id", ids);
  return ids.map(id =>
    rows.find(r => r.id === id));
});

const bookResolvers = {
  Book: {
    author:  (b) => authorLoader.load(b.authorId),
    reviews: (b) => reviewsByBookLoader.load(b.id),
    // ...repeat per relation
  },
};
```

::right::

**After** — one liner

```ts
import { Book } from "src/entities";
import { entityResolver } from "src/resolvers/utils";

export const bookResolvers: BookResolvers = {
  // Maps entity fields/relations by default, including
  // m2o, o2m, o2o, m2m (only if exposed in the schema)
  ...entityResolver(Book),

  // Implement one-off field resolvers as/if needed
};
```

<!--

We have many other features, but after those main killer features, let's now look
quickly at the "GraphQL layer on top".

For our query resolvers, the left is an example of hand-writing field resolvers by hand,
doing the right dataloader setup, etc

With Joist, instead we have mostly one-liners that "put the entity on the
wire", for 90% of the fields & relations that are 1:1.

-->


---
layout: two-cols-header
---

# 7. Mutation Resolvers

<div class="text-lg -mt-1">Map inputs to entities, validation on the model</div>

::left::

**Before** — hand-wired partial updates

```ts
async function saveAuthor(_, { input }, ctx) {
  const a = input.id
    ? await ctx.em.load(Author, input.id)
    : ctx.em.create(Author, {});
  if (input.name !== undefined) a.name = input.name;
  if (input.bio  !== undefined) a.bio  = input.bio;
  // ...20 more fields
  if (input.bookIds)
    a.books.set(
      await ctx.em.loadAll(Book, input.bookIds));
  await validateAuthor(a);
  await db.insert(authors).values(a).returning();
  return a;
}
```

::right::

**After** — `saveEntity`

```ts
import { saveEntity } from "src/resolvers/utils";

export const saveAuthor = {
  async saveAuthor(_, args, { em }) {
    // Upsert and copy fields that map 1:1 automatically
    const author = await saveEntity(em, Author, args.input);
    // Run validations, reactions, and issue SQL calls
    await em.flush();  
    return { author };
  },
};
```

<!--

Same thing for our mutation resolvers, while on the left is a mutation that
is dealing with input fields one-by-one, figuring out which updates to do,
figuring out what validation to do/not do, figuring out which SQL updates to do.

With Joist, instead we drop the input into the graph, for the fields that map 1:1,
and then call our `flush` method which just tells the graph to do it's thing.

Run validation, recalc any derived values, and if everything is kosher,
persists the data into the database. Short & sweet.

-->

---
layout: center
class: text-center
---

# Building a Great Backend DX

<div class="mt-4 text-2xl text-left inline-block leading-relaxed">

1. Build an Entity graph first
2. Solve batching, validation, and business logic in the graph
3. Then layer GraphQL on top as the wire format

</div>

<div v-click>

<div class="mt-8 text-3xl">
  ...or just use <span class="text-joist font-bold">Joist 🚀</span>
</div>

<div class="mt-12 grid grid-cols-3 gap-6 text-sm">
  <div>
    <a href="https://joist-orm.io" class="text-joist font-bold">Docs</a>
    <div class="opacity-85">joist-orm.io</div>
  </div>
  <div>
    <a href="https://github.com/joist-orm/joist-orm" class="text-joist font-bold">GitHub</a>
    <div class="opacity-85">joist-orm/joist-orm</div>
  </div>
  <div>
    <a href="https://joist-orm.io/discord" class="text-joist font-bold">Discord</a>
    <div class="opacity-85">joist-orm.io/discord</div>
  </div>
</div>

<div class="mt-12 text-joist text-3xl">
Thanks!
</div>

<div class="mt-2 text-lg opacity-75">
Stephen Haberman
</div>

</div>

<!--

And that's it for the whirlwind tour.

Stepping back from Joist itself, my assertion is that if you want a great backend DX,
you should focus on building an entity graph first.

Solve batching, validation, and business logic in the graph, and then layer GraphQL on top
as the wire format.

You can write all of that on your own, or you can come use Joist.

We've been working on this for a few years now, and having a lot of fun doing it.

Thanks!

-->
