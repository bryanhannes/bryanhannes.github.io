---
layout: post
title:  "Reactively storing and retrieving URL state in Angular"
date:   2023-01-07 10:00:00 +0100
published: true
comments: true
categories: Angular
cover: assets/reactively-storing-and-retrieving-url-state-in-angular/reactively-storing-and-retrieving-url-state-in-angular.png
---

In this article. we'll explore different techniques you can use to store state in the URL, including query parameters and route parameters. We also build an application that saves the search filters in the URL.
By the end of this article, you'll have a good understanding of how to implement URL state management reactively in Angular.

Storing state in the URL can improve the user experience of an application because it allows the state to be preserved when the page is refreshed, bookmarked, or shared. 

But we don't want to save all types of state to the URL, some type of state is more suitable to save to the URL than others. Some state that is perfect to store in the URL is:
- Filters
- Sort criteria
- Page Numbers
- Ids (for detail pages)

## 2 Types of states in the URL
There are 2 types of states in the URL

1. Query Parameters
2. Route Parameters

### Query Parameters
Query parameters are key-value pairs that are added to the end of the URL after a `?` character and every parameter is added after this in a `key=value` format, separated by a `&`.

```url
https://blog.bryanhannes.com?category=angular&page=2
```

### Route parameters
Route parameters are mostly used to identify a specific resource. A route parameter is part of the URL and it is not per se added to the end of the URL.
They have a `:key` syntax.

```url
https://blog.bryanhannes.com/products/:productId
```

So in this example, `angular` is the productId.
```url
https://blog.bryanhannes.com/products/angular
```


## The car catalog application
We are going to build a simple car catalog application in Angular 15 where we can search for cars. The search filters will be stored in the URL as query parameters.

*Note: we are using standalone components in the code.*

This is what the application looks like:

<img src="/assets/reactively-storing-and-retrieving-url-state-in-angular/car-catalog-application.png" alt="Overview of the car catalog application" alt="Overview of the car catalog application">


We use the `app.component` as the container or smart component. If we have a look at the template we notice that there are 2 UI components: `SearchFilterComponent` and `SearchResultsComponent`.


```html
<!-- app.component.html -->

<ng-container *ngIf="vm$ | async as vm">
  <h1>Car catalog</h1>

  <app-search-filter
    [name]="vm.name"
    [brand]="vm.brand"
    [color]="vm.color"
    (filterChanged)="filterChanged($event)"
  ></app-search-filter>

  <button type="button" (click)="clear()">Clear</button>

  <app-search-results [cars]="vm.results"></app-search-results>
</ng-container>
```

First, let's see what the `SearchFilterComponent` and `SearchResultsComponent` look like.


### The SearchFilterComponent
The search filter component renders the filters: `name`, `brand` and `color`.
This component also emits an event when a filter changes `filterChanges`.

The component takes in three inputs (`name`, `brand` and `color`) which are used to set the value of the input (`name`) and selects (`brand` and `color`).

```html
<!-- search-filter.component.html -->

<div class="filter">
  <label for="name">Name:</label>
  <input type="text" id="name" [ngModel]="name" (keyup)="updateName($event)" />
</div>

<div class="filter">
  <label for="brand">Brand:</label>
  <select
    name="brand"
    id="brand"
    [ngModel]="brand"
    (ngModelChange)="updateBrand($event)"
  >
    <option value="Toyota">Toyota</option>
    <option value="Ford">Ford</option>
    <option value="Volkswagen">Volkswagen</option>
  </select>
</div>

<div class="filter">
  <label for="name">Color:</label>

  <select
    name="color"
    id="color"
    [ngModel]="color"
    (ngModelChange)="updateColor($event)"
  >
    <option value="black">Black</option>
    <option value="blue">Blue</option>
    <option value="red">Red</option>
    <option value="green">Green</option>
  </select>
</div>

```

