# Acts As Votable (aka Acts As Likeable)

[![Build Status](https://travis-ci.org/ryanto/acts_as_votable.svg?branch=master)](https://travis-ci.org/ryanto/acts_as_votable)
[![Code Climate](https://codeclimate.com/github/ryanto/acts_as_votable.png)](https://codeclimate.com/github/ryanto/acts_as_votable)

Acts As Votable is a Ruby Gem specifically written for Rails/ActiveRecord models.
The main goals of this gem are:

- Allow any model to be voted on, like/dislike, upvote/downvote, etc.
- Allow any model to be voted under arbitrary scopes.
- Allow any model to vote.  In other words, votes do not have to come from a user,
  they can come from any model (such as a Group or Team).
- Provide an easy to write/read syntax.

## Installation

### Supported Ruby and Rails versions

* Ruby >= 2.3.0
* Rails >= 4

### Install

Just add the following to your Gemfile to install the latest release.

```ruby
gem 'acts_as_votable', '~> 0.12.0'
```

And follow that up with a ``bundle install``.

### Database Migrations

Acts As Votable uses a votes table to store all voting information.  To
generate and run the migration just use.

    rails generate acts_as_votable:migration
    rake db:migrate

You will get a performance increase by adding in cached columns to your model's
tables.  You will have to do this manually through your own migrations.  See the
caching section of this document for more information.

## Usage

### Votable Models

```ruby
class Post < ActiveRecord::Base
  acts_as_votable
end

@post = Post.new(:name => 'my post!')
@post.save

@post.liked_by @user
@post.votes_for.size # => 1
```

### Like/Dislike Yes/No Up/Down

Here are some voting examples.  All of these calls are valid and acceptable.  The
more natural calls are the first few examples.

```ruby
@post.liked_by @user1
@post.downvote_from @user2
@post.vote_by :voter => @user3
@post.vote_by :voter => @user4, :vote => 'bad'
@post.vote_by :voter => @user5, :vote => 'like'
```

By default all votes are positive, so `@user3` has cast a 'good' vote for `@post`.

`@user1`, `@user3`, and `@user5` all voted in favor of `@post`.

`@user2` and `@user4` on the other had has voted against `@post`.


Just about any word works for casting a vote in favor or against post.  Up/Down,
Like/Dislike, Positive/Negative... the list goes on-and-on.  Boolean flags `true` and
`false` are also applicable.

Revisiting the previous example of code.

```ruby
# positive votes
@post.liked_by @user1
@post.vote_by :voter => @user3
@post.vote_by :voter => @user5, :vote => 'like'

# negative votes
@post.downvote_from @user2
@post.vote_by :voter => @user2, :vote => 'bad'

# tally them up!
@post.votes_for.size # => 5
@post.weighted_total => 5
@post.get_likes.size # => 3
@post.get_upvotes.size # => 3
@post.get_dislikes.size # => 2
@post.get_downvotes.size # => 2
@post.weighted_score => 1
```

Active Record scopes are provided to make life easier.

```ruby
@post.votes_for.up.by_type(User)
@post.votes_for.down
@user1.votes.up
@user1.votes.down
@user1.votes.up.for_type(Post)
```

Once scoping is complete, you can also trigger a get for the
voter/votable

```ruby
@post.votes_for.up.by_type(User).voters
@post.votes_for.down.by_type(User).voters

@user.votes.up.for_type(Post).votables
@user.votes.up.votables
```

You can also 'unvote' a model to remove a previous vote.

```ruby
@post.liked_by @user1
@post.unliked_by @user1

@post.disliked_by @user1
@post.undisliked_by @user1
```

Unvoting works for both positive and negative votes.

### Examples with scopes

You can add a scope to your vote

```ruby
# positive votes
@post.liked_by @user1, :vote_scope => 'rank'
@post.vote_by :voter => @user3, :vote_scope => 'rank'
@post.vote_by :voter => @user5, :vote => 'like', :vote_scope => 'rank'

# negative votes
@post.downvote_from @user2, :vote_scope => 'rank'
@post.vote_by :voter => @user2, :vote => 'bad', :vote_scope => 'rank'

# tally them up!
@post.find_votes_for(:vote_scope => 'rank').size # => 5
@post.get_likes(:vote_scope => 'rank').size # => 3
@post.get_upvotes(:vote_scope => 'rank').size # => 3
@post.get_dislikes(:vote_scope => 'rank').size # => 2
@post.get_downvotes(:vote_scope => 'rank').size # => 2

# votable model can be voted under different scopes
# by the same user
@post.vote_by :voter => @user1, :vote_scope => 'week'
@post.vote_by :voter => @user1, :vote_scope => 'month'

@post.votes_for.size # => 2
@post.find_votes_for(:vote_scope => 'week').size # => 1
@post.find_votes_for(:vote_scope => 'month').size # => 1
```
### Adding weights to your votes

You can add weight to your vote. The default value is 1.

```ruby
# positive votes
@post.liked_by @user1, :vote_weight => 1
@post.vote_by :voter => @user3, :vote_weight => 2
@post.vote_by :voter => @user5, :vote => 'like', :vote_scope => 'rank', :vote_weight => 3

# negative votes
@post.downvote_from @user2, :vote_scope => 'rank', :vote_weight => 1
@post.vote_by :voter => @user2, :vote => 'bad', :vote_scope => 'rank', :vote_weight => 3

# tally them up!
@post.find_votes_for(:vote_scope => 'rank').sum(:vote_weight) # => 6
@post.get_likes(:vote_scope => 'rank').sum(:vote_weight) # => 6
@post.get_upvotes(:vote_scope => 'rank').sum(:vote_weight) # => 6
@post.get_dislikes(:vote_scope => 'rank').sum(:vote_weight) # => 4
@post.get_downvotes(:vote_scope => 'rank').sum(:vote_weight) # => 4
```

### The Voter

You can have your voters `acts_as_voter` to provide some reserve functionality.

```ruby
class User < ActiveRecord::Base
  acts_as_voter
end

@user.likes @article

@article.votes.size # => 1
@article.likes.size # => 1
@article.dislikes.size # => 0
```

To check if a voter has voted on a model, you can use ``voted_for?``.  You can
check how the voter voted by using ``voted_as_when_voted_for``.

```ruby
@user.likes @comment1
@user.up_votes @comment2
# user has not voted on @comment3

@user.voted_for? @comment1 # => true
@user.voted_for? @comment2 # => true
@user.voted_for? @comment3 # => false

@user.voted_as_when_voted_for @comment1 # => true, user liked it
@user.voted_as_when_voted_for @comment2 # => false, user didnt like it
@user.voted_as_when_voted_for @comment3 # => nil, user has yet to vote
```

You can also check whether the voter has voted up or down.

```ruby
@user.likes @comment1
@user.dislikes @comment2
# user has not voted on @comment3

@user.voted_up_on? @comment1 # => true
@user.voted_down_on? @comment1 # => false

@user.voted_down_on? @comment2 # => true
@user.voted_up_on? @comment2 # => false

@user.voted_up_on? @comment3 # => false
@user.voted_down_on? @comment3 # => false
```

Aliases for methods `voted_up_on?` and `voted_down_on?` are: `voted_up_for?`, `voted_down_for?`, `liked?` and `disliked?`.

Also, you can obtain a list of all the objects a user has voted for.
This returns the actual objects instead of instances of the Vote model.
All objects are eager loaded

```ruby
@user.find_voted_items

@user.find_up_voted_items
@user.find_liked_items

@user.find_down_voted_items
@user.find_disliked_items
```

Members of an individual model that a user has voted for can also be
displayed. The result is an ActiveRecord Relation.

```ruby
@user.get_voted Comment

@user.get_up_voted Comment

@user.get_down_voted Comment
```

### Registered Votes

Voters can only vote once per model.  In this example the 2nd vote does not count
because `@user` has already voted for `@shoe`.

```ruby
@user.likes @shoe
@user.likes @shoe

@shoe.votes # => 1
@shoe.likes # => 1
```

To check if a vote counted, or registered, use `vote_registered?` on your model
after voting.  For example:

```ruby
@hat.liked_by @user
@hat.vote_registered? # => true

@hat.liked_by => @user
@hat.vote_registered? # => false, because @user has already voted this way

@hat.disliked_by @user
@hat.vote_registered? # => true, because user changed their vote

@hat.votes.size # => 1
@hat.positives.size # => 0
@hat.negatives.size # => 1
```

To permit duplicates entries of a same voter, use option duplicate. Also notice that this
will limit some other methods that didn't deal with multiples votes, in this case, the last vote will be considered.

```ruby
@hat.vote_by voter: @user, :duplicate => true
```

## Caching

To speed up perform you can add cache columns to your votable model's table.  These
columns will automatically be updated after each vote.  For example, if we wanted
to speed up @post we would use the following migration:

```ruby
class AddCachedVotesToPosts < ActiveRecord::Migration
  def change
    change_table :posts do |t|
      t.integer :cached_votes_total, default: 0
      t.integer :cached_votes_score, default: 0
      t.integer :cached_votes_up, default: 0
      t.integer :cached_votes_down, default: 0
      t.integer :cached_weighted_score, default: 0
      t.integer :cached_weighted_total, default: 0
      t.float :cached_weighted_average, default: 0.0
    end

    # Uncomment this line to force caching of existing votes
    # Post.find_each(&:update_cached_votes)
  end
end
```

If you have a scope for your vote, let's say `subscribe`, your migration will be slightly different like below:

```ruby
class AddCachedVotesToPosts < ActiveRecord::Migration
  def change
    change_table :posts do |t|
      t.integer :cached_scoped_subscribe_votes_total, default: 0
      t.integer :cached_scoped_subscribe_votes_score, default: 0
      t.integer :cached_scoped_subscribe_votes_up, default: 0
      t.integer :cached_scoped_subscribe_votes_down, default: 0
      t.integer :cached_weighted_subscribe_score, default: 0
      t.integer :cached_weighted_subscribe_total, default: 0
      t.float :cached_weighted_subscribe_average, default: 0.0
    end
  end
end
```

`cached_weighted_average` can be helpful for a rating system, e.g.:

Order by average rating:

```ruby
Post.order(:cached_weighted_average => :desc)
```

Display average rating:

```erb
<%= post.weighted_average.round(2) %> / 5
<!-- 3.5 / 5 -->
```
## updated_at at Votable model

You can control whether `updated_at` column of votable model will be touched or
not by passing `cacheable_strategy` option to `acts_as_votable` method.

By default, `update` strategy is used. Pass `:update_columns` as
`cacheable_strategy` if you don't want to touch model's `updated_at` column.
```ruby
class Post < ActiveRecord::Base
  acts_as_votable cacheable_strategy: :update_columns
end
```

NOTE: this option does not work for ActiveRecord < 3.1


## Testing

All tests follow the RSpec format and are located in the spec directory.
They can be run with:

```
rake spec
```

## Changes

### Fixes for votable voter model

In version 0.8.0, there are bugs for a model that is both votable and voter.
Some name-conflicting methods are renamed:
+ Renamed Votable.votes to votes_for
+ Renamed Votable.vote to vote_by,
+ Removed Votable.vote_by alias (was an alias for :vote_up)
+ Renamed Votable.unvote_for to unvote_by
+ Renamed Votable.find_votes to find_votes_for
+ Renamed Votable.up_votes to get_upvotes
  + and its aliases :get_true_votes, :get_ups, :get_upvotes, :get_likes, :get_positives, :get_for_votes
+ Renamed Votable.down_votes to get_downvotes
  + and its aliases :get_false_votes, :get_downs, :get_downvotes, :get_dislikes, :get_negatives


## License

Acts as votable is released under the [MIT
License](http://www.opensource.org/licenses/MIT).

## Next steps

- Pass in a block of options when creating acts_as.  Allow for things
  like disabling the aliasing

- Smarter language syntax.  Example: `@user.likes` will return all of the votables
that the user likes, while `@user.likes @model` will cast a vote for @model by
@user.

- The aliased methods are referred to by using the terms 'up/down' and/or
'true/false'. Need to come up with guidelines for naming these methods.

- Create more aliases. Specifically for counting votes and finding votes.
