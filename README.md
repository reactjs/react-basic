# React - Basic Theoretical Concepts

This document is my attempt to formally explain my mental model of React. The intention is to describe this in terms of deductive reasoning that lead us to this design.

There may certainly be some premises that are debatable and the actual design of this example may have bugs and gaps. This is just the beginning of formalizing it. Feel free to send a pull request if you have a better idea of how to formalize it. The progression from simple -> complex should make sense along the way without too many library details shining through.

The actual implementation of React.js is full of pragmatic solutions, incremental steps, algorithmic optimizations, legacy code, debug tooling and things you need to make it actually useful. Those things are more fleeting, can change over time if it is valuable enough and have high enough priority. The actual implementation is much more difficult to reason about.

I like to have a simpler mental model that I can ground myself in.

## Transformation

The core premise for React is that UIs are simply a projection of data into a different form of data. The same input gives the same output. A simple pure function.

```js
function NameBox(name) {
  return { fontWeight: 'bold', labelContent: name };
}
```

```
'Sebastian Markbåge' ->
{ fontWeight: 'bold', labelContent: 'Sebastian Markbåge' };
```

## Abstraction

You can't fit a complex UI in a single function though. It is important that UIs can be abstracted into reusable pieces that don't leak their implementation details. Such as calling one function from another.

```js
function FancyUserBox(user) {
  return {
    borderStyle: '1px solid blue',
    childContent: [
      'Name: ',
      // Embed the render output of `NameBox`.
      NameBox(user.firstName + ' ' + user.lastName)
    ]
  };
}
```

```
{ firstName: 'Sebastian', lastName: 'Markbåge' } ->
{
  borderStyle: '1px solid blue',
  childContent: [
    'Name: ',
    { fontWeight: 'bold', labelContent: 'Sebastian Markbåge' }
  ]
};
```

## Composition

To achieve truly reusable features, it is not enough to simply reuse leaves and build new containers for them. You also need to be able to build abstractions from the containers that *compose* other abstractions. The way I think about "composition" is that they're combining two or more different abstractions into a new one.

```js
function FancyBox(children) {
  // `FancyBox` doesn't need to know what's inside it.
  // Instead, it accepts `children` as an argument.
  return {
    borderStyle: '1px solid blue',
    children: children
  };
}

function UserBox(user) {
  // Now we can put different `children` inside `FancyBox` in different parts of UI.
  // For example, `UserBox` is a `FancyBox` with a `NameBox` inside.
  return FancyBox([
    'Name: ',
    NameBox(user.firstName + ' ' + user.lastName)
  ]);
}

function MessageBox(message) {
  // However a `MessageBox` is a `FancyBox` with a message.
  return FancyBox([
    'You received a new message: ',
    message
  ]);
}
```

## State

A UI is NOT simply a replication of server / business logic state. There is actually a lot of state that is specific to an exact projection and not others. For example, if you start typing in a text field. That may or may not be replicated to other tabs or to your mobile device. Scroll position is a typical example that you almost never want to replicate across multiple projections.

We tend to prefer our data model to be immutable. We thread functions through that can update state as a single atom at the top.

```js
function FancyNameBox(user, likes, onClick) {
  return FancyBox([
    'Name: ', NameBox(user.firstName + ' ' + user.lastName),
    'Likes: ', LikeBox(likes),
    LikeButton(onClick)
  ]);
}

// Implementation Details

var likes = 0;
function addOneMoreLike() {
  likes++;
  rerender();
}

// Init

FancyNameBox(
  { firstName: 'Sebastian', lastName: 'Markbåge' },
  likes,
  addOneMoreLike
);
```

*NOTE: These examples use side-effects to update state. My actual mental model of this is that they return the next version of state during an "update" pass. It was simpler to explain without that but we'll want to change these examples in the future.*

## Memoization

Calling the same function over and over again is wasteful if we know that the function is pure. We can create a memoized version of a function that keeps track of the last argument and last result. That way we don't have to reexecute it if we keep using the same value.

