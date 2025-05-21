---
title: ts utils
cover: /img/useState.webp
---

### Parameters  提取函数参数

```ts
type Parameters<T extends (...args: any) => any> = T extends (...args: infer P) => any ? P : never;

const func = (name:string,age:number) => {};

type params = Parameters<typeof func>
```

### keyof 强制类型校验

```ts
const data = {
    name:'xxx',
    age:18
}

function getData<T extends object ,K extends keyof T>(o:T,key:K):T[K]{
    return o[key]
}

getData(data,'name')
```

### 部分参数可选

```ts
type User = {
     id:number,
     name:string,
     age:number
 }

// Pick选取原对象匹配到传入key的值 key只能是原对象存在的
type Pick<T, K extends keyof T> = {
    [P in K]: T[P];
};

// Exclude选取原对象匹配到传入key的值 key可以是原对象不存在的
type Exclude<T, U> = T extends U ? never : T;

// Omit 拿到原对象没有匹配到传入的key的值
type Omit<T, K extends keyof any> = Pick<T, Exclude<keyof T, K>>;

// Partial将原对象的值变为可选
type Partial<T> = {
    [P in keyof T]?: T[P];
};

// SelectPartial将原对象变为部分参数可选
type SelectPartial<T,K extends keyof T> = Partial<Pick<T,K>> & Omit<T,K>

type Final = SelectPartial<User,'id' | 'name'>
```

### 交集和并集

```ts
interface FirstType {
     id:number,
     firstName:string,
     lastName:string
 }

interface SecondType {
     id:number,
     address:string,
     name:string
 }

type ExtractType = Extract<keyof FirstType,keyof SecondType>
type ExcludeType = Exclude<keyof FirstType,keyof SecondType> 
```

### Record 定义对象的key value

```ts
type Record<K extends keyof any, T> = {
    [P in K]: T;
};


 interface ProductCar {
     size:string,
     length:number,
     type:string
 }

 type Product = ProductCar[]

 interface ProductCarIn {
     [key:string]:ProductCar
 }

 class CarModel {
     ProductCar :Record<string,ProductCar> = {
         'X5':{
             size:'big',
             length:5080,
             type:'suv'
         }
     }
 }
```

### NonNullable排除null undefined

```ts
 type NonNullableType = string | number | null | undefined

 const nonNull:NonNullable<NonNullableType> = 'ts-node'  
```

### ts唯一属性

```ts
const PROD = Symbol('production mode');
const DEV = Symbol('development mode');
```


### 构造函数参数类型 构造函数返回值类型

```ts
// 构造函数返回值类型
type InstanceType<T extends abstract new (...args: any) => any> = T extends abstract new (...args: any) => infer R ? R : any;

// 构造函数参数类型
type ConstructorParameters<T extends abstract new (...args: any) => any> = T extends abstract new (...args: infer P) => any ? P : never;

class UserOne {
    constructor(public name:string){}
}

interface IConstruct<T extends new (...arg:any) => any> {
    type:new (...arg:ConstructorParameters<T>) => InstanceType<T>;
}

type UserConstruct = IConstruct<typeof UserOne>

const constr:UserConstruct = {
    type:UserOne
}

// constr.type ==> new (name:string) => UserOne

const userInstance = new constr.type('jiong')

console.log(userInstance.name)

```

### is toArray

```ts
function isString(a:unknown):a is string{
    return typeof a === 'string'
}

type ToArray<T> = T extends unknown[] ? T : T[]

const arrData:ToArray<string> = Array.from('123')
```


### 强制类型转换 <const> as const 元组转数组

```ts
function tuplify <T extends unknown[]>(...eles:T):T{
    return eles
}
function test(){
    const respone :string = 'jiong';
    const age:number = 30;
    return tuplify(respone,age) 
    // return[respone,age]
}
const item = test();
const [respone] = item
```

### hasKey

```ts
function hasKey<O extends object>(obj:O,key:PropertyKey):key is keyof O{
    return obj.hasOwnProperty(key)
}
```


### 实现PickPromise，能够获取泛型的类型

```ts
type PickPromise<T extends Promise<any>> = T extends Promise<infer P> ? P : never
type A = Promise<number>
type B = PickPromise<A> 
```


### 把对象的值的类型 提取成元组

```ts

const objVol = {
    name:'xxx',
    age:18,
    add:(a:number,b:number) =>{ return a + b},
}

type GetObjVal<T extends object> = T[keyof T]
type GetObjKey<T extends object> = {
    [K in keyof T]:K
}[keyof T]

type ObjKey = GetObjKey<typeof objVol>
type ObjVal = GetObjVal<typeof objVol>
```


### 泛型放前面

```ts
interface C {
    <T>(arg:T):T
}

type CC = <T>(arg:T) => number

const getDataA:C = <T>(arg:T):T => {
    return arg
}

getDataA<string>('xxx')


const getDataB:CC = <T>(arg:T):number => {
    return <number>(arg as unknown)
}
```


### 实现一个ts的工具函数 GetOnlyFnProps<T> 提取泛型T中字段类型是函数类型的工具函数，其中T属于一个对象。


```ts
type GetOnlyFnKeys<T extends object> = {
    [K in keyof T]: T[K] extends Function ? K : never
}[keyof T]
   
type GetOnlyFnProps<T extends object> = {
    [K in GetOnlyFnKeys<T>]: T[K]
}

type SomeObj = {
    name: string,
    age:number,
    add():void,
    des():void,
}

type CCC = GetOnlyFnKeys<SomeObj>

```
