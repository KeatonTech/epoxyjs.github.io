---
description: >-
  Why not impose some structure on your Epoxy use? It'll make it easier to
  debug, faster, and optionally atomic!
---

# Transactions & Batching

## Batching

If you have a function that makes a bunch of different changes to an object, you may want those changes to be logically grouped together and traceable back to the culprit. If you're a fan of stretched analogies, you could think of Batch Operations in Epoxy like Actions in Redux -- they are discreet, named things that change your state in predictable ways.

Beyond the added convenience for debugging, Batch Operations also increase efficiency. Epoxy holds off on emitting any mutations until the operation has finished, which means your code can run faster because it's not tied up in re-calculating computed properties or updating the UI or anything like that. Once your function completes Epoxy will automatically optimize the changes you made into a minimal set of mutations. For example, if your batch operation added 100 elements to the end of an array one-by-one, Epoxy will convert those 100 separate single-item mutations into 1 mutation with 100 items. That transformation again saves processing time for everything that depends on your data.

In Typescript you create a batch operation like this:

```typescript
import { BatchOperation } from 'epoxy';

@BatchOperation()
pushABunchOfStuffToAList(list: Array<number>) {
    for (let i = 0; i < 100; i++) {
        list.push(i);
    }
}
```

By default the operation takes its name from the function definition, but this may not be desirable in cases where debugging is likely to happen on minified code. Therefore it is also possible to explicitly give a name to the batch operation.

```javascript
@BatchOperation("Push Stuff to a List")
```

## Transactions

Transaction Operations are just batch operations with one particularly useful feature added on: If an error is thrown from any part of the operation, your state will not be changed at all. This is a property called **atomicity** that is common in databases.

```typescript
import {Transaction, makeListenable} from 'epoxy';

const listenable = makeListenable([]);

// Adds 100 items to the list but then throws an error
@Transaction()
static pushABunchOfStuffToAList() {
    for (let i = 0; i < 100; i++) {
        listenable.push(i);
    }
    throw new Error('test error');
}

pushABunchOfStyuffToAList();

// Note that ultimately no items were added to the list
expect(listenable.length).equals(0);
```

## For Javascript Users

In addition to the `@BatchOperation` and `@Transaction` decorators, Epoxy provides the `runInBatch` and `runTransaction` functions that can be used from older Javascript versions that do not support decorators. They support the same features and have largely the same syntax.

```javascript
runTransaction('BulkAdd', () => {
    for (let i = 0; i < 100; i++) {
        list.push(i);
    }
});
```

## Strict Operations Mode

For larger teams with more structured projects we recommend putting Epoxy into strict operations mode, which simply disallows mutations from happening outside of a batched operation \(or transaction\). This significantly cleans up the debugger view and allows linter rules to be added to restrict mutations to a certain set of files.

```typescript
EpoxyGlobalState.strictBatchingMode = true;
const listenableArray = makeListenable([]);
expect(() => listenableArray.push('Try this')).throws();
```

