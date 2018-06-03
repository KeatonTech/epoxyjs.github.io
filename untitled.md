---
description: >-
  Objects are one of the two core data structures in Javascript (alongside
  Arrays). Epoxy objects work the same way as any other object – with some
  special tricks up their figurative sleeves.
---

# Epoxy Objects

## Creating Epoxy Objects

To create an Epoxy object, simply pass an existing Javascript object into the makeListenable\(\) function.

```javascript
import {makeListenable} from 'epoxyjs';

const baseScore = {
    hits: 10,
    bonuses: 0,
    multiplier: 1.0,
};

const listenableScore = makeListenable(baseScore);
```

The type information of the input object is preserved, so Typescript users can take advantage of Epoxy objects without losing any type safety.

```typescript
import {makeListenable} from 'epoxyjs';

interface PlayerScore {
    hits: number;
    bonuses: number;
    multiplier: number;    
}

const listenableScore = makeListenable({
    hits: 10,
    bonuses: 100,
    multiplier: 3.5,
} as PlayerScore);

// listenableScore has type `PlayerScore & Listenable`
```

Epoxy does not require objects to have an explicitly-defined set of properties, so it's possible to use them as maps – again, just as you would with vanilla Javascript objects.

```typescript
interface PlayerMap {
    [playerName: string]: PlayerScore
}

const listenablePlayerMap = makeListenable({} as PlayerMap);
listenablePlayerMap['alice'] = {
    hits: 1000,
    bonuses: 1,
    multiplier: 0.5,
};
```

Ultimately, the secret of Epoxy objects is that they _are_ just normal Javascript objects. Epoxy just wraps them in a lightweight proxy layer that detects whenever you add, edit, or delete a property. This means you can pass Epoxy objects to any existing Javascript framework that expects an object as an input and everything will work!

## Computed values

Obviously none of the previous examples on this page are particularly interesting. Thus far you haven't actually seen the `makeListenable` function _do_ anything, it seems as though it just gives you back the original object and sets you on your merry way. In a sense, that's by design. The goal of Epoxy is to stay out of your way until you need it.

So, let's look at some examples of what you can do with listenable objects. The most common use case is to create a computed value that depends on properties of an object.

```typescript
import {computed, makeListenable} from 'epoxyjs';

const scoreObject = makeListenable({
    hits: 10,
    bonuses: 0,
    multiplier: 1.0,
});

const totalScore$ = computed(() => {
    return (scoreObject.hits + scoreObject.bonuses) * scoreObject.multiplier;
});

totalScore$.subscribe((newTotalScore) => {
    console.log(`Score: ${newTotalScore}`);
});
                            // => Score: 10
scoreObject.hits = 11;      // => Score: 11
scoreObject.bonuses = 4;    // => Score: 15
scoreObject.multiplier = 2; // => Score: 30
```

The `computed` function creates an rxjs Observable that updates whenever one of the values it depends on is modified. This works by keeping track of which properties were accessed across all Epoxy data structures, so a single computed observable could depend on multiple Epoxy objects and Epoxy arrays. Crucially, the observable will _not_ update when a value it doesn't depend on is changed.

```typescript
totalScore.unnecessaryField = "Why am I here?" // => Nothing is logged
```

### Computed properties

One special feature of Epoxy objects is that you can create properties from rxjs Observables. The property value will always equal the most recent value from the Observable, at least until you explicitly set the property to something else.

```typescript
import {makeListenable} from 'epoxyjs';
import {BehaviorSubject} from 'rxjs';

const value = new BehaviorSubject<number>(10);
const store = makeListenable({});
store.value = value.asObservable();

expect(store.value).equals(10); // The Observable is unwrapped.

value.next(11);
expect(store.value).equals(11); // and the value is kept up to date
```

**But wait!** If computed properties are rxjs Observables and rxjs Observables can be used to control object values, then...

