---
author: Keqi Chen
pubDatetime: 2025-11-01T21:40:00Z
modDatetime: 2025-11-01T00:16:00Z
title: React Caching Mechanism — useMemo and useCallback
slug: usememo-and-usecallback
featured: true
draft: false
tags:
  - ReactJs
description: useMemo and useCallback are caching mechanisms built into React. In this post, I'll explain how they work, when they're actually useful (and when they're not).
---

## Table of contents

## Why Does useMemo Exist?

By design, React re-renders a component every time a **local** state or prop changes, or its **parent** re-renders. This helps keep state synced and updated, but it can become a problem when:

- The computation is expensive (e.g., filtering thousands of items)
- New objects or functions are created on every render, causing unnecessary child re-renders

That's where `useMemo` and `useCallback` come in — they allow you to optimise performance by **caching** derived values or function references to avoid redundant work.

## How useMemo Works

`useMemo` acts like a small in-memory cache in React. Like any cache, it works based on cache keys. If the cache key hasn't changed, it returns the cached value. Otherwise, it recalculates.

The cache key for `useMemo` is its **dependency array**. Here's an example of memoising an expensive calculation:

```ts
const expensiveValue = useMemo(() => {
  return items.filter(item => item.price > 100).length;
}, [items]); // Only recalculates when 'items' changes — otherwise it returns the cached value from the previous render
```

When memoising a component, the cache keys are the **props** passed to it:

```ts
const Component = ({ count }: Props) => {
  return <div>Count: {count}</div>;
};

export default React.memo(Component);
// Only re-renders when 'count' prop changes
```

This diagram represents the flow clearly:

```ts
Component is about to re-render
      ↓
Compare dependency array/props
      ↓
Changed? ───▶ Yes → Recompute and store new result
      │
      └────▶ No  → Return cached value
```

## How useCallback Works

`useCallback` is similar to `useMemo`, but it caches the **reference** of a function instead of a computed value.

This matters because functions in JavaScript are reference types. Every time a component re-renders, functions defined inside it get new references — even if their implementation is identical. This can cause child components to re-render unnecessarily.

```ts
const handleClick = useCallback(() => {
  setCount(c => c + 1);
}, []); // Function reference stays the same across re-renders
```

## When to Use Them

`useMemo` and `useCallback` are most helpful in two cases:

### 1. Expensive Calculations - to avoid recomputing derived data unnecessarily

```ts
// Filtering/sorting large datasets (1000+ items)
const filteredItems = useMemo(() => {
  return items
    .filter(item => item.category === selectedCategory)
    .sort((a, b) => b.price - a.price);
}, [items, selectedCategory]);

// Complex mathematical computations
const statistics = useMemo(() => {
  return {
    mean: calculateMean(data),
    median: calculateMedian(data),
    stdDev: calculateStandardDeviation(data)
  };
}, [data]);

// Deep object transformations
const transformedData = useMemo(() => {
  return rawData.map(item => ({
    ...item,
    nested: processNestedStructure(item.children)
  }));
}, [rawData]);
```

**Without `useMemo`:** These calculations run on every render, even if `items`, `data`, or `rawData` haven't changed.

**With `useMemo`:** Calculations only run when dependencies actually change.

### 2. Stable references — to prevent avoidable re-renders of memoised children

```ts
const Parent = () => {
  const [count, setCount] = useState(0);

  const handleClick = () => setCount(count + 1);

  return <Child onClick={handleClick} />;
};
```

When `handleClick` is triggered, here's what happens:

1. `Parent`'s `count` state updates
2. `Parent` re-renders
3. `handleClick` gets a **new reference**
4. `Child` receives a different `onClick` prop
5. `Child` re-renders (even though the function does the same thing)

### The Solution: Both useCallback AND React.memo

To prevent unnecessary child re-renders, you need **both**:

```ts
const Parent = () => {
  const [count, setCount] = useState(0);

  // 1. Stabilise the function reference
  const handleClick = useCallback(() => {
    setCount(c => c + 1);
  }, []);

  return <Child onClick={handleClick} />;
};

// 2. Memoise the child component
const Child = React.memo(({ onClick }: Props) => {
  return <button onClick={onClick}>Click me</button>;
});
```

Now when `Parent` re-renders:

- `handleClick` keeps the same reference (thanks to `useCallback`)
- `Child`'s props don't change (same reference)
- `React.memo` prevents `Child` from re-rendering

Note that `useCallback` alone doesn’t stop re-renders — it only stabilises the function reference. The parent component will still re-render when its state changes, and without `React.memo`, the child will re-render too. That's why Both are needed to make this optimisation effective.

### A Better Approach: Avoid Passing setState Down

While the above solution works, please be aware that **it's generally not good practice to pass `setState` functions to child components** in the first place. A better pattern is to lift the state down or use composition:

```ts
// Better: Keep state local to where it's used
const Parent = () => {
  return <Child />;
};

const Child = () => {
  const [count, setCount] = useState(0);
  const handleClick = () => setCount(count + 1);

  return <button onClick={handleClick}>Count: {count}</button>;
};
```

Or use composition to avoid prop drilling:

```ts
const Parent = () => {
  const [count, setCount] = useState(0);

  return (
    <div>
      <Display count={count} />
      <button onClick={() => setCount(c => c + 1)}>Increment</button>
    </div>
  );
};
```

## Should You Memoise by Default?

Some teams choose to memoise by default — wrapping every component in `React.memo` and using `useCallback` everywhere — arguing that CPU and memory costs are negligible and that consistency outweighs the overhead of case-by-case decisions.

Others prefer to optimise only when profiling shows a real benefit, keeping the codebase cleaner and easier to reason about because they believe **readability and maintainability** matters more. Consider this example:

```ts
// Over-memoised code is harder to read and maintain
const Component = () => {
  const value1 = useMemo(() => prop1 + prop2, [prop1, prop2]);
  const value2 = useMemo(() => value1 * 2, [value1]);
  const handleClick = useCallback(() => doSomething(), []);
  const handleChange = useCallback(() => doSomethingElse(), []);
  const data = useMemo(() => ({ value1, value2 }), [value1, value2]);

  return <Child data={data} onClick={handleClick} onChange={handleChange} />;
};
```

This approach may:

- ❌ Makes code harder to understand at a glance
- ❌ Increases cognitive load when debugging
- ❌ Creates more opportunities for bugs (stale closures, missing dependencies)
- ❌ Makes the codebase less approachable for new team members

I won't say one approach is _definitively better_ than the other — both perspectives have a valid point. What matters most is understanding what you're trading off and developing an optimisation strategy accordingly.

## Conclusion

`useMemo` and `useCallback` are optimisation tools with trade-offs worth considering.

**When they're useful:**

- Passing callbacks to memoised child components
- Expensive calculations
- Confirmed performance issues

**What you trade off:**

- Code readability and simplicity
- Easier debugging and maintenance
- Lower cognitive load for your team

## Useful Links

- [**How to useMemo and useCallback: you can remove most of them**](https://www.developerway.com/posts/how-to-use-memo-use-callback) —  
  A deep dive explaining why overusing these hooks often doesn’t help and how React’s reconciliation already optimises many cases.

- [**Why We Memo All the Things**](https://attardi.org/why-we-memo-all-the-things/) —  
  A counterpoint perspective arguing for consistent memoisation to reduce cognitive cost in large teams.

- [**Overusing useMemo and useCallback — Kent C. Dodds**](https://kentcdodds.com/blog/usememo-and-usecallback) —  
  Kent’s explanation on why these hooks should be used sparingly and how to measure if they’re actually improving performance.

- [**Dan Abramov — When to useMemo and useCallback (Twitter Thread)**](https://twitter.com/dan_abramov/status/1305855005932818432) —  
  A concise summary by React’s co-author clarifying when memoisation is worth it and when it’s just noise.

- [**React Performance — Avoid Premature Optimisation**](https://react.dev/learn/keeping-components-pure#avoid-premature-optimizations) —  
  From the official docs: start with clean, pure components first, then optimise based on profiling data.
