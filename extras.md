---
description: Epoxy provides some extra goodies to simplify common use cases.
---

# Extras

## Actors

### Use Case

A common use case of Epoxy is creating a class that synchronizes the state of an Epoxy data structure with another data store, such as an IndexedDb table. In these cases the class needs to edit the data structure \(to load the initial set of data in and add more pages as necessary\) and respond to changes that external classes make so they can be synced back.

The problem here is that, by default, the synchronizer class will get all of its own modifications when it listens to changes in the structure. In the best case this just creates a little bit of extra processing overhead. In the worst case this leads to infinite loops of data modifications, potentially even involving network calls.

### Using Actors

Actors are wrappers around an Epoxy data structure that filter out their own modifications, meaning that the `.listen()`, `.asObservable()` and `.observables()` functions only resolve with new data when a _different_ actor changes the object \(or when code that doesn't use the Actors paradigm modifies it\).

```typescript
const listenable = makeListenable([]);
const listenableActor = asActor('test', listenable);

listenableActor.listen().subscribe((mutation) => {
    // Do things with the mutation
});

listenable.push(1); // The mutation listener code runs

listenableActor.push(2); // The mutation listener code does not run
```

## Readonly Data Structures

Sometimes it's necessary to pass data structure references to external code, or to code that could be error-prone. If that code starts making changes to your data that can make your formerly-elegant state flow into a spaghettified mess. The usual solution to this problem is to pass immutable references around, which can never be modified by anybody, ever. This paradigm doesn't jibe with Epoxy's design goal of only ever having one instance of your state data. Luckily, Epoxy has an even better - lighter weight and more transparent - solution to this problem.

```typescript
const baseArray = makeListenable([1, 2, 3, 4, 5, 6]);
const readonly = baseArray.asReadonly();

baseArray.push(7);
// readonly is now equal to [1, 2, 3, 4, 5, 6, 7];

readonly.push(8);
// readonly throws a ReadonlyException and nothing is modified.
```

Readonly objects present the same interface as normal Epoxy observables. They can be listened to and observed, they can be used in computed properties, and they can be used with non-modifying methods like `Array.map()`.

Behind the scenes they are implemented as another layer of Proxies around the original data structure, making them efficient since they never actually copy the underlying data.



