# antoinealb.github.io

## Kasper theme

This is a port of Ghost's default theme [Casper](https://github.com/tryghost/casper) for Jekyll.
Feel free to fork, change, modify and re-use it.
It is released under the MIT license.

Kasper theme includes:

* Pagination
* Rss
* Google Analytics Tracking code
* Code Syntax Highlight
* Author's profile with picture
* Disqus comments

## Install dev tools

The only external requirement is Ruby>=2.0 and the `bundler` Gem (can be installed via `gem install bundler`).
To install all other needed tools by running `bundle install --path vendor --binstubs`.

## Running the site locally

```bash
# Normal version
./bin/jekyll serve

# With drafts posts
./bin/jekyll serve --drafts
```
