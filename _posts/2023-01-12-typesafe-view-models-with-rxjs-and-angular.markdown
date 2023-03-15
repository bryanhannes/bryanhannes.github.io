---
layout: post
title:  "Typesafe view models with RxJS and Angular"
date:   2023-01-13 10:00:00 +0100
published: true
comments: true
categories: Angular
cover: assets/typesafe-view-models-with-rxjs-and-angular/typesafe-view-models-with-rxjs-and-angular
tags: [Angular, RxJS]
type: article
---

In this article, we are going to improve the code from
a [previous blog article](/reactively-storing-and-retrieving-url-state-in-angular/) by making the view model typesafe.
We will do this with an NPM package specially created for this article
called [`@bryanhannes/typed-view-model`](https://www.npmjs.com/package/@bryanhannes/typed-view-model){:target="_blank"}.
In this NPM package is a function that will help us to create typesafe view models.

## Forking the repository

We are going to use the Car catalog application from a previous article. We can fork the repository
from [Stackblitz](https://stackblitz.com/edit/angular-ivy-y8vtzw?file=src%2Fapp%2Fapp.component.ts){:target="_blank"}
or [GitHub](https://github.com/bryanhannes/car-catalog){:target="_blank"}. In this article, we are going to use
the `Stackblitz repository`.

## Installing the NPM package

After forking the repository, we have to install the NPM package `@bryanhannes/typed-view-model`.

In StackBlitz we can do this by adding `@bryanhannes/typed-view-model` to the `dependencies` section.

![Adding @bryanhannes/typed-view-model as dependency](/assets/typesafe-view-models-with-rxjs-and-angular/typed-view-models-1.png)

If you checked out the source code from Github, you can install the package with:

```shell
npm install @bryanhannes/typed-view-model
```

## Typesafe view model

This `@bryanhannes/typed-view-model` NPM package exposes 1 function called `vm<T>()`. `vm` stands for `view model`.

So let's import this function in the `app.component.ts`.

```typescript
import {vm} from '@bryanhannes/typed-view-model';
```

Now we want to use this `vm` function to create the `vm$` observable. In the documentation of the NPM package is shown
the following example:

```typescript
interface PageViewModel {
    name: string;
    number: number;
    stringArray: string[];
}

const name$$ = new Subject<string>();
const number$$ = new Subject<number>();
const stringArray$$ = new Subject<string[]>();

const vm$ = vm<PageViewModel>({
    name: {observable: name$$, initialValue: 'John Snow'},
    number: {observable: number$$, initialValue: 5000},
    stringArray: {observable: stringArray$$, initialValue: ['a', 'b', 'c']},
});
```

From the example, we can see

- that the `vm()` function takes in an object with the same property keys as `PageViewModel`. If we add a property that
  is not in `PageViewModel` we get a compilation error. That is the typesafe view model we are talking about.
- that the value object has 2 properties
    - `observable` should be of type `Observable<K>` where `K` is of the same type as the corresponding property in
      the `PageViewModel`
    - `initialValue` should be of the same type as the corresponding property in the `PageViewModel`

```typescript
interface PageViewModel {
    name: string;
}

const color$$ = new Subject<string>();
const number$$ = new Subject<number>();

// This does not compile
const vm$ = vm<PageViewModel>({
    color: {observable: color$$, initialValue: ''}, // Example 1: This line will not compile because 'color' is not a property of `PageViewModel`
    name: {
        observable: number$$, // Example 2: Does not compile because the type of the Subject is not assignable to the type of the property. Observable<number> is not assignable to Observable<string>
        initialValue: [], // Example 3: Does not compile because a string is expected here and not an array
    },
});
```

Now let's test this `vm()` function out.
First, let's refactor the name `queryParams$` to `filter$` because this name is more appropriate.
We add the `vm()` function in the same manner as in the example.
We also added `map` operators to map the fields of `CarFilter` to the corresponding property.

```typescript
// Renamed from `queryParams$` to `filter$`
private readonly
filter$: Observable < CarFilter > =
    this.activatedRoute.queryParams.pipe(
        map((params: Params) => ({
            name: params.name,
            brand: params.brand,
            color: params.color,
        }))
    );

...

public readonly
vm$ = vm<PageViewModel>({
    name: {
        observable: this.filter$.pipe(map((filter) => filter.name)), // Only need the name property here
        initialValue: null,
    },
    color: {
        observable: this.filter$.pipe(map((filter) => filter.color)), // Only need the color property here
        initialValue: null,
    },
    brand: {
        observable: this.filter$.pipe(map((filter) => filter.brand)), // Only need the brand property here
        initialValue: null,
    },
    results: {observable: this.results$, initialValue: []},
});
```

Congratulations, we just created a typesafe view model!

## Creating custom RxJS `map` operators

A final improvement we can do to improve the readability of our code is adding custom RxJS `map` operators.

```typescript
// The first generic of `UnaryFunction` is the type of the value that is passed in the operator
// The second generic of `UnaryFunction` is the return type of the operator
// In the `mapToName` operator we pass in a `CarFilter` and return `string` or `null`
export const mapToName = (): UnaryFunction<Observable<CarFilter>,
    Observable<string | null>> => pipe(map((filter: CarFilter) => filter.name));

...

public readonly
vm$ = vm<PageViewModel>({
    name: {
        observable: this.queryParams$.pipe(mapToName()), // Using the `mapToName()` operator
        initialValue: null,
    },
    ...
});
```

## Demo

Check out the full demo
on [StackBlitz](https://stackblitz.com/edit/angular-ivy-k4t8eb?file=src%2Fapp%2Fapp.component.ts){:target="_blank"}

## Conclusion

- Now we know how to create typesafe view models.
- If we assign types that are not allowed, we get compilation errors.
- We can use the `UnaryFunction` to create custom RxJS operators.

The source code of the `@bryanhannes/typed-view-model` NPM package can be found
on [Github](https://github.com/bryanhannes/typed-view-model){:target="_blank"}
