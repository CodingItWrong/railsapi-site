# Getting Started With JSONAPI::Resources

JSON:API is a web service standard that provides a lot of advantages over rolling your own format or using GraphQL. One of the best ways to see the benefits of JSON:API is to build a web service with Ruby on Rails and the [JSONAPI::Resources][jr] gem. If you haven't used Rails before, though, this can be daunting.

To overcome this challenge, let's do a quick walkthrough of just the essential bits of Rails to set up a JSON:API web service. I think you'll find that even if you don't know Rails it's still far easier than building a web service from scratch in your framework of choice!

## Creating the App

Let's create a web service for tracking video games we own.

First, [install Ruby on Rails][install-rails].

Create a new Rails app:

```bash
$ rails new --api video_games
```

This will create an app configured to store data in a [SQLite][sqlite] database, which is just a flat file. This is the simplest way to go for experimentation purposes. If you'd like to use another SQL database like Postgres or MySQL, the `--database=` flag can be used. Run `rails new --help` to see a list of valid options for the flag.

Let's make sure our app will work. Run the rails server:

```bash
$ rails server
```

Then in a browser go to `http://localhost:3000`. You should see the “Yay! You’re on Rails!" page.

## Models
Rails persists data to the database using classes called Models. JSONAPI::Resources uses the same models, so to start building our app we’ll create models in the typical Rails way.

First let’s create a model representing a video game system. Run the following command in the terminal:

```bash
$ rails generate model system name:string
```

This tells Rails to create a new model called `system` and to define one field on it: a `string` field called `name`.

You’ll see output like the following:

```bash
Running via Spring preloader in process 13955
      invoke  active_record
      create    db/migrate/20190522002434_create_systems.rb
      create    app/models/system.rb
      invoke    test_unit
      create      test/models/system_test.rb
      create      test/fixtures/systems.yml
```

The generator created a number of files; let’s take a look at a few of them. First, open the file in `db/migrate` that ends with `_create_systems.rb` — the date on the file will be different than mine, showing the time you ran the command.

```ruby
class CreateSystems < ActiveRecord::Migration[5.2]
  def change
    create_table :systems do |t|
      t.string :name

      t.timestamps
    end
  end
end
```

This file contains a *migration*, a class that tells Rails how to make a change to a database. This file will `create_table :systems`, which is just what it sounds like. The `do` keyword introduces a block passed to the `create_table` method. It receives a parameter `t`, representing a table. `t.string` creates a new string column, and `t.timestamps` creates `created_at` and `updated_at` columns that Rails will manage for us automatically. Rails will also create a primary key on the table; we don’t need to specify anything for it to do so.

The `systems` table hasn’t actually been set up yet; the migration file just records *how* to set it up. You can run it on your computer, when a coworker pulls it down she can run it on hers, and you can run it on the production server as well. Run the migration now with this command:

```bash
$ rails db:migrate
```

You’ll see the following output:

```bash
== 20190522002434 CreateSystems: migrating ====================================
-- create_table(:systems)
   -> 0.0012s
== 20190522002434 CreateSystems: migrated (0.0021s) ===========================
```

Next let’s look at the `app/models/system.rb` file created:

```ruby
class System < ApplicationRecord
end
```

That’s…pretty empty. We have a `System` class that inherits from `ApplicationRecord`, but nothing else. This represents a System record, but how does it know what columns are available? Rails will automatically inspect the table to see what columns are defined on it and make those columns available; no configuration is needed.

The generator also created a few test files in the `test/` directory, but we’ll ignore those for the sake of this tutorial.

Now let’s set up the model for a video game itself. As a shorthand for `rails generate`, you can just type `rails g`:

```bash
$ rails g model game title:string year:integer system:references
```

You’ve seen a `string` column before, and you can probably guess what `integer` does, but what about `references`? This creates a foreign key column that references another model. By default the column name and the name of the other model are the same, in this case `system`.

