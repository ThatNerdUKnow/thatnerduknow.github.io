+++
title = "Typescript Is Magic"
date = "2022-09-01T17:26:00-05:00"
author = "Brandon Piña"
authorTwitter = "" #do not include @
cover = ""
tags = ["programming", "typescript"]
keywords = ["typescript"]
description = "Using *Intersection Types* and *Template Literals* in a record type allows us to derive an object's type based off of it's key. This is especially useful when you don't know how many keys may exist on this object at build time with the caveat that the keys must follow a predictable pattern"
showFullContent = false
hideComments = false
+++
Typescript somehow always manages to find new and exciting ways to surprise me. It's no secret that investing time in defining good types for your projects is well worth it as you get better intellisense and lower cognitive load when dealing with your custom types. (*What properties exist on this object again?*) However, sometimes it can be difficult to define a type for an upstream service, like an API. If you know what keys/types to expect beforehand you might define your types as such:

```ts
type APIResponse = {
    data: APIData | APIError; // Data returned by the api could be valid or not
    headers: Record<string, string>;
    timestamp: Date;
    motd: string;
}

// Therefore we need to know what properties to expect
type APIData = {
    foo: 'bar'
}

type APIError = {
    message: 'You done goofed'
}
```
## The problem
But what if you don't know what keys exist at build time? Like in the case an api batch endpoint or a configuration file? What if certain key patterns map to certain types?  Enter ***Template Literals*** and ***Intersection types***.
```ts
// Define individual record types with a key matching a pattern 
// defined by a template literal and the corresponding type
type stringRecord = Record<`String${number}`,string> // Properties with keys matching "String${number}" will be of type string
type numberRecord = Record<`Number${number}`,number> // Properties with keys matching "Number${number}" will be of type number

// An intersection type rather than a union type is necessary
// Because the record types need to coexist
// rather than being of one type or the other
type intersectionRecord = stringRecord & numberRecord;

let foo: intersectionRecord = {};

foo[`String${1}`] = "Bar" // Valid
foo[`Number${1}`] = 1 // Also Valid
```
Here's where things get interesting. *Typescript can warn us of invalid uses of our type* on both our keys and values of the record.
```ts
// Invalid assignment to valid key
foo[`Number${2}`] = "Baz" // Type 'string' is not assignable to type 'number'.
foo[`String${2}`] = 1 // Type 'number' is not assignable to type 'string'.

// Assignment to invalid key
foo[`NumberA`] = 3 // Property 'NumberA' does not exist on type 'intersectionRecord'.
foo[`StringA`] = "Foo" //  Property 'StringA' does not exist on type 'intersectionRecord'.
foo[`Invalid Key`] //  Property 'Invalid Key' does not exist on type 'intersectionRecord'.
```
## Okay, so what?
Imagine if you will our api provides us a batch endpoint which we may make multiple requests and make subsequent requests based on other requests in the batch. We may request different ***types*** of resources and a single union type record: `Record<string, ResourceA|ResourceB|etc.>` type may not be suitable for our needs. If we can derive the type we expect from the ***key*** of the object instead of using a union type record, we can provide an extra level of type safety!

## TLDR
Using *Intersection Types* and *Template Literals* in a record type allows us to derive an object's type based off of it's key. This is especially useful when you don't know how many keys may exist on this object at build time with the caveat that the keys must follow a predictable pattern