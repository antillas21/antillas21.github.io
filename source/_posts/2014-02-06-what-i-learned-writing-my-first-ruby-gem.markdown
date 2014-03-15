---
layout: post
title: "What I learned writing my first Ruby gem"
date: 2014-03-15 14:56:18 -0800
comments: true
categories:  [gems]
---

Weeks ago, Marvel Comics announced the release of its Comics API. I've always been fond of comics, and in the spirit of learning how to consume APIs, I thought this would be a great pet-project to work on: A gem that interacts with this API. You can access the [end product here](https://github.com/antillas21/marvelite).

So, I began working on an initial draft (paper mostly) after reading Marvel's Developer portal instructions about API requests.

My first steps into pulling data from Marvel's API was through the browser just passing query params in the url bar, and watching a JSON response appear in the window (I highly recommend the [JSON View](https://chrome.google.com/webstore/detail/jsonview/chklaanhfefbnpoihckbnefhakgolnmc) browser extension to enhance the experience).

My first gem release was nothing fancy or to be really proud of, merely two endpoints available (`/characters` and `/characters/:id`) and a rough sketch of how to add more endpoints in time... but hey, I released something into the public and just shortly after the API was available.

After one week of iterating over the gem design, working merely at night for a couple of hours each time, here are the lessons I learned from this:

1. Agree on a public API for (potential) users.
2. Metaprogram for fun and profit.
3. Do you want people to contribute? Add tests!
4. Provide detailed documentation to users.
5. Add tests. Seriously, add tests!

Let's elaborate on each of these points.

### Agree on a public API for users

Perhaps it's because my programming experience is heavily influenced by Rails, when I started drafting how I wanted to expose methods in this gem, I decided to use sort of Rails-y conventions:

```ruby
comics(123)
characters(234)

# and why not, the option to pass a character's name to retrieve it,
# because we don't know ids (not yet at least).
character('Spider-Man')
```

When I realized I needed to implement a small router that should provide an API client with available routes, I thought it would be a good idea to implement them like this:

```ruby
comics_path
comic_path(123)
character_comics_path(234)

# yes, a Rails like router ;)
```

These decisions influenced every line of code that came after them.

### Metaprogram for fun and profit

