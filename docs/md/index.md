> :Banner


Fast and precise reactive state management, in a flexible and unopinionated manner. Make changes
to any part of your state tree, track changes, subscribe to specific node/sub-tree, track changes by entity keys, etc.

```bash
npm i rxdeep
```

---

# Quick Tour

Create state:

```ts
import { State } from 'rxdeep';
const state = new State([ { name: 'John' }, { name: 'Jack' }, { name: 'Jill' } ]);
```

<br>

Listen to sub-state:

```ts
state.sub(1).sub('name').subscribe(console.log); // --> subscribes to property `name` of object at index 1 of the array
```

<br>

Modify root state:
```ts
state.value = [ { name: 'Julia' }, ...state.value ]; // --> logs `John`, since `John` is index 1 now
```

... or mid-level states:

```ts
state.sub(1).value = { name: 'Josef' };              // --> logs `Josef`
```

... or leaf-states on the same address:
```ts
state.sub(1).sub('name').value = 'Jafet';            // --> logs `Jafet`
```

<br>

A `State` is an [`Observer`](https://rxjs.dev/guide/observer):

```ts
import { interval } from 'rxjs';
import { map } from 'rxjs/operators';

interval(1000)
.pipe(map(i => ({ name: `Jarvis #${i}`})))
.subscribe(state.sub(1));                            // --> logs `Jarvis #0`, `Jarvis #1`, `Jarvis #2`, ...
```

<br>

A `State` is an [`Observable`](https://rxjs.dev/guide/observable):

```ts
import { debounceTime } from 'rxjs/operators';

state.sub(1).pipe(debounceTime(1000)).subscribe(console.log); // --> debounces changes for 1 second
```

<br>

Track `keys` instead of indexes:

```ts
import { State, KeyedState } from 'rxdeep';

const state = new State([{ id: 101, name: 'Jill' }, { id: 102, name: 'Jack' }]);
const keyed = new KeyedState(state, p => p.id);

keyed.key(101).sub('name').subscribe(console.log);     // --> logs `Jill`

state.value = [state.value[1], state.value[0]];      // --> no log
state.sub(1).sub('name').value = 'John';             // --> logs `John`
```

<br>

Track index of a specific `key`:

```ts
keyed.index(101).subscribe(console.log);      // --> logs 0, 1
```

<br>

Verify changes to the state:

```ts
import { State, VerifiedState } from 'rxdeep';

const s = new State(12);
const v = new VerifiedState(s, change => change.from < change.to); // --> only increasing numbers

v.subscribe(console.log);

v.value = 10; // --> logs 12
v.value = 14; // --> logs 14
v.value = 9;  // --> logs 14
v.value = 13; // --> logs 14
v.value = 15; // --> logs 15
```

---

# Features

Here are the design goals/features of **RxDeep**, setting aside from other reactive state management libraries:

<br>

## Performance

**RxDeep** is extremely fast and light-weight in terms of memory consumption and computation, utilizing pure mappings
on the root of the state-tree for reading/writing on the whole tree.

The only performance hotspot are `KeyedState`s, as they conduct an [O(n)](:Formula) operation on change emission
from state-tree upstream, involving a user-provided _key_ function. To mitigate potential performance hits, `Keyed` states
do share this computation across subscriptions.

<br>

## Precision

**RxDeep** enables subscribing to a particular sub-state of the state tree. These sub-states (mostly) only
emit values when the value of the sub-state has changed (or a change is issued on same tree address).
So you could subscribe heavy-weight operations (such as DOM re-rendering)
on sub-states.

Sometimes **RxDeep** is unable to determine the address of a change and needs
to rely on equality checks to be able to properly down-propagate changes.
Equality operator `===` is used by defaul, which might result in redundant emissions on complex objects.
If you need absolute precision,
you can provide custom equality checks (e.g. [`lodash.isEqual()`](https://lodash.com/docs/4.17.15#isEqual)), trading
performance for precision.

<br>

## Flexibility

**RxDeep**, unlike libraries such as [Redux](https://redux.js.org/), doesn't require your changes to be funneled
through specific channels. You can freely issue changes to any part of the state-tree, so for example you can only
expose relevant parts of the state-tree to modules/components.

The only limitation (similar to [Redux](https://redux.js.org/)) is that you need to respect object immutability.
Basically do not change an object without changing its reference.

**DON'T**:

```ts
state.value.push(x);                   // --> WRONG!
state.value.x = y;                     // --> WRONG!
```


**DO**:
```ts
state.value = state.value.concat(x);   // --> CORRECT!
state.sub('x').value = y;              // --> CORRECT!
state.value = { ...state.value, x: y } // --> CORRECT!
```

<br>

## Change History

State tree is kept in sync by tracking changes (via `Change` objects). This simply means you can track changes
directly, record them, replay them, etc.

```ts
state.downstream.subscribe(console.log);    // --> Log changes
state.sub(1).sub('name').value = 'Dude';

// This object will be logged:
{ 
  value: [{...}, { name: 'Dude', ... }, ...],
  trace: {
    head: { sub: 1 },
    rest: { head: { sub: 'name' } }
  },
  from: ..., to: 'Dude'
}
```

Furthermore, `KeyedState`s provide detailed array changes, i.e. additions/deletions on particular indexes,
or items being moved from one index to another.

```ts
const state = new State([{ id: 101, name: 'Jack' }, { id: 102, name: 'Jill' }]);
const keyed = new KeyedState(state, p => p.id);

keyed.changes().subscribe(console.log);        // --> Log changes

state.value = [
  { id: 102, name: 'Jill' },
  { id: 101, name: 'Jack' },
  { id: 103, name: 'Jafet' }
];

// This object will be logged:
{
  additions: [{
    index: 2,
    item: { id: 103, name: 'Jafet' }
  }],
  deletions:[],
  moves:[
    { oldIndex: 0, newIndex:1, item: { id: 101, name: 'Jack'} },
    { oldIndex: 1, newIndex:0, item: { id: 102, name: 'Jill'} }
  ]
}
```

<br>

## Change Verification

You can verify changes occuring on the state-tree (or on a particular sub-tree). **RxDeep** will utilize
the change history to revert unverified changes on affected sub-states:

```ts
const s = new State([{ val: 21 }, { val: 22 }, { val: 23 }]);
const v = new VerifiedState(s, change => change.value.reduce((t, i) => t + i.val) % 2 === 0);

v.sub(0).sub('val').value = 22; // --> change denied, local changes automatically reverted
v.sub(0).sub('val').value = 23; // --> change accepted and routed through the state-tree
```

<br>

## Extensibility

Each `State` is an [`Observable`](https://rxjs.dev/guide/observable) 
and an [`Observer`](https://rxjs.dev/guide/observer) at the same time, 
providing great inter-operability with [RxJS](https://rxjs.dev), its strong operators, and any
tool working with observables.

Additionally, each `State` is tied to the rest of the state tree via a downstream (`Observable<Change>`),
from which the state receives changes from higher-up in the tree,
and an upstream (`Observer<Change>`), to which the state reports changes occuring to it. The upstream
and the downstream can be ANY observable/observer pair. This, for example, means you can easily create
state trees distributed across a network, states trees remaining in sync with some REST API, etc.

<br>

## Thin and Type Safe

**RxDeep** includes minimal surface area, focusing only on
effectively tracking and propagating changes across a reactive state tree. Its bundle size is roughly `~1.5KB`,
not including dependencies. Including dependencies (which is [RxJS](https://rxjs.dev), and hence most probably
already included in your bundle), it would be `~6.5KB`.

**RxDeep** is written in [TypeScript](https://www.typescriptlang.org/) with detailed type annotations, 
which should greatly improve development experience even if you use it in JavaScript (error highlighting, autocompletes, etc).


> :ToCPrevNext