```typescript
// search-filter.component.ts

import {
  ChangeDetectionStrategy,
  Component,
  EventEmitter,
  Input,
  Output,
} from '@angular/core';
import { FormsModule } from '@angular/forms';
import { CarFilter } from '../model/car-filter';

@Component({
  selector: 'app-search-filter',
  templateUrl: './search-filter.component.html',
  styleUrls: ['./search-filter.component.css'],
  imports: [FormsModule],
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class SearchFilterComponent {
  @Input() public name: string = '';
  @Input() public brand: string = '';
  @Input() public color: string = '';

  @Output() public readonly filterChanged = new EventEmitter<CarFilter>();

  public updateName(event: KeyboardEvent): void {
    const value = (event.target as HTMLInputElement).value;
    this.filterChanged.emit({ name: value });
  }

  public updateBrand(value: string): void {
    this.filterChanged.emit({ brand: value });
  }

  public updateColor(value: string): void {
    this.filterChanged.emit({ color: value });
  }
}

```

### Search Results component
The search results component renders the table of cars, this component has one input called `cars`.

```html
<!-- search-results.component.html -->

<table>
  <thead>
    <tr>
      <th>Name</th>
      <th>Brand</th>
      <th>Color</th>
    </tr>
  </thead>

  <tbody>
    <tr *ngFor="let car of cars">
      <td>{{ car.name }}</td>
      <td>{{ car.brand }}</td>
      <td>{{ car.color }}</td>
    </tr>
    <tr *ngIf="cars?.length === 0">
      <td colspan="3">No cars found</td>
    </tr>
  </tbody>
</table>
```

```typescript
// search-results.component.ts

import { CommonModule, JsonPipe } from '@angular/common';
import { ChangeDetectionStrategy, Component, Input } from '@angular/core';
import { Car } from '../model/car';

@Component({
  selector: 'app-search-results',
  templateUrl: './search-results.component.html',
  styleUrls: ['./search-results.component.css'],
  imports: [CommonModule],
  changeDetection: ChangeDetectionStrategy.OnPush,
  standalone: true,
})
export class SearchResultsComponent {
  @Input() public cars: Car[] = [];
}
```

### AppComponent
The `AppComponent` is the place where the most interesting stuff will happen:
- Retrieve the query parameters with `ActivatedRoute`
- Automatically fetch the cars from the API via the `CarService` when the filters change
- ViewModel to show/pass data in the template


#### Retrieving the query parameters
The first thing we have to do in the AppComponent is to retrieve the query parameters from the URL and map the `Params` to a `CarFilter`. 

We can use the `ActivatedRoute` service to retrieve the query parameters from the current URL. We can transform the `Params` to a `CarFilter` with the help of the `map()` RXJS operator.

```typescript

activatedRoute = inject(ActivatedRoute);

queryParams$: Observable<CarFilter> = this.activatedRoute.queryParams.pipe(
  map((params) => ({
    name: params.name,
    brand: params.brand,
    color: params.color,
  }))
);
```
To ensure that the search results are automatically updated whenever the query parameters change, we can use the `pipe()` and `switchMap()` operators on the `queryParams$` observable. The `switchMap()` operator is used because it allows us to return another observable (in this case, the observable of search results returned by the `carService.findCars()` method).

```typescript
carService = inject(CarService);

results$: Observable<Car[]> = this.queryParams$.pipe(
  switchMap((carFilter: CarFilter) => {
    return this.carService.findCars(carFilter);
  })
);
```