Go ahead and migrate the database again:

```bash
$ rails db:migrate
```

Then check the `app/models/game.rb` file:

```ruby
class Game < ApplicationRecord
  belongs_to :system
end
```

There’s one change in this file: using `references` *does* actually result in another line of code being added to the model, a call to `belongs_to`. This indicates that a `Game` belongs to a `System`: that is, it has a foreign key pointing to it. This is a many-to-one relationship: many games can belong to one system.

We can set up the reverse one-to-many relationship as well: the fact that one system has many games. Rails doesn’t do this for us: we need to do it manually. Add the following line to `system.rb`:

```diff
 class System < ApplicationRecord
+  has_many :games
 end
```

Now that our models are set up, we can create some records. You could do it by hand, but Rails has the concept of a `seeds.rb` file, which allows you to “seed” your database with sample data. Let’s use that to set up some data. Replace the contents of `db/seeds.rb` with the following:

```ruby
ps = System.create!(name: 'PlayStation')
wii = System.create!(name: 'Wii')
xb_360 = System.create!(name: 'Xbox 360')

ps.games.create!(title: 'Castlevania: Symphony of the Night', year: 1997)
ps.games.create!(title: 'Final Fantasy 7', year: 1997)
wii.games.create!(title: 'Okami', year: 2006)
xb_360.games.create!(title: 'Fallout 3', year: 2008)
xb_360.games.create!(title: 'Portal', year: 2007)
```

These are my five all-time favorite games—if you’re a gamer, feel free to add your own!

Note that we can just pass the attributes to the `create()` method by name. Notice, too, that we can access the `games` relationship for a given system, and `create!` a record on that relationship—that way Rails knows what foreign key value to provide for the `system` relationship.

## Setting Up the Web Service

Now that we’ve got our data all set, let’s set up JSONAPI::Resources (JR) so we can access it via a web service.

Ruby application dependencies are specified in the file `Gemfile` at the root of the project. Add the following line anywhere in that file other than inside a “group”:

```ruby
gem 'jsonapi-resources'
```

Then run the following command in the terminal:

```bash
$ bundle install
```

`bundle` is the command for Bundler, a Ruby library that handles dependencies. `bundle install` will ensure all the dependencies specified in your `Gemfile` are installed. It will record the exact version installed in `Gemfile.lock`.

To set up a JR web service, first we need to create a “resource”, which represents a model in an end-user-facing way. Run the following commands:

```bash
$ rails g jsonapi:resource system
$ rails g jsonapi:resource game
```

JR hooks in to Rails’ command line to add commands to generate resource files. First take a look at `app/resources/system_resource.rb`:

```ruby
class SystemResource < JSONAPI::Resource
end
```

Once again, a pretty straightforward file. This time we’ll have to configure it, because JR doesn’t want to make any assumptions about the data we want to expose to end users; we need to explicitly tell it. Inside the `class` declaration, add these lines:

```ruby
attributes :name

has_many :games
```

As you can probably guess, this means the `name` attribute and `games` relationship will be exposed to the end user. Add the following to `game_resource.rb`:

```ruby
attributes :title, :year

has_one :system
```

Notice that while we used `belongs_to` in the model, in the resource JR uses `has_one` instead.

Now that our resources are set, we need to create controllers that handle the HTTP requests for systems and games. JR provides a generator that will give us controllers set for use with JR:

```bash
$ rails g jsonapi:controller system
$ rails g jsonapi:controller game
```

Open `app/controllers/systems_controller.rb` and you’ll see:

```ruby
class SystemsController < JSONAPI::ResourceController
end
```

The controller inherits from `JSONAPI::ResourceController`, which provides it with almost everything it needs; there’s just one Rails security feature we need to turn off. By default Rails enables an authenticity token feature that prevents Cross-Site Request Forgery attacks. This works when you use Rails to render forms on the server, but for APIs it won’t work, so we need to turn it off. We can do so by adding the following line inside each of the two controller classes:

```ruby
skip_before_action :verify_authenticity_token
```

The last piece of the puzzle is hooking up the routes. Open `routes.rb` and you’ll see the following:

```ruby
Rails.application.routes.draw do
  # For details on the DSL available within this file, see http://guides.rubyonrails.org/routing.html
end
```

This is where all the routes for your app are configured. As you might guess, JR provides helpers that will set up routes the way JR needs. Add the following inside the `do` block:

```ruby
jsonapi_resources :systems
jsonapi_resources :games
```

This will set up all necessary routes. For example, for systems, the following main routes are created:

- GET /systems — lists all the systems
- POST /systems — creates a new system
- GET /systems/:id — gets one system
- PATCH or PUT /systems/:id — updates a system
- DELETE /systems/:id — deletes a system

That’s a lot we’ve gotten without having to write almost any code!

## Trying It Out

Now let’s give it a try. Start the Rails server with `rails server`, or `rails s` for short:

```bash
$ rails s
```

Visit `http://localhost:3000/systems/1` in your browser. You should see something like the following:

```json
{
  "data": {
    "id": "1",
    "type": "systems",
    "links": {
      "self": "http://localhost:3000/systems/1"
    },
    "attributes": {
      "name": "PlayStation"
    },
    "relationships": {
      "games": {
        "links": {
          "self": "http://localhost:3000/systems/1/relationships/games",
          "related": "http://localhost:3000/systems/1/games"
        }
      }
    }
  }
}
```

If you’re using [Firefox][firefox] you should see the JSON data nicely formatted. If your browser doesn’t automatically format JSON, you may be able to find a browser extension to do so. For example, for Chrome you can use [JSONView][jsonview].

This is a JSON:API response for a single record. Let’s talk about what’s going on here:

- The top-level `data` property contains the main data for the response. In this case it’s one record; it can also be an array.
- The record contains an `id` property giving the record’s publicly-exposed ID, which by default is the database integer ID. But JSON:API IDs are always exposed as strings, to allow for the possibility of slugs or UUIDs.
- Even if you can infer the type of the record from context, JSON:API records always have a `type` field recording which type they are. In some contexts, records of different types will be intermixed in an array, so this keeps them distinct.
- `attributes` is an object containing all the attributes we exposed. They are nested instead of directly on the record to avoid colliding with other standard JSON:API properties like `type`.
- `relationships` provides data on the relationships for this record. In this case, the record has a `games` relationship. Two `links` are provided to get data related to that relationship:
	- The `self` link conceptually provides the relationships themselves, which is to say just the IDs of the related records
	- The `related` link provides the full related records.

Try visiting the `related` link, `http://localhost:3000/systems/1/games`, in the browser. You’ll see the following:

```json
{
  "data": [
    {
      "id": "1",
      "type": "games",
      "links": {
        "self": "http://localhost:3000/games/1"
      },
      "attributes": {
        "title": "Castlevania: Symphony of the Night",
        "year": 1997
      },
      "relationships": {
        "system": {
          "links": {
            "self": "http://localhost:3000/games/1/relationships/system",
            "related": "http://localhost:3000/games/1/system"
          }
        }
      }
    },
    {
      "id": "2",
      "type": "games",
      "links": {
        "self": "http://localhost:3000/games/2"
      },
      "attributes": {
        "title": "Final Fantasy 7",
        "year": 1997
      },
      "relationships": {
        "system": {
          "links": {
            "self": "http://localhost:3000/games/2/relationships/system",
            "related": "http://localhost:3000/games/2/system"
          }
        }
      }
    }
  ]
}
```

Note that this time the `data` is an array of two records. Each of them also has their own `relationships` getting back to the `system` associated with the record. These relationships are where JR really shines. Instead of having to manually build routes, controllers, and queries for all of these relationships, JR exposes them for you. And because it uses the standard JSON:API format, there are prebuilt client tools that can save you the same kind of code on the frontend!