```js
function memoize(fn) {
  var cachedArg;
  var cachedResult;
  return function(arg) {
    if (cachedArg === arg) {
      return cachedResult;
    }
    cachedArg = arg;
    cachedResult = fn(arg);
    return cachedResult;
  };
}

// Has the same API as NameBox but caches its result if its single argument
// has not changed since the last time `MemoizedNameBox` was called.
var MemoizedNameBox = memoize(NameBox);

function NameAndAgeBox(user, currentTime) {
  return FancyBox([
    'Name: ',
    MemoizedNameBox(user.firstName + ' ' + user.lastName),
    'Age in milliseconds: ',
    currentTime - user.dateOfBirth
  ]);
}

// We calculate the output of `NameAndAgeBox` twice, so it will call `MemoizedNameBox` twice.
// However `NameBox` is only going to be called once because its argument has not changed.
const sebastian = { firstName: 'Sebastian', lastName: 'Markbåge' };
NameAndAgeBox(sebastian, Date.now());
NameAndAgeBox(sebastian, Date.now());
```

## Lists

Most UIs are some form of lists that then produce multiple different values for each item in the list. This creates a natural hierarchy.

To manage the state for each item in a list we can create a Map that holds the state for a particular item.

```js
function UserList(users, likesPerUser, updateUserLikes) {
  return users.map(user => FancyNameBox(
    user,
    likesPerUser.get(user.id),
    () => updateUserLikes(user.id, likesPerUser.get(user.id) + 1)
  ));
}

var likesPerUser = new Map();
function updateUserLikes(id, likeCount) {
  likesPerUser.set(id, likeCount);
  rerender();
}

UserList(data.users, likesPerUser, updateUserLikes);
```

*NOTE: We now have multiple different arguments passed to FancyNameBox. That breaks our memoization because we can only remember one value at a time. More on that below.*

## Continuations

Unfortunately, since there are so many lists of lists all over the place in UIs, it becomes quite a lot of boilerplate to manage that explicitly.

We can move some of this boilerplate out of our critical business logic by deferring execution of a function. For example, by using "currying" ([`bind`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/bind) in JavaScript). Then we pass the state through from outside our core functions that are now free of boilerplate.

This isn't reducing boilerplate but is at least moving it out of the critical business logic.

```js
function FancyUserList(users) {
  // `UserList` needs three arguments: `users`, `likesPerUser`, and `updateUserLikes`.

  // We want `FancyUserList` to be ignorant of the fact that `UserList` also
  // needs `likesPerUser` and `updateUserLikes` so that we don't have to wire
  // the arguments for bookkeeping this state through `FancyUserList`.
  
  // We can cheat by only providing the first argument for now:
  const children = UserList.bind(null, users)
  
  // Unlike in the previous examples, `children` is a partially applied function
  // that still needs `likesPerUser` and `updateUserLikes` to return the real children.

  // However, `FancyBox` doesn't "read into" its children and just uses them in its output,
  // so we can let some kind of external system inject the missing arguments later.
  return FancyBox(children);
}

// The render output is not fully known yet because the state is not injected.
const box = FancyUserList(data.users);
// `box.children()` is a function, so we finally inject the state arguments.
const resolvedChildren = box.children(likesPerUser, updateUserLikes);
// Now we have the final render output.
const resolvedBox = {
  ...box,
  children: resolvedChildren
};
```

## State Map

We know from earlier that once we see repeated patterns we can use composition to avoid reimplementing the same pattern over and over again. We can move the logic of extracting and passing state to a low-level function that we reuse a lot.

