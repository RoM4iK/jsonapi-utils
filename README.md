# JSONAPI::Utils

JSON::Utils is a simple way to get a full-featured [JSON API](jsonapi.org) serialization in your
controller's responses. This gem works on top of the awesome gem [jsonapi-resources](https://github.com/cerebris/jsonapi-resources),
bringing to controllers a Rails-native way to render data.

## Installation

Add these lines to your application's Gemfile:

```ruby
gem 'jsonapi-resources', '~> 0.5.7'
gem 'jsonapi-utils'
```

And then execute:

```shell
$ bundle
```

## Macros

* `jsonapi_render`: it works like ActionController's `render` method, receiving model objects and
rendering them into JSON API's data format.

* `jsonapi_serialize`: in the backstage, it's the method that actually parsers model objects or hashes and builds JSON data.
It can be called anywhere in controllers, concerns etc.

## Options

Those macros accept the following options:

* `resource`: explicitly points the resource to be used in the serialization. By default, JSONAPI::Utils will
select resources by inferencing from controller's name.

```ruby
# If in UsersController for some reason it needs to render a different resource:
jsonapi_render json: Post.all, options: { resource: PostResource }
```

* `count`: explicitly points the total count of records for the request, in order to build a proper pagination. By default, JSONAPI::Utils will
count the total number of records for a given resource.

```ruby
users = User.all
jsonapi_render json: users, options: { count: users.size }
```

## Usage

Let's say we have a Rails app for a super simple blog.

### Models

```ruby
# app/models/user.rb
class User < ActiveRecord::Base
  has_many :posts
  validates :first_name, :last_name, presence: true
end

# app/models/user.rb
class Post < ActiveRecord::Base
  belongs_to :author, class_name: 'User', foreign_key: 'user_id'
  validates :title, :body, presence: true
end
```

### Resources

Here is where we define how the serialization will behave:

```ruby
# app/resources/user_resource.rb
class UserResource < JSONAPI::Resource
  attributes :first_name, :last_name, :full_name, :birthday
  attribute :full_name

  has_many :posts

  def full_name
    "#{@model.first_name} #{@model.last_name}"
  end
end

# app/resources/post_resource.rb
class PostResource < JSONAPI::Resource
  attributes :title, :body
  has_one :author
end
```

### Routes & Controllers

Let's define our routes using the `jsonapi_resources` and `jsonapi_links` macros provied by the `jsonapi-resources` gem:

```ruby
Rails.application.routes.draw do
  jsonapi_resources :users do
    jsonapi_resources :posts
    jsonapi_links :posts
  end
end
```

And a base controller to include the features from `jsonapi-resources` and `jsonapi-utils`:

```ruby
# app/controllers/base_controller.rb
class BaseController < JSONAPI::ResourceController
  include JSONAPI::Utils
  protect_from_forgery with: :null_session
  rescue_from ActiveRecord::RecordNotFound, with: :jsonapi_render_not_found
end
```

For this example, let's get focused only on read actions. After including `JSONAPI::Utils` we can use the `jsonapi_render` method
in order to generate responses which follow the JSON API's standards.

```ruby
# app/controllers/users_controller.rb
class UsersController < BaseController
  before_action :load_user, only: [:show]

  # GET /users
  def index
    users = User.all
    jsonapi_render json: users, options: { count: users.size }
  end

  # GET /users/:id
  def show
    jsonapi_render json: @user
  end

  private

  def load_user
    @user = User.find(params[:id])
  end
end
```

And:

```ruby
# app/controllers/posts_controller.rb
class PostsController < BaseController
  before_action :load_user, only: [:index, :show]
  before_action :load_post, only: [:show]

  # GET /users/:user_id/posts
  def index
    jsonapi_render json: @user.posts, options: { count: @user.posts.size }
  end

  # GET /users/:user_id/posts/:id
  def show
    jsonapi_render json: @post
  end

  private

  def load_user
    @user = User.find(params[:user_id])
  end

  def load_post
    @post = @user.posts.find(params[:id])
  end
end
```

### Erros

#### Not found

As you might have seen in BaseController this line will handle all errors related to not found resources:

```ruby
rescue_from ActiveRecord::RecordNotFound, with: :jsonapi_render_not_found
```

