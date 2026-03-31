# Usage

Examples of how SDM should be used.

For the following examples, we'll be Engineers for an online Bookstore.

# Before Testing

Simply create a `domain.sdm.ts`:

```ts
class Author {
  id: string
  name: string
  books: Book[]
  series: Series[]
}

class Book {
  id: string
  title: string
  author: Author
  series?: Series             // optional
  isPublished: boolean = true // defaults to true
}

class Series {
  id: string
  author: Author
  books: Book[]
}
```

Then run `npm run sdm:generate` with `"sdm:generate": "sdm generate --files ./domain.sdm.ts"` to create your Factories

# For Testing

SDM's goal is that test data should highlight what's important **and nothing else**. Your tests should read like exactly what you need with 0 clutter.

```ts
// "I need an author with two books"
const author = newAuthor({ books: [{}, {}] })
// "I need a book"
const book = newBook({})
// "I need a series with three books"
const series = newSeries({ books: [{}, {}, {}] })
```

Books need authors; your tests don't. All of the above factories quietly generated and linked the required relationships silently. But it's important to understand how.

And somewhere in `setupTests.ts` (vite, jest) `afterEach(() => sdm.reset())`

## Relation Linking

Entities with a non-optional relationship will generate it. Books needs authors, so if you call `newBook({})` it will call `newAuthor({})` under the hood. Books do not, however, "need" a series, so `newSeries({})` does not get invoked since we declared `series?: Series` in our above Book class.

That only scales so far. Consider:

```ts
newBook({}) // needs an Author, so generate it
newBook({}) // needs an Author, so... generate a second one?
// If ☝️ were true, then what would these do:
newAuthor({ books: [{}, {}] }) // does this generate 3 authors (newAuthor itself + two books?)
newSeries({ author: {}, books: [{}, {}, {}] }) // does this generate 4?
```

How does it _actually_ work?

If a Book needs an Author and...
1. NONE exist - Create it
1. ONE exists - Reuse it
1. 2+ exist - Ambiguous, can't possibly know which to reuse, create another

Entity auto-generation follows some basic guidelines based on observed usage. By "observed usage" we mean "Any time this question comes up, LOOK at the test and try to estimate what the developer would've wanted." Overwhelmingly it either "doesn't matter" (I was creating books, not Authors) or it's plainly evident it should've linked to the same author.

If each book should really have its own author, simply call `newBook({ author: {} })`. Providing `author: {}` will force new Author creation.

> [!CAUTION]
> Consider these cases:
> - `newBook({ author: {}, series: { /* needs an author */ } })` how do we tell `series` to reuse our new `author`
> - `newBook({ author: {}, series: { author: {} }})` what is this even attempting to say?
> - `twoOf(() => newBook({ author: {}, series: {} }))` "two books from two different authors and on two different series" how do we tell `series` to reuse _that book's_ author?

## Multiple Ways to Declare

Under this framework it's very easy to describe the same thing multiple ways:

```ts
// "I need an author with two books"
const a = newAuthor({ books: [{}, {}] })
const [b1, b2] = a.books
// I need two books
const b1 = newBook({}) // creates Author#1
const b2 = newBook({}) // reuses Author#1
```

Or maybe with collections:

```ts
// "I need an author with 1 series with 1 book
newAuthor({ series: [{ books: [{}] }] })
// or
newBook({ series: {} }) // Needs an Author, also creates a Series, links everything as expected
// or
newSeries({ books: [{}] }) // Needs an author, creates 1 book
```

As always, your test itself should dictate what "makes sense" to call. If you want to test how your utility will handle processing a Series with two books, your test should read like that.

## Keep it Tight

A single call to `newBook({ author: {} })` and `newBook({})` are logically equivalent (multiple calls may not be). Only define the author when: For some reason it reads well in your test, or you intentionally want to create multiple authors (for multiple books).

It could also make sense to call `newBook({ author: { name: "CS Lewis" } })` to give the author a specific name. They're your tests - write what makes sense.

# What SDM Isn't

## Not for App Testing

SDM came about from a desire to test libraries and utilities with "realish" data, not for full Backend or Frontend applications. YMMV

We were writing an internal library for some data processing. The library itself could be plugged into Frontend or Backend code and normal testing (GraphQL Factories, Swagger/OpenAPI Factories, Faker, MSW mocks, etc.) could proceed with integration tests. But the library itself needed tests too. We wanted to showcase "What would using these tools 'realistically' look like," so we generated some "Authors & Books" tests. Shortly thereafter, we realized we could just use our _actual_ Domain (Projects, TaskLists, and Tasks) to model how this library could be used, but we wanted to keep it lightweight. "It should be easy," we told ourselves, "to define a Simple Domain Model to showcase via unit tests how to use these tools." Thus SDM was born.

# Coming Eventually, Maybe

## Faker

```ts
class User {
  emailAddress: string
  birthdate: Date
  isAdmin: boolean = false

  static factoryDefaults = {
    emailAddress: () => faker.internet.email(),
    birthdate: () => faker.date.birthdate({ mode: 'age', min: 18, max: 65 }),
  }
}
```
