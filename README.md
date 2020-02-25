# Angular Route Resolver
- Resolver is a service that has to be [provided] in the root module.
- Basically, a Resolver acts like middleware, which can be executed before a component is loaded.

#### Why Use Route Resolvers in Angular?
## Scenario
let's think about the scenario when you are using ```*ngIf="some condition"```, and your logic relies on the length of an array, which is manipulated when an API call is completed. For example, you may want to display in a component the items of this array which were just fetched in an unordered list.

```javascript
<ul>
  <li *ngFor="let item of items">{{item.description}}</li>
</ul>
```

In this situation, you might get into a problem because your data will come up after the component is ready. Items in the array do not really exist yet. Here, the Route Resolver comes in handy. Angular’s Route Resolver class will fetch your data before the component is ready. Your conditional statements will work smoothly with the Resolver.

## Resolve Interface
First see how the Resolve interface looks like.
```javascript
export interface Resolve<T> {
  resolve(route: ActivatedRouteSnapshot, state: RouterStateSnapshot): Observable<T> | Promise<T> | T {
  return 'Data resolved here...'
  }
}
```
If you want to create a Route Resolver, you have to create a new class that will implement the above interface. This interface provides us with a resolve function, which gets you two parameters in case you need them. The first one is the route, which is of type ActivatedRouteSnapshot, and the second one is the state, of type RouterStateSnapshot. Here, you can make an API call that will get the data you need before your component is loaded.

Via the route parameter, you can get route parameters that may be used in the API call. The resolve  method can return an Observable, a promise or just a custom type.

**Note:** It is important to mention that only resolved data will be returned via this method. 

## Implementation

**Step 1:** Sending Data to the Router via a Resolver
```javascript
items: any[] = [
  { description: 'Item 1' },
  { description: 'Item 2' },
  { description: 'Item 3' },
];
getData(): Observable<any[]> {
  const observable = Observable.create(observer => {
    observer.next(this.items)
  });
  return observable;
}
```

So, now, you will see that the subscription that we have, never hits. This is caused because you are not sending the data correctly. You are not complete the subscription. To fix that error, you need to complete the subscription by adding one more line of code.

```javascript
observer.complete();
```

**Step 2:** Implementing a Route Resolver

First of all, we will need a service that will fetch the user data for us. In this service, we have a function called getUsers() that returns an observable.

```javascript

@Injectable({
providedIn: 'root'
})
export class FakeApiService {
  constructor(private http: HttpClient) { }
  private usersEndpoint = "https://jsonplaceholder.typicode.com/users";
  getUsers(): Observable<any> {
    // We do not subscribe here! We let the resolver take care of that...
    return this.http.get(this.usersEndpoint);
  }
}
```

It is important no to subscribe to the function getUsers. The route resolver called UserResolver will take care of this for you. The next step is to create a new service called UserResolver which will implement the resolve function of the Resolve interface of the router.

```javascript
@Injectable({
  providedIn: 'root'
})
export class UserResolverService implements Resolve<any> {
  constructor(private fakeApi: FakeApiService) { }
  resolve(route: ActivatedRouteSnapshot, state: RouterStateSnapshot) {
    return this.fakeApi.getUsers().pipe(
      catchError((error) => {
      return empty();
      });
    );
  }
}
```
- This service, UserResolver, will subscribe automatically to the getUsers observable and provide the router with the fetched data. In case of an error, while fetching the data, you can send an empty observable and the router will not proceed to the route.
- The navigation will be terminated at this point. This last step is to create a component that will be called when the user goes to the /users route. Typically, without a Resolver, you will need to fetch the data on the ngOnInit hook of the component and handle the errors caused by ‘no data’ exists. The user's component is a simple one. It just gets the user's data from the ActivatedRoute and displays them into an unordered list.
- After you have created the user's component, you need need to define the routes and tell the router to use a resolver ( UserResolver). This could be achieved with the following code into the  app-routing.modulte.ts.

```javascript
const routes: Routes = [
  { path: 'users', component: UsersComponent, resolve: { users: UserResolverService } }
];
@NgModule({
  imports: [
  CommonModule,
  FormsModule,
  HttpClientModule,
  RouterModule.forRoot(routes)],
  exports: [RouterModule]
})
export class AppRoutingModule { }
```

You need to set the resolve property into the user's route and declare the UserResolver. The data will be passed into an object with a property called users. After that, you are almost done. There is only one thing you need to do. You must get the fetched data into the users' component via the ActivatedRoute with the following code.

```javascript
constructor(private activatedRoute: ActivatedRoute) { }
users: any[];
ngOnInit() {
  this.activatedRoute.data.subscribe((data: { users: any }) => {
  this.users = data.users;
  });
}
```
Then, you can just display them into HTML without any *ngIf statements ```( *ngIf=”users && users.length > 0 )``` because the data will be there before the component is loaded.

```html
<h2>Fetched Users:</h2>
<ul>
  <li *ngFor="let user of users">{{ user.name }}</li>
</ul>
```
###### Reference
https://dzone.com/articles/understanding-angular-route-resolvers-by-example
