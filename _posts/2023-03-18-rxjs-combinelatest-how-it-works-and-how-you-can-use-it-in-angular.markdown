---
layout: post
title:  "RxJS combineLatest: how it works and how you can use it in Angular"
date:   2023-03-18 05:00:00 +0100
published: true
comments: true
categories: Angular
cover: "assets/rxjs-combinelatest-how-it-works-and-how-you-can-use-it-in-angular/rxjs-combinelatest-how-it-works-and-how-you-can-use-it-in-angular"
tags: [Angular, RxJS]
type: article
---

The `combineLatest` operator combines multiple observables into one and emits them all when one of them changes. 
This makes it a very powerful operator, but there are some things we need to be aware of when using `combineLatest`.

In this article, we'll cover:
- how the `combineLatest` operator works
- how we can use it in our Angular applications 
- pitfalls when using `combineLatest`

## How `combineLatest` works
The `combineLatest` operator accepts an array of source observables as input and returns a new observable calculated based on the latest values from each source observable. Doesn't sound to hard, does it? But that's not all, we should be aware of the following:
- All the source observables need to emit at least once before `combineLatest` starts emitting. 
- Every time a new emission happens on one of the source observables, `combineLatest` recalculates the output observable.
This can cause a lot of emissions and should be dealt with carefully. 
- Most of the time you'll want to use multiple, long-lived observables with `combineLatest`.

### Pitfalls when using `combinelatest`

Unfortunately, there are some things we have to keep in mind when using `combineLatest`:

- When any of the source observables throws an error, the whole `combineLatest` observable completes
- `combineLatest` can make your code really complex, especially when you're using multiple `combineLatest` operators that depend on each other
- Multiple emissions: `combineLatest` emits a value every time one of its source observables emits. <br> ➡ In the case of Angular this can trigger change detection way more that it should

## Using `combineLatest` in Angular

We can use `combineLatest` in Angular to:
- Enriching data from multiple sources
- Create a View Model
- Client-side search filter

### Example 1: enriching data from multiple sources
```typescript
type User = {
    username: string;
    roleId: string;
}

type Role = {
    id: string;
    name: string;
}

type UserWithRole = User & { role: Role };

@Component({
    ...
    template: `
        <ul>
            <li *ngFor="let user of usersWithRole$ | async">{{ user.username }} - {{ user.role.name }}</li>
        </ul>
    `
})
export class CombineLatestComponent {
    private readonly roleService = inject(RoleService);
    private readonly userService = inject(UserService);

    public readonly usersWithRole$: Observable<UserWithRole> = combineLatest([
        // We need to add the startWith() operator to make sure both the users$ and roles$ observables emit at least once
        // If we don't do this, combineLatest won't emit anything
        this.userService.getAllUsers().pipe(startWith([])), 
        this.roleService.getAllRoles().pipe(startWith([]))
    ]).pipe(
        map(([users, roles]) => users.map(user => ({
            ...user,
            role: roles.find(role => role.id === user.roleId)
        })))
    );
}
```

### Example 2: Using `combineLatest` to create a View Model
We can use `combineLatest` to create a View Model that we can use in our template. 
There is also a really simple way to create your own typed view models, which I explain in this article: [Typesafe view models with RxJS and Angular](https://blog.bryanhannes.com/typesafe-view-models-with-rxjs-and-angular/)

```typescript
@Component({
    ...
    template: `
        <div *ngIf="vm$ | async as vm">
            <h1>Welcome {{ vm.currentUser }}</h1>
            <ul>
                <li *ngFor="let user of vm.allUsers">{{ user }}</li>
            </ul>
        </div>
    `
})
export class AppComponent {
    private readonly authService = inject(AuthService);
    private readonly userService = inject(UserService);
    
    // By using a view model we have only async pipe in our template = only 1 subscription 
    // ✅ Less subscriptions
    // ✅ Less change detection
    // ✅ Cleaner template
   public readonly vm$ = combineLatest([
        this.authService.currentUser$,
        this.userService.getAllUsers().pipe(startWith([]))
    ]).pipe(
        // We can use the map operator to transform the array into an object
        map(([currentUser, allUsers]) => ({ currentUser, allUsers }))
    );
} 
```

### Example 3: Creating a client-side search filter
```typescript
@Component({
    template: `
        <input #searchInput type="text"/>
        <ul>
            <li *ngFor="let user of filteredUsers$ | async">{{ user.name }}</li>
        </ul>
    `
})
export class CombineLatestComponent {
    private readonly userService = inject(UserService);

    // We need to add the startWith() operator to make sure both the users$ and searchTerm$ observables emit at least once
    // If we don't do this, combineLatest won't emit anything
    private readonly users$ = this.userService.getAllUsers().pipe(startWith([]));
    private readonly searchTerm$ = fromEvent(
        this.searchInput.nativeElement,
        'input'
    ).pipe(
        map((event: any) => event.target.value),
        startWith('')
    );

    public readonly filteredUsers$ = combineLatest([
        this.users$,
        this.searchTerm$,
    ]).pipe(
        map(([users, searchTerm]) =>
            users.filter((user) => user.name.includes(searchTerm))
        )
    );
}
```

## Conclusion

- `combineLatest` starts emitting when all the source observables have emitted at least once
- `combineLatest` can be hard to debug and can make our code really complex
- Use `combineLatest` with caution and only when you really need it