Next, let’s take a look at the systems list view. Visit `http://localhost:3000/systems` and you’ll see all the records returned.

Next, let’s try creating a record. We won’t be able to do this in the browser; we’ll need a more sophisticated web service client to do so. One good option is [Postman][postman]—download it and start it up.

You can use Postman for GET requests as well: set up a GET request to `http://localhost:3000/systems` and see how it displays the same data as the browser.

Next, let’s create a POST request to the same URL, `http://localhost:3000/systems`. Go to the Headers tab and enter key “Content-Type” and value “application/vnd.api+json”—this is the content type JSON:API requires.

Next, in the Body tab, enter the following:

```json
{
	"data": {
    "type": "systems",
	  "attributes": {
	    "name": "PlayStation 4"
	  }
	}
}
```

Notice that we don't have to provide an `id` because we’re relying on the server to generate it. And we don’t have to provide the `relationships` or `links`, just the `attributes` we want to set on the new record.

Now that our request is set up, click Send and you should get a “201 Created” response, with the following body:

```JSON
{
    "data": {
        "id": "4",
        "type": "systems",
        "links": {
            "self": "http://localhost:3000/systems/4"
        },
        "attributes": {
            "name": "PlayStation 4"
        },
        "relationships": {
            "games": {
                "links": {
                    "self": "http://localhost:3000/systems/4/relationships/games",
                    "related": "http://localhost:3000/systems/4/games"
                }
            }
        }
    }
}
```

Our new record is created and the data is returned to us!

Let’s see how we can create related data as well. To add a new game associated with system 1, `POST` to `http://localhost:3000/games`:

```json
{
	"data": {
	  "type": "games",
	  "attributes": {
	    "title": "Metal Gear Solid",
	    "year": 1998
	  },
	  "relationships": {
		"system": {
		  "data": {
			"type": "systems",
			"id": "1"
		  }
		}
	  }
	}
}
```

Notice that now, instead of `links` inside the relationship, we provide `data` that specifies the type and ID of the record the game is related to.

If you’d like to try out updating and deleting records:

- Make a `PUT` request to `http://localhost:3000/systems/1`, passing in updated `attributes`.
- Make a `DELETE` request to `http://localhost:3000/systems/1` with no body to delete the record.

## There’s More
We’ve seen a ton of JSONAPI\:\:Resources has provided us: the ability to create, read, update, and delete records, including record relationships. But it offers a lot more too! It automatically exposes Rails validation errors, allows you to request only a subset of the fields you need, allows you to include related records in the response, as well as sorting, filtering, and pagination. We’ll give all these features a try in a future post, but in the meantime check out [the JSONAPI\:\:Resources Guide][jr].

Now that you have a JSON:API backend, you should try connecting to it from the frontend! [Ember.js][ember] is a modern frontend framework with a built-in data layer that's designed from the ground up for JSON:API; if you like the productivity of JR you'll be able to maximize that productivity by choosing Ember. Alternatively, I've created a JSON:API [client for React][reststate-mobx] and [one for Vue][reststate-vuex] that attempt to take the same zero-config approach as JR takes on the server side. You can see other clients on the [JSON:API Implementations page][jsonapi-implementations].

[ember]: https://emberjs.com/
[firefox]: https://getfirefox.com
[install-rails]: https://guides.rubyonrails.org/getting_started.html
[jr]: http://jsonapi-resources.com/v0.9/guide/resources.html
[jsonapi-implementations]: https://jsonapi.org/implementations/
[jsonview]: https://chrome.google.com/webstore/detail/jsonview/chklaanhfefbnpoihckbnefhakgolnmc
[postman]: https://www.getpostman.com/
[reststate-mobx]: https://mobx.reststate.org/
[reststate-vuex]: https://vuex.reststate.org/
[sqlite]: https://www.sqlite.org/index.html
