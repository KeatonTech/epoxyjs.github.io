---
description: >-
  Arrays, like Objects, are one of the two core Javascript data structures.
  Epoxy Arrays are just normal Javascript Arrays, but with some extra magic
  bolted on.
---

# Epoxy Arrays

## Creating Epoxy Arrays

To create an Epoxy array, simply pass an existing Javascript array into the makeListenable\(\) function.

```javascript
import {makeListenable} from 'epoxyjs';
const listenablePrices = makeListenable([1, 1, 2, 3]);
```

The type information of the input object is preserved, so Typescript users can take advantage of Epoxy arrays without losing any type safety.

```typescript
import {makeListenable} from 'epoxyjs';

interface ToDoItem {
    name: string;
    complete: boolean;
}

const toDoList: ToDoItem[] = [];

const listenableList = makeListenable(toDoList);

// listenableList has type `ToDoItem[] & Listenable`
```

Epoxy arrays are just normal Javascript arrays, so they have all the functions you know and loathe, like `.splice()`, `.push()`, `.unshift()`, `.map()`, and everything else.

Note that objects within arrays, just like objects within objects, are automatically converted to be listenable. This means that not only will the listenableList array detect when a new item is added or removed, it will also detect when any individual item is modified \(for example, to mark an individual task as completed\). This is also true of arrays within arrays, by the way.

## Computed values

The computed\(\) function works with arrays in the same way it works with objects.

```typescript
import {computed, makeListenable} from 'epoxyjs';

const prices = makeListenable([1, 1, 2, 3]);

const sum$ = computed(() => prices.reduce((i, a) => i + a));
sum$.subscribe((newSum) => console.log(`Subtotal: ${newSum}`));

                            // => Subtotal: 7
prices.push(10);            // => Subtotal: 17
prices.splice(0, 1);        // => Subtotal: 16
prices[0] = 3;              // => Subtotal: 18
```

The `computed` function creates an rxjs Observable that updates whenever one of the values it depends on is modified. This works by keeping track of which properties were accessed across all Epoxy data structures, so a single computed observable could depend on multiple Epoxy objects and Epoxy arrays.

### Reactive fibonacci

If you read the docs on [Epoxy objects](untitled.md#computed-values) you'll remember that rxjs Observables can be used to drive object properties. This is also true for values in arrays! And, just like with objects, this means that values within arrays can be computed from other values in the array. Here we use this trick to make a reactive fibonacci sequence where the entire sequence updates whenever either of the two base states \(1 and 1 for the usual sequence\) are changed.

```typescript
import {computed, makeListenable} from 'epoxyjs';

const fibonnaci = makeListenable([1, 1]);
for (let i = 0; i < 10; i++) {
    const n = i; // Make a new variable for each iteration.
    fibonnaci.push(computed(() => fibonnaci[n] + fibonnaci[n + 1]));
}

expect(fibonnaci).eql([1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89, 144]);

fibonnaci[0] = 0;
expect(fibonnaci).eql([0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89]);

fibonnaci[1] = 0;
expect(fibonnaci).eql([0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0]);
```

## Listening for changes

There are a few different ways to directly listen for changes on an array without creating computed properties. The simplest way to listen for changes is simply to convert the entire array into an observable.

```typescript
import {makeListenable} from 'epoxyjs';

const prices = makeListenable([1, 1, 2, 3]);
const prices$ = prices.asObservable();
prices$.subscribe((newPrices) => console.log(newPrices));

                            // => [1, 1, 2, 3]
prices.push(5);             // => [1, 1, 2, 3, 5]
prices.splice(0, 1);        // => [1, 2, 3, 5]
prices[3] = 4;              // => [1, 2, 3, 4]
```

The `prices$` observable updates with the entire \(copied\) state of the array whenever anything about it changes, including individual items and their sub-properties. You can also listen to changes in the value of just one index.

```typescript
import {makeListenable} from 'epoxyjs';

const prices = makeListenable([1, 1, 2, 3]);
const firstPrice$ = prices.observables()[0];
firstPrice$.subscribe((newPrice) => console.log(newPrice));

                            // => 1
prices[0] = 0;              // => 0
prices[1] = 0;              // Nothing is logged
prices.splice(0, 1);        // => 1 (the first item is now what used to be the second)
```

These techniques are great if you just want to know when something changes, but sometimes it's necessary to know exactly _what_ changed. Thankfully, Epoxy's mutations work for arrays just as they do for objects, and are even more powerful here.

### Mutations

```typescript
import {makeListenable, ArraySpliceMutation, SubpropertyMutation} from 'epoxyjs';

interface ToDoItem {
    name: string;
    complete: boolean;
}

const toDoList: ToDoItem[] = [
    {name: 'Finish Epoxy Docs', complete: false},
    {name: 'Finish ngEpoxy', complete: false},
];

const listenableList = makeListenable(toDoList);

const listMutations$ = listenableList.listen();
listMutations$.subscribe((mutation) => console.log(mutation));

listenableList.push({
    name: 'Demonstrate ArraySpliceMutation',
    complete: true,
});                         // => ArraySpliceMutation {
                            //        key: 2,   // The index of the inserted item
                            //        deleted: [],
                            //        inserted: [{
                            //            name: 'Demonstrate ArraySpliceMutation',
                            //            complete: true
                            //        }]}

listenableList.splice(2, 1);
                            // => ArraySpliceMutation {
                            //        key: 2,   // The index of the deleted item
                            //        deleted: [{
                            //            name: 'Demonstrate ArraySpliceMutation',
                            //            complete: true
                            //        }],
                            //        inserted: [] }
                                                        
listenableList[0].complete = true;
                            // => SubpropertyMutation {
                            //        key: 0, 
                            //        mutation: PropertyMutation {
                            //            key: 'complete',
                            //            newValue: true,
                            //            oldValue: false,
                            //        }}
```

