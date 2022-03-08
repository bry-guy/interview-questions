### What tool, framework, language, or computing concept do you know the best? Explain it to me like I’m a new grad.

Right now, I think I'd answer "Ruby on Rails web apps". I've spent more purposeful time learning about it than other languages/frameworks I've used in the past, and my past experiences have been helpful in quickly grokking parts of it. To be clear; I'm a generalist, not an expert, so I'm not going to give you maintainer-level-knowledge. I'm going to be mistaken; please correct me where I am wrong, it'll help me improve my model.

My mental model of Ruby on Rails is probably easiest to explain from outside-in. Let's say we're hosting a Rails app at https://foo.com. When you navigate to there, your browser finds the IP by DNS and issues an HTTP GET to $IP. At $IP, there will _usually_ be a web server, such as nginx, reverse proxying (and sometimes load balancing) many apps. nginx's config determines which Rails app to forward the request to at $RAILS_IP, and does so. This means https://foo.com/bar can actually hit an entirely different application (doesn't have to be Rails).

So our request hits our Rails app. What does that mean? A Rails app is a Ruby application. From bottom-to-top, Rails apps include:
- A "rackable" application server such as `unicorn` (forked) or `puma` (threaded)
- `rack` configuration, a middleware that maps Rails applications logic to the application server logic
- Rails MVC logic, such as an `MyResourceController` which maps methods like `#show`  and models like `MyResource` to the `MyResourceView/show.html` view, presenting a server-side rendered HTML view.
- _Usually_ a relational DB server acts as the datastore to round out the 3-tier application model, like Postgres or MySQL.

This all runs over HTTP, which runs on TCP. The Rails app is a Ruby process - which is an interpreted VM process - and the rack middleware converts between the Ruby process to whatever runtime the app servers are written in, usually C. 

So, if we request https://foo.com/my_resource/1/, by REST convention (which Rails is designed around), we're asking "show me `my_resource` #1". The Rails application has a `MyResourceController#show`, a `MyResourceView/show.html`, and a `MyResource.rb` model.

Nginx passes the request to $RAILS_IP, and that host - usually a Linux host - opens a socket, accepts the bytes from Nginx, and passes those along to the application server (this is configured by Rails/Rack). The application server says "hey, this is an HTTP GET request for resource `my_resource/1/`", and hands that off to Rack, which then invokes the Rails logic, asking for `MyResourceController#show`.

The Ruby process (ignoring caching) interprets the code at `MyResourceController#show`, executing it in the Ruby VM. Importantly, any single Ruby process will only ever one thread at once - this is the "Global Interpreter Lock" - so concurrency is limited to I/O (waiting for network responses) for a single Ruby process.

The Ruby VM finds the `#show` method, interprets the logic, and says "hey we need to find `MyResource` id 1". Rails models are `ActiveRecord` models, meaning they are an object-relational mapping to the DB. Ruby interprets the `ActiveRecord` code, issues a SQL query over the DB connection that looks like `select * from my_resource where id 1`. I/O wait happens - we release the GIL lock, and other Ruby threads may execute now - and then the SQL query returns us our results, which Ruby interprets into a Ruby model mapped in memory. That `ActiveRecord` model gets referenced to our `MyResourceView/show.html.erb`, which has a template typically invoking some values from our model. Ruby interprets this to an HTML result, serializes that, hands it off to rack, which hands it back to our app server, which hands it back to nginx, which hands it back to our browser, which renders the HTML.

My mental model is an approximation, and I've glossed over many important topics like:
- How does routing work?
- How does the rack-to-app-server magic work?
- How do CSS and JavaScript get included?
- How do we configure Rails?
- Background jobs, instrumentation, serialization (expensive)

There's a ton; I'm constantly looking this stuff up, because I can't model it all in my head, and I haven't been exposed to it all. My model fundamentally comes down to:
- A single request invokes all our code
- There's some non-trivial internet tooling that underlies our application
- Requests are served by an "app-server" which (with Rack) translates internet-land to Ruby-land
- Our code is dynamically interpreted in the Ruby VM, and one VM thread runs at a time
- We're server-side rendering HTML (or JSON if we're an API)

The utility of this model means:
- I can reason about our operations across the stack; I can understand if our IP routing is wrong, and I could trace a critical bug outside of Rails and into an app-server if needed
- I can write concurrency-informed code
- It's very quick for me to expand my model with documentation, meaning Rails is very accessible to me

It's not perfect but it works well for my day-to-day life as a Rails app dev.
