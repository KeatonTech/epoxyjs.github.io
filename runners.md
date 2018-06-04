---
description: >-
  Instead of listening to changes directly you can use Epoxy runners to
  automatically react.
---

# Runners

## Computed Values

`computed()` is probably the most straightforward of the Epoxy runners. It outputs an observable whose value updates whenever any of the dependencies change \(where dependencies here means 'any property of an Epoxy data structure that was accessed while calculating the value\).

```typescript
import {makeListenable, computed} from 'epoxyjs';

const state = makeListenable({
    a: 1,
    b: 2,
    c: 3,
});

const sum$ = computed(() => state.a + state.b);

sum$.subscribe((newSum) => console.log(newSum)); // => 3

state.a = 2;                                     // => 4
state.b = 0;                                     // => 2
state.c = 10;                                    // Nothing is logged
delete state.a;                                  // => 0
```

Epoxy keeps track of the dependencies on every run, so conditional statements are possible.

```typescript
import {makeListenable, computed} from 'epoxyjs';

const switcher = makeListenable({
    showA: true,
});

const state = makeListenable({
    a: 'I am a',
    b: 'This is b',
});

const value$ = computed(() => switcher.showA ? state.a : state.b);
value$.subscribe((value) => console.log(value));     // => 'I am a'

state.a = 'a is great!'                              // => 'a is great!'
state.b = 'b is great!'                              // Nothing is logged

switcher.showA = false;                              // => 'b is great!'
state.a = 'a is great!'                              // Nothing is logged
state.b = 'I love b'                                 // => 'I love b'
```

Note that in this case the computed value relies on two different Epoxy objects. Epoxy also tracks cases where an entire object or data structure is used, such as when the `.reduce` function is called on a listenable array.

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

### Optionally Computed

Sometimes you may find yourself creating a computed property that doesn't actually rely on any Epoxy variables \(this usually happens when user input is involved, or when you're building a general-purpose framework that wraps existing code in Epoxy runners\). The `optionallyComputed()` function works the same was as `computed()` but only returns an Observable object if the value relies on at least one Epoxy value. If it doesn't, `optionallyComputed()` simply returns the primitive value.

## Autorun

`autorun()` is similar to computed\(\) except that it doesn't return a value, it simply re-runs the entire code block whenever any of the dependencies update. This is useful for side effects like integrating Epoxy with frontend JS frameworks.

```typescript
import {makeListenable, autorun} from 'epoxyjs';

const state = makeListenable({
    value: 4,
});

let lastStateValue: number;
autorun(() => {
    lastStateValue = state.value;
})
expect(lastStateValue).eqls(4);

state.value = 5;
expect(lastStateValue).eqls(5);
```

The `autorun` function returns another function that cancels the autorun when it is called, similar to how setTimeout returns a token that can be used to cancel it.

```typescript
import {makeListenable, autorun} from 'epoxyjs';

const state = makeListenable({
    value: 4,
});

let lastStateValue: number;
const cancelAutorun = autorun(() => {
    lastStateValue = state.value;
})
expect(lastStateValue).eqls(4);

cancelAutorun();
state.value = 5;
expect(lastStateValue).eqls(4);
```

All of the same concepts of getter tracking apply to `autorun` as apply to `computed`, in a sense it's just an API for doing the same thing but without involving rxjs. Autorun does have one extra trick up its sleeve though:

### autorunTree

Autorun functions can contain nested `autorunTree` functions inside them. These functions can use variables from the outer function's scope, but keep their epoxy dependencies contained. In other words: `autorunTree` can divide up a large `autorun` function into smaller parts that can be auto-run independently, preventing unnecessary work from being done.

For example, this function creates an HTML `<ol>` containing the items of a listenable list. When the value of an individual list item is updated it can simply update the one HTML element that was affected. When the length of the list changes the entire list will be re-rendered.

```typescript
import {makeListenable, autorun, autorunTree} from 'epoxyjs';

const shoppingList = makeListenable([
    'Bananas',
    'Gravy',
    'A hat'
]);

const orderedList = document.createElement('ol');
document.body.appendChild(orderedList);

autorun(() => {
    orderedList.innerHTML = ''; // Clear the existing list.
    
    for (let i = 0; i < shoppingList.length; i++) {
        const itemElement = document.createElement('li');
        orderedList.appendChild(itemElement);
        
        autorunTree(() => {
            orderedList.innerText = shoppingList[i];
        });
    }
});
```

Cancelling the outer `autorun` function will also cancel any nested `autorunTree` functions.

