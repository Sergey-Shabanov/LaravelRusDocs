git eaeef461d4fc2ef00aee94ea6f9f504bf4748b8d

---

# Laravel Scout

- [Введение](#introduction)
- [Установка](#installation)
    - [Требования к драйверам](#driver-prerequisites)
    - [Очередь](#queueing)
- [Настройка](#configuration)
    - [Настройка индексов моделей](#configuring-model-indexes)
    - [Настройка поисковых данных](#configuring-searchable-data)
    - [Настройка идентификатора модели](#configuring-the-model-id)
    - [Идентификация пользователей](#identifying-users)
- [Локальная разработка](#local-development)
- [Индексирование](#indexing)
    - [Пакетный импорт](#batch-import)
    - [Добавление записей](#adding-records)
    - [Обновление записей](#updating-records)
    - [Удаление записей](#removing-records)
    - [Приостановка индексации](#pausing-indexing)
    - [Экземпляры моделей с условным поиском](#conditionally-searchable-model-instances)
- [Поиск](#searching)
    - [Условия Where](#where-clauses)
    - [Постраничная разбивка данных (Pagination)](#pagination)
    - [Псевдоудаление](#soft-deleting)
    - [Настройка поискового движка](#customizing-engine-searches)
- [Разработка поискового движка](#custom-engines)
- [Собственные методы поиска](#builder-macros)

<a name="introduction"></a>
## Введение

Laravel Scout предоставляет простое решение на основе драйверов для добавления полнотекстового поиска в ваши [модели Eloquent](/docs/{{version}}/eloquent). Используя наблюдателей моделей, Scout будет автоматически синхронизировать ваши поисковые индексы с данными моделей Eloquent.

В настоящее время Scout поставляется с драйверами [Algolia](https://www.algolia.com/) и [MeiliSearch](https://www.meilisearch.com). Кроме того, Scout включает поисковый драйвер «коллекций», предназначенный для использования в локальной разработке и не требующий каких-либо внешних зависимостей или сторонних сервисов. Кроме того, написать собственные драйверы просто, и вы можете свободно расширять Scout своими собственными реализациями поиска.

<a name="installation"></a>
## Установка

Сначала установите Scout через менеджер пакетов Composer:

    composer require laravel/scout

После установки Scout вы должны опубликовать файл конфигурации Scout с помощью Artisan-команды `vendor:publish`. Эта команда добавит файл конфигурации `scout.php` в каталог `config` вашего приложения:

    php artisan vendor:publish --provider="Laravel\Scout\ScoutServiceProvider"

Наконец, добавьте трейт (trait) `Laravel\Scout\Searchable` к модели, которую вы хотите сделать доступной для поиска. Этот трейт зарегистрирует наблюдателя модели, который будет автоматически синхронизировать модель с вашим драйвером поиска:

    <?php

    namespace App\Models;

    use Illuminate\Database\Eloquent\Model;
    use Laravel\Scout\Searchable;

    class Post extends Model
    {
        use Searchable;
    }

<a name="driver-prerequisites"></a>
### Требования к драйверам

<a name="algolia"></a>
#### Algolia

При использовании драйвера Algolia вы должны настроить учетные данные Algolia `id` и `secret` в файле конфигурации `config/scout.php`. После того как ваши учетные данные будут настроены, вам также необходимо будет установить Algolia PHP SDK через диспетчер пакетов Composer:

    composer require algolia/algoliasearch-client-php

<a name="meilisearch"></a>
#### MeiliSearch

При использовании драйвера MeiliSearch вам необходимо установить MeiliSearch PHP SDK через менеджер пакетов Composer:

    composer require meilisearch/meilisearch-php http-interop/http-factory-guzzle

Затем установите переменную среды `SCOUT_DRIVER`, а также учетные данные вашего MeiliSearch `host` и `key` в файле` .env` вашего приложения:

    SCOUT_DRIVER=meilisearch
    MEILISEARCH_HOST=http://127.0.0.1:7700
    MEILISEARCH_KEY=masterKey

Для получения дополнительной информации обратитесь к [документации MeiliSearch](https://docs.meilisearch.com/learn/getting_started/quick_start.html).

> {tip} Если вы не знаете, как установить MeiliSearch на свой локальный компьютер, вы можете использовать [Laravel Sail](/docs/{{version}}/sail#meilisearch), официально поддерживаемую Laravel среду разработки Docker.

<a name="queueing"></a>
### Очередь

Хотя при использовании Scout не является строго обязательным, но вам следует серьезно подумать о настройке [драйвера очереди](/docs/{{version}}/queues) перед использованием библиотеки. Запуск обработчика очереди позволит Scout ставить в очередь все операции, которые синхронизируют информацию вашей модели с вашими поисковыми индексами, обеспечивая гораздо лучшее время отклика для веб-интерфейса вашего приложения.

После того как вы настроили драйвер очереди, установите значение опции `queue` в вашем конфигурационном файле `config/scout.php` равным `true`:

    'queue' => true,

<a name="configuration"></a>
## Настройка

<a name="configuring-model-indexes"></a>
### Настройка индексов моделей

Каждая модель Eloquent синхронизируется с заданным поисковым «индексом», который содержит все доступные для поиска записи для этой модели. Другими словами, вы можете думать о каждом индексе как о таблице MySQL. По умолчанию каждая модель будет сохранена в индексе, соответствующем типичному «табличному» имени модели. Обычно это форма множественного числа от названия модели; однако вы можете настроить индекс, переопределив метод `searchableAs` в модели:

    <?php

    namespace App\Models;

    use Illuminate\Database\Eloquent\Model;
    use Laravel\Scout\Searchable;

    class Post extends Model
    {
        use Searchable;

        /**
         * Переопределение имени индекса модели по умолчанию
         *
         * @return string
         */
        public function searchableAs()
        {
            return 'posts_index';
        }
    }

<a name="configuring-searchable-data"></a>
### Настройка поисковых данных

По умолчанию вся форма `toArray` данной модели будет сохранена в ее поисковом индексе. Если вы хотите настроить данные, которые синхронизируются с поисковым индексом, вы можете переопределить метод `toSearchableArray` в модели:

    <?php

    namespace App\Models;

    use Illuminate\Database\Eloquent\Model;
    use Laravel\Scout\Searchable;

    class Post extends Model
    {
        use Searchable;

        /**
         * Переопределение массива индекса модели по умолчанию
         *
         * @return array
         */
        public function toSearchableArray()
        {
            $array = $this->toArray();

            // Customize the data array...

            return $array;
        }
    }

<a name="configuring-the-model-id"></a>
### Настройка идентификатора модели

По умолчанию Scout будет использовать первичный ключ модели в качестве уникального идентификатора / ключа модели, который хранится в поисковом индексе. Если вам нужно настроить это поведение, вы можете переопределить методы `getScoutKey` и `getScoutKeyName` в модели:

    <?php

    namespace App\Models;

    use Illuminate\Database\Eloquent\Model;
    use Laravel\Scout\Searchable;

    class User extends Model
    {
        use Searchable;

        /**
         * Переопределение значения ключа индекса модели по умолчанию
         *
         * @return mixed
         */
        public function getScoutKey()
        {
            return $this->email;
        }

        /**
         * Переопределение имени ключа индекса модели по умолчанию
         *
         * @return mixed
         */
        public function getScoutKeyName()
        {
            return 'email';
        }
    }

<a name="identifying-users"></a>
### Идентификация пользователей

Scout также позволяет автоматически идентифицировать пользователей при использовании [Algolia](https://algolia.com). Связывание аутентифицированного пользователя с операциями поиска может быть полезно при просмотре аналитики поиска на панели инструментов Algolia. Вы можете включить идентификацию пользователя, определив для переменной среды `SCOUT_IDENTIFY` значение `true` в файле `.env` вашего приложения:

    SCOUT_IDENTIFY=true

Включение этой функции также передаст IP-адрес запроса и основной идентификатор вашего аутентифицированного пользователя в Algolia, поэтому эти данные будут связаны с любым поисковым запросом, сделанным пользователем.

<a name="local-development"></a>
## Локальная разработка

Хотя вы можете использовать поисковые системы Algolia или MeiliSearch во время локальной разработки, вам может быть удобнее начать работу с поисковым драйвером «коллекций». Драйвер коллекций данных будет использовать условие «where» и фильтрацию набора результатов из вашей существующей базы данных, чтобы определить применимые результаты поиска для запроса. При использовании этого механизма нет необходимости «индексировать» доступные для поиска модели, поскольку они будут просто извлечены из локальной базы данных.

Чтобы использовать драйвер коллекций, вы можете просто установить для переменной среды `SCOUT_DRIVER` значение `collection` или указать драйвер `collection` непосредственно в файле конфигурации `scout` вашего приложения:

```ini
SCOUT_DRIVER=collection
```

После того как вы указали драйвер коллекции в качестве предпочтительного, вы можете начать [выполнение поисковых запросов](#searching) по вашим моделям. Индексирование поисковой системой, необходимое для заполнения индексов Algolia или MeiliSearch, не требуется при использовании драйвера коллекций.

<a name="indexing"></a>
## Индексирование

<a name="batch-import"></a>
### Пакетный импорт

If you are installing Scout into an existing project, you may already have database records you need to import into your indexes. Scout provides a `scout:import` Artisan command that you may use to import all of your existing records into your search indexes:

    php artisan scout:import "App\Models\Post"

The `flush` command may be used to remove all of a model's records from your search indexes:

    php artisan scout:flush "App\Models\Post"

<a name="modifying-the-import-query"></a>
#### Modifying The Import Query

If you would like to modify the query that is used to retrieve all of your models for batch importing, you may define a `makeAllSearchableUsing` method on your model. This is a great place to add any eager relationship loading that may be necessary before importing your models:

    /**
     * Modify the query used to retrieve models when making all of the models searchable.
     *
     * @param  \Illuminate\Database\Eloquent\Builder  $query
     * @return \Illuminate\Database\Eloquent\Builder
     */
    protected function makeAllSearchableUsing($query)
    {
        return $query->with('author');
    }

<a name="adding-records"></a>
### Добавление записей

Once you have added the `Laravel\Scout\Searchable` trait to a model, all you need to do is `save` or `create` a model instance and it will automatically be added to your search index. If you have configured Scout to [use queues](#queueing) this operation will be performed in the background by your queue worker:

    use App\Models\Order;

    $order = new Order;

    // ...

    $order->save();

<a name="adding-records-via-query"></a>
#### Adding Records Via Query

If you would like to add a collection of models to your search index via an Eloquent query, you may chain the `searchable` method onto the Eloquent query. The `searchable` method will [chunk the results](/docs/{{version}}/eloquent#chunking-results) of the query and add the records to your search index. Again, if you have configured Scout to use queues, all of the chunks will be imported in the background by your queue workers:

    use App\Models\Order;

    Order::where('price', '>', 100)->searchable();

You may also call the `searchable` method on an Eloquent relationship instance:

    $user->orders()->searchable();

Or, if you already have a collection of Eloquent models in memory, you may call the `searchable` method on the collection instance to add the model instances to their corresponding index:

    $orders->searchable();

> {tip} The `searchable` method can be considered an "upsert" operation. In other words, if the model record is already in your index, it will be updated. If it does not exist in the search index, it will be added to the index.

<a name="updating-records"></a>
### Обновление записей

To update a searchable model, you only need to update the model instance's properties and `save` the model to your database. Scout will automatically persist the changes to your search index:

    use App\Models\Order;

    $order = Order::find(1);

    // Update the order...

    $order->save();

You may also invoke the `searchable` method on an Eloquent query instance to update a collection of models. If the models do not exist in your search index, they will be created:

    Order::where('price', '>', 100)->searchable();

If you would like to update the search index records for all of the models in a relationship, you may invoke the `searchable` on the relationship instance:

    $user->orders()->searchable();

Or, if you already have a collection of Eloquent models in memory, you may call the `searchable` method on the collection instance to update the model instances in their corresponding index:

    $orders->searchable();

<a name="removing-records"></a>
### Удаление записей

To remove a record from your index you may simply `delete` the model from the database. This may be done even if you are using [soft deleted](/docs/{{version}}/eloquent#soft-deleting) models:

    use App\Models\Order;

    $order = Order::find(1);

    $order->delete();

If you do not want to retrieve the model before deleting the record, you may use the `unsearchable` method on an Eloquent query instance:

    Order::where('price', '>', 100)->unsearchable();

If you would like to remove the search index records for all of the models in a relationship, you may invoke the `unsearchable` on the relationship instance:

    $user->orders()->unsearchable();

Or, if you already have a collection of Eloquent models in memory, you may call the `unsearchable` method on the collection instance to remove the model instances from their corresponding index:

    $orders->unsearchable();

<a name="pausing-indexing"></a>
### Приостановка индексации

Sometimes you may need to perform a batch of Eloquent operations on a model without syncing the model data to your search index. You may do this using the `withoutSyncingToSearch` method. This method accepts a single closure which will be immediately executed. Any model operations that occur within the closure will not be synced to the model's index:

    use App\Models\Order;

    Order::withoutSyncingToSearch(function () {
        // Perform model actions...
    });

<a name="conditionally-searchable-model-instances"></a>
### Экземпляры моделей с условным поиском

Sometimes you may need to only make a model searchable under certain conditions. For example, imagine you have `App\Models\Post` model that may be in one of two states: "draft" and "published". You may only want to allow "published" posts to be searchable. To accomplish this, you may define a `shouldBeSearchable` method on your model:

    /**
     * Determine if the model should be searchable.
     *
     * @return bool
     */
    public function shouldBeSearchable()
    {
        return $this->isPublished();
    }

The `shouldBeSearchable` method is only applied when manipulating models through the `save` and `create` methods, queries, or relationships. Directly making models or collections searchable using the `searchable` method will override the result of the `shouldBeSearchable` method.

<a name="searching"></a>
## Поиск

You may begin searching a model using the `search` method. The search method accepts a single string that will be used to search your models. You should then chain the `get` method onto the search query to retrieve the Eloquent models that match the given search query:

    use App\Models\Order;

    $orders = Order::search('Star Trek')->get();

Since Scout searches return a collection of Eloquent models, you may even return the results directly from a route or controller and they will automatically be converted to JSON:

    use App\Models\Order;
    use Illuminate\Http\Request;

    Route::get('/search', function (Request $request) {
        return Order::search($request->search)->get();
    });

If you would like to get the raw search results before they are converted to Eloquent models, you may use the `raw` method:

    $orders = Order::search('Star Trek')->raw();

<a name="custom-indexes"></a>
#### Custom Indexes

Search queries will typically be performed on the index specified by the model's [`searchableAs`](#configuring-model-indexes) method. However, you may use the `within` method to specify a custom index that should be searched instead:

    $orders = Order::search('Star Trek')
        ->within('tv_shows_popularity_desc')
        ->get();

<a name="where-clauses"></a>
### Условия Where

Scout позволяет добавлять в поисковые запросы простые условия "where" ("где"). В настоящее время эти условия поддерживают только базовые проверки числового равенства и в первую очередь полезны для определения области поисковых запросов по идентификатору владельца. Поскольку поисковый индекс не является реляционной базой данных, более сложные условия "where" в настоящее время не поддерживаются:

    use App\Models\Order;

    $orders = Order::search('Star Trek')->where('user_id', 1)->get();

<a name="pagination"></a>
### Постраничная разбивка данных (Pagination) (Пагинация)

In addition to retrieving a collection of models, you may paginate your search results using the `paginate` method. This method will return an `Illuminate\Pagination\LengthAwarePaginator` instance just as if you had [paginated a traditional Eloquent query](/docs/{{version}}/pagination):

    use App\Models\Order;

    $orders = Order::search('Star Trek')->paginate();

You may specify how many models to retrieve per page by passing the amount as the first argument to the `paginate` method:

    $orders = Order::search('Star Trek')->paginate(15);

Once you have retrieved the results, you may display the results and render the page links using [Blade](/docs/{{version}}/blade) just as if you had paginated a traditional Eloquent query:

```html
<div class="container">
    @foreach ($orders as $order)
        {{ $order->price }}
    @endforeach
</div>

{{ $orders->links() }}
```

Of course, if you would like to retrieve the pagination results as JSON, you may return the paginator instance directly from a route or controller:

    use App\Models\Order;
    use Illuminate\Http\Request;

    Route::get('/orders', function (Request $request) {
        return Order::search($request->input('query'))->paginate(15);
    });

<a name="soft-deleting"></a>
### Псевдоудаление

Если ваши проиндексированные модели [псевдоудалены](/docs/{{version}}/eloquent#soft-deleting) и вам нужно выполнить поиск по своим псевдоудаленным моделям, установите параметр `soft_delete` в файле `config/scout.php` на `true`:

    'soft_delete' => true,

When this configuration option is `true`, Scout will not remove soft deleted models from the search index. Instead, it will set a hidden `__soft_deleted` attribute on the indexed record. Then, you may use the `withTrashed` or `onlyTrashed` methods to retrieve the soft deleted records when searching:

    use App\Models\Order;

    // Include trashed records when retrieving results...
    $orders = Order::search('Star Trek')->withTrashed()->get();

    // Only include trashed records when retrieving results...
    $orders = Order::search('Star Trek')->onlyTrashed()->get();

> {tip} When a soft deleted model is permanently deleted using `forceDelete`, Scout will remove it from the search index automatically.

<a name="customizing-engine-searches"></a>
### Настройка поискового движка

If you need to perform advanced customization of the search behavior of an engine you may pass a closure as the second argument to the `search` method. For example, you could use this callback to add geo-location data to your search options before the search query is passed to Algolia:

    use Algolia\AlgoliaSearch\SearchIndex;
    use App\Models\Order;

    Order::search(
        'Star Trek',
        function (SearchIndex $algolia, string $query, array $options) {
            $options['body']['query']['bool']['filter']['geo_distance'] = [
                'distance' => '1000km',
                'location' => ['lat' => 36, 'lon' => 111],
            ];

            return $algolia->search($query, $options);
        }
    )->get();

<a name="custom-engines"></a>
## Разработка поискового движка

<a name="writing-the-engine"></a>
#### Writing The Engine

If one of the built-in Scout search engines doesn't fit your needs, you may write your own custom engine and register it with Scout. Your engine should extend the `Laravel\Scout\Engines\Engine` abstract class. This abstract class contains eight methods your custom engine must implement:

    use Laravel\Scout\Builder;

    abstract public function update($models);
    abstract public function delete($models);
    abstract public function search(Builder $builder);
    abstract public function paginate(Builder $builder, $perPage, $page);
    abstract public function mapIds($results);
    abstract public function map(Builder $builder, $results, $model);
    abstract public function getTotalCount($results);
    abstract public function flush($model);

You may find it helpful to review the implementations of these methods on the `Laravel\Scout\Engines\AlgoliaEngine` class. This class will provide you with a good starting point for learning how to implement each of these methods in your own engine.

<a name="registering-the-engine"></a>
#### Registering The Engine

Once you have written your custom engine, you may register it with Scout using the `extend` method of the Scout engine manager. Scout's engine manager may be resolved from the Laravel service container. You should call the `extend` method from the `boot` method of your `App\Providers\AppServiceProvider` class or any other service provider used by your application:

    use App\ScoutExtensions\MySqlSearchEngine
    use Laravel\Scout\EngineManager;

    /**
     * Bootstrap any application services.
     *
     * @return void
     */
    public function boot()
    {
        resolve(EngineManager::class)->extend('mysql', function () {
            return new MySqlSearchEngine;
        });
    }

Once your engine has been registered, you may specify it as your default Scout `driver` in your application's `config/scout.php` configuration file:

    'driver' => 'mysql',

<a name="builder-macros"></a>
## Собственные методы поиска

Если вы хотите назначить собственный метод построения поиска Scout, вы можете использовать метод `macro` класса `Laravel\Scout\Builder`. Как правило, "макросы" следует определять в методе `boot` [сервисного провайдера](/docs/{{version}}/providers):

    use Illuminate\Support\Facades\Response;
    use Illuminate\Support\ServiceProvider;
    use Laravel\Scout\Builder;

    /**
     * Bootstrap any application services.
     *
     * @return void
     */
    public function boot()
    {
        Builder::macro('count', function () {
            return $this->engine()->getTotalCount(
                $this->engine()->search($this)
            );
        });
    }

The `macro` function accepts a macro name as its first argument and a closure as its second argument. The macro's closure will be executed when calling the macro name from a `Laravel\Scout\Builder` implementation:

    use App\Models\Order;

    Order::search('Star Trek')->count();
