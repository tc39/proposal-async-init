# Async Initialization

* Stage: 1
* Champion: Bradley Farias (GoDaddy)

## Motivation

Class features are unable to properly integrate with some async workflows in JS.
This proposal seeks to aid async initialization of class instances.

As stage 1, the current effort is into finding which design would be appealing
given our goals.

## Goals

* Encapsulate incomplete initialization
* Allow a reasonable subclass workflow

## Potential Paths

### Async Classes

Mark an entire class as `async` as well as the `constructor`:

```mjs
async class Query {
  async constructor() {
    this.data = await performQuery('select * from table;');
  }
}
```

This is somewhat nice as it does open a path towards `await` during field
initialization:

```mjs
const connection = db.connect();
async class Table {
  db = await connection;
  query(q) {
    return db.query(q);
  }
}
```

### Async Constructors

Only make `constructor` marked as `async`.

```mjs
class Query {
  async constructor() {
    await performQuery('select * from table;')
  }
}
```

This is minimal, but due to timing concerns.

### Private Constructors

```mjs
class Query {
  #constructor(data) {
    this.data = data;
  }
  async create() {
    return new Query.prototype.#constructor(
      await performQuery('select * from table;')
    );
  }
}
```

This is also somewhat minimal, but does not allow for subclasses to easily
extend the initialization since it is private.

## Common Problems Throughout

All the solutions face some level of commonality for problems:

### `await super()`

`await super()` is already valid so a new syntax would be needed in order to
differentiate how `this` should be assigned. For bike shed mitigation, I will
be using `await.super()` as a placeholder for now. It cannot be placed as a
meta-property on `super` because `super.*` is also already valid syntax.

### Timing and resumption of super()

Considering that many classes can perform meaningful actions prior to accessing 
superclass initialized behavior, it should be possible to queue such actions.

## FAQ

What other languages support `await` during construction and encapsulate the instance until construction finishes?

None found so far, but will happily document any. Lots of languages have various patterns for dealing with this problem space though. It seems somewhat common to have the following:

```mjs
class Instance {}
async function instanceFactory() {
  // awaits go here
  f(new Instance(data));
}
```

## How will this interact with abstract sub/super classes?

Unclear, but it seems that abstract classes would need to know timing.
Timing is needed to avoid races such as performing an action too early and
`async` initialization may affect the timing by which subclass `super` resumes.

This leans towards banning mixing of `async` & non-`async` initialization to
some extent. It seems unsafe for a non-`async` initialization to subclass an
async initialization, but the opposite has not yet seen a problem.
