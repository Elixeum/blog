# Elixeum Blog

This is tech blog of Elixeum development team built on amazing [Jekyll](https://jekyllrb.com) site generator (including nice Minimal Mistakes theme!) and hosted by [Github Pages](https://pages.github.com) :raised_hands:

## How To Create a Post

Just create a new `.md` file and place it to `/_posts/` folder. Then fill required meta information like `date` or `title` and - of course - your amazing blog post text! :sunglasses:

# Development

As this blog is based on Jekyll - various parts of it can be modified and changed, feel free to make changes and submit PR!

## Requirements

Following tools needs to be installed on your development machine:
- Ruby
  - Tested with Ruby 3.1.* on macOS installed using [Homebrew](https://brew.sh)
  - `brew install ruby`
  - Don't forget to add new Ruby to `PATH`, as Homebrew will not do this due bundled macOS version of Ruby

## Start Local Instance

If you want to start local environment just use following:

```shell
bundle install # Install all dependencies see Gemfile for more
bundle exec jekyll serve --livereload # Starts local HTTP server with live reload enabled
```

