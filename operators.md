---
description: >-
  Manipulate Epoxy objects using functional programming syntax – while
  maintaining their reactive qualities.
---

# Operators

Epoxy comes with the ```epoxyjs/operators``` library which includes collection manipulation functions common in functional programming, like map\(\) and filter\(\). The difference between these and the normal Javascript map\(\) and filter\(\) functions is that the output is also a listenable collection, so it will change as the original collection changes. This is accomplished iteratively behind the scenes -- Epoxy is not just re-running the function every time any change happens, it intelligently applies only the necessary changes to the manipulated collection.

## Map

`Array.map` is probably the most common, and the most basic, of the functional operations. It takes every element in an array and runs it through a transform function to produce a new array of the same length but with different values. In Epoxy it is possible to do this using Javascript's standard Array.map function inside of a computed\(\) property.

```typescript
const listenableArray = makeListenable([1, 1]);

// This works but don't actually do this! Keep reading for a better way.
const bigger = computed(() => listenableArray.map((val) => val * 10));

bigger.subscribe((biggerVal) => console.log(biggerVal));
                                // => [10, 10]
listenableArray.push(2);        // => [10, 10, 20]
```

The problem is this means every time any value in the array changes every single item will need to be re-mapped, which means running the mapper function again. In most cases this is probably fine since arrays in Javascript tend to be small and mapping functions tend to be simple.

Still, this should bother you on a deep level, like fingernails on a chalkboard. We know _exactly_ which properties are changing and when – so why are we blindly rerunning the map function for everything all the time?

That's a good question. Let's not!

```typescript
const things = makeListenable([
    "EpoxyJS",
    "Typescript",
    "Github"
]);

const awesomeThings = listenableMap(
    things,
    (thing) => `${thing} is awesome!`
);

expect(awesomeThings).eqls([
    "EpoxyJS is awesome!",
    "Typescript is awesome!",
    "Github is awesome!"
]);

things.push("listenableMap");
expect(awesomeThings).eqls([
    "EpoxyJS is awesome!",
    "Typescript is awesome!",
    "Github is awesome!",
    "listenableMap is awesome!"
]);
```

The `listenableMap` function here works the same way as `Array.map` but is driven by the underlying Mutation events so it can do a minimal amount of work in keeping the mapped data structure up to date. This also means that instead of outputting an Observable like the naive approach listed above it outputs a normal Epoxy data structure, complete with its own mutations stream.

{% hint style="info" %}
The data structures created by Operators are all [readonly](extras.md), meaning they will stay up to date with the input structure but will throw an error if you attempt to modify them directly.
{% endhint %}

### Computed Mapper Functions

The mapper function passed into `listenableMap` can rely on other Epoxy values, which will cause the output map to automatically update when those values are changed. This works using the same underlying mechanism as `computed()`.

```typescript
const things = makeListenable([
    "EpoxyJS",
    "Typescript",
    "Github"
]);

const state = makeListenable({
    adjective: "awesome",
});

const adjectiveThings = listenableMap(
    things,
    (thing) => `${thing} is ${state.adjective}!`
);

expect(adjectiveThings).eqls([
    "EpoxyJS is awesome!",
    "Typescript is awesome!",
    "Github is awesome!"
]);

state.adjective = "cool";
expect(adjectiveThings).eqls([
    "EpoxyJS is cool!",
    "Typescript is cool!",
    "Github is cool!"
]);
```

## Filter

Just like with map, using Javascript's `Array.filter()` function inside of `computed()` will work just fine – but isn't very efficient and loses all of the original mutation information. Generally though, the syntax of Epoxy's filter function is just like that of JS.

```typescript
const baseArray = makeListenable([1, 2, 3, 4, 5, 6]);
const filteredArray = filter(baseArray, (val) => val % 2 == 0);
expect(filteredArray).eqls([2, 4, 6]);
```

Unlike Javascript's built-in filter function, Epoxy's `filter()` works on objects \(this is also true of map\)!

```typescript
const epoxyObject: TypedObject<String> = makeListenable({
    "a": "Some Uppercase",
    "b": "all-lowercase",
    "c": "more-lowercase",
    "d": "A",
});
const filtered = filter(epoxyObject, (val) => val.toLowerCase() == val);
expect(filtered).eqls({
    "b": "all-lowercase",
    "c": "more-lowercase",
});

epoxyObject["e"] = "e";
expect(filtered).eqls({
    "b": "all-lowercase",
    "c": "more-lowercase",
    "e": "e",
});
```

Note that unlike map\(\), the filter function does not support computed filter functions, so if your filter function relies on Epoxy values it will not automatically update. This functionality might be added in the future.

## ToObject

Sometimes it is useful to be able to look up items in an array in constant-time, which usually necessitates turning the array into some sort of HashMap. Epoxy doesn't yet support the actual JS Map class, but Javascript objects are usually HashMaps under the hood \(sometimes they're tree maps but those are pretty good too\). Like all the other operators, this not only converts a listenable array to an object but also keeps it up to date as the array changes.

```typescript
const messages = makeListenable([
    {id: 'message1', text: 'Hello'},
    {id: 'message2', text: 'Hello to you too!'},
    ...
]);

const messagesById = toObject(messages, (message) => message.id);
expect(messagesById).eqls({
    "message1": {id: 'message1', text: 'Hello'},
    "message2": {id: 'message2', text: 'Hello to you too!'},
    ...
});
```

The second argument of the toObject function is a method that generates a key for the given array row, usually by pulling some sort of unique key field out of an object. Note that keys **must be unique**. Trying to insert an item into an array that has the same key as an existing item will result in an error.



