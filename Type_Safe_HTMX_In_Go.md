# Type-Safe HTMX Routes in Go

Over the past few months, I've been porting my [T3](https://create.t3.gg/) apps to Golang and HTMX. If you aren't familiar, T3 is a app template that incorporates Typescript, [tRPC](https://trpc.io/), React, and some other tools that make building React apps much better. It works great and I'd still recommend it if you're forced to use React, but it's time for me to move on. 

I want developing my websites to be simple again. I'm tired of all the new frameworks, new paradigms, and seemingly pointless changes the JS ecosystem requires these days.

_If only there were a language and framework for caveman that just want to build interesting and interactive websites that have a native-app-like user experience and are genuinely type-safe but are too dumb to understand barrow checkers and server vs client components..._

Obviously, this article is about Go and HTMX. I'll save all the reasons for that combination for another article discussing the entire rewriting process of my 60,000 line React code base, but today I want to focus on the primary pain point of HTMX: remembering all the routes and their parameters in a complicated app. Did I call that route `/user/{id}`, or `/users/{id}`, or `/user/{userId}`, or `/users/{userID}`?

To those that say this is a skill issue and you should use better naming conventions and simplify your api, I say, "Sure. But also build something more complicated than a todo app." I'm certain better rules would help this issue, with a complicated application like I'm building, I'll end up with hundreds of end points for different interactions on the pages and it's really hard to keep them all straight. This is exactly the problem tRPC solves and it's the main reason people love it. You don't need to remember all these endpoints anymore. Your LSP knows them for you and you just need to select the correct auto-complete.

Wouldn't be nice to have something like this: `<div ... hx-get={users.url(userId)}/>`? You'd never have to remember or type the full path again and you would know exactly which parameters you need and what they are named.

The above isn't valid Go code, so we need to choose our tooling smartly. My preferred HTML generation solution is Gomponents. It's nothing more than a set of functions that generate byte slices that get sent to the client as HTML. It looks like this:

```go
func MyComponent (userId string) g.Node {
    return Div(
        HX.Get("/users/"+userId), // this turns into hx-get="/users/user123"
        H1(
            g.Text("This is a heading text within the h1 tag.")
        ),
    )
}
```

Ignore the specific types and `g.` stuff for now. The point is Gomponents is JSX with function syntax instead of HTML syntax and all of it executes on the backend. It has all the same utilities of HTML and while the syntax does take a minute to get used to reading, it has all the same information and enables all the same functionality.

It also provides with a huge advantage when dealing with our type-safe route problem. We can use all of our Go functionality when creating our components because they are nothing more than Go functions already. We don't have to worry about pre-processing or any of the other challenges that come with some other templating solutions. My type-safe route solution can still be used with those other tools, but the routes will need to be sent to the template as a property instead of inlined like we're going to do with Gomponents.

So now that we have an easy way to create HTML using Go that will make our solution easier to create. This is what we want to end up with:

```go
func (api *Api) MyComponent (userId string) g.Node {
    return Div(
        HX.Get(api.Routes.User.Url(userId)), // again, this turns into hx-get="/users/user123"
        H1(
            g.Text("This is a heading text within the h1 tag.")
        ),
    )
}
```

If we could do this, we would never need to remember a route again. Like tRPC, we can just let our auto-complete create the routes for us and we would always know what parameters we need and how they are named.

This is our target. Let's build it.

```go
// routeBuilder.go
type param map[string]string

type pathBuilder struct {
	template    string
	params      param
	queryParams param
}

func newPath(template string) *pathBuilder {
	return &pathBuilder{
		template:    template,
		params:      param{},
		queryParams: param{},
	}
}

func (pb *pathBuilder) newParams(keys ...string) {
	for _, key := range keys {
		pb.params[key] = ""
	}
}

func (pb *pathBuilder) newQueryParams(keys ...string) {
	for _, key := range keys {
		pb.queryParams[key] = ""
	}
}
```

We'll start with defining our basic route `struct` and then create a couple helper functions that just make it easier to create the routes. You could certainly get by without these, but I find them helpful.

Now we'll create a function that generates our route:

```go
// usersApi.go
type User struct {
	pathBuilder
}

func createUser() *User {
	p := &User{*newPath("/users/{id}")}
	p.newParams("id")
	return p
}
```

One cool thing we are doing here is defining the `User struct` as a `pathBuilder`. This gives us all the properties of the `pathBuilder` with a different name so we can generate specific properties for `User` without messing with the underlying `struct`. This is what lets us do `routes.User.url()`. It's dangerously close to inheritance, but this is as close as we'll ever get to that sun, so don't worry too much.

We generated this route as a function rather than directly because we need an easy way to add it to our Api type like so:

```go

type Routes struct {
	User   *User
}


func CreateRoutes() Routes {
	return Routes{
		User:  createUser(),
    }
}
```

And then when we build our Api we'll add the routes to it using the `CreateRoutes` function:

```go

type Api struct {
    //...
	Routes Routes
}

func CreateApi() Api {
    // add your database connection and whatever else you need here
	return Api{
        //...
		Routes: CreateRoutes(),
	}
}
```

This is where we start to see the verbosity of Go come out. Again, Copilot makes this easy, but it's still not as terse as I would like. That said, we won't have to install any packages, rely on and dependencies, and we'll fully understand everything that's going on. Can you _akshuallee_ say that about tRPC?

Yes, we'll need to create `struct`, `createFunction`, and add them both to the `Routes struct` for every route. It's a pain, less than ideal, but that's the price of type-safety in Go.

With everything above, we can do this: `api.Routes.User` in our components for every route we add. Now we need to add some methods to generate the url with the proper parameters applied.

One more helper function for all of our paths:

```go
// routeBuilder.go
func (pb *pathBuilder) genBuilder(params, queryParams param) string {
	result := pb.template
	if len(params) > 0 {
		for key, value := range params {
			placeholder := "{" + key + "}"
			result = strings.Replace(result, placeholder, value, 1)
		}
	}
	if len(queryParams) > 0 {
		result += "?"
		for key, value := range queryParams {
			if value != "" {
				result += key + "=" + value + "&"
			}
		}
		result = result[:len(result)-1]
	}
	return result
}
```

This simply cycles through the parameters we pass in and adds them to the template so we end up with the proper output in our HTMX url.

Then we create a `User` specific `Url` function that will tell us to pass the `userId` and nothing else:

```go
// usersApi.go
type User struct {
	pathBuilder
}

func createUser() *User {
	p := &User{*newPath("/users/{id}")}
	p.newParams("id")
	return p
}
func (p *User) Url(userId string) string {
	params := param{"id": id}
	queryParams := param{}
	return p.pathBuilder.genBuilder(params, queryParams)
}
```

I repeated initial definitions so you can see everything in one place to properly assess if this pattern is getting out of control yet.

With this function in place, we achieved our goal! In our component, we can now use `api.routes.User.Url("user123")` and it will produce the correct url and add the parameters in the correct place for us.

This pattern has saved me seconds of going back and forth to my router to verify every route. It's so much nicer to just hit tab on the next auto-complete and continuing on than it is to hit escape to get back to normal mode, space 1 to harpoon back to my router, type /user to search for the path I'm looking for, verify the spelling and then space 2 to harpoon back to the original file and another i keystroke to get back to insert mode. If you don't see the beauty in writing 14 lines of code to save 10 keystrokes, we can't be friends.

Here is one more extension on this pattern that makes it even better.

```go
func (p *User) Handler(a *Api) (string, http.HandlerFunc) {
	return p.template, a.Auth(func(w http.ResponseWriter, r *http.Request) {
        // Handle it.
	})
}
```

I add this function right below the `url` function. Writing my handler function in this way gives me an easy way to access the rest of my `Api struct` so I have easy access to my db and other utilities, and the output of this function is the type my router expects so when I need to register this route in my router it looks like this:

```go
	a := api.CreateApi()
	R := chi.NewRouter()

	R.Get(a.Routes.User.Handler(&a))
```

This reduces one more place for me to make a typo and makes it super easy to register new routes.

All in all, ignoring the `pathBuilder` definition and its utility functions, each route does take an extra 15-20 repetitive lines of code, but everything related to that route is in exactly the same place. The path, the handler, and the parameter labels are all in one place and I have full type safety when I'm adding the route to my components. After getting used to the boilerplate, it feels like a really good compromise between Go and HTMX's simplicity and the benefits of tools like tRPC. Copilot makes it super easy to generate new routes and I honestly don't think it takes me any extra time to build my routes using this pattern

Is this pattern overkill for the one route we build above? Yes, 100%. If you only have 5 routes, there is no need to use this pattern. The second time a button doesn't work because you missed typed the path when you were writing the HTMX is when you should start thinking about this pattern.

It also becomes more attractive when your marketing guy tells you the payment page url should be `/sign-up` instead of `/payment` and you have to find and replace every instance in your code base and risk missing one instead of changing it in one place. I promise I've never had a marketing guy ask me to do that.

I'm still relatively new at Go, so this is unlikely to be the best implementation of this. If you have ideas on how to improve, please let me know.
