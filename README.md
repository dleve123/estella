# stella

[![Build Status](https://travis-ci.org/artsy/stella.svg?branch=master)](https://travis-ci.org/artsy/stella)
[![License Status](https://git.legal/projects/3493/badge.svg)](https://git.legal/projects/3493)

Builds on [elasticsearch-model](https://github.com/elastic/elasticsearch-rails/tree/master/elasticsearch-model) to make your Ruby objects searchable with Elasticsearch. Provides fine-grained control of fields, analysis, filters, weightings and boosts.

## Installation

```
gem 'stella', github: 'artsy/stella'
```

The module will try to use Elasticsearch on `localhost:9200` by default. You can configure your global ES client like so:

```ruby
Elasticsearch::Model.client = Elasticsearch::Client.new host: 'foo.com', log: true
```

It is also configurable on a per model basis, see the [doc](https://github.com/elastic/elasticsearch-rails/tree/master/elasticsearch-model#the-elasticsearch-client).

## Indexing

Just include the `Stella::Searchable` module and add a `searchable` block in your ActiveRecord or Mongoid model declaring the fields to be indexed like so:

```ruby
class Artist < ActiveRecord::Base
    include Stella::Searchable

    searchable do
      field :name, type: :string, analysis: Stella::Analysis::FULLTEXT_ANALYSIS, factor: 1.0
      field :keywords, type: :string, analysis: ['snowball', 'shingle'], factor: 0.5
      field :bio, using: :biography, type: :string, index: :not_analyzed
      field :birth_date, type: :date
      field :follows, type: :integer
      field :published, type: :boolean, filter: true
      boost :follows, modifier: 'log1p', factor: 1E-3
    end
    ...
end
```

For a full understanding of the options available for field mappings, see the Elastic [mapping documentation](https://www.elastic.co/guide/en/elasticsearch/reference/2.4/mapping.html). 

The `filter` option allows the field to be used as a filter at search time.

You can optionally provide field weightings to be applied at search time using the `factor` option. These are multipliers. 

Document-level boosts can be applied with the `boost` declaration, see the [field_value_factor](https://www.elastic.co/guide/en/elasticsearch/reference/2.4/query-dsl-function-score-query.html#function-field-value-factor) documentation for boost options. 

While `filter`, `boost` and `factor` are query options, Stella allows for their static declaration in the `searchable` block for simplicity - they will be applied at query time by default when using `#stella_search`.

You can now create your index mappings with this migration: 

```ruby
Artist.reload_index!
```

This uses a default index naming scheme based on your model name, which you can override simply by declaring the following in your model:

```ruby
index_name 'my_index_name'
```

Start indexing documents simply by creating or saving them:

```ruby
Artist.create(name: 'Frank Stella', keywords: ['art', 'minimalism'])
```

Stella adds `after_save` and `after_destroy` callbacks for inline indexing, override these callbacks if you'd like to do your indexing in a background process. For example:

```ruby
class Artist < ActiveRecord::Base
  include Stella::Searchable

  # disable stella inline callbacks
  skip_callback(:save, :after, :es_index)
  skip_callback(:destroy, :after, :es_delete)

  # declare your own
  after_save :delay_es_index
  after_destroy :delay_es_delete
  
  ...
end
```

## Custom Analysis

Stella defines `standard`, `snowball`, `ngram` and `shingle` analysers by default. These cover most search contexts, including auto-suggest. In order to enable full-text search for a field, use:

```ruby
analysis: Stella::Analysis::FULLTEXT_ANALYSIS
```

Or alternatively select your analysis by listing the analysers you want enabled for a given field:

```ruby
es_field :keywords, type: :string, analysis: ['snowball', 'shingle']
```

The searchable block takes a `settings` hash in case you require custom analysers or sharding (see [doc](https://www.elastic.co/guide/en/elasticsearch/guide/current/configuring-analyzers.html)):

```ruby
my_analysis = {
  tokenizer: {
    ...
  },
  filter: {
    ...
  }
}

my_settings = { 
  analysis: my_analysis,
  index: { 
    number_of_shards: 1, 
    number_of_replicas: 1 
  } 
}

searchable my_settings do
  ...
end
```

It will otherwise use Stella defaults.

## Searching

Finally perform full-text search:

```ruby
Artist.stella_search(term: 'frank')
Artist.stella_search(term: 'minimalism')
```

Stella searches all analysed text fields by default, using a [multi_match](https://www.elastic.co/guide/en/elasticsearch/guide/current/multi-match-query.html) search. The search will return an array of database records in score order. If you'd like access to the raw Elasticsearch response data use the `raw` option:

```ruby
Artist.stella_search(term: 'frank', raw: true)
```

Stella supports filtering on `filter` fields and pagination:

```ruby
Artist.stella_search(term: 'frank', published: true)
Artist.stella_search(term: 'frank', size: 10, from: 5)
```

If you'd like to customize your query further, you can extend `Stella::Query` and override the `query_definition`:

```ruby
class MyQuery < Stella::Query
  def query_definition
    {
      multi_match: {
        ...
      }
    }
  end
end
```

And then override class method `stella_search_query` to direct Stella to use your query object:

```ruby
class Artist < ActiveRecord::Base
  include Stella::Searchable

  searchable do
    ...
  end

  def self.stella_search_query
    MyQuery
  end
end

Artist.stella_search (term: 'frank')
```

For further search customization, see the [elasticsearch dsl](https://github.com/elastic/elasticsearch-rails/tree/master/elasticsearch-model#the-elasticsearch-dsl). 

Stella works with any ActiveRecord or Mongoid compatible data models.

## Contributing

Just fork the repo and submit a pull request.

## License

Copyright (c) 2017 Artsy Inc., [MIT License](LICENSE).
