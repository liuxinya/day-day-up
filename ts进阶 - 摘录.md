# infer

* 表示待推断类型

```ts
type ParamType<T> = T extends (...args: infer P) => any ? P : T;
```
* 如果`T`能赋值给`(...args: infer P) => any`, 结果是`(...args: infer P) => any` 否则返回`T`
```ts
interface User {
  name: string;
  age: number;
}

type Func = (user: User) => void;

type Param = ParamType<Func>; // Param = User
type AA = ParamType<string>; // string
```

# in 遍历数组

* Q - 实现 `TupleToObject`
```ts
const tuple = ['tesla', 'model 3', 'model X', 'model Y'] as const

type result = TupleToObject<typeof tuple> // expected { tesla: 'tesla', 'model 3': 'model 3', 'model X': 'model X', 'model Y': 'model Y'}
```

* A - `T[number]`
```ts
type TupleToObject<T extends ReadonlyArray<string | number>> = {
    [p in T[number]]: p
}
```

# [First of Array](https://github.com/type-challenges/type-challenges/blob/main/questions/00014-easy-first/README.md)

* Q 
```ts
type arr1 = ['a', 'b', 'c']
type arr2 = [3, 2, 1]

type head1 = First<arr1> // expected to be 'a'
type head2 = First<arr2> // expected to be 3
```

* A
```ts
type First<T extends any[]> = T extends [] ? never : T[0]
type First<T extends any[]> = T['length'] extends 0 ? never : T[0]
type First<T extends any[]> = T extends [infer P, ...infer Rest] ? P : never
```