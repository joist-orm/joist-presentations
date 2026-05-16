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
where we don't sell SaaS, we actually build the homes; we have GC licenses in several
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
- GraphQL "fat shapes" / adhoc subgraphs are super-easy to render

<!--
No denying that frontends love GraphQL...
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
Which is curious, what about Facebook?
-->

---
layout: two-cols-header
---

# How did Facebook do this? 🤔

::left::

<div class="mt-15">

- **Ent** — a rich entity/domain model in Hack
- **GraphQL** — a *wire format* for querying Ent
- Probably fewer, lightweight resolvers
- Graph-based traversals, graph-based auth, etc.
- The graph came first

</div>

::right::

<div v-click="1">

...at Homebound, we built

- A rich entity/domain model in TypeScript
- Using **GraphQL** as a *wire format* for the graph
- Fewer, lightweight resolvers
- Graph-based traversals, graph-based auth, etc.
- The graph comes first

😅

</div>

<div v-click="2">

**Joist** &mdash; a TypeScript ORM for Majestic Monoliths

</div>

<!--
I haven't worked at Facebook, but my understanding is that Ent already existed.

Coincidentally, at Homebound we...
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

**3. Schema**

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

---
layout: two-cols-header
---

# 1. Safe Graph Traversal

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

Starting with the domain mode., Joist's first killer feature is...

If your `someHelperMethod` does a `load`, it's still batched -- big change from
most ORMs, like ActiveRecord, where you have to hoist loading/populate hints to the top.

-->

---
layout: two-cols-header
---

# 2. Succinct Graph Traversal

<div class="text-lg -mt-1">"Load in a loop" is safe but ugly &mdash; instead populate shapes</div>

::left::

**Before** — `Promise.all` soup

```ts
const author = await em.load(Author, id);

// By default, relations force await usage
const books = await author.books.load();

// ...fine but boilerplately
const reviews = (await Promise.all(
  books.map(b => b.reviews.load())
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

---
layout: two-cols-header
---

# 3. Graph Validation

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

<!-- Because of the robust domain models, we can put entities on the wire ... and it's fun. -->
