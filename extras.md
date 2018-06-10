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

## Transactions

{% hint style="info" %}
This feature is still under active development. It currently only works for users of the `.asObservable()` stream. The `.listen()` stream is not yet batched.
{% endhint %}

Sometimes it's necessary to make a bunch of different changes to the state at once. This works fine in Epoxy, of course, but it's a little inefficient because any agents listening to changes in the data structure will get triggered for every modification. For example if you add 100 items to an array, the `.asObservable()` stream on that array will trigger 100 times.

Transactions batch all of these operations together so that the listener only runs once, and also makes debugging easier by attaching a name to the agent that caused the mutations.

### In Javascript

```javascript
const listenable = makeListenable([]);
let lastMutation: Mutation<any>;
listenable.listen().subscribe((mutation) => lastMutation = mutation);

class TestFuncs {
    static pushABunchOfStuffToAList(list: Array<number>) {
        runTransaction('BulkAdd', () => {
            for (let i = 0; i < 100; i++) {
                list.push(i);
            }
        });
    }
}

TestFuncs.pushABunchOfStuffToAList(listenable);
expect(lastMutation.fromBatch).eqls('BulkAdd');
```

### In Typescript with Decorators

```typescript
const listenable = makeListenable([]);
let mutationCount = 0;
listenable.asObservable().subscribe(() => mutationCount++);

class TestFuncs {
    @Transaction()
    static pushABunchOfStuffToAList(list: Array<number>) {
        for (let i = 0; i < 100; i++) {
            list.push(i);
        }
    }
}

TestFuncs.pushABunchOfStuffToAList(listenable);
expect(mutationCount).equals(1);
```

{% hint style="info" %}
In the future this feature will also support error handling, so if a @Transaction function throws an error then all of the changes it made will be thrown out.
{% endhint %}



