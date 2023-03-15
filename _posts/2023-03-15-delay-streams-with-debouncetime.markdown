---
layout: post
title:  "Delaying streams with RxJS debounceTime"
date:   2023-03-15 05:00:00 +0100
published: true
comments: true
categories: Angular
cover: "assets/delaying-streams-with-debouncetime/delaying-streams-with-debouncetime"
tags: [Angular, RxJS]
type: article
---

The RxJS `debounceTime` operator delays values emitted by a source Observable. It does this by waiting for a specified amount of time called (in milliseconds) and then emitting the value when the specified amount of time has elapsed. 

Unless a new value is emitted from the source Observable before the delay time is up, then it:
- drops the previous value
- emits the new value from the source Observable
- resets the timer and starts waiting again for the specified time

This behavior can be used to filter out the unwanted values or to only emit the last value of a series of rapidly emitted values.
This makes the `debounceTime` operator very helpful when creating a [real-time search]('#real-time-search-with-debouncetime') and [autosave]('#autosave-with-debouncetime').

## Basic Usage
```typescript
const result = fromEvent(document, 'click')
.pipe(
    // Click event is delayed for 1000ms or 1s
    // It emits the most recent click event, even after burst clicking
    debounceTime(1000) 
);

result.subscribe(value => console.log(value));
```


## Real-time search with `debounceTime`
When the user types in a search bar, we can use `debounceTime` to prevent sending a request for every keystroke and instead wait for a time to elapse before sending the request to the server. 

In a previous blog post, I created a real-life example of how to create a [typeahead with the `switchMap` and `debounceTime` operator](/real-life-use-cases-for-rxjs-switchmap-in-angular/#typeahead-with-switchmap-and-reactiveforms).

```typescript
@Component({
  ...
  template: `
  <input [formControl]="searchBar" type="text">
  `,
})
export class AppComponent {
  public readonly searchBar = new FormControl();

  public readonly searchResults$ = this.searchBar.valueChanges.pipe(
    debounceTime(1000), // Waits for 1000ms or 1s before emitting
    distinctUntilChanged() // doesn't emit if current value = previous value
  );

  constructor() {
    this.searchResults$.subscribe((query) => {
      console.log(query);
    });
  }
}
```

## Autosave with `debounceTime`
When a user is typing in a form, we can use `debounceTime` to prevent saving after every keystroke and instead wait for a time before saving the changes to the database.
 
 ```typescript
@Component({
  ...
  template: `
    <form [formGroup]="form" (ngSubmit)="save()">
        <input type="text" formControlName="firstName">
        <input type="text" formControlName="lastName">
        <button type="submit">Save</button>
    </form>
  `,
})
export class UserFormComponent implements OnInit {
  ...

  public readonly form = this.fb.group(...);

  ngOnInit() {
    this.form.valueChanges
      // Wait for 500ms after each keystroke before saving the user
      .pipe(debounceTime(500))
      .subscribe((user: User) => {
        this.userService.saveUser(user);
      });
  }

  ...
}
```

Check out the full example on [Stackblitz](https://stackblitz.com/edit/angular-nntsy9?file=src/main.ts){:target="_blank"}

## Conclusion
- `debounceTime` delays emitted values from a source Observable
- previous emitted values are dropped when a new value comes in before the delay is up
- `debounceTime` is a helpful operator when creating a real-time search and autosave