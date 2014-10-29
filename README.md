# Bouncy

Elasticsearch is a great search engine, but it takes some work to transform its results to an easy to use dataset. Bouncy does exactly that: it maps Elasticsearch results to Eloquent models, so you can keep using the same logic with some special enhancements. In addition, it handles indexing, either manually or automatically on model creation, update or deletion.

This package was created for a personal project and it's still a work in progress. I don't expect it's API to change however.

I was inspired and most of the implementation is based on [Elasticquent](https://github.com/adamfairholm/Elasticquent/). Without it, I would have struggled a lot more trying to understand how Eloquent works under the hood.

## Installation

- Add the package to your `composer.json` file and run `composer update`:
```json
{
    "require": {
        "fadion/bouncy": "dev-master"
    }
}
```

- Add the service provider to your `app/config/app.php` file, inside the `providers` array: `'Fadion\Bouncy\BouncyServiceProvider'`

- Publish the config file by running the following command in the terminal: `php artisan config:publish fadion/bouncy`

- Edit the config files (located in `app/config/packages/bouncy/`) and set the Elasticsearch index name, server configuration, etc.

## Setting Up

There's only one step to tell your models that they should use Bouncy. Just add a trait! I'll be using a fictional `Product` model for the examples.

```php
// file Product.php
use Fadion\Bouncy\BouncyTrait;

class Product extends Eloquent {
    
    use BouncyTrait;

}
```

## Indexing

Before doing any real work, you'll need to index your data. Bouncy handles indexes on models or collections.

Index all records:
```php
Product::all()->index();
```

Index a partial collection of models:
```php
Product::where('sold', true)->get()->index();
```

Index an individual model:
```php
$product = Product::find(10);
$product->index();
```

Collection indexes will be added in bulk, which Elasticsearch handles quite fast. However, keep in mind that indexing a big collection is an exhaustive process. Hitting the sql database and iterating over each row needs time and resources, so try to keep the collection relatively small. You'll have to experiment with the number of data you can index at a time, depending on your server resources and configuration.

## Updating Indexes

Updating doesn't change much from indexing, except that it's usually more safe, as Elasticsearch reduces the chances of version conflicts. It's just a good idea to update the index when it exists, and Bouncy does that automatically. When updating, it will check if it exists and will either update or index it.

Updating a model's index:
```php
$product = Product::find(10);
$product->price = 100;
$product->updateIndex();
```

Updating a model's index using custom attributes. Usually, you'll want to keep the database and Elasticsearch's index in sync and the above method for updating is safer. However, for those occassions where you want manual control, you have the option:
```php
$product = Product::find(10);
$product->updateIndex([
    'price' => 120,
    'sold' => false
]);
```

## Removing Indexes

As in indexing, removing them works the same on both models or collections of models.

Removing the indexes of a collection:
```php
Product::where('quantity', '<', 25)->get()->removeIndex();
```

Removing the index of a single model:
```php
$product = Product::find(10);
$product->removeIndex();
```

The method is intentionally prefixed with 'remove' instead of 'delete', so you don't mistake it with Eloquent's delete() method.

## Automatic Indexes

Bouncy knows when a model is created, saved or deleted and it will reflect those changes to the indexes. Except for the initial index creation of an existing database, you'll generally won't need to use the above methods to manipulate indexes. Any new model's index will be added automatically, will be updated on save and removed when the model is deleted.

The only cases where Bouncy can't update or delete indexes are when doing mass updates or deletes. Those queries run directly on the query builder and it's impossible to override them. I'm investigating for a good way of doing this, but for now, the following queries don't reflect changes on indexes:

```php
Product::where('price', 100)->update(['price' => 110]);
// or
Product::where('price', 100)->delete();
```

You can still call the indexing methods manually and work the limitation. It will add an extra database query, but at least it will keep data in sync.
```php
Product::where('price', 100)->get()->updateIndex(['price' => 110]);
Product::where('price', 100)->update(['price' => 110]);
// or
Product::where('price', 100)->get()->removeIndex();
Product::where('price', 100)->delete();
```

## Searching

Now on the real deal! Searching is where Elasticsearch shines and why you're bothering with it. Bouncy doesn't get in the way, allowing you to build any search query you can imagine in exactly the same way you do with Elasticsearch's client. This allows for great flexibility, while providing your results with a collection of Eloquent models.

An example match query:
```php
$params = [
    'query' => [
        'match' => [
            'title' => 'github'
        ]
    ],
    'size' => 20
];

$products = Product::search($params);

foreach ($products as $product) {
    echo $product->title;
}
```

The `$params` array is exactly as Elasticsearch expects for it to build a JSON request. Nothing new here! You can easily build whatever search query you want, be it a match, multi_match, more_like_this, etc. As you may have noticed, the only missing parameters are 'index' and 'type'. Those are passed automatically by Bouncy. The index is read from the config file, while the type is taken from the model in the same way as Eloquent does.

## Highlights

Highlights are a nice feature to visualize the search query in your printed data. Bouncy augments the models with the highlighted fields when there are ones present. We'll use the same example as above, but this time we'll define a highlighted field.

```php
$params = [
    'query' => [
        'match' => [
            'title' => 'github'
        ]
    ],
    'highlight' => [
        'fields' => [
            'title' => new \stdClass
        ]
    ]
];

$products = Product::search($params);

foreach ($products as $product) {
    echo $product->highlightedTitle;
}
```

The convention is that any highlight field with be appended with `highlighted` and be in camelCase. For example, a 'price' field will be named to 'highlightedPrice', while 'offer_price' will be named 'highlightedOfferPrice'.

## Pagination

Paginated results are important in an application and it's generally a pain with raw Elasticsearch results. Another good reason for using Bouncy! It paginates results in exactly the same way as Eloquent does, so you don't have to learn anything new.

Paginate to 15 models per page (default):
```php
$products = Product::search($params)->paginate();
```

Paginate to an arbitrary number:
```php
$products = Product::search($params)->paginate(30);
```

In your views, you can show pagination links exactly as you've done before:

```php
$products->links();
```

## Limit

For performance, limiting your search results should be done on the Elasticsearch parameter list with the `size` keyword. However, for easy limiting, Bouncy provides that functionality.

Limit to 50 results:
```php
$products = Product::search($params)->limit(50);
```

Or with the alias:
```php
$products = Product::search($params)->take(50);
```

## Searching Shorthands

Having flexibility and all is great, but there are occassions when you just need to run a simple match query and get the job done, without writting a full parameters array. Bouncy offers some shorthand methods for the most common search queries. They work and handle results in the same way as `search()` does, so all of the above applies to them too.

match query:
```php
$products = Product::match($title, $query)
```

multi_match query:
```php
$products = Product::multiMatch(Array $fields, $query)
```

fuzzy query:
```php
$products = Product::fuzzy($field, $value, $fuzziness = 'AUTO')
```

geoshape query:
```php
$products = Product::geoshape($field, Array $coordinates, $type = 'envelope')
```

ids query:
```php
$products = Product::ids(Array $values)
```

more_like_this query
```php
$products = Product::moreLikeThis(Array $fields, Array $ids, $minTermFreq = 1, $percentTermsToMatch = 0.5, $minWordLength = 3)
```

## Elasticsearch Client Facade

Finally, when you'll need it, you can access Elasticsearch's native client in Laravel fashion using a Facade. For this step to work, you'll need to add an alias in `app/config/app.php` in the aliases array: `Fadion\Bouncy\Facades\Elastic`.

```php
Elastic::index();
Elastic::get();
Elastic::search();
Elastic::indices()->create();

// and any other method it provides
```