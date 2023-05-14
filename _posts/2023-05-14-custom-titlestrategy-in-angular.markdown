---
layout: post
title:  "Custom TitleStrategy in Angular"
date:   2023-05-14 05:00:00 +0100
published: true
comments: true
categories: Angular
cover: "assets/custom-titlestrategy-in-angular/custom-titlestrategy-in-angular"
tags: [Angular]
type: article
---

The page title is an essential element for both SEO and user experience. In Angular, there are a few ways to set a static page title, such as in the `index.html` file, where the page title will be the same for the whole application. Or, we can set a title per route, where the title will be different for each route.

```typescript
// Title per route with the "title" property
const myRoutes = [
    { path: 'home', component: HomeComponent, title: 'Where it all starts' },
    { path: 'about', component: AboutComponent, title: 'About our awesome team' },
]
``` 

However, sometimes we need to set a dynamic page title based on the current route or data. In such cases, we can achieve this by using a Custom Title Strategy in Angular.
## Using a Custom Title Strategy
Setting up a Custom Title Strategy is easy. All we need to do is create a service that extends the TitleStrategy class from @angular/router. Here is an example of a CustomTitleStrategy that sets the page title to the current route title if it's set; otherwise, it will use the default title from index.html and add a suffix to the title:

```typescript
@Injectable({
    providedIn: 'root',
})
export class CustomTitleStrategy extends TitleStrategy {
    private readonly title: Title = inject(Title);

    public updateTitle(snapshot: RouterStateSnapshot): void {
        // PageTitle is equal to the "Title" of a route if it's set
        // If its not set it will use the "title" given in index.html
        const pageTitle = this.buildTitle(snapshot) || this.title.getTitle();
        this.title.setTitle(`${pageTitle} - My awesome application`);
    }
}

```

We also need to tell Angular that we want to use our `CustomTitleStrategy` class instead of the default `TitleStrategy` class. We can achieve this using `useClass`. Here's an example:
```typescript
bootstrapApplication(App, {
  providers: [
      // The app should have routes configured or the TitleStrategy will not be used
      provideRouter(appRoutes), 
    {
      provide: TitleStrategy, 
      // Will tell Angular DI to inject our CustomTitleStrategy class when TitleStrategy is requested 
      useClass: CustomTitleStrategy, 
    },
  ],
});
```

In conclusion, using a Custom Title Strategy in Angular is a quick win to improve the applications SEO and user experience. You can find the full code example on [StackBlitz](https://stackblitz.com/edit/angular-ttf65p?file=src/main.ts){:target="_blank"} .