When I was working on the gem, I had the opportunity to pair-program with [Ismael Marín](https://github.com/igmarin), a friend from the Bajío on Rails community. During our session, he pointed out that my (at the moment) current codebase could benefit from using metaprogramming tricks, which in the end will result in reducing the maintenance burden of adding new endpoints and functionality as the Marvel API makes them available.

And so, I went from manually-defined methods like this:

```ruby
## WARNING!: The following is fugly code. It has been refactored FTW :)

# code from the Marvelite::API::Router class

def characters_path
  "/#{api_version}/public/characters"
end

def character_path(id)
  "/#{api_version}/public/characters/#{id}"
end

def character_comics_path(id)
  "/#{api_version}/public/characters/#{id}/comics"
end

# code from the Marvelite::API::Client class

def characters(query_params = {})
  response = self.class.get("/#{api_version}/#{router.characters_path}",
    :query => params(query_params))
  build_response_object(response)
end

def character_comics(id, query_params = {})
  if id.is_a?(String)
    character = find_character_by_name(id)
    return false unless character
    id = character.id
  end

  response = self.class.get("/#{api_version}/#{router.character_comics_path(id)}",
    :query => params(query_params))
  build_response_object(response)
end
```
You can already start discussing the burden this will create for maintenance, just by manually adding new methods to tackle access to all available API endpoints.

With just cause, you will be talking some sense to me to fix this ASAP. Just want to point out in my defense, that this was only exploratory code to have something out-the-door.

After some thinking, tweaking and refactoring, I ended up with something like this:

```ruby
# code from the Marvelite::API::Router class

def add_route(name, endpoint)
  routes["#{name}_path".to_sym] = { :name => name, :endpoint => endpoint }
end

def method_missing(method, *args, &block)
  if routes.keys.include?(method)
    endpoint = "#{routes[method][:endpoint]}"
    params = *args
    if params.any?
      params[0].each do |p_key, p_value|
        endpoint.gsub!(":#{p_key.to_s}", p_value.to_s)
      end
    end
    "/#{api_version}#{endpoint}"
  else
    super
  end
end

# code from the Marvelite::API::Client class

def build_methods
  @router.routes.each do |_, hash|
    name = hash[:name]
    self.class.send(:define_method, name) do |*args|
      params = process_arguments(*args)
      response = fetch_response(name, params)
      build_response_object(response)
    end
  end
end
# just call build_methods on initialization, and voilá, our methods are there.
```
You can see the finished files here: [Marvelite::API::Router](https://github.com/antillas21/marvelite/blob/master/lib/marvelite/api/router.rb) and [Marvelite::API::Client](https://github.com/antillas21/marvelite/blob/master/lib/marvelite/api/client.rb).

The benefit of it all? I would only need to have a file where I would map the original API endpoints to a route-like structure my gem would understand.

There's still work to do, I must confess, but I think this is a good first step in the right direction. This way, I reduced the maintenance burden adding new API endpoints, just add a new line and it's done, how cool is that?

### Do you want people to contribute? Add tests!

Since day one I started working on this, I invited my friends from the Mexicali Open Source community to join in and contribute if they thought like it.

Question was, how to be sure things work while having contributors get up to speed understanding the pieces of the gem? Easy, add tests and be descriptive about the functionality in place.

I think this would be better explained through an image, from one of the pull requests I received.

{% img image /images/marvelite-pr-contribution.jpg %}

Thinking back, someone commenting you have done adding endpoints an easy task, is the best compliment to decisions made when building the project... and definitively, that made my day.

### Provide detailed documentation to users

As a developer, I spend most of the time making things that work and then refactor them... and most of the time, only me or another developer understands what I ended up with, either because you assume they'll consult specs or read the source.

So, I set as a personal goal, that as I would have no control over who may end up using my gem (maybe a seasoned developer that lives and breathes RTFM or a newbie developer, who knows), to write human, non-geek, non-developer exclusively, understandable documentation on how to use the gem.

I'd like to say that I made it right, but I can't be sure. Although if I judge by the numbers in this image, I like to think, documentation has made a difference.

{% img image /images/marvelite-rubygems-stats.jpg %}

### Add tests. Seriously, add tests!

You would think that in the Ruby/Rails community, where in every conference you attend to, everyone spreads the Gospel of Testing... truth be told, not everyone practices this.

I could count with the fingers in one hand, how many projects that I've worked on, really abide by test-driven development. Yes, that few. And have been working on development projects for nearly 5 years now.

I will not argue about this point, and I will not say whoever doesn't practice TDD is wrong and even less I will not say I am strict practitioner of it, because truth be told, I am not.

But hey, not being a strict practitioner does not mean I don't want to become one. So, I set as a skill-building exercise to add tests to prove me (and potential contributors) that things worked as expected.

#### Was it hard? was it fun? did it help?

Yes, yes and yes.

Remember when I was discussing refactoring to make use of metaprogramming? Well, that wouldn't be possible nor an enjoyable task if it wasn't because of the tests in place to let me know when things broke in the course of refactoring.

Also, remember the pull request from above? It would have not been possible without, yes you guessed, the tests in place that explained another person what endpoint was being tested and how to replicate it to test another endpoint.


Overall, I must confess this has been an interesting journey and experience. I believe I have learned more in the course of a week working on a pet-project, than in months of contract work.

I'm really looking forward for Marvel to add new endpoints or release a newer version of its API, to get back to the drawing board and address whatever issues may arise from the change.

The main topic after all this, I liked how pair-programming went out. And honestly, would like to experience more of it. So, if you have a pet project yourself and think maybe pairing could help you, let me know. I for sure, know it will be great for me.