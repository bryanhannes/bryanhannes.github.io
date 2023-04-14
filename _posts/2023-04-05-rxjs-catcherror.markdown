---
layout: post
title:  "RxJS catchError: error handling"
date:   2023-04-13 05:00:00 +0100
published: true
comments: true
categories: Angular
cover: "assets/rxjs-catcherror/rxjs-catcherror"
tags: [Angular, RxJS]
type: article
---
Error handling literally makes or breaks an application. In synchronous code, we can use try/catch blocks, but when working with RxJs we can't.
When working with RxJS we have to use error handling operators, one of them is the `catchError` operator.

The `catchError` operator takes in an observable that *might* emit an error and
- if it doesn't, it will just return the original observable.
- if it does, it will catch the error and pass it to the error handling function that you provide.
  - This error handling function can return a new observable or can rethrow the error.


Example of a try/catch block:
```typescript
try {
  // Some code that might throw an error
    const foo = someFunctionThatMightThrowAnError();
} catch (err) {
  // Handle the error
}
```

Example of `catchError`
```typescript
// Some observable that *might* emit an error 
const someObservable$ = throwError(() => new Error('I am an error'));
someObservable$
    .pipe(
        // Input observable throws an error, and original observable completes
        // catchError returns a new observable that emits 1 value -> the message of the error: 
        // "I am an error"
        catchError((error) => of(error.message))
    )
.subscribe({
    // Next callback is called in this example:
    // It will console.log the newly created observable from catchError
    next: (v) => console.log('Value:', v), 
    error: (e) => console.error('Error:', e), // Error callback is not called in this example
    complete: () => console.info('Completed'), // Complete callback is called in this example
});

// Outputs:
// Value: I am an error
// Completed
```

In this blog post about `catchError` we'll cover:
- How to catch errors and replace them with a fallback observable
- How to catch errors and rethrow them
- Conclusion

## How to catch errors and replace them with a fallback observable
Sometimes we want to cover up a thrown error and replace it with a fallback value. 

In this example we'll replace an HTTP error with an empty array.

```typescript
// Import the catchError operator
import { catchError, ... } from 'rxjs/operators';
...

// Some observable that performs an HTTP call and that *might* emit an error
const httpCall$ = fromFetch('/api/some-endpoint');

httpCall$
    .pipe(
        // Mimic an HTTP error
        switchMap(() => throwError(() => new Error('HTTP error'))),
        // Catch the error and return a new Observable that emits 1 value of an empty array []
        // Here we pass in the 'error handling function'
        catchError((error) => {
            // Here you would put your custom error handling logic, eg:
            // - Log the error
            // - Send the error to a logging service
            // - Show a toast message

            // Return a new Observable that emits 1 value of an empty array []
            return of([]);
        })
    )
    .subscribe({
        // Will console.log the newly created observable from catchError
        next: (v) => console.log('Value', v),
        // Error callback is not called in this example because we return a new Observable that emits 1 value of []
        error: (e) => console.error('Error', e.message), 
        complete: () => console.info('complete'),
    });

// Outputs: 
// Value []
// complete
```

## How to catch errors and rethrow them
In some use cases we want to catch thrown errors and perform some custom error handling logic and/or rethrow the error with a new message.

In this example we'll catch an HTTP error and rethrow it with a new message.

```typescript
// Import the catchError operator
import { catchError, ... } from 'rxjs/operators';
...

// Some observable that performs an HTTP call and that *might* emit an error
const httpCall$ = fromFetch('/api/some-endpoint');

httpCall$
    .pipe(
        // Mimic an HTTP error
        switchMap(() => throwError(() => new Error('HTTP error'))),
        // Catch the error and perform custom error handling logic and then rethrow the error 
        // with a new message
        catchError((error) => {
            // Here you would put your custom error handling logic, eg:
            // - Log the error
            // - Send the error to a logging service
            // - Show a toast message

            // Throw a new error with a new message: 'Something went wrong'
            return throwError(() => new Error('Something went wrong'));
        })
    )
    .subscribe({
        // Next callback is not called in this example
        next: (v) => console.log('Value', v),
        // Error callback is called in this example because we rethrow the error
        error: (e) => console.error('Error', e.message),
        complete: () => console.info('complete'),
    });

// Outputs: 
// Error Something went wrong
```


## Conclusion
- `catchError` is an error handling operator that takes in an observable that *might* emit an error
- `catchError` should return an Observable either a fallback observable or a rethrown error