```typescript
import {computed, makeListenable} from 'epoxyjs';

const myScore = makeListenable({
    hits: 10,
    bonuses: 0,
    multiplier: 1.0,
});

myScore.total = computed(() => {
    return (scoreObject.hits + scoreObject.bonuses) * scoreObject.multiplier;
});

expect(myScore.total).equals(10);

myScore.bonuses = 3;
expect(scoreObject.total).equals(13);
```

And these new computed values also work in _other_ computed values!

```typescript
// Winner is an Observable that updates with the name of the player who is
// currently winning (assuming the players are logically named 'me' and 'you').
const winner$ = computed(() => {
    if (myScore.total > yourScore.total) {
        return 'me';
    } else {
        return 'you';
    }
});
```

## Listening for changes

There are a few different ways to directly listen for changes on an object, without creating computed properties. These techniques can be used in much the same way but are technically more efficient because they don't rely on global getter tracking \(although in practice it usually doesn't matter too much\). They also allow you more flexibility in figuring out exactly what changed and responding to it accordingly.

The simplest way to listen for changes is simply to convert the object into an observable.

```typescript
import {makeListenable} from 'epoxyjs';

const myScore = makeListenable({
    hits: 10,
    multiplier: 1.0,
});

const myScore$ = myScore.asObservable();
myScore$.subscribe((newScore) => console.log(newScore));

                            // => {hits: 10, multiplier: 1.0}
myScore.hits = 11;          // => {hits: 11, multiplier: 1.0}
myScore.bonus = 5;          // => {hits: 11, multiplier: 1.0, bonus: 5}
delete myScore.multiplier;  // => {hits: 11, bonus: 5}
```

The `myScore$` observable updates with the entire \(copied\) state of the object whenever anything about it changes, including properties and sub-properties. You can also listen to changes in the value of just one property.

```typescript
import {makeListenable} from 'epoxyjs';

const myScore = makeListenable({
    hits: 10,
    multiplier: 1.0,
});

const hits$ = myScore.observables().hits;
hits$.subscribe((newHits) => console.log(newHits));

                            // => 10
myScore.hits = 11;          // => 11
myScore.multiplier = 2.0;   // Nothing is logged
delete myScore.hits;        // => undefined
```

These techniques are great if you just want to know when something changes, but sometimes it's necessary to know exactly _what_ changed. Thankfully, Epoxy has a solution for that in the form of Mutations.

### Mutations

```typescript
import {makeListenable, PropertyMutation} from 'epoxyjs';

const myScore = makeListenable({
    hits: 10,
    multiplier: 1.0,
});

const myScoreMutations$ = myScore.listen();
myScoreMutations$.subscribe((mutation) => console.log(mutation));

myScore.hits = 11;          // => PropertyMutation {
                            //        key: 'hits', 
                            //        newValue: 11,
                            //        oldValue: 10 }

myScore.bonus = 5;          // => PropertyMutation {
                            //        key: 'bonus', 
                            //        newValue: 5,
                            //        oldValue: undefined }
                                                        
delete myScore.multiplier;  // => PropertyMutation {
                            //        key: 'multiplier', 
                            //        newValue: undefined,
                            //        oldValue: 1.0 }
```

Epoxy also tracks mutations that occur inside nested objects, making things like this possible.

```typescript
import {makeListenable, PropertyMutation, SubpropertyMutation} from 'epoxyjs';

const scores = makeListenable({
    player1: {
        hits: 10,
        multiplier: 1.0,
    },
    player2: {
        hits: 6,
        multiplier: 1.0,
    }
});

const scoreMutations$ = scores.listen();
scoreMutations$.subscribe((mutation) => console.log(mutation));

scores.player1.hits = 11;   // => SubpropertyMutation {
                            //        key: 'player1', 
                            //        mutation: PropertyMutation { 
                            //            key: 'hits',
                            //            newValue: 11,
                            //            oldValue: 10 }}
```

Mutations are the way Epoxy tracks changes internally, and all of it is available to you for your change-detecting pleasure.

