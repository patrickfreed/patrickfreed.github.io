---
layout: post
title: "Unlocking greater performance in the MongoDB Rust Driver via raw BSON and zero copy deserialization"
date: 2022-04-27
category: rust
tags: [rust, bson, mongodb, benchmarking, profiling, performance]
---

The `2.2.0` release of the Rust BSON library (the [`bson`](https://crates.io/crates/bson) crate) introduced a "raw" BSON API, which enabled us to achieve some internal performance improvements in the Rust MongoDB driver (the [`mongodb`](https://crates.io/crates/mongodb) crate) and, in some cases, can be leveraged by users to dramatically improve performance of their queries, including via the use of [`serde`](https://serde.rs/)'s zero-copy deserialization functionality. In this post, I'll demonstrate how to use this new API and provide some examples of where it can help speed up your reads.

## Index
- [What is "raw" BSON?](#what-is-raw-bson)
- [Overview of the raw BSON API](#overview-of-the-raw-bson-api)
- [Speeding up your queries using raw BSON](#speeding-up-your-queries-using-raw-bson)
    - [Further speedups via zero-copy deserialization](#further-speedups-using-zero-copy-deserialization)
- [Notable differences between `RawDocumentBuf` and `Document`](#notable-differences-between-rawdocumentbuf-and-document)
    - [Validating BSON](#validating-bson)
    - [Looking up keys](#looking-up-keys)
    - [Inserting new elements](#inserting-new-elements)
- [Conclusion](#conclusion)
- [Acknowledgments](#acknowledgments)

## What is "raw" BSON?

Before we jump into any examples, I first want to clarify what I mean by "raw" BSON. To take a step even further back, I want to describe what BSON is generally. [BSON](https://bsonspec.org) stands for "Binary JSON" (kind of), and it's a binary format describing ordered maps of strings to various types, notably more types than JSON supports (for example, BSON has datetimes). It's used by MongoDB to store data, to communicate with drivers, and in its query language. The `bson` crate prior to `v2.2.0` already had the `Bson` and `Document` types to model BSON values and maps (called documents) respectively, but they aren't "raw" in the sense that their actual bytes in memory aren't in BSON. For example, a `Document` actually contains an [`indexmap::IndexMap<String, Bson>`](https://docs.rs/indexmap/latest/indexmap/index.html), which itself contains a hash table and other Rust data structures, and while Document can be serialized to and deserialized from its raw BSON equivalent, it doesn’t actually contain those bytes itself. The new "raw" BSON document type, which was introduced in `2.2.0` as `RawDocumentBuf`, instead just contains a `Vec<u8>` of bytes that correspond to actual BSON. The main benefit of this type is that when reading a document from the database, the bytes can be just copied as-is instead of being deserialized key-by-key to a Rust data structure like a `Document`, which can be very time consuming.

## Overview of the raw BSON API

As mentioned above, `v2.2.0` of `bson` introduces the [`RawDocumentBuf`](https://docs.rs/bson/latest/bson/raw/struct.RawDocumentBuf.html) type, which is an owned raw document type. Because it owns the BSON bytes that back it, it can be mutated via the `append` method, which adds a new key value pair. `RawDocumentBufs` can also be created via the `rawdoc!` macro, which behaves similarly to the existing `doc!` macro.

``` rust
let mut doc = RawDocumentBuf::new();
doc.append("a key", "a value");
doc.append("an integer", 12i32);
println!("{:?}", doc.get("a key")); // prints "Ok(Some(String("a value")))"

let other = rawdoc! {
    "a key": "a value",
    "an integer": 12i32
};
assert_eq!(doc, other);
```

To allow for accessing borrowed documents, including subdocuments borrowed from other documents, there's also the unsized [`RawDocument`](https://docs.rs/bson/latest/bson/raw/struct.RawDocument.html) type, which is only used via a wide-pointer `&RawDocument`. This type includes all the same methods that `RawDocumentBuf` does minus the mutating ones, since again it is only used as an immutable reference. Instances of this can be created via `RawDocument::from_bytes` or via the `get` / `get_document` methods on other documents.

``` rust
let doc = RawDocument::from_bytes(b"\x13\x00\x00\x00\x02hi\x00\x06\x00\x00\x00y'all\x00\x00")?;
assert_eq!(doc.get("hi")?, Some(RawBsonRef::String("y'all")));

let top = rawdoc! {
    "top_key": {
        "some_key": 12,
        "other": true
    }
};

// no clones are performed when retrieving the subdocument nor when iterating over it
let subdoc = top.get_document("top_key")?;
for kvp in subdoc {
    let (k, v) = kvp?;
    println!("{} = {}", k, v);
}
```

Individual values are modeled via the [`RawBson`](https://docs.rs/bson/latest/bson/raw/enum.RawBson.html) enum, which is similar to the existing `Bson` enum except that all the variants are "raw" (e.g. `RawBson::Document` contains a `RawDocumentBuf` instead of a `Document`). The reference version of this type is `RawBsonRef<'a>`, and it only contains references to owned raw BSON values. Instances of `RawBson` can be created via the `rawbson!` macro, which behaves similarly to the existing `bson!` macro.

``` rust
let mut doc = RawDocumentBuf::new();
doc.append("key", RawBson::String("a".to_string());
doc.append("other", rawbson!("a"));
let s = doc.get("key")?; // gets a reference, no copy here
println!("{:?}", s); // prints "Some(String("a"))"
```

The raw BSON API also contains support for borrowed deserialization via [`serde`](https://serde.rs/), which can greatly speed up populating structs from BSON by skipping expensive copies.

``` rust
let doc = rawdoc! {
    "key": "value",
};

#[derive(Deserialize)]
struct Data<'a> {
    #[serde(borrow)]
    key: &'a str
}

// borrows from the input bson rather than copying
let d: Data = bson::from_slice(doc.as_bytes())?;
assert_eq!(d.key, "value");
```

## Speeding up your queries using raw BSON

Simply by upgrading to the `2.2.0` release of the driver, you'll already have sped them up quite a bit! This was due to some optimizations we were able to make internally to the driver by using the new types. For example, [this benchmark](https://github.com/patrickfreed/raw-bson-benchmarks/blob/main/benches/rawbson.rs#L52-L74), which finds all 10,000 documents in a collection, performs 12% faster after bumping the version of `mongodb` from `2.1.0` to `2.2.0` without any changes in the code! (For more info on how that benchmark works and profiling Rust code in general, check out my [previous blog post](https://patrickfreed.github.io/rust/2021/10/15/making-slow-rust-code-fast.html)).

If you'd like to leverage the raw BSON API to unlock further performance improvements, you can do so by using `RawDocumentBuf` as the generic type of your collection and the cursors returned from it. This will greatly speed up queries where you don't need to perform lots of key lookups, since in those cases using raw BSON allows the driver to avoid parsing the individual key value pairs more than necessary (e.g. a situation where you just serialize the results of the query straight to JSON to be served to the frontend).

For example, given the following code:

``` rust
let coll = db.collection::<Document>("docs");
let docs = coll.find(None, None).await?.try_collect::<Vec<_>>().await?;
serde_json::to_string_pretty(&docs)?
```
Simply updating `Document` to `RawDocumentBuf` results in a 70% speedup while still yielding the exact same result!

``` rust
let coll = db.collection::<RawDocumentBuf>("docs");
let docs = coll.find(None, None).await?.try_collect::<Vec<_>>().await?;
serde_json::to_string_pretty(&docs)?
```

See [here](https://github.com/patrickfreed/raw-bson-benchmarks/blob/main/benches/rawbson.rs#L76-L103) for benchmarks demonstrating this. Note that `RawDocumentBuf` works well here because no key lookups are performed on the result documents. If the document needs to be inspected a lot, it's still preferable, ergonomically and for performance reasons, to use a struct which implements `Deserialize` that models your data instead.

Also note that there is such a big difference in this case because we're returning the entire collection (10k documents). In cases where only a few results need to be returned, using `RawDocumentBuf` may not yield as much or any improvements. 

### Further speedups via zero-copy deserialization

The `serde` framework includes support for [borrowing data from the input during deserialization](https://serde.rs/lifetimes.html#understanding-deserializer-lifetimes) (a.k.a. zero-copy deserialization), which in some cases can be used to avoid large copies and greatly speed things up. These cases are generally when the documents being deserialized from are really big and/or include fields that are really large (e.g. big strings or binary values). Starting in `2.2.0`, the `Cursor` type now includes methods for borrowing from the underlying result set, enabling users to take advantage of this functionality:

``` rust
#[derive(Debug, Deserialize)]
struct Stuff<'a> {
    #[serde(borrow)]
    name: &'a str,
    #[serde(borrow)]
    bio: &'a str
}

let coll = db.collection::<Stuff>("stuff");
let mut cursor = coll.find(None, None).await?;
while cursor.advance()?.await {
    println!("{:?}", cursor.deserialize_current()?);
}
```

Note that for a lot of workloads, the time spent server-side processing the query and the network latency dwarf the time spent during deserialization, so borrowing during deserialization won't make a big difference in those cases. However, in cases that involve huge documents, the difference can be quite significant. For a benchmark demonstrating this, see [here](https://github.com/patrickfreed/raw-bson-benchmarks/blob/main/benches/rawbson.rs#L165-L187).

## Notable Differences between `RawDocumentBuf` and `Document`

When considering whether the raw BSON API is right for your use case, it may be helpful to consider the following differences between `RawDocumentBuf` and `Document`.

### Validating BSON

In order to construct a `Document` from BSON, the bytes need to be parsed into Rust types (`String` and `Bson`) up front. For `RawDocumentBuf`, the parsing happens lazily as the document is iterated, meaning invalid BSON bytes could be encountered during iteration. As a result, the various ways of accessing elements in a raw BSON document can fail and thus return `Result<RawBsonRef>`. This differs from `Document`, which can return just `Bson` (and equivalents), since all the parsing was done up front.

``` rust
// ensures that bytes contains valid BSON, parses it all out 
let doc: Document = bson::from_slice(bson_bytes)?;

// only performs simple bounds checking, may have invalid bytes in the middle encountered during iteration
let raw_doc: RawDocumentBuf = bson::from_slice(bson_bytes)?;

for (k, v) in doc {
    println!("{}: {:?}", k, v); // prints <key>: Bson::<type>
}

for kvp in &raw_doc {
    println!("{:?}", kvp); // prints Ok(RawBsonRef) or Err(...)
}
```

### Looking up keys

Because `RawDocumentBuf` contains BSON instead of a hashmap-like type, lookups by key involve traversing the whole document from the front (i.e. linear time complexity). This means that the performance of `RawDocumentBuf::get` is potentially a lot slower than that of `Document::get`, especially if there are a lot of keys and the key being looked up is at the end of the document.

### Inserting new elements

Because key lookups can be so expensive, inserting a new element to a `RawDocumentBuf` would be expensive too if it first checked to see if the key existed already (which `Document` does). To avoid this, instead of an `insert` method, `RawDocumentBuf` only has an `append` method that just tacks the new key-value-pair to the end without checking to see if the key exists in the document already. This is super fast, but users have to be careful not to accidentally append two of the same key.

## Conclusion

The raw BSON API is included in version `2.2.0` of the `bson` crate, and support for working with it was introduced in the driver in its version `2.2.0`. Check them out and let us know how they're working for you by filing an issue on [GitHub](https://github.com/mongodb/bson-rust) or on our [Jira project](https://jira.mongodb.org/browse/RUST). Thanks!

## Acknowledgments

Huge shout out to community member [@jcdyer](https://github.com/jcdyer) (author of the [`rawbson`](https://crates.io/crates/rawbson) crate from which much of the raw BSON code in `bson` is derived) who has long spearheaded this effort!
