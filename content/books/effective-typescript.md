---
title: Effective TypeScript
---

**Abstractions and Type Hierarchies**
Prefer relying on abstractions rather than concrete classes. Think in terms of sets and subsets: `<T extends Foo>` means `T` is a subset of `Foo`, just like subclasses are more specific versions of their superclass. Generics are essentially functions for types, and you constrain them using `extends`.

**Type Assertions and Narrowing**
Use type assertions when you know more than TypeScript does — for instance, to convert from one type to another if either is a subset of the other, or when you're certain a value isn’t null. Type assertions bypass excess property checks, so be intentional. Also, assertions like `as const` can help narrow down to literal or tuple types.

When dealing with `unknown`, it’s not assignable to anything without a type assertion. It's useful when you expect a value but don't know its shape. You can narrow `unknown` using `instanceof`, `'key' in obj`, or a custom type guard (`foo is SomeType`).

**Function Types and Utilities**
If you repeat function signatures, define function types. To mimic an existing function type, use `typeof fn`. You can also extract a function’s return type and parameters using `ReturnType<typeof fn>` and `Parameters<typeof fn>`. Conditional types let you create functions that adapt to multiple input types and return accordingly.

**Interfaces and Objects**
Interfaces are flexible: they can be extended or even augmented (declaration merging). If you need a subset, use `Pick`. To handle dynamic data (like parsed CSVs), use index signatures such as `[key: string]: string`. For safety and clarity, annotate object literals and return types explicitly.

**Loops, Arrays, and Mutability**
For performance, avoid `for-in` loops — `for-of` or traditional loops are faster. `number[]` is a subtype of `readonly number[]`, so you can assign mutable arrays to readonly ones, but not the other way around. If a function doesn’t modify its parameters, mark them `readonly`.

**Literal and Narrow Types**
Writing `as const` after a value tells TypeScript to infer the narrowest possible type, including tuple types for arrays. This is great for reducing unwanted widening.

**TypeScript vs JavaScript**
Remember that JavaScript’s `typeof` isn't the same as TypeScript’s type system. Don’t conflate the two.

**Enums, Unions, and Access Modifiers**
Prefer unions of literal types over enums — enums transpile messily into JavaScript. When it comes to private fields, use `#` instead of `private`. It enforces privacy at runtime, not just in types.

**Miscellaneous Best Practices**
Localize your use of `any` — avoid typing entire variables as `any`; instead, use `as any` in narrow contexts where you’re confident. Arrays can widen as you add elements, so be careful. Try the `type-coverage` package with the `--detail` flag for a deep dive on what’s covered.

**Destructuring Defaults**
When destructuring objects, you can set defaults inline: `const { foo = 'foo' } = obj.vars`.
