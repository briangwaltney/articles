# Type-Safe HTMX Routes in Go

Over the past few months, I've been porting my [T3](https://create.t3.gg/) apps to Golang and HTMX. If you aren't familiar, T3 is a app template that incorporates Typescript, [tRPC](https://trpc.io/), React, and some other tools that make building React apps much better. It works great and I'd still recommend it if you're forced to use React, but it's time for me to move on.

I want developing my websites to be simple again. I'm tired of all the new frameworks, new paradigms, and seemingly pointless changes in the JS ecosystem.

Obviously, this article is about Go and HTMX. I'll save all the reasons for that combination for another article discussing the rewriting process of my 60,000 line React code base, but today I want to focus on the primary pain point of HTMX: remembering all the routes and their parameters in a complicated app. Did I call that route `/user/{id}`, or `/users/{id}`, or `/user/{userId}`, or `/users/{userID}`?

To those that say, "Git gud, loser. Use better naming conventions and simplify your api," I say, "Sure. But also, build something more complicated than a todo app." I'm certain better rules would help me, but complicated applications end up with hundreds of endpoints for different interactions on pages and it's really hard to keep them all straight no matter how many rules I have in place. This is exactly the problem tRPC solves and it's the main reason people love it. You don't need to remember all these endpoints anymore. Your LSP recognizes them and you just need to select the correct suggestion.

Wouldn't it be nice to have something like this: `<div ... hx-get={users.url(userId)}/>` in your go application? Well you can't! This is Go, not some [mocha choca bullshit](https://youtu.be/B9IYeVJiWLQ?si=xEmHu-xZ43h3XlSu&t=118) language. Well, we can. But we need to build it ourselves.

Before we get into the coding our routes, we need to choose our tooling smartly. [Gomponents](https://www.gomponents.com/) is the easiest answer here. It's nothing more than a set of functions that generate byte slices that get sent to the client as HTML. It looks like this:

```go
func MyComponent (userId string) g.Node {
    return Div(
        HX.Get("/users/"+userId),
        H1(
            g.Text("Heading Text")
        ),
    )
}
// This results in
// <div hx-get="/users/user123">
//   <h1>Heading Text</h1>
// </div>
```

Gomponents is like JSX with function syntax instead of HTML syntax and all of it executes on the backend and doesn't require any pre-processing. It has all the same utilities of HTML and while the syntax does take a minute to get used to reading, it has all the same information and enables all the same functionality.

It also provides with a huge advantage when dealing with our type-safe route problem. The entire component is just a collection of Go functions, so we can use valid Go code within it and everything will work fine.

Our components can look like this and will just work™️:

```go
func (api *Api) MyComponent (userId string) g.Node {
	return Div(
		HX.Get(api.Routes.User.Url(userId)),
		H1(
			g.Text("Heading Text")
		),
	)
}
```

Now that we have our target. Let's build it.

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

We'll start with defining our basic route `struct` and then create a couple helper functions that just make it easier to create the parameters. You could certainly get by without these, but I find them helpful. This will be the template for all our paths.

Now we'll create a function that generates our `user` route:

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

One cool thing we are doing here is defining the `User struct` as a `pathBuilder`. This gives us all the properties of the `pathBuilder` with a different name so we can generate specific properties for `User` without messing with the underlying `struct`. This is what will let us do `Routes.User.Url()` shortly. It's dangerously close to inheritance, but this is as close as we'll ever get to that sun, so don't worry too much.

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

type Api struct {
    // db or other stuff
	Routes Routes
}

func CreateApi() Api {
    // create your db connection
	return Api{
        // db or other stuff
		Routes: CreateRoutes(),
	}
}
```

Yes, we'll need to create `struct`, `createFunction`, and add them both to the `Routes struct` and the `CreateRoutes` function for every route. It's a pain, but that's the price of type-safety in Go and it's still easier to understand than `getServerSideProps`. I split the `Routes struct` and the `CreateRoutes` function into a separate file because I think it's cleaner as the app grows, but do what you think is best.

The verbosity of Go is really starting to come out now. That said, we won't have to install any packages, rely on and dependencies, and we fully understand everything that's going on. Can you _akshuallee_ say that about tRPC?

With everything above, we can do this: `api.Routes.User` in our components, but we need to add a method to generate the URL with the proper parameters applied.

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

This simply loops through the parameters we pass in and adds them to the template so we end up with the proper output URL. Again, you can get by without this if you want to generate the string yourself. This just makes life a little easier.

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
	return p.genBuilder(params, queryParams)
}
```

I repeated initial definitions so you can see everything in one place to properly assess if this is getting out of control yet.

With this function in place, we achieved our goal!

In our component, we can now use `api.Routes.User.Url("user123")` and it will produce the correct URL and add the parameter in the correct place for us. We also have all the tools in place for more complicated routes. You can add more arguments to the Url function to accept more parameters or query parameters or even accept a custom `struct` if you have an especially ugly path that has some optional query parameters. The `genBuilder` function is going to ignore all the `""` values for you and leave you with the clean URL.

This pattern has saved me ***seconds*** of going back and forth to my router definitions to verify paths. It's so much nicer to accept the next auto-complete and continuing on than it is to hit `<ESC>` to get back to normal mode, `<leader>1` to harpoon back to my router, `/` `user` to search for the path I'm looking for, verify the spelling and then `<leader>2` to harpoon back to the original file and another `i` keystroke to get back to insert mode. If you don't see the beauty in writing 14 lines of code to save 12 keystrokes, we can't be friends.

Here is one more extension on this pattern that makes it even better.

```go
func (p *User) Handler(a *Api) (string, http.HandlerFunc) {
	return p.template, a.AuthMiddleware(func(w http.ResponseWriter, r *http.Request) {
        // backend magic stuff
        a.MyComponent("user69").Render(w)
	})
}
```

I add this function right below the `Url` function so everything is together. It's easy for me to see the path, parameters, and handler all in one place. One added benefit of having everything colocated like this is that changing the url for something means changing one line of code. All of my links, buttons, and HTMX calls will update automatically. They all reference the same path I wrote in the `createUser` function.

Writing my handler function in this way also gives me an easy way to access the rest of my `Api struct` so I have easy access to my middleware, db, and anything else I've added for easy access, and the `(string, http.HandlerFunc)` output of this function is the type my router expects so registering my route is as simple as:

```go
	a := api.CreateApi()
	R := chi.NewRouter()

	R.Get(a.Routes.User.Handler(&a))
	// more routes

	srv := &http.Server{
		Handler: R,
		Addr:    "localhost:8000",
	}

	err := srv.ListenAndServe()

```

This reduces one more place for me to make a typo and makes it super easy to register new routes.

All in all, ignoring the `pathBuilder` definition and its utility functions, each route requires an extra 15-20 repetitive lines of code, but the type-safety has been worth it to me. It feels like a good compromise between Go and HTMX's simplicity and the benefits of tools like tRPC. Most of the extra typing is the type of boilerplate Copilot is perfect for. After creating my first route, Copilot generates the rest for me nearly automatically and it takes almost no extra time to write new paths.

For small applications, this is overkill and you should only do it if using your LSP to its fullest gets you to half mast like it does me. Let's be honest, T3 is overkill for many small applications as well. This pattern really starts to shine when your routes get more complicated and you're using them in multiple places. My rule is wait until I've created 2 HTMX calls that didn't work because I had a typo in the path and then go back and adding this to my application. One nice feature of this pattern is you can add it incrementally. We're just dealing with strings so all your old ones will still work.

If you try this pattern, let me know what you think of it or if you find a way to make it more terse. I would love to see your solutions to this problem.

Don't forget to subscribe to the channel. The name is the Type-Safe HTMXagen.