To use both the search results (`results$`) and the search filters (`queryParams$`) in the template, we can create a [ViewModel](https://blog.simplified.courses/reactive-viewmodels-for-ui-components-in-angular/){:target="_blank"} that `combineLatest`()` with both observables.

Using the `map()` operator, we can transform the array of params and result into a `PageViewModel` object.

```typescript

interface PageViewModel {
  name: string;
  brand: string;
  color: string;
  results: Car[];
}

vm$: Observable<PageViewModel> = combineLatest([
    this.queryParams$,
    this.results$,
  ]).pipe(
    map(([params, results]) => ({
      name: params.name,
      brand: params.brand,
      color: params.color,
      results,
    }))
  );
```

Thanks to the `vm$` View Model we have only 1 `async` pipe in the template and so only 1 subscription. It also makes the template a lot cleaner.

```html
<!-- app.component.html -->

<ng-container *ngIf="vm$ | async as vm">
  <h1>Car catalog</h1>

  <app-search-filter
    [name]="vm.name"
    [brand]="vm.brand"
    [color]="vm.color"
    (filterChanged)="filterChanged($event)"
  ></app-search-filter>

  <button type="button" (click)="clear()">Clear</button>

  <app-search-results [cars]="vm.results"></app-search-results>
</ng-container>

```

The last thing we need to do in the `AppComponent` is, add the filters (name, brand and name) to the URL as query parameters whenever the filters change. Luckily we have defined an `@Output()` in the `SearchFilterComponent` which emits whenever the filters change.

To update the query parameters of the current route without navigating to a new route, we can use the `Router` service and pass an empty array as the first parameter. The property queryParamsHandling: 'merge' allows us to merge the new query parameters with the existing ones. 

```typescript
  public filterChanged(filter: CarFilter): void {
    this.router.navigate([], {
      queryParams: filter,
      queryParamsHandling: 'merge',
    });
  }
```


Our final `AppComponent` looks like this:

```typescript
// app.component.ts

import { Component, inject } from '@angular/core';
import { SearchFilterComponent } from './search-filter/search-filter.component';
import { ActivatedRoute, Router, Params } from '@angular/router';
import { combineLatest, map, Observable, switchMap } from 'rxjs';
import { CommonModule } from '@angular/common';
import { CarService } from './services/car.service';
import { SearchResultsComponent } from './search-results/search-results.component';
import { CarFilter } from './model/car-filter';
import { Car } from './model/car';

interface PageViewModel {
  name: string;
  brand: string;
  color: string;
  results: Car[];
}

@Component({
  selector: 'my-app',
  templateUrl: './app.component.html',
  styleUrls: ['./app.component.css'],
  imports: [CommonModule, SearchFilterComponent, SearchResultsComponent],
  providers: [CarService],
  standalone: true,
})
export class AppComponent {
  private readonly activatedRoute = inject(ActivatedRoute);
  private readonly router = inject(Router);
  private readonly carService = inject(CarService);

  // Source
  private readonly queryParams$: Observable<CarFilter> =
    this.activatedRoute.queryParams.pipe(
      map((params: Params) => ({
        name: params.name,
        brand: params.brand,
        color: params.color,
      }))
    );

  // Intermediate
  private readonly results$: Observable<Car[]> = this.queryParams$.pipe(
    switchMap((carFilter: CarFilter) => {
      return this.carService.findCars(carFilter);
    })
  );

  // Presentation
  public readonly vm$: Observable<PageViewModel> = combineLatest([
    this.queryParams$,
    this.results$,
  ]).pipe(
    map(([params, results]) => ({
      name: params.name,
      brand: params.brand,
      color: params.color,
      results,
    }))
  );

  // We are using queryParamsHandling: 'merge', because we want to merge all query parameters
  // Let's say brand is already selected then our URL looks like this:
  // example.com?brand=ford
  // When we select a color we want to add the color (merge) to the already existing query parameters
  // example.com?brand=ford&color=red
  // If we don't set queryParamsHandling to 'merge' then the latest filter will override the previous one and will there be only one query parameter
  public filterChanged(filter: CarFilter): void {
    this.router.navigate([], {
      queryParams: filter,
      queryParamsHandling: 'merge',
    });
  }

  public clear() {
    this.router.navigate([]);
  }
}

```


## Conclusion
- Now we know the different techniques for storing state in the URL: query and route parameters.
- To subscribe to changes in query parameters, we can use the `ActivatedRoute` service.
- We can use the `queryParamsHandling` property to specify how the `Router` should merge new query parameters with the existing ones when navigating to a new route. By setting `queryParamsHandling: 'merge'`, we can merge the new query parameters with the existing ones.
- We saw how the `ViewModel` approach makes our templates cleaner.


Here you check out the full Stackblitz demo:
<iframe src="https://stackblitz.com/edit/angular-ivy-y8vtzw" width="100%" height="500px"></iframe>

