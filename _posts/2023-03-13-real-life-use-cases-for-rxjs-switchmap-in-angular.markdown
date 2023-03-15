---
layout: post
title:  "Real-life use cases for RxJS SwitchMap in Angular"
date:   2023-03-13 05:00:00 +0100
published: true
comments: true
categories: Angular
cover: "assets/real-life-use-cases-for-rxjs-switchmap-in-angular/real-life-use-cases-for-rxjs-switchmap-in-angular"
tags: [Angular, RxJS]
type: article
---

The `switchMap()` operator transforms an observable into another observable, while cancelling the previous subscription. By cancelling the previous subscription, the `switchMap()` operator prevents memory leaks and ensures that only the latest observable is subscribed to.

Because of this, `switchMap()` should be avoided in scenarios where every subscription needs to complete, for example when using the `switchMap()` operator with database-written HTTP requests (POST, PUT, PATCH, DELETE). In this case `mergeMap()` should be used instead.

In this article, we’ll discuss several real-life examples of how the RxJS `switchMap()` operator can be used in Angular.

Some typical use cases for the `switchMap()` operator are:
- Fetching Details from an API on a Routed Detail Screen
- Typeahead Search

## Detail screen with `switchMap()`

```typescript 
@Component({
  selector: 'my-car-brand-detail',
  standalone: true,
  imports: [CommonModule],
  providers: [CarBrandService],
  template: `
  Details:
  <ng-container *ngIf="carBrand$ | async as brand">
    <p>Brand ID: {{brand.id}}</p>
    <p>Brand Name: {{brand.name}}</p>
  </ng-container>
   `,
})
export class CarBrandDetailComponent {
  private readonly carBrandService = inject(CarBrandService);
  private readonly activatedRoute = inject(ActivatedRoute);

  // Listen for changes on the Router Params and map the params to an ID value
  private readonly id$ = this.activatedRoute.params.pipe(
    map((params) => params['id'])
  );

  // Every time the id$ changes, the carBand will be fetched
  public readonly carBrand$: Observable<CarBrand> = this.id$.pipe(
    switchMap(
      // SwitchMap returns an observable, in this case an Observable<CarBrand>
      (id) => this.carBrandService.getById(id)
    )
  );
}
```
Check out the full example on [Stackblitz](https://stackblitz.com/edit/angular-fehknb?file=src/car-detail.component.ts){:target="_blank"}.

## Typeahead with `switchMap()` and `ReactiveForms`

```typescript
@Component({
  selector: 'my-app',
  standalone: true,
  imports: [CommonModule, FormsModule, ReactiveFormsModule], 
  providers: [CarService],
  template: `
  <input [formControl]="searchTermControl" type="text">
  <ul>
    <li *ngFor="let brand of filteredCarBrands$ | async">{{ brand }}</li>
  </ul>
`,
})
export class AppComponent {
  private readonly carBrandService = inject(CarService);

  public readonly searchTermControl = new FormControl('');

  public readonly filteredCarBrands$: Observable<string[]> =
  // Every time the value changes of the fornControl this stream will be executed again
    this.searchTermControl.valueChanges.pipe(
      startWith(''), // Make observable hot => show all the car brands initially
      debounceTime(300), // Wait 300ms before searching
      switchMap((searchTerm) =>
        // Returns a list of Car Brands
        this.carBrandService.searchCarBrands(searchTerm)
      )
    );
}
```
Check out the full example on [Stackblitz](https://stackblitz.com/edit/angular-l3vtt1?file=src/main.ts){:target="_blank"}.



## Typeahead search with `switchMap()` and `BehaviorSubject`

```typescript
@Component({
    selector: 'my-app',
    standalone: true,
    imports: [CommonModule, FormsModule],
    providers: [CarService],
    template: `
      <input [ngModel]="searchTerm$$.value" (ngModelChange)="searchTerm$$.next($event)" type="text">
      <ul>
        <li *ngFor="let brand of filteredCarBrands$ | async">{{ brand }}</li>
      </ul>
`,
})
export class AppComponent {
    private readonly carBrandService = inject(CarService);

    public readonly searchTerm$$ = new BehaviorSubject('');
    private readonly searchTerm$ = this.searchTerm$$.asObservable();

    // Every time the searchTerm is changed then the carBrandService.searchCarBrands function is called
    public readonly filteredCarBrands$: Observable<string[]> =
        this.searchTerm$.pipe(
            debounceTime(300), // Wait 300ms before searching
            switchMap((searchTerm) =>
                // Returns a list of Car Brands
                this.carBrandService.searchCarBrands(searchTerm)
            )
        );
}
```

Check out the full example on [Stackblitz](https://stackblitz.com/edit/angular-ygswzq?file=src/main.ts){:target="_blank"}.

## Conclusion 
- The `switchMap()` operator is used to transform an observable into another observable.
- `switchMap()` cancels the previous subscription.
- Typically used for fetching data from an API on a routed detail screen or for a typeahead search.
