---
tags:
- 前端
title: Typescript动态类型推断
date: 2020-10-12
updated: 2020-10-12
issueid: 51
---
### 实现一个功能，根据第一个字段输入的值来动态更改其余字段的类型

比如如下代码

```ts
interface InputProps {
    label: string;
}
interface SelectProps {
    name: string;
}
export interface InputType {
    input: InputProps;
    select: SelectProps;
}
export type FormItemDefinition = {
    type: 'input' | 'select';
    componentProps?: any
}
```
<!--more-->
我希望当 `type === 'input'` 时 `componentProps` 为 `InputProps`，同理当为 `'select'`时 `componentProps` 为 `'SelectProps'`

#### 初步实现我们的需求

```ts
type InputProps = {
    label: string;
}
type SelectProps = {
    name: string;
}
export type InputType = {
    input: InputProps;
    select: SelectProps;
}
type FormItemDefinition<T extends keyof InputType = keyof InputType> = {
    type: T;
    componentProps?: InputType[T];
}
const a: FormItemDefinition = {
    type: 'input',
    componentProps: {
        
    }
}
export const ItemCreateList: FormItemDefinition[] = [
    {
        type: 'input',
        componentProps: {
            
        }
    }
]
```

上述代码有个问题，componentProps包含了全部的类型

[具体看这个Playground](https://www.typescriptlang.org/play?noImplicitAny=false&alwaysStrict=false#code/C4TwDgpgBAkgdmArsACgJwPZgM5QLxQDeAUFGVADYCGARhBQFxTbBoCWcA5gNzEC+xUJCgBlehADGqTDnxFS5OFQC2EJi3ZdeAiAA8wGNMChDo8JMAAq4aARLkoHC03PJ0WbLwfZxUpmIpJaQ9tQRsoADFDZRhgCGUAEQgAMw42YDYMOAAeSyg9OLgAE1wAawgQDGTYBGRrYQJyyurXKxsAPjl7clMmSy9yCQxlAzgIOGCcAH4XWrbIAG1LAF1QobgWKComKLQYuMSUtIysroUHMl6oAHInZGuAGnOLoZGs8cnsJm6Li+eHAQA4h6AxGKDrTaxeIAYTQECocQAMmwWDtolDDqk4OlMnAFss5Atnj8LldbnNHv9BsNRh93DhvlTfkyoIDyAJlsQgA)

```ts
componentProps?: InputProps | SelectProps | undefined
```
而目标是

```ts
componentProps?: InputProps | undefined
```

#### 进一步优化

```ts
type InputProps = {
    label: string;
}
type SelectProps = {
    name: string;
}
export type InputType = {
    input: InputProps;
    select: SelectProps;
}
// T是一个类型而keyof InputType是字符串，所以要用extends产生新类型
// 如果是 T = keyof InputType则会出现 Type 'T' cannot be used to index type 'InputType'.(2536) 错误
type FormItemDefinition<T extends keyof InputType> = {
    type: T;
    componentProps?: InputType[T];
}
// 最关键在这，定义一个数组，这个数组类型是 T extends keyof InputType ? FormItemDefinition<T> : any，如果T是InputType的其中一个最后类型就是 FormItemDefinition<T>
export type FormItems<T = keyof InputType> = (T extends keyof InputType ? FormItemDefinition<T> : any)[]

export const ItemCreateList: FormItems = [
    {
        type: 'input',
        componentProps: {
            label:''
        },
    },
    {
        type: 'select',
        componentProps: {
            name: ''
        },
    },
];
```

[以上代码可以在这里找到](https://www.typescriptlang.org/play?noImplicitAny=false&alwaysStrict=false&ssl=11&ssc=27&pln=11&pc=34#code/C4TwDgpgBAkgdmArsACgJwPZgM5QLxQDeAUFGVADYCGARhBQFxTbBoCWcA5gNzEC+xUJCgBlehADGqTDnxFS5OFQC2EJi3ZdeAiAA8wGNMChDo8JMAAq4aARLkoHC03PJ0WbLwfZxUpmIpJaQ9tQRsoADFDZRhgCGUAEQgAMw42YDYMOAAeSyg9OLgAE1wAawgQDGTYBGRrSAA+OXtyUyZLL3IJDGUDOAg4YJwAfhdaqxsAbUsAXVC9AyMTcKi0GLjlbFy5csrq1wnGuQAKPIKBkqhdqpqLeuhhyOjY+KTUuHTMnMsmpio4EAASkmM2IxAWhmM3TgLFgGwAwmgIFQ4gAZNgsJirdbxXAESYKMgtBxkNpQADkTmQ5IANISSd1elkBkNsExiSSHNQ6IxyeT6Q4+HTBcLyByHGTyT5AlJaQKuj0+iz3Dh2fLOUpVEw+eqoEL6fq5kA)

`FormItemDefinition` 不能设置联合类型的默认值，要不然 `componentProps` 也都是联合类型了，就无法区分了，把默认值提取出去，然后逐个给 `FormItemDefinition` 限制对应类型的泛型，就是不同类型的集合了