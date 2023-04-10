---
layout: post
title:  "RxJS distinctUntilChanged: filtering out duplicate emissions"
date:   2023-04-10 05:00:00 +0100
published: true
comments: true
categories: Angular
cover: "assets/rxjs-distinctuntilchanged/rxjs-distinctuntilchanged"
tags: [Angular, RxJS]
type: article
---

The RxJS `distinctUntilChanged` operator is a filter operator, it filters out duplicate emissions. It compares each emission with the previous and only emits the new value if it is different from the previous value.
By default, it uses `===` (strict equality) to compare the new and old value, but you can also provide a custom comparison function.

In this blog post we'll cover:
- Basic usage of `distinctUntilChanged`
- Provinding a custom comparison function
- Real-world use cases with `distinctUntilChanged` in Angular
- Conclusion

## Basic usage of `distinctUntilChanged`
The  `distinctUntilChanged` operator is often used to optimize our streams. It will only emit a new value if it is different to the previous value. 
In this example, we can see how we optimized the `alphabet$` Observable by using `distinctUntilChanged`:

```typescript
// Import the distinctUntilChanged operator
import { distinctUntilChanged } from 'rxjs/operators';

// Observable that emits a, b, b, c, c, c, d, d, d, d
const alphabet$ = of('a', 'b', 'b', 'c', 'c', 'c', 'd', 'd', 'd', 'd');

// Without optimization of distinctUntilChanged
alphabet$.subscribe(console.log);
// Output: a, b, b, c, c, c, d, d, d, d

// With optimization of distinctUntilChanged
alphabet$.pipe(distinctUntilChanged()).subscribe(console.log);
// Output: a, b, c, d
```

## Providing a custom comparison function
By default, the `distinctUntilChanged` operator uses `===` (strict equality) to compare the new and old value.
But you can also provide a custom comparison function to compare the new and old value.

In the example below, we have an Observable `users$` that emits users one by one. We want to filter out subsequent duplicate emissions based on their name.

```typescript
type User = { id: string; name: string; };

const users$ = of(
  { id: '1', name: 'John' },
  { id: '2', name: 'John' }, // will be filtered out
  { id: '3', name: 'John' }, // will be filtered out
  { id: '4', name: 'Jane' },
  { id: '5', name: 'Jane' }, // will be filtered out
  { id: '6', name: 'Bryan' },
  { id: '7', name: 'Bryan' }, // will be filtered out
);

// Custom comparison function that compares the name of the users (case-insensitive)
const customCompare = (a: User, b: User) => a.name.toLowerCase() === b.name.toLowerCase();

// distinctUntilChanged with custom comparison function
users$.pipe(distinctUntilChanged(customCompare)).subscribe(console.log);

// Output:
// { id: '1', name: 'John' }
// { id: '4', name: 'Jane' }
// { id: '6', name: 'Bryan' }
```

### Filtering based on 1 key with `distinctUntilKeyChanged`
We can simplify the previous example with the `distinctUntilKeyChanged` operator. This operator compares emissions based on a specific property (key) instead of the whole object.

```typescript
type User = { id: string; name: string; };

const users$ = of(
  { id: '1', name: 'John' },
  { id: '2', name: 'John' }, // will be filtered out
  { id: '3', name: 'John' }, // will be filtered out
  { id: '4', name: 'Jane' },
  { id: '5', name: 'Jane' }, // will be filtered out
  { id: '6', name: 'Bryan' },
  { id: '7', name: 'Bryan' }, // will be filtered out
);

// distinctUntilKeyChanged that filters out emissions based on the name property
// please note that the comparison is case-sensitive
users$.pipe(distinctUntilKeyChanged('name')).subscribe(console.log);

// Output:
// { id: '1', name: 'John' }
// { id: '4', name: 'Jane' }
// { id: '6', name: 'Bryan' }
```


## Real-world use case with `distinctUntilChanged` in Angular

### Filtering out duplicate HTTP requests

In this example, we have a search input that emits every time the user types something in the search bar. 
We want to filter out duplicate search terms and only send a request to the server if the search term is different from the previous search term.

All of the emissions (keypresses in the search bar) that happens within 1 second are thrown away and only the last value is emitted because of the [`debounceTime`](/delay-streams-with-debouncetime) operator.
We use [`switchMap`](/real-life-use-cases-for-rxjs-switchmap-in-angular) to perform the actual search request.

```typescript
@Component({
    ...
    template: `<input [formControl]="searchBar" type="text">`
})
export class AppComponent {
    ...
    public readonly searchBar = new FormControl();

    public readonly searchResults$ = this.searchBar.valueChanges.pipe(
        debounceTime(1000), // Waits for 1000ms or 1s before emitting
        distinctUntilChanged(), // Only emits if the new value is different from the previous value
        switchMap((query) => this.searchService.search(query)) // Performs the search request
    );

    constructor() {
        this.searchResults$.subscribe((query) => {
            console.log(query);
        });
    }
}
```

### Optimizing View Models

In a previous article about RxJS `combineLatest`, we saw how we can create a [view model](/rxjs-combinelatest-how-it-works-and-how-you-can-use-it-in-angular/#example-2-using-combinelatest-to-create-a-view-model). 
We should also optimize the view model by using `distinctUntilChanged` to make sure to only emit when one of the properties in the view model has actually changed.

```typescript
@Component({
    ...
    template: `
        <div *ngIf="vm$ | async as vm">
            <h1>{{ vm.title }}</h1>
            <p>Current user: {{ vm.currentUsername }}</p>
        </div>
    `
})
export class AppComponent {
    private readonly titleService = inject(TitleService);
    private readonly authService = inject(AuthService);
    
    // distinctUntilChanged operator makes sure that the vm$ is 
    // only emitted when one of the values has actually changed
   public readonly vm$ = combineLatest([
        this.titleService.getTitle().pipe(distinctUntilChanged()),
        this.userService.getCurrentUsername().pipe(distinctUntilChanged())
    ]).pipe(
        map(([title, currentUsername]) => ({ currentUser, currentUsername }))
    );
} 
```


## Conclusion
The `distinctUntilChanged` filters out duplicate emissions, it performs a strict equality check (===) by default but we can provide a custom comparsion function.
When we want to filter on a specific property of a type we can use the `distinctUntilKeyChanged` operator.