It will produce the following error payload:

```json
{
  "errors": [
    {
      "title": "Record not found",
      "detail": "The record identified by 3 could not be found.",
      "id": null,
      "href": null,
      "code": 404,
      "source": null,
      "links": null,
      "status": "not_found"
    }
  ]
}
```

### Initializer

In order to enable a proper pagination, record count etc, let's define an initializer such as:

```ruby
# config/initializers/jsonapi_resources.rb
JSONAPI.configure do |config|
  config.json_key_format = :underscored_key
  config.route_format = :dasherized_route

  config.operations_processor = :active_record

  config.allow_include = true
  config.allow_sort = true
  config.allow_filter = true

  config.raise_if_parameters_not_allowed = true

  config.default_paginator = :paged

  config.top_level_links_include_pagination = true

  config.default_page_size = 10
  config.maximum_page_size = 20

  config.top_level_meta_include_record_count = true
  config.top_level_meta_record_count_key = :record_count

  config.use_text_errors = false

  config.exception_class_whitelist = []

  config.always_include_to_one_linkage_data = false
end
```

You may want a different configuration for your API. For more information check [this](https://github.com/cerebris/jsonapi-resources/#configuration).

### Requests & Responses

Here's some examples of requests – based on those sample [controllers](#routes--controllers) – and their respective JSON responses.

* [Collection](#collection)
* [Collection (options)](#collection-options)
* [Record](#record)
* [Record (options)](#record-options)
* [Relationships (identifier objects)](#relationships-identifier-objects)
* [Nested resources](#nested-resources)

#### Collection

Request:

```
GET /users HTTP/1.1
Accept: application/vnd.api+json
```

Response:

```json
HTTP/1.1 200 OK
Content-Type: application/vnd.api+json

{
  "data": [
    {
      "id": "1",
      "type": "users",
      "links": {
        "self": "http://api.myawesomeblog.com/users/1"
      },
      "attributes": {
        "first_name": "Tiago",
        "last_name": "Guedes",
        "full_name": "Tiago Guedes",
        "birthday": null
      },
      "relationships": {
        "posts": {
          "links": {
            "self": "http://api.myawesomeblog.com/users/1/relationships/posts",
            "related": "http://api.myawesomeblog.com/users/1/posts"
          }
        }
      }
    },
    {
      "id": "2",
      "type": "users",
      "links": {
        "self": "http://api.myawesomeblog.com/users/2"
      },
      "attributes": {
        "first_name": "Douglas",
        "last_name": "André",
        "full_name": "Douglas André",
        "birthday": null
      },
      "relationships": {
        "posts": {
          "links": {
            "self": "http://api.myawesomeblog.com/users/2/relationships/posts",
            "related": "http://api.myawesomeblog.com/users/2/posts"
          }
        }
      }
    }
  ],
  "meta": {
    "record_count": 2
  },
  "links": {
    "first": "http://api.myawesomeblog.com/users?page%5Bnumber%5D=1&page%5Bsize%5D=10",
    "last": "http://api.myawesomeblog.com/users?page%5Bnumber%5D=1&page%5Bsize%5D=10"
  }
}
```

#### Collection (options)

Request:

```
GET /users?include=posts&fields[users]=full_name,posts&fields[posts]=title&page[number]=1&page[size]=1 HTTP/1.1
Accept: application/vnd.api+json
```

Response:

```json
HTTP/1.1 200 OK
Content-Type: application/vnd.api+json

{
  "data": [
    {
      "id": "1",
      "type": "users",
      "links": {
        "self": "http://localhost:3000/users/1"
      },
      "attributes": {
        "full_name": "Tiago Guedes"
      },
      "relationships": {
        "posts": {
          "links": {
            "self": "http://localhost:3000/users/1/relationships/posts",
            "related": "http://localhost:3000/users/1/posts"
          },
          "data": [
            {
              "type": "posts",
              "id": "1"
            }
          ]
        }
      }
    }
  ],
  "included": [
    {
      "id": "1",
      "type": "posts",
      "links": {
        "self": "http://localhost:3000/posts/1"
      },
      "attributes": {
        "title": "An awesome post"
      }
    }
  ],
  "meta": {
    "record_count": 2
  },
  "links": {
    "first": "http://localhost:3000/users?fields%5Bposts%5D=title&fields%5Busers%5D=full_name%2Cposts&include=posts&page%5Bnumber%5D=1&page%5Bsize%5D=1",
    "next": "http://localhost:3000/users?fields%5Bposts%5D=title&fields%5Busers%5D=full_name%2Cposts&include=posts&page%5Bnumber%5D=2&page%5Bsize%5D=1",
    "last": "http://localhost:3000/users?fields%5Bposts%5D=title&fields%5Busers%5D=full_name%2Cposts&include=posts&page%5Bnumber%5D=2&page%5Bsize%5D=1"
  }
}
```

#### Single record

```
GET /users/1 HTTP/1.1
Accept: application/vnd.api+json

{
  "data": {
    "id": "1",
    "type": "users",
    "links": {
      "self": "http://localhost:3000/users/1"
    },
    "attributes": {
      "first_name": "Tiago",
      "last_name": "Guedes",
      "full_name": "Tiago Guedes",
      "birthday": null
    },
    "relationships": {
      "posts": {
        "links": {
          "self": "http://localhost:3000/users/1/relationships/posts",
          "related": "http://localhost:3000/users/1/posts"
        }
      }
    }
  }
}
```

#### Single record (options)

```
GET /users/1?include=posts&fields[users]=full_name,posts&fields[posts]=title HTTP/1.1
Accept: application/vnd.api+json

{
  "data": {
    "id": "1",
    "type": "users",
    "links": {
      "self": "http://localhost:3000/users/1"
    },
    "attributes": {
      "full_name": "Tiago Guedes"
    },
    "relationships": {
      "posts": {
        "links": {
          "self": "http://localhost:3000/users/1/relationships/posts",
          "related": "http://localhost:3000/users/1/posts"
        },
        "data": [
          {
            "type": "posts",
            "id": "1"
          }
        ]
      }
    }
  },
  "included": [
    {
      "id": "1",
      "type": "posts",
      "links": {
        "self": "http://localhost:3000/posts/1"
      },
      "attributes": {
        "title": "An awesome post"
      }
    }
  ]
}
```

#### Relantionships (identifier objects)

```
GET /users/1/relationships/posts HTTP/1.1
Accept: application/vnd.api+json

{
  "links": {
    "self": "http://localhost:3000/users/1/relationships/posts",
    "related": "http://localhost:3000/users/1/posts"
  },
  "data": [
    {
      "type": "posts",
      "id": "1"
    }
  ]
}
```

#### Nested resources

```
GET /users/1/posts HTTP/1.1
Accept: application/vnd.api+json

{
  "data": [
    {
      "id": "1",
      "type": "posts",
      "links": {
        "self": "http://localhost:3000/posts/1"
      },
      "attributes": {
        "title": "An awesome post",
        "body": "Lorem ipsum dolot sit amet"
      },
      "relationships": {
        "author": {
          "links": {
            "self": "http://localhost:3000/posts/1/relationships/author",
            "related": "http://localhost:3000/posts/1/author"
          }
        }
      }
    }
  ],
  "meta": {
    "record_count": 1
  },
  "links": {
    "first": "http://localhost:3000/posts?page%5Bnumber%5D=1&page%5Bsize%5D=10",
    "last": "http://localhost:3000/posts?page%5Bnumber%5D=1&page%5Bsize%5D=10"
  }
}
```

## Development

After checking out the repo, run `bin/setup` to install dependencies. Then, run `rake rspec` to run the tests. You can also run `bin/console` for an interactive prompt that will allow you to experiment.

To install this gem onto your local machine, run `bundle exec rake install`. To release a new version, update the version number in `version.rb`, and then run `bundle exec rake release`, which will create a git tag for the version, push git commits and tags, and push the `.gem` file to [rubygems.org](https://rubygems.org).

## Contributing

Bug reports and pull requests are welcome on GitHub at https://github.com/[USERNAME]/jsonapi-utils. This project is intended to be a safe, welcoming space for collaboration, and contributors are expected to adhere to the [Contributor Covenant](contributor-covenant.org) code of conduct.


## License

The gem is available as open source under the terms of the [MIT License](http://opensource.org/licenses/MIT).


