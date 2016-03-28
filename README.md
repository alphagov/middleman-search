# Middleman::Search

LunrJS-based search for Middleman.

## Installation

Add this line to your application's Gemfile:

    gem 'middleman-search'

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install middleman-search

## Usage

You need to activate the module in your `config.rb`, telling the extension how to index your resources:

```ruby
activate :search do |search|

  search.resources = ['blog/', 'index.html', 'contactus/index.html']

  search.index_path = 'search/lunr-index.json' # defaults to `search.json`

  search.fields = {
    title:   {boost: 100, store: true, required: true},
    content: {boost: 50},
    url:     {index: false, store: true},
    author:  {boost: 30}
  }
end
```

Where `resources` is a list of the beginning of the URL of the resources to index (tested with `String#start_with?`), `index_path` is the relative path of the generated index file in your site, and `fields` is a hash with one entry for each field to be indexed, with a hash of options associated:

- `boost` Specifies lunr relevance boost when searching this field
- `store` Whether to store this field in the document map (see below), defaults to false
- `index` Whether to index this field, defaults to true
- `required` The resource will not be indexed if a field marked as required has an empty or null value

Note that a special field `id` is included automatically, with an autogenerated identifier to be used as the `ref` for the document.

All fields values are retrieved from the resource `data` (i.e. its frontmatter), or from the `options` in the `resource.metadata` (i.e. any options specified in a `proxy` page), except for:
- `url` which is the actual resource url
- `content` the text extracted from the rendered resource, without including its layout

### Manual index manipulation

You can fully customise the content to be indexed and stored per resource by defining a `before_index` callback:

```ruby
activate :search do |search|
  search.before_index = Proc.new do |to_index, to_store, resource|
    if author = resource.data.author
      to_index[:author] = data.authors[author].name
    end
  end
end
```

This option accepts a callback that will be executed for each resource, and will be executed with the document to be indexed and the map to be stored, in the `index` and `docs` objects of the output respectively (see below), as well as the resource being processed. You can use this callback to modify either of those, or `throw(:skip)` to skip the resource in question.

### Lunr pipeline configuration

In some cases, you may want to add new function to the lunr pipeline, both for creating the indexing and then for searching. You can do this by providing a `pipeline` hash with function names and body, for example:

```ruby
activate :search do |search|
  search.pipeline = {
    tildes: <<-JS
      function(token, tokenIndex, tokens) {
        return token
          .replace('á', 'a')
          .replace('é', 'e')
          .replace('í', 'i')
          .replace('ó', 'o')
          .replace('ú', 'u');
      }
    JS
  }
end
```

This will register the `tildes` function in the lunr pipeline and add it when building the index. From the Lunr documentation:

> Functions in the pipeline are called with three arguments: the current token being processed; the index of that token in the array of tokens, and the whole list of tokens part of the document being processed. This enables simple unigram processing of tokens as well as more sophisticated n-gram processing.
>
> The function should return the processed version of the text, which will in turn be passed to the next function in the pipeline. Returning undefined will prevent any further processing of the token, and that token will not make it to the index.

Note that if you add a function to the pipeline, it will also be loaded when de-serialising the index, and lunr will fail with an `Cannot load un-registered function: tildes` error if it has not been re-registered. You can either register them manually, or simply include the following in a `.js.erb` file to be executed __before__ loading the index:
```erb
<%= search_lunr_js_pipeline %>
```


## Index file

The generated index file contains a JSON object with two properties:
- `index` contains the serialised lunr.js index, which you can load via `lunr.Index.load(lunrData.index)`
- `docs` is a map from the autogenerated document ids to an object that contains the attributes configured for storage

You will typically load the `index` into a lunr index instance, and then use the `docs` map to look up the returned value and present it to the user.

You should also `require` the `lunr.min.js` file in your main sprockets javascript file (if using the asset pipeline) to be able to actually load the index:

```javascript
//= require lunr.min
```

### Asset pipeline

The Middleman pipeline (if enabled) does not include `json` files by default, but you can easily modify this by adding `.json` to the `exts` option of the corresponding extensions, such as `gzip` and `asset_hash`:

```ruby
activate :asset_hash do |asset_hash|
  asset_hash.exts << '.json'
end
```

Note that if you run the index json file through the asset hash extension, you will need to retrieve the actual destination URL when loading the file in the browser for searching, using the `search_index_path` view helper:

```javascript
var lunrIndex = null;
var lunrData  = null;

// Download index data
$.ajax({
  url: "<%= search_index_path %>",
  cache: true,
  method: 'GET',
  success: function(data) {
    lunrData = data;
    lunrIndex = lunr.Index.load(lunrData.index);
  }
});
```

## Acknowledgments

A big thank you to:
- [Octo-Labs](https://github.com/Octo-Labs)'s [jagthedrummer](https://github.com/jagthedrummer) for his [`middleman-alias`](https://github.com/Octo-Labs/middleman-alias) extension, in which we based for developing this one.
- [jnovos](https://github.com/jnovos) and [256dpi](https://github.com/256dpi), for their [`middleman-lunrjs`](https://github.com/jnovos/middleman-lunrjs) and [`middleman-lunr`](https://github.com/256dpi/middleman-lunr) extensions, which served as inspirations for making this one.
- [olivernn](https://github.com/olivernn) and all [`lunr.js`](http://lunrjs.com/) [contributors](https://github.com/olivernn/lunr.js/graphs/contributors)
- [The Middleman](https://middlemanapp.com/) [team](https://github.com/orgs/middleman/people) and [contributors](https://github.com/middleman/middleman/graphs/contributors)
