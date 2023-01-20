---
title: "HTMX with a fake backend - hypermedia-server"
date: 2023-01-20T14:53:31-05:00
draft: false
type: "post"
---

[HTMX](https://htmx.org/) is an awesome tool to quickly add some dynamic behaviour to the frontend of applications, without alot of the machinery associated with feature rich frameworks like react, svelte or vue.

When starting projects, I’m often looking to translate a rough design or flow into interactive pages. Figma can be great first step to move towards a clickable prototype, but what’s the next? HTMX fits in really nicely to this stage, where I’d like to shift over into real markup and begin fleshing out some of the high-level model objects. The question I ask myself is often, how do I make an experiment real without alot of complexity? When is it time to reach for django/rails? Similarly, do I really need to start projects with React? Svelte? Next.js?

I think the usual decision for people is whatever you’re comfortable with, in my case Django and Lit or React. But lately I’ve been exploring this in a bit more detail. What if I just… didn’t introduce these tools at this stage? How far can I get without this stuff? What’s missing or time-consuming to get going?

In this post I’m going to introduce an extension of the API faking library `json-server` that adds hypermedia responses. Instead of a RESTful api returning JSON, we render that same data through through simplistic templates, returning html blocks. When working with HTMX and markup, I find this has been a great tool to explore ideas quickly.

To pull it down:

```bash
git clone [https://github.com/zachgoldstein/hypermedia-server.git](https://github.com/zachgoldstein/hypermedia-server.git)`
```

Run it locally via `npx`

```bash
npx babel-node -- src/cli/bin ./examples/boops/db.json --hypermedia --template ./examples/boops/templates.json
```

Let’s serve our little example page quickly:

```bash
cd hypermedia-server/examples/boops
python3 -m http.server
```

Now if we go to [`http://localhost:8000/page.html`](http://localhost:8000/page.html), we have a running CRUD app with no backend!

{{< figure src="/posts/hypermedia-server/boop-demo.gif" class="center">}}

Let’s dig into this toy example in a bit more detail. Here’s what it’s doing with HTMX and a fake backend library:

- Click the boop button and a `POST` request to `/boops` will create a new boop model object.
- The `POST` request returns an html block for the newly created boop, which is shown as some confirmation text under our boop button.
- A `<ul>` element shows all our boop model objects by issuing a `GET` request to the plural `/boops` endpoint. It’s triggered after our initial page load and when clicking the boop button.
- The `/boops` endpoint returns an html block with A new `<ul>` element, htmx is swapping our `<ul>` object’s contents with the newly returned html block representing all boops.
- Each `<li>` element representing a boop has a button on it to delete, update or get that boop model object.
- Clicking the `DELETE` button on a boop issues a `DELETE` request, deleting the boop and then swapping in the empty html block that is getting returned.
- Clicking the `UPDATE` button on a boop issues a `PUT` request with a new timestamp, updating the boop and then swapping in the returned html block with the updated boop.
- Clicking the `GET` button on a boop issues a `GET` request for that specific boop, returning it’s html representation and swapping in the returned html block with the updated boop.

Our page’s markup looks like this:

```html
<body>
		<button class="boop-btn" 
        hx-post="/boops" 
        hx-trigger="click" 
        hx-target=".new-boop">
        Click to BOOP</button>
    <div class="new-boop"></div>

    <div class="boops" 
        hx-get="/boops" 
        hx-trigger="load, click from:.boop-btn" 
        hx-target=".boop-list" 
        hx-swap="outerHTML">
        <h3>Boops:</h3>
        <ul class="boop-list"></ul>
    </div>
</body>
```

Full page at [https://github.com/zachgoldstein/hypermedia-server/blob/master/examples/boops/page.html](https://github.com/zachgoldstein/hypermedia-server/blob/master/examples/boops/page.html)

Our hypermedia returning fake API will respond to a plural `GET` request with an html block of all the boops in the `db.json` file we provide, which acts as a very simple DB via [lowdb](https://github.com/typicode/lowdb). In that html block, each boop object has buttons to `GET`, `UPDATE` and `DELETE` the resource.

The `get-all-boops.ejs` template, returning a list of all boops:
```html
<ul class="boop-list">
  <% top.forEach(function(boop) { %>
  <li id="boop-<%= boop.id %>">
    <p><strong>Boop: <%= boop.id %></strong> time updated: <%= boop.time %></p>
    <button 
        hx-delete="/boops/<%= boop.id %>" 
        hx-target="#boop-<%= boop.id %>"
        hx-swap="outerHTML">
        Delete
    </button>
    <button 
        hx-put="/boops/<%= boop.id %>" 
        hx-vals='js:{time: Date.now()}' 
        hx-target="#boop-<%= boop.id %>">
        Update
    </button>
    <button 
        hx-target="#boop-<%= boop.id %>"
        hx-get="/boops/<%= boop.id %>">
        Get
    </button>  
  </li>
  <% }); %>
</ul>
```

And for a single boop model object we have `get-boop.ejs`:

```html
<li id="boop-<%= top.id %>">
    <p><strong>Boop: <%= top.id %></strong> time updated: <%= top.time %></p>
    <button 
        hx-delete="/boops/<%= top.id %>" 
        hx-target="#boop-<%= top.id %>"
        hx-swap="outerHTML">
        Delete
    </button>
    <button 
        hx-put="/boops/<%= top.id %>" 
        hx-vals='js:{time: Date.now()}' 
        hx-target="#boop-<%= top.id %>">
        Update
    </button>
    <button 
        hx-target="#boop-<%= top.id %>"
        hx-get="/boops/<%= top.id %>">
        Get
    </button>
</li>
```

For these templates to work, we provide a mapping file, `templates.json` that looks like this:

```json
{
	"boops":{
		"POST":"post-boop.ejs",
		"GET":"get-boop.ejs",
		"GET-ALL":"get-all-boops.ejs",
		"PUT":"get-boop.ejs",
		"DELETE":"delete-boop.ejs"
	}
}
```

You can see both the `PUT` and `GET` endpoints are using the same template for a single boop object, since they’re both meant to return an html representation of a single boop object.

And then for each of the other templates:

`post-boop.ejs`: Returning a simple status message that the boop was created

```html
<div>Created a boop! ID:<%= top.id %></div>
```

`delete-boop.ejs` returns nothing, so swapping the results of DELETE will remove an element.

A slight gotcha with htmx, in order for this to work you need to tell htmx to add the location of your locally-running hypermedia faking api to the request path:

```jsx
document.body.addEventListener('htmx:configRequest', function (evt) {
    evt.detail.path = "http://localhost:3000" + evt.detail.path
});
```

And voila! A dead simple CRUD application with htmx and a faked hypermedia API.

Caveats galore! This is not a real application

- The data is pretty ephemeral, sitting in a simple json file.
- The target usecase here is really CRUD applications, anything requiring more dynamic behaviour might be more tricky.
- You can’t really build non-trivial backend business logic.
- There isn’t really much “best practise” here, it’s more of an approach to quickly sketch out ideas.

See [https://htmx.org/essays/when-to-use-hypermedia/](https://htmx.org/essays/when-to-use-hypermedia/) for a much more detailed take on when/when not to use hypermedia and htmx.

I think the best way to use something like this is an early way to get dynamic prototypes in front of people. You can throw this online with something like [ngrok](https://ngrok.com/) and then use it for an early research session. Beyond coached demos it’s probably time to pull in the longer-term tools.

My belief is that in using these incredibly pared down tools, you remove alot of the effort when experimenting with web applications. You can focus on learning about whether or not your main flows and data models make sense with minimal effort before tackling anything really meaty.

Lots of other stuff to explore in the future! A few ideas:

- Write up how to use `ngrok` and `python3 -m http.server` to get something public that would actually be interactive for somebody else. Hint: Use a `config.yml` file with two tunnels, then change the page’s `htmx:configRequest` listener to point at the endpoint ngrok give you for the faked api.
- How would we reuse the templates as we shift to a full stack framework like Django?
- How do we evolve the frontend markup to longer-term solutions? How would a full featured framework like react get pulled in?
- If we wanted to make individual buckets of markup & functionality more maintainable, can we move incrementally into web-components?
- What would this look like if we replaced this faked API with the API wrapper of an excel-sheet-esque system like airtable?