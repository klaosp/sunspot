# Sunspot

![Build Status](https://secure.travis-ci.org/sunspot/sunspot.png)

<http://outoftime.github.com/sunspot>

Sunspot is a Ruby library for expressive, powerful interaction with the Solr
search engine. Sunspot is built on top of the RSolr library, which
provides a low-level interface for Solr interaction; Sunspot provides a simple,
intuitive, expressive DSL backed by powerful features for indexing objects and
searching for them.

Sunspot is designed to be easily plugged in to any ORM, or even non-database-backed
objects such as the filesystem.

## Wiki

New to Sunspot and using Rails? Check out [Adding Sunspot search to Rails in 5
minutes or less](https://github.com/sunspot/sunspot/wiki/Adding-Sunspot-search-to-Rails-in-5-minutes-or-less).

This README is intended as a quick primer on what Sunspot is capable of; for
detailed treatment of Sunspot's full feature range, check out the wiki:
<http://wiki.github.com/sunspot/sunspot>.

The [API documentation](http://outoftime.github.com/sunspot/docs/) is also
complete and up-to-date; however, because of the way Sunspot is structured,
it's not the easiest way for new users to get to know the library.

## Features

* Define indexing strategy for each searchable class using intuitive block-based
  API
* Clean separation between keyword-searchable fields and fields for
  scoping/ordering
* Define fields based on existing attributes or "virtual fields" for custom
  indexing
* Indexes each object's entire superclass hierarchy, for easy searching for all
  objects inheriting from a parent class
* Intuitive DSL for scoping searches, with all the usual boolean operators
  available
* Intuitive interface for requesting facets on indexed fields
* Extensible adapter architecture for easy integration of other ORMs or
  non-model classes
* Refine search using field facets, date range facets, or ultra-powerful
  query facets
* Built in pagination support similar to will_paginate or kaminari
* Ordering by field value, relevance, geographical distance, or random

## Installation

    gem install sunspot

Optionally install the packaged Solr installation (recommended for development):

    gem install sunspot_solr

In order to start the packaged Solr installation, run:

    sunspot-solr start -- [-d /path/to/data/directory] [-p port] [-s path/to/solr/home] [--pid-dir=path/to/pid/dir]

If you don't specify a data directory, your Solr index will be stored in your
operating system's temporary directory.

If you specify a solr home, the directory must contain a <code>conf</code>
directory, which should contain at least <code>schema.xml</code> and
<code>solrconfig.xml</code>. Be sure to copy the <code>schema.xml</code> out of
the Sunspot gem's <code>solr/solr/conf</code> directory. Sunspot relies on the
field name patterns defined in the packaged <code>schema.xml</code>, so those
cannot be modified.

You can also run your own instance of Solr wherever you'd like; just
copy the solr/config/schema.xml file out of the gem's solr into your
installation.  You can change the URL at which Sunspot accesses Solr by
setting <code>SOLR_URL</code> in your application's environment, or
assigning it to Sunspot's configuration directly:

    Sunspot.config.solr.url = 'http://solr.my.host:9818/solr'

## Rails Integration

The Sunspot::Rails plugin makes integrating Sunspot into Rails drop-in easy.

    gem install sunspot_rails

See the README for that gem or the Sunspot Wiki for more information.

## Using Sunspot

### Define an index

    class Post
      #...
    end

    Sunspot.setup(Post) do
      text :title, :description, :stored => true
      string :author_name
      integer :blog_id
      integer :category_ids
      float :average_rating, :using => :ratings_average
      time :published_at
      string :sort_title do
        title.downcase.sub(/^(an?|the)\W+/, ''/) if title = self.title
      end
    end

See Sunspot.setup for more information.

Note that in order for a class to be searchable, it must have an adapter
registered for itself or one of its subclasses. Adapters allow Sunspot to load
objects out of persistent storage, and to determine their primary key for
indexing. [Sunspot::Rails](http://github.com/outoftime/sunspot_rails) comes with
an adapter for ActiveRecord objects, but for other types of models you will need
to define your own. See Sunspot::Adapters for more information.

### Search for objects

    @search = Sunspot.search Post do
      keywords 'great pizza' do
        highlight :title, :description
      end
      with :author_name, 'Mark Twain'
      with(:blog_id).any_of [2, 14]
      with(:category_ids).all_of [4, 10]
      with(:published_at).less_than Time.now
      any_of do
        with(:expired_at).greater_than(Time.now)
        with(:expired_at, nil)
      end
      without :title, 'Bad Title'
      without bad_instance # specifically exclude this instance from results

      paginate :page => 3, :per_page => 15
      order_by :average_rating, :desc

      facet :blog_id
    end

See Sunspot.search for more information.

### Work with search results

    .facets
      %ul.blog_facet
        - @search.facet(:blog_id).rows.each do |row|
          %li.facet_row
            = link_to(row.instance.name, params.merge(:blog_id => row.value))
            %span.count== (#{row.count})

    .page_info
      %h4
        %span.count= pluralize(@search.total, 'result')
        %span.pages Page #{@search.hits.current_page} of #{@search.hits.total_pages} 

    - @search.each_hit_with_result do |hit, post|
      .search_result
        %h3.title
          = link_to(h(hit.stored(:title)), post_url(post))
        - if hit.score
          %span.relevance== (#{hit.score})
        %p= hit.highlight(:description).format { |word| "<span class=\"highlight\">#{word}</span>" }
    .pagination= will_paginate(@search.hits)

## API Documentation

All of the methods documented in the RDoc are considered part of Sunspot's
public API. Methods that are not part of the public API are documented in the
code, but excluded from the RDoc. If you find yourself needing to access methods
that are not part of the public API in order to do what you need, please contact
me so I can rectify the situation!

## Dependencies

1. RSolr
2. Java 1.5+

Sunspot has been tested with MRI 1.8.6 and 1.8.7, REE 1.8.6, YARV 1.9.1, and
JRuby 1.2.0

## Bugs

Please submit bug reports to <https://github.com/sunspot/sunspot/issues>.

## Contribution Guidelines

Contributions are very welcome - both new features, enhancements, and bug fixes.
Bug reports with a failing regression test are also lovely. In order to keep the
contribution process as organized and smooth as possible, please follow these
guidelines:

* Contributions should be submitted as pull requests on the Sunspot
  Github project: <https://github.com/sunspot/sunspot/pulls>
* Patches should not make any changes to the gemspec task other than
  adding/removing dependencies (e.g., changing the name, version, email,
  description, etc.)
* Patches should not include any changes to the gemspec itself.
* Document any new methods, options, arguments, etc.
* Write tests.
* As much as possible, follow the coding and testing styles you see in existing
  code. One could accuse me of being nitpicky about this, but consistent code is
  easier to read, maintain, and enhance.
* Don't make any massive changes to the structure of library or test code. If
  you think something needs a huge refactor or rearrangement, shoot me a
  message; trying to apply that kind of patch without warning opens the door to
  a world of conflict hurt.

## Help and Support

### Asking for Help

* Sunspot Discussion: [ruby-sunspot@googlegroups.com](mailto:ruby-sunspot@googlegroups.com) / <http://groups.google.com/group/ruby-sunspot>
* IRC: [#sunspot-ruby @ Freenode](irc://chat.freenode.net/#sunspot-ruby)

### Tutorials and Articles

* [Full Text Searching with Solr and Sunspot](http://collectiveidea.com/blog/archives/2011/03/08/full-text-searching-with-solr-and-sunspot/) (Collective Idea)
* [Full-text search in Rails with Sunspot](http://tech.favoritemedium.com/2010/01/full-text-search-in-rails-with-sunspot.html) (Tropical Software Observations)
* [Sunspot Full-text Search for Rails/Ruby](http://therailworld.com/posts/23-Sunspot-Full-text-Search-for-Rails-Ruby) (The Rail World)
* [A Few Sunspot Tips](http://blog.trydionel.com/2009/11/19/a-few-sunspot-tips/) (spiral_code)
* [Sunspot: A Solr-Powered Search Engine for Ruby](http://www.linux-mag.com/id/7341) (Linux Magazine)
* [Sunspot Showed Me the Light](http://bennyfreshness.com/2010/05/sunspot-helped-me-see-the-light/) (ben koonse)
* [RubyGems.org — A case study in upgrading to full-text search](http://blog.websolr.com/post/3505903537/rubygems-search-upgrade-1) (Websolr)
* [How to Implement Spatial Search with Sunspot and Solr](http://codequest.eu/articles/how-to-implement-spatial-search-with-sunspot-and-solr) (Code Quest)
* [Sunspot 1.2 with Spatial Solr Plugin 2.0](http://joelmats.wordpress.com/2011/02/23/getting-sunspot-1-2-with-spatial-solr-plugin-2-0-to-work/) (joelmats)
* [rails3 + heroku + sunspot : madness](http://anhaminha.tumblr.com/post/632682537/rails3-heroku-sunspot-madness) (anhaminha)
* [How to get full text search working with Sunspot](http://cookbook.hobocentral.net/recipes/57-how-to-get-full-text-search) (Hobo Cookbook)
* [Full text search with Sunspot in Rails](http://hemju.com/2011/01/04/full-text-search-with-sunspot-in-rail/) (hemju)
* [Using Sunspot for Free-Text Search with Redis](http://masonoise.wordpress.com/2010/02/06/using-sunspot-for-free-text-search-with-redis/) (While I Pondered...)
* [Fuzzy searching in SOLR with Sunspot](http://www.pipetodevnull.com/past/2010/8/5/fuzzy_searching_in_solr_with_sunspot/) (pipe :to => /dev/null)
* [Default scope with Sunspot](http://www.cloudspace.com/blog/2010/01/15/default-scope-with-sunspot/) (Cloudspace)
* [Index External Models with Sunspot/Solr](http://www.medihack.org/2011/03/19/index-external-models-with-sunspotsolr/) (Medihack)
* [Chef recipe for Sunspot in production](http://gist.github.com/336403)
* [Testing with Sunspot and Cucumber](http://collectiveidea.com/blog/archives/2011/05/25/testing-with-sunspot-and-cucumber/) (Collective Idea)
* [Cucumber and Sunspot](http://opensoul.org/2010/4/7/cucumber-and-sunspot) (opensoul.org)
* [Testing Sunspot with Cucumber](http://blog.trydionel.com/2010/02/06/testing-sunspot-with-cucumber/) (spiral_code)
* [Running cucumber features with sunspot_rails](http://blog.kabisa.nl/2010/02/03/running-cucumber-features-with-sunspot_rails) (Kabisa Blog)
* [Testing Sunspot with Test::Unit](http://timcowlishaw.co.uk/post/3179661158/testing-sunspot-with-test-unit) (Type Slowly)
* [How To Use Twitter Lists to Determine Influence](http://www.untitledstartup.com/2010/01/how-to-use-twitter-lists-to-determine-influence/) (Untitled Startup)
* [Sunspot Quickstart](http://wiki.websolr.com/index.php/Sunspot_Quickstart) (WebSolr)
* [Solr, and Sunspot](http://www.kuahyeow.com/2009/08/solr-and-sunspot.html) (YT!)
* [The Saga of the Switch](http://mrb.github.com/2010/04/08/the-saga-of-the-switch.html) (mrb -- includes comparison of Sunspot and Ultrasphinx)

## Contributors

* Mat Brown (mat@patch.com)
* Peer Allan (peer.allan@gmail.com)
* Dmitriy Dzema (dima@dzema.name)
* Benjamin Krause (bk@benjaminkrause.com)
* Marcel de Graaf (marcel@slashdev.nl)
* Brandon Keepers (brandon@opensoul.org)
* Peter Berkenbosch (peterberkenbosch@me.com)
* Brian Atkinson
* Tom Coleman (tom@thesnail.org)
* Matt Mitchell (goodieboy@gmail.com)
* Nathan Beyer (nbeyer@gmail.com)
* Kieran Topping
* Nicolas Braem (nicolas.braem@gmail.com)
* Jeremy Ashkenas (jashkenas@gmail.com)
* Dylan Vaughn (dylanvaughn@yahoo.com)
* Brian Durand (brian@embellishedvisions.com)
* Sam Granieri (sam@samgranieri.com)
* Nick Zadrozny (nick@onemorecloud.com)
* Jason Ronallo (jronallo@gmail.com)

## License

Sunspot is distributed under the MIT License, copyright (c) 2008-2009 Mat Brown