```js
// `FancyBoxWithState` receives `children` that are not resolved yet.
// Each child contains a `continuation`. It is a partially applied function
// that would return the child's output, given the child's state and a function to update it.
// The children also contain unique `key`s so that their state can be kept in a map.
function FancyBoxWithState(
  children,
  stateMap,
  updateState
) {
  // Now that we have the `stateMap`, inject it into all the continuations
  // provided by the children to get their resolved rendering output.
  const resolvedChildren = children.map(child => child.continuation(
    stateMap.get(child.key),
    updateState
  ));

  // Pass the rendered output to `FancyBox`.
  return FancyBox(resolvedChildren);
}

function UserList(users) {
  // `UserList` returns a list of children that expect their state
  // to get injected at a later point. We don't know their state yet,
  // so we return partially applied functions ("continuations").
  return users.map(user => {
    continuation: FancyNameBox.bind(null, user),
    key: user.id
  });
}

function FancyUserList(users) {
  // `FancyUserList` returns a `continuation` that expects the state
  // to get injected at a later point. This state will be passed on
  // to `FancyBoxWithState` which needs it to resolve its stateful children.
  const continuation = FancyBoxWithState.bind(null, UserList(users));
  return continuation;
}

// The render output of `FancyUserList` is not ready to be rendered yet.
// It's a continuation that still expects the state to be injected.
const continuation = FancyUserList(data.users);

// Now we can inject the state into it.
const output = continuation(likesPerUser, updateUserLikes);

// `FancyUserList` will forward the state to `FancyBoxWithState`, which will pass
// the individual entries in the map to the `continuation`s of its `children`.

// Those `continuations` were generated in `UserList`, so they will pass
// the state into the individual `FancyNameBox`es in the list.
```

## Memoization Map

Once we want to memoize multiple items in a list memoization becomes much harder. You have to figure out some complex caching algorithm that balances memory usage with frequency.

Luckily, UIs tend to be fairly stable in the same position. The same position in the tree gets the same value every time. This tree turns out to be a really useful strategy for memoization.

We can use the same trick we used for state and pass a memoization cache through the composable function.

```js
function memoize(fn) {
  // Note how in the previous memoization example, we kept the cached argument and the
  // cached result as a local variable inside `memoize`. This is not useful for lists
  // because in a list, the function will be called many times with a different argument.

  // Now the function returned by `memoize` accepts the `memoizationCache` as an argument
  // in the hope that the list containing a component can supply a "local" cache for each item.
  return function(arg, memoizationCache) {
    if (memoizationCache.arg === arg) {
      return memoizationCache.result;
    }
    const result = fn(arg);
    memoizationCache.arg = arg;
    memoizationCache.result = result;
    return result;
  };
}

function FancyBoxWithState(
  children,
  stateMap,
  updateState,
  memoizationCacheMap
) {
  return FancyBox(
    children.map(child => child.continuation(
      stateMap.get(child.key),
      updateState,
      // When the UI changes, it usually happens just in some parts of the screen.
      // This means that most children with the same keys will likely render to the same output.
      // We give each child each own memoization map, so that in the common case its output can be memoized.
      memoizationCacheMap.get(child.key)
    ))
  );
}

const MemoizedFancyNameBox = memoize(FancyNameBox);
```

## Algebraic Effects

It turns out that it is kind of a PITA to pass every little value you might need through several levels of abstractions. It is kind of nice to sometimes have a shortcut to pass things between two abstractions without involving the intermediates. In React we call this "context".

Sometimes the data dependencies don't neatly follow the abstraction tree. For example, in layout algorithms you need to know something about the size of your children before you can completely fulfill their position.

Now, this example is a bit "out there". I'll use [Algebraic Effects](http://math.andrej.com/eff/) as [proposed for ECMAScript](https://esdiscuss.org/topic/one-shot-delimited-continuations-with-effect-handlers). If you're familiar with functional programming, they're avoiding the intermediate ceremony imposed by monads.

```js
function ThemeBorderColorRequest() { }

function FancyBox(children) {
  // This will propagate through the caller stack, like "throw"
  const color = raise new ThemeBorderColorRequest();
  return {
    borderWidth: '1px',
    borderColor: color,
    children: children
  };
}

function BlueTheme(children) {
  return try {
    children();
  } catch effect ThemeBorderColorRequest -> [, continuation] {
    // However, unlike "throw", we can resume the child function and pass some data
    continuation('blue');
  }
}

function App(data) {
  return BlueTheme(
    FancyUserList.bind(null, data.users)
  );
}
```

