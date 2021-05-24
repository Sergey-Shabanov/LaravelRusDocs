git bbe4f5210f58fddb828c4e16261c552b8db59ef5

---

# Laravel Passport

- [Введение](#introduction)
    - [Passport или Sanctum?](#passport-or-sanctum)
- [Установка](#installation)
    - [Развертывание Passport](#deploying-passport)
    - [Настройка миграции](#migration-customization)
    - [Обновление Passport](#upgrading-passport)
- [Настройка](#configuration)
    - [Хеширование секретного ключа клиента](#client-secret-hashing)
    - [Срок жизни токена](#token-lifetimes)
    - [Переопределение моделей по умолчанию](#overriding-default-models)
- [Выдача токенов доступа](#issuing-access-tokens)
    - [Управление клиентами](#managing-clients)
    - [Запрос токенов](#requesting-tokens)
    - [Обновление токенов](#refreshing-tokens)
    - [Отзыв токенов](#revoking-tokens)
    - [Удаление токенов](#purging-tokens)
- [Предоставление кода авторизации с помощью PKCE](#code-grant-pkce)
    - [Создание клиента](#creating-a-auth-pkce-grant-client)
    - [Запрос токенов](#requesting-auth-pkce-grant-tokens)
- [Парольные токены](#password-grant-tokens)
    - [Создание токенов](#creating-a-password-grant-client)
    - [Запрос токенов](#requesting-password-grant-tokens)
    - [Запрос токена для всех областей](#requesting-all-scopes)
    - [Настройка пользовательского провайдера](#customizing-the-user-provider)
    - [Настройка поля имени пользователя](#customizing-the-username-field)
    - [Настройка проверки пароля пользователя](#customizing-the-password-validation)
- [Неявные токены](#implicit-grant-tokens)
- [Токены учетных данных](#client-credentials-grant-tokens)
- [Токены персонального доступа](#personal-access-tokens)
    - [Создание токенов](#creating-a-personal-access-client)
    - [Управление токенами](#managing-personal-access-tokens)
- [Защита маршрутов](#protecting-routes)
    - [Через посредников](#via-middleware)
    - [Через передачу токена](#passing-the-access-token)
- [Области токенов](#token-scopes)
    - [Определение областей](#defining-scopes)
    - [Области по-умолчанию](#default-scope)
    - [Назначение областей токенам](#assigning-scopes-to-tokens)
    - [Проверка областей](#checking-scopes)
- [Использование API через JavaScript](#consuming-your-api-with-javascript)
- [События](#events)
- [Тестирование](#testing)

<a name="introduction"></a>
## Введение

Laravel Passport обеспечивает полную реализацию сервера OAuth2 для вашего приложения Laravel за считанные минуты. Passport построен на основе [League OAuth2](https://github.com/thephpleague/oauth2-server), который поддерживается Энди Миллингтоном (Andy Millington) и Саймоном Хэмпом (Simon Hamp).

> {Примечание} В этой документации предполагается, что вы уже знакомы с OAuth2. Если вы ничего не знаете о OAuth2, перед продолжением ознакомьтесь с общей [терминологией](https://oauth2.thephpleague.com/terminology/) и функциями OAuth2.

<a name="passport-or-sanctum"></a>
### Passport или Sanctum?

Прежде чем начать, вы можете определиться, будет ли ваше приложение лучше обслуживаться через Laravel Passport или [Laravel Sanctum](/docs/{{version}}/sanctum). Если вашему приложению необходима поддержка OAuth2, то следует использовать Laravel Passport.

Однако, если вы пытаетесь аутентифицировать одностраничное приложение, мобильное приложение или выдавать токены API, вам следует использовать [Laravel Sanctum](/docs/{{version}}/sanctum). Laravel Sanctum не поддерживает OAuth2; однако он обеспечивает гораздо более простой опыт разработки аутентификации API.

<a name="installation"></a>
## Установка

Для начала установите Passport через менеджер пакетов Composer:

    composer require laravel/passport

[Сервис-провайдер](/docs/{{version}}/providers) Passport регистрирует свой собственный каталог миграции базы данных, поэтому вам следует перенести свою базу данных после установки пакета. При миграции паспорта будут созданы таблицы, необходимые вашему приложению для хранения клиентов OAuth2 и токенов доступа:

    php artisan migrate

Затем вы должны выполнить Artisan-команду `passport:install`. Эта команда создаст ключи шифрования, необходимые для создания токенов безопасного доступа. Кроме того, команда создаст клиентов "personal access" и "password grant", которые будут использоваться для генерации токенов доступа:

    php artisan passport:install

> {Примечание} Если вы хотите использовать UUID в качестве значения первичного ключа модели Passport `Client` вместо автоматически увеличивающихся целых чисел, установите Passport, используя [the `uuids` option](#client-uuids).

После выполнения команды `passport:install` добавьте [трейт](https://www.php.net/manual/ru/language.oop5.traits.php) `Laravel\Passport\HasApiTokens` в свою модель `App\Models\User`. Этот трейт предоставит вашей модели несколько вспомогательных методов, которые позволят вам проверить токен и области аутентифицированного пользователя:

    <?php

    namespace App\Models;

    use Illuminate\Database\Eloquent\Factories\HasFactory;
    use Illuminate\Foundation\Auth\User as Authenticatable;
    use Illuminate\Notifications\Notifiable;
    use Laravel\Passport\HasApiTokens;

    class User extends Authenticatable
    {
        use HasApiTokens, HasFactory, Notifiable;
    }

Затем вы должны вызвать метод `Passport::routes` в методе `boot` вашего `App\Providers\AuthServiceProvider`. Этот метод зарегистрирует маршруты, необходимые для выдачи токенов, отзыва токенов, токенов персонального доступа и клиентов:

    <?php

    namespace App\Providers;

    use Illuminate\Foundation\Support\Providers\AuthServiceProvider as ServiceProvider;
    use Illuminate\Support\Facades\Gate;
    use Laravel\Passport\Passport;

    class AuthServiceProvider extends ServiceProvider
    {
        /**
         * Сопоставление политик приложения.
         *
         * @var array
         */
        protected $policies = [
            'App\Models\Model' => 'App\Policies\ModelPolicy',
        ];

        /**
         * Регистрация сервисов аутентификации и авторизации.
         *
         * @return void
         */
        public function boot()
        {
            $this->registerPolicies();

            if (! $this->app->routesAreCached()) {
                Passport::routes();
            }
        }
    }

Наконец, в файле конфигурации приложения `config/auth.php` вы должны установить для параметра `driver` раздела `api` значение `passport`. Это укажет вашему приложению использовать Passport `TokenGuard` при аутентификации входящих запросов API:

    'guards' => [
        'web' => [
            'driver' => 'session',
            'provider' => 'users',
        ],

        'api' => [
            'driver' => 'passport',
            'provider' => 'users',
        ],
    ],

<a name="client-uuids"></a>
#### Клиентские UUID

Вы также можете запустить команду `passport:install` с опцией `--uuids`. Эта опция укажет Passport, что вы хотели бы использовать UUID вместо автоматически увеличивающихся целых чисел в качестве значений первичного ключа модели Passport `Client`. После выполнения команды `passport:install` с параметром `--uuids` вы получите дополнительные инструкции по отключению миграций по умолчанию для Passport:

    php artisan passport:install --uuids

<a name="deploying-passport"></a>
### Развертывание Passport

При первом развертывании Passport на серверах вашего приложения вам, вероятно, потребуется выполнить команду `passport:keys`. Эта команда генерирует ключи шифрования, необходимые Passport для создания токенов доступа. Сгенерированные ключи обычно не хранятся в системе контроля версий:

    php artisan passport:keys

При необходимости вы можете указать путь, откуда должны быть загружены ключи Passport. Для этого вы можете использовать метод `Passport::loadKeysFrom`. Обычно этот метод следует вызывать из метода `boot` класса `App\Providers\AuthServiceProvider` вашего приложения:

    /**
     * Регистрация сервисов аутентификации и авторизации.
     *
     * @return void
     */
    public function boot()
    {
        $this->registerPolicies();

        Passport::routes();

        Passport::loadKeysFrom(__DIR__.'/../secrets/oauth');
    }

<a name="loading-keys-from-the-environment"></a>
#### Загрузка ключей из окружения

В качестве альтернативы вы можете опубликовать файл конфигурации Passport с помощью Artisan-команды `vendor:publish`:

    php artisan vendor:publish --tag=passport-config

После публикации файла конфигурации вы можете загрузить ключи шифрования вашего приложения, определив их как переменные среды:

```bash
PASSPORT_PRIVATE_KEY="-----BEGIN RSA PRIVATE KEY-----
<private key here>
-----END RSA PRIVATE KEY-----"

PASSPORT_PUBLIC_KEY="-----BEGIN PUBLIC KEY-----
<public key here>
-----END PUBLIC KEY-----"
```

<a name="migration-customization"></a>
### Настройка миграции

Если вы не собираетесь использовать миграции Passport по умолчанию, вам следует вызвать метод `Passport::ignoreMigrations` в методе `register` вашего класса `App\Providers\AppServiceProvider`. Вы можете экспортировать миграции по умолчанию, используя Artisan-команду `vendor:publish`:

    php artisan vendor:publish --tag=passport-migrations

<a name="upgrading-passport"></a>
### Обновление Passport

При обновлении до новой основной версии Passport важно внимательно изучить [руководство по обновлению](https://github.com/laravel/passport/blob/master/UPGRADE.md).

<a name="configuration"></a>
## Настройка

<a name="client-secret-hashing"></a>
### Хеширование секретного ключа клиента

Если вы хотите, чтобы секретные ключи клиента хешировались при хранении в базе данных, вы должны вызвать метод `Passport::hashClientSecrets` в методе `boot` класса `App\Providers\AuthServiceProvider`:

    use Laravel\Passport\Passport;

    Passport::hashClientSecrets();

После включения все секретные ключи будут отображаться пользователю один раз, только после их создания. Поскольку значение секретного ключа в виде обычного текста никогда не хранится в базе данных, невозможно восстановить значение ключа, если оно утеряно.

<a name="token-lifetimes"></a>
### Срок жизни токена

По умолчанию Passport выдает долговременные токены доступа, срок действия которых истекает через год. Если вы хотите настроить более длительный / более короткий срок жизни токена, вы можете использовать методы `tokensExpireIn`, `refreshTokensExpireIn` и `personalAccessTokensExpireIn`. Эти методы следует вызывать из метода `boot` класса `App\Providers\AuthServiceProvider` вашего приложения:

    /**
     * Регистрация сервисов аутентификации и авторизации.
     *
     * @return void
     */
    public function boot()
    {
        $this->registerPolicies();

        Passport::routes();

        Passport::tokensExpireIn(now()->addDays(15));
        Passport::refreshTokensExpireIn(now()->addDays(30));
        Passport::personalAccessTokensExpireIn(now()->addMonths(6));
    }

> {Примечание} Поля `expires_at` в таблицах базы данных Passport доступны только для чтения и только для отображения. При выпуске токенов Passport сохраняет информацию об истечении срока действия в подписанных и зашифрованных токенах. Если вам нужно сделать токен недействительным, вы должны [отозвать его](#revoking-tokens).

<a name="overriding-default-models"></a>
### Переопределение моделей по умолчанию

Вы можете свободно расширять модели, используемые внутри Passport, определяя свою собственную модель и расширяя соответствующую модель Passport:

    use Laravel\Passport\Client as PassportClient;

    class Client extends PassportClient
    {
        // ...
    }

После определения модели вы можете указать Passport использовать вашу пользовательскую модель через класс `Laravel\Passport\Passport`. Как правило, вы должны сообщить Passport о ваших пользовательских моделях в методе `boot` класса `App\Providers\AuthServiceProvider` вашего приложения:

    use App\Models\Passport\AuthCode;
    use App\Models\Passport\Client;
    use App\Models\Passport\PersonalAccessClient;
    use App\Models\Passport\Token;

    /**
     * Регистрация сервисов аутентификации и авторизации.
     *
     * @return void
     */
    public function boot()
    {
        $this->registerPolicies();

        Passport::routes();

        Passport::useTokenModel(Token::class);
        Passport::useClientModel(Client::class);
        Passport::useAuthCodeModel(AuthCode::class);
        Passport::usePersonalAccessClientModel(PersonalAccessClient::class);
    }

<a name="issuing-access-tokens"></a>
## Выдача токенов доступа

Использование OAuth2 через коды авторизации — это то, через что большинство разработчиков знакомится с OAuth2. При использовании кодов авторизации клиентское приложение перенаправит пользователя на ваш сервер, где он либо утвердит, либо отклонит запрос на выдачу токена доступа клиенту.

<a name="managing-clients"></a>
### Управление клиентами

Во-первых, разработчики, которым необходимо взаимодействовать с API вашего приложения, должны будут зарегистрировать свое приложение в вашем, создав «клиента» (client). Обычно это состоит из указания имени своего приложения и URL-адреса, на который ваше приложение может перенаправить после того, как пользователи одобрят свой запрос на авторизацию.

<a name="the-passportclient-command"></a>
#### Команда `passport:client`

Самый простой способ создать клиента — использовать Artisan-команду `passport:client`. Эта команда может использоваться для создания ваших собственных клиентов для тестирования вашей функциональности OAuth2. Когда вы запускаете команду `client`, Passport запросит у вас дополнительную информацию о вашем клиенте и предоставит вам идентификатор клиента и секретный ключ:

    php artisan passport:client

**URL-адреса перенаправления**

Если вы хотите разрешить несколько URL-адресов перенаправления для своего клиента, вы можете указать их, используя список с разделителями-запятыми, когда вам будет предложено ввести URL-адрес командой `passport:client`. Любые URL-адреса, содержащие запятые, должны быть закодированы:

```bash
http://example.com/callback,http://examplefoo.com/callback
```

<a name="clients-json-api"></a>
#### JSON API

Поскольку пользователи вашего приложения не смогут использовать команду `client`, Passport предоставляет JSON API, который вы можете использовать для создания клиентов. Это избавляет вас от необходимости вручную кодировать контроллеры для создания, обновления и удаления клиентов.

Однако вам нужно будет связать JSON API Passport с вашим собственным интерфейсом, чтобы предоставить вашим пользователям панель управления для управления своими клиентами. Ниже мы рассмотрим все конечные точки API для управления клиентами. Для удобства мы будем использовать [Axios](https://github.com/axios/axios), чтобы продемонстрировать выполнение HTTP-запросов к конечным точкам.

JSON API защищен посредниками `web` и `auth`; поэтому его можно вызывать только из вашего собственного приложения. Он не может быть вызван из внешнего источника.

<a name="get-oauthclients"></a>
#### `GET /oauth/clients`

Этот маршрут возвращает всех клиентов для аутентифицированного пользователя. Это в первую очередь полезно для перечисления всех клиентов пользователя, чтобы пользователи могли редактировать или удалять их:

    axios.get('/oauth/clients')
        .then(response => {
            console.log(response.data);
        });

<a name="post-oauthclients"></a>
#### `POST /oauth/clients`

Этот маршрут используется для создания новых клиентов. Для этого требуются два параметра: `name` - имя клиента и `redirect` - URL-адрес перенаправления. URL-адрес `redirect` - это то, куда пользователь будет перенаправлен после утверждения или отклонения запроса на авторизацию.

Когда клиент будет создан, ему будет выдан идентификатор клиента и секретный ключ. Эти значения будут использоваться при запросе токенов доступа из вашего приложения. Маршрут создания клиента вернет новый экземпляр клиента:

    const data = {
        name: 'Client Name',
        redirect: 'http://example.com/callback'
    };

    axios.post('/oauth/clients', data)
        .then(response => {
            console.log(response.data);
        })
        .catch (response => {
            // List errors on response...
        });

<a name="put-oauthclientsclient-id"></a>
#### `PUT /oauth/clients/{client-id}`

Этот маршрут используется для обновления клиентов. Для этого требуются два параметра: `name` - имя клиента и `redirect` - URL-адрес перенаправления. URL-адрес `redirect` - это то, куда пользователь будет перенаправлен после утверждения или отклонения запроса на авторизацию. Маршрут вернет обновленный экземпляр клиента:

    const data = {
        name: 'New Client Name',
        redirect: 'http://example.com/callback'
    };

    axios.put('/oauth/clients/' + clientId, data)
        .then(response => {
            console.log(response.data);
        })
        .catch (response => {
            // List errors on response...
        });

<a name="delete-oauthclientsclient-id"></a>
#### `DELETE /oauth/clients/{client-id}`

Этот маршрут используется для удаления клиентов:

    axios.delete('/oauth/clients/' + clientId)
        .then(response => {
            //
        });

<a name="requesting-tokens"></a>
### Запрос токенов

<a name="requesting-tokens-redirecting-for-authorization"></a>
#### Перенаправление для авторизации

После создания клиента разработчики могут использовать свой идентификатор клиента и секретный ключ, чтобы запросить код авторизации и токен доступа из вашего приложения. Во-первых, приложение-потребитель должно сделать запрос перенаправления на маршрут вашего приложения `/oauth/authorize` следующим образом:

    use Illuminate\Http\Request;
    use Illuminate\Support\Str;

    Route::get('/redirect', function (Request $request) {
        $request->session()->put('state', $state = Str::random(40));

        $query = http_build_query([
            'client_id' => 'client-id',
            'redirect_uri' => 'http://third-party-app.com/callback',
            'response_type' => 'code',
            'scope' => '',
            'state' => $state,
        ]);

        return redirect('http://passport-app.com/oauth/authorize?'.$query);
    });

> {Примечание} Помните, что маршрут `/oauth/authorize` уже определен методом `Passport::routes`. Вам не нужно вручную определять этот маршрут.

<a name="approving-the-request"></a>
#### Подтверждение запроса

При получении запросов на авторизацию Passport автоматически отображает шаблон для пользователя, позволяющий подтвердить или отклонить запрос авторизации. Если они подтвердят запрос, то будут перенаправлены обратно на адрес `redirect_uri`, который был указан приложением-потребителем. Адрес `redirect_uri` должен совпадать с URL-адресом `redirect`, который был указан при создании клиента.

Если вы хотите настроить экран утверждения авторизации, вы можете опубликовать макет Passport с помощью Artisan-команды `vendor: publish`. Опубликованные макеты будут помещены в каталог `resources/views/vendor/passport`:

    php artisan vendor:publish --tag=passport-views

Иногда вам может потребоваться пропустить запрос авторизации, например, при авторизации основного клиента. Вы можете добиться этого, [расширив модель `Client`](#overriding-default-models) и определив метод `skipsAuthorization`. Если `skipsAuthorization` возвращает `true`, клиент будет одобрен, и пользователь будет немедленно перенаправлен обратно в `redirect_uri`:

    <?php

    namespace App\Models\Passport;

    use Laravel\Passport\Client as BaseClient;

    class Client extends BaseClient
    {
        /**
         * Определите, должен ли клиент пропускать запрос авторизации.
         *
         * @return bool
         */
        public function skipsAuthorization()
        {
            return $this->firstParty();
        }
    }

<a name="requesting-tokens-converting-authorization-codes-to-access-tokens"></a>
#### Преобразование кодов авторизации в токены доступа

Если пользователь одобряет запрос авторизации, он будет перенаправлен обратно в приложение-потребитель. Потребитель должен сначала сверить параметр `state` со значением, которое было сохранено до перенаправления. Если параметр `state` совпадает, то потребитель должен отправить вашему приложению запрос `POST`, чтобы запросить токен доступа. Запрос должен включать код авторизации, который был выдан вашим приложением, когда пользователь утвердил запрос авторизации:

    use Illuminate\Http\Request;
    use Illuminate\Support\Facades\Http;

    Route::get('/callback', function (Request $request) {
        $state = $request->session()->pull('state');

        throw_unless(
            strlen($state) > 0 && $state === $request->state,
            InvalidArgumentException::class
        );

        $response = Http::asForm()->post('http://passport-app.com/oauth/token', [
            'grant_type' => 'authorization_code',
            'client_id' => 'client-id',
            'client_secret' => 'client-secret',
            'redirect_uri' => 'http://third-party-app.com/callback',
            'code' => $request->code,
        ]);

        return $response->json();
    });

Маршрут `/oauth/token` вернет ответ JSON, содержащий атрибуты `access_token`, `refresh_token` и `expires_in`. Атрибут `expires_in` содержит количество секунд до истечения срока действия токена доступа.

> {Примечание} Как и маршрут `/oauth/authorize`, маршрут `/oauth/token` определяется для вас методом `Passport::routes`. Нет необходимости определять этот маршрут вручную.

<a name="tokens-json-api"></a>
#### JSON API

Passport также включает JSON API для управления авторизованными токенами доступа. Вы можете связать это со своим собственным интерфейсом, чтобы предложить своим пользователям панель управления для управления токенами доступа. Для удобства мы будем использовать [Axios](https://github.com/mzabriskie/axios), чтобы продемонстрировать выполнение HTTP-запросов к конечным точкам. JSON API защищен посредниками `web` и `auth`; поэтому его можно вызывать только из вашего собственного приложения.

<a name="get-oauthtokens"></a>
#### `GET /oauth/tokens`

Этот маршрут возвращает все токены доступа, созданные аутентифицированным пользователем. Это в первую очередь полезно для просмотра всех токенов пользователя, чтобы он мог их отозвать:

    axios.get('/oauth/tokens')
        .then(response => {
            console.log(response.data);
        });

<a name="delete-oauthtokenstoken-id"></a>
#### `DELETE /oauth/tokens/{token-id}`

Этот маршрут может использоваться для отзыва токенов доступа и связанных с ними токенов обновления:

    axios.delete('/oauth/tokens/' + tokenId);

<a name="refreshing-tokens"></a>
### Обновление токенов

Если ваше приложение выдает недолговечные токены доступа, пользователям потребуется обновить свои токены доступа с помощью токена обновления, предоставленного им при выдаче токена доступа:

    use Illuminate\Support\Facades\Http;

    $response = Http::asForm()->post('http://passport-app.com/oauth/token', [
        'grant_type' => 'refresh_token',
        'refresh_token' => 'the-refresh-token',
        'client_id' => 'client-id',
        'client_secret' => 'client-secret',
        'scope' => '',
    ]);

    return $response->json();

Этот маршрут `/oauth/token` вернет ответ JSON, содержащий атрибуты `access_token`, `refresh_token` и `expires_in`. Атрибут `expires_in` содержит количество секунд до истечения срока действия токена доступа.

<a name="revoking-tokens"></a>
### Отзыв токенов

Вы можете отозвать токен с помощью метода `revokeAccessToken` в `Laravel\Passport\TokenRepository`. Вы можете отозвать токены обновления токена с помощью метода `revokeRefreshTokensByAccessTokenId` в `Laravel\Passport\RefreshTokenRepository`. Эти классы могут быть разрешены с помощью [сервисного контейнера](/docs/{{version}}/container) Laravel:

    use Laravel\Passport\TokenRepository;
    use Laravel\Passport\RefreshTokenRepository;

    $tokenRepository = app(TokenRepository::class);
    $refreshTokenRepository = app(RefreshTokenRepository::class);

    // Revoke an access token...
    $tokenRepository->revokeAccessToken($tokenId);

    // Revoke all of the token's refresh tokens...
    $refreshTokenRepository->revokeRefreshTokensByAccessTokenId($tokenId);

<a name="purging-tokens"></a>
### Удаление токенов

Когда токены были отозваны или срок их действия истек, вы можете удалить их из базы данных. Команда `passport:purge` Artisan, содержащаяся в Passport, может сделать это за вас:

    # Удалить отозванные и просроченные токены, и коды авторизации ...
    php artisan passport:purge

    # Удалить только отозванные токены и коды авторизации ...
    php artisan passport:purge --revoked

    # Удалить только просроченные токены и коды авторизации ...
    php artisan passport:purge --expired

Вы также можете настроить [запланированное задание](/docs/{{version}}/scheduling) в классе вашего приложения `App\Console\Kernel` для автоматического удаления токенов по расписанию:

    /**
     * Определите расписание приложения.
     *
     * @param  \Illuminate\Console\Scheduling\Schedule  $schedule
     * @return void
     */
    protected function schedule(Schedule $schedule)
    {
        $schedule->command('passport:purge')->hourly();
    }

<a name="code-grant-pkce"></a>
## Предоставление кода авторизации с помощью PKCE

Предоставление кода авторизации с `Proof Key for Code Exchange` (PKCE) - это безопасный способ аутентификации одностраничных приложений или собственных приложений для доступа к вашему API. Это разрешение следует использовать, когда вы не можете гарантировать, что секретный ключ клиента будет храниться конфиденциально, или, чтобы уменьшить угрозу перехвата кода авторизации злоумышленником. Комбинация `code verifier` и `code challenge` заменяет секретный ключ клиента при замене кода авторизации на токен доступа.

<a name="creating-a-auth-pkce-grant-client"></a>
### Создание клиента

Прежде чем ваше приложение сможет выдавать токены через предоставление кода авторизации с помощью PKCE, вам необходимо создать клиента с поддержкой PKCE. Вы можете сделать это с помощью Artisan-команды `passport:client` с параметром `--public`:

    php artisan passport:client --public

<a name="requesting-auth-pkce-grant-tokens"></a>
### Запрос токенов

<a name="code-verifier-code-challenge"></a>
#### Code Verifier & Code Challenge

Поскольку это разрешение на авторизацию не предоставляет секретный ключ клиента, разработчикам необходимо сгенерировать комбинацию `code verifier` и `code challenge`, чтобы запросить токен.

Средство проверки кода должно представлять собой случайную строку от 43 до 128 символов, содержащую буквы, цифры и символы `"-"`, `"."`, `"_"`, `"~"`, как определено в [спецификации RFC 7636](https://tools.ietf.org/html/rfc7636).

Итоговым результатом должна быть строка в кодировке Base64 с URL-адресом и безопасными для имени файла символами. Завершающие символы '=' должны быть удалены, и не должно быть разрывов строк, пробелов или других дополнительных символов.

    $encoded = base64_encode(hash('sha256', $code_verifier, true));

    $codeChallenge = strtr(rtrim($encoded, '='), '+/', '-_');

<a name="code-grant-pkce-redirecting-for-authorization"></a>
#### Перенаправление для авторизации

После создания клиента вы можете использовать идентификатор клиента и сгенерированный `code verifier` и `code challenge`, чтобы запросить код авторизации и токен доступа из вашего приложения. Во-первых, приложение-потребитель должно сделать запрос перенаправления на маршрут вашего приложения `/oauth/authorize`:

    use Illuminate\Http\Request;
    use Illuminate\Support\Str;

    Route::get('/redirect', function (Request $request) {
        $request->session()->put('state', $state = Str::random(40));

        $request->session()->put(
            'code_verifier', $code_verifier = Str::random(128)
        );

        $codeChallenge = strtr(rtrim(
            base64_encode(hash('sha256', $code_verifier, true))
        , '='), '+/', '-_');

        $query = http_build_query([
            'client_id' => 'client-id',
            'redirect_uri' => 'http://third-party-app.com/callback',
            'response_type' => 'code',
            'scope' => '',
            'state' => $state,
            'code_challenge' => $codeChallenge,
            'code_challenge_method' => 'S256',
        ]);

        return redirect('http://passport-app.com/oauth/authorize?'.$query);
    });

<a name="code-grant-pkce-converting-authorization-codes-to-access-tokens"></a>
#### Преобразование кодов авторизации в токены доступа

Если пользователь одобряет запрос авторизации, он будет перенаправлен обратно в приложение-потребитель. Потребитель должен сверить параметр `state` со значением, которое было сохранено до перенаправления, как в стандартном предоставлении кода авторизации.

Если параметр состояния совпадает, потребитель должен отправить вашему приложению запрос `POST`, чтобы запросить токен доступа. Запрос должен включать код авторизации, который был выдан вашим приложением, когда пользователь утвердил запрос авторизации, вместе с первоначально сгенерированным верификатором кода:

    use Illuminate\Http\Request;
    use Illuminate\Support\Facades\Http;

    Route::get('/callback', function (Request $request) {
        $state = $request->session()->pull('state');

        $codeVerifier = $request->session()->pull('code_verifier');

        throw_unless(
            strlen($state) > 0 && $state === $request->state,
            InvalidArgumentException::class
        );

        $response = Http::asForm()->post('http://passport-app.com/oauth/token', [
            'grant_type' => 'authorization_code',
            'client_id' => 'client-id',
            'redirect_uri' => 'http://third-party-app.com/callback',
            'code_verifier' => $codeVerifier,
            'code' => $request->code,
        ]);

        return $response->json();
    });

<a name="password-grant-tokens"></a>
## Парольные токены

Предоставление пароля OAuth2 позволяет другим сторонним клиентам, таким как мобильное приложение, получать токен доступа, используя адрес электронной почты / имя пользователя и пароль. Это позволяет вам безопасно выдавать токены доступа своим основным клиентам, не требуя от пользователей прохождения всего потока перенаправления кода авторизации OAuth2.

<a name="creating-a-password-grant-client"></a>
### Создание токенов

Прежде чем ваше приложение сможет выдавать токены с помощью предоставления пароля, вам необходимо создать клиент предоставления пароля. Вы можете сделать это с помощью Artisan-команды `passport:client` с параметром `--password`. **Если вы уже выполнили команду `passport:install`, вам не нужно запускать эту команду:**

    php artisan passport:client --password

<a name="requesting-password-grant-tokens"></a>
### Запрос токенов

После того как вы создали клиента для предоставления пароля, вы можете запросить токен доступа, отправив запрос `POST` по маршруту `/oauth/token` с адресом электронной почты и паролем пользователя. Помните, что этот маршрут уже зарегистрирован методом `Passport::routes`, поэтому нет необходимости определять его вручную. Если запрос будет успешным, вы получите от сервера `access_token` и `refresh_token` в ответе JSON:

    use Illuminate\Support\Facades\Http;

    $response = Http::asForm()->post('http://passport-app.com/oauth/token', [
        'grant_type' => 'password',
        'client_id' => 'client-id',
        'client_secret' => 'client-secret',
        'username' => 'taylor@laravel.com',
        'password' => 'my-password',
        'scope' => '',
    ]);

    return $response->json();

> {Примечание} Помните, токены доступа по умолчанию являются долгоживущими. Однако вы можете [настроить максимальное время жизни токена доступа](#configuration), если это необходимо.

<a name="requesting-all-scopes"></a>
### Запрос токена для всех областей

При использовании доступа по паролю или доступа с учетными данными клиента вы можете авторизовать токен для всех областей, поддерживаемых вашим приложением. Вы можете сделать это, указав `*` в параметре `scope`. При этом метод `can` экземпляра токена всегда будет возвращать `true`. Эта расширенная область может быть назначена только токену, который выпущен с использованием разрешений `password` или `client_credentials`:

    use Illuminate\Support\Facades\Http;

    $response = Http::asForm()->post('http://passport-app.com/oauth/token', [
        'grant_type' => 'password',
        'client_id' => 'client-id',
        'client_secret' => 'client-secret',
        'username' => 'taylor@laravel.com',
        'password' => 'my-password',
        'scope' => '*',
    ]);

<a name="customizing-the-user-provider"></a>
### Настройка пользовательского провайдера

Если ваше приложение использует более одного [провайдера аутентификации пользователя](/docs/{{version}}/authentication#introduction), вы можете указать, какой провайдер использует клиент предоставления пароля, указав параметр `--provider` при создании клиента через команду `artisan passport:client --password`. Указанное имя провайдера должно соответствовать допустимому провайдеру, определенному в файле конфигурации приложения `config/auth.php`. Затем вы можете [защитить свой маршрут с помощью посредника](#via-middleware), чтобы гарантировать, что авторизованы только пользователи из указанного провайдера.

<a name="customizing-the-username-field"></a>
### Настройка поля имени пользователя

При аутентификации с использованием предоставления пароля Passport будет использовать атрибут `email` вашей аутентифицируемой модели в качестве "username". Однако вы можете настроить это поведение, определив метод `findForPassport` в своей модели:

    <?php

    namespace App\Models;

    use Illuminate\Foundation\Auth\User as Authenticatable;
    use Illuminate\Notifications\Notifiable;
    use Laravel\Passport\HasApiTokens;

    class User extends Authenticatable
    {
        use HasApiTokens, Notifiable;

        /**
         * Возвращает экземпляр пользователя для переданного имени.
         *
         * @param  string  $username
         * @return \App\Models\User
         */
        public function findForPassport($username)
        {
            return $this->where('username', $username)->first();
        }
    }

<a name="customizing-the-password-validation"></a>
### Настройка проверки пароля пользователя

При аутентификации с использованием предоставления пароля Passport будет использовать атрибут `password` модели для проверки пароля. Если модель не имеет атрибута `password` или вы хотите настроить логику проверки пароля, вы можете определить метод `validateForPassportPasswordGrant` в своей модели:

    <?php

    namespace App\Models;

    use Illuminate\Foundation\Auth\User as Authenticatable;
    use Illuminate\Notifications\Notifiable;
    use Illuminate\Support\Facades\Hash;
    use Laravel\Passport\HasApiTokens;

    class User extends Authenticatable
    {
        use HasApiTokens, Notifiable;

        /**
         * Проверьте пароль пользователя для предоставления разрешения.
         *
         * @param  string  $password
         * @return bool
         */
        public function validateForPassportPasswordGrant($password)
        {
            return Hash::check($password, $this->password);
        }
    }

<a name="implicit-grant-tokens"></a>
## Неявные токены

Неявное разрешение аналогично предоставлению кода авторизации; однако токен возвращается клиенту без обмена кодом авторизации. Этот разрешение чаще всего используется для JavaScript или мобильных приложений, где учетные данные клиента не могут быть надежно сохранены. Чтобы включить разрешение, вызовите метод `enableImplicitGrant` в методе `boot` класса `App\Providers\AuthServiceProvider` вашего приложения:

    /**
     * Регистрация сервисов аутентификации и авторизации.
     *
     * @return void
     */
    public function boot()
    {
        $this->registerPolicies();

        Passport::routes();

        Passport::enableImplicitGrant();
    }

Once the grant has been enabled, developers may use their client ID to request an access token from your application. The consuming application should make a redirect request to your application's `/oauth/authorize` route like so:

    use Illuminate\Http\Request;

    Route::get('/redirect', function (Request $request) {
        $request->session()->put('state', $state = Str::random(40));

        $query = http_build_query([
            'client_id' => 'client-id',
            'redirect_uri' => 'http://third-party-app.com/callback',
            'response_type' => 'token',
            'scope' => '',
            'state' => $state,
        ]);

        return redirect('http://passport-app.com/oauth/authorize?'.$query);
    });

> {tip} Remember, the `/oauth/authorize` route is already defined by the `Passport::routes` method. You do not need to manually define this route.

<a name="client-credentials-grant-tokens"></a>
## Client Credentials Grant Tokens

The client credentials grant is suitable for machine-to-machine authentication. For example, you might use this grant in a scheduled job which is performing maintenance tasks over an API.

Before your application can issue tokens via the client credentials grant, you will need to create a client credentials grant client. You may do this using the `--client` option of the `passport:client` Artisan command:

    php artisan passport:client --client

Next, to use this grant type, you need to add the `CheckClientCredentials` middleware to the `$routeMiddleware` property of your `app/Http/Kernel.php` file:

    use Laravel\Passport\Http\Middleware\CheckClientCredentials;

    protected $routeMiddleware = [
        'client' => CheckClientCredentials::class,
    ];

Then, attach the middleware to a route:

    Route::get('/orders', function (Request $request) {
        ...
    })->middleware('client');

To restrict access to the route to specific scopes, you may provide a comma-delimited list of the required scopes when attaching the `client` middleware to the route:

    Route::get('/orders', function (Request $request) {
        ...
    })->middleware('client:check-status,your-scope');

<a name="retrieving-tokens"></a>
### Retrieving Tokens

To retrieve a token using this grant type, make a request to the `oauth/token` endpoint:

    use Illuminate\Support\Facades\Http;

    $response = Http::asForm()->post('http://passport-app.com/oauth/token', [
        'grant_type' => 'client_credentials',
        'client_id' => 'client-id',
        'client_secret' => 'client-secret',
        'scope' => 'your-scope',
    ]);

    return $response->json()['access_token'];

<a name="personal-access-tokens"></a>
## Personal Access Tokens

Sometimes, your users may want to issue access tokens to themselves without going through the typical authorization code redirect flow. Allowing users to issue tokens to themselves via your application's UI can be useful for allowing users to experiment with your API or may serve as a simpler approach to issuing access tokens in general.

<a name="creating-a-personal-access-client"></a>
### Creating A Personal Access Client

Before your application can issue personal access tokens, you will need to create a personal access client. You may do this by executing the `passport:client` Artisan command with the `--personal` option. If you have already run the `passport:install` command, you do not need to run this command:

    php artisan passport:client --personal

After creating your personal access client, place the client's ID and plain-text secret value in your application's `.env` file:

```bash
PASSPORT_PERSONAL_ACCESS_CLIENT_ID="client-id-value"
PASSPORT_PERSONAL_ACCESS_CLIENT_SECRET="unhashed-client-secret-value"
```

<a name="managing-personal-access-tokens"></a>
### Managing Personal Access Tokens

Once you have created a personal access client, you may issue tokens for a given user using the `createToken` method on the `App\Models\User` model instance. The `createToken` method accepts the name of the token as its first argument and an optional array of [scopes](#token-scopes) as its second argument:

    use App\Models\User;

    $user = User::find(1);

    // Creating a token without scopes...
    $token = $user->createToken('Token Name')->accessToken;

    // Creating a token with scopes...
    $token = $user->createToken('My Token', ['place-orders'])->accessToken;

<a name="personal-access-tokens-json-api"></a>
#### JSON API

Passport also includes a JSON API for managing personal access tokens. You may pair this with your own frontend to offer your users a dashboard for managing personal access tokens. Below, we'll review all of the API endpoints for managing personal access tokens. For convenience, we'll use [Axios](https://github.com/mzabriskie/axios) to demonstrate making HTTP requests to the endpoints.

The JSON API is guarded by the `web` and `auth` middleware; therefore, it may only be called from your own application. It is not able to be called from an external source.

<a name="get-oauthscopes"></a>
#### `GET /oauth/scopes`

This route returns all of the [scopes](#token-scopes) defined for your application. You may use this route to list the scopes a user may assign to a personal access token:

    axios.get('/oauth/scopes')
        .then(response => {
            console.log(response.data);
        });

<a name="get-oauthpersonal-access-tokens"></a>
#### `GET /oauth/personal-access-tokens`

This route returns all of the personal access tokens that the authenticated user has created. This is primarily useful for listing all of the user's tokens so that they may edit or revoke them:

    axios.get('/oauth/personal-access-tokens')
        .then(response => {
            console.log(response.data);
        });

<a name="post-oauthpersonal-access-tokens"></a>
#### `POST /oauth/personal-access-tokens`

This route creates new personal access tokens. It requires two pieces of data: the token's `name` and the `scopes` that should be assigned to the token:

    const data = {
        name: 'Token Name',
        scopes: []
    };

    axios.post('/oauth/personal-access-tokens', data)
        .then(response => {
            console.log(response.data.accessToken);
        })
        .catch (response => {
            // List errors on response...
        });

<a name="delete-oauthpersonal-access-tokenstoken-id"></a>
#### `DELETE /oauth/personal-access-tokens/{token-id}`

This route may be used to revoke personal access tokens:

    axios.delete('/oauth/personal-access-tokens/' + tokenId);

<a name="protecting-routes"></a>
## Protecting Routes

<a name="via-middleware"></a>
### Via Middleware

Passport includes an [authentication guard](/docs/{{version}}/authentication#adding-custom-guards) that will validate access tokens on incoming requests. Once you have configured the `api` guard to use the `passport` driver, you only need to specify the `auth:api` middleware on any routes that should require a valid access token:

    Route::get('/user', function () {
        //
    })->middleware('auth:api');

<a name="multiple-authentication-guards"></a>
#### Multiple Authentication Guards

If your application authenticates different types of users that perhaps use entirely different Eloquent models, you will likely need to define a guard configuration for each user provider type in your application. This allows you to protect requests intended for specific user providers. For example, given the following guard configuration the `config/auth.php` configuration file:

    'api' => [
        'driver' => 'passport',
        'provider' => 'users',
    ],

    'api-customers' => [
        'driver' => 'passport',
        'provider' => 'customers',
    ],

The following route will utilize the `api-customers` guard, which uses the `customers` user provider, to authenticate incoming requests:

    Route::get('/customer', function () {
        //
    })->middleware('auth:api-customers');

> {tip} For more information on using multiple user providers with Passport, please consult the [password grant documentation](#customizing-the-user-provider).

<a name="passing-the-access-token"></a>
### Passing The Access Token

When calling routes that are protected by Passport, your application's API consumers should specify their access token as a `Bearer` token in the `Authorization` header of their request. For example, when using the Guzzle HTTP library:

    use Illuminate\Support\Facades\Http;

    $response = Http::withHeaders([
        'Accept' => 'application/json',
        'Authorization' => 'Bearer '.$accessToken,
    ])->get('https://passport-app.com/api/user');

    return $response->json();

<a name="token-scopes"></a>
## Token Scopes

Scopes allow your API clients to request a specific set of permissions when requesting authorization to access an account. For example, if you are building an e-commerce application, not all API consumers will need the ability to place orders. Instead, you may allow the consumers to only request authorization to access order shipment statuses. In other words, scopes allow your application's users to limit the actions a third-party application can perform on their behalf.

<a name="defining-scopes"></a>
### Defining Scopes

You may define your API's scopes using the `Passport::tokensCan` method in the `boot` method of your application's `App\Providers\AuthServiceProvider` class. The `tokensCan` method accepts an array of scope names and scope descriptions. The scope description may be anything you wish and will be displayed to users on the authorization approval screen:

    /**
     * Регистрация сервисов аутентификации и авторизации.
     *
     * @return void
     */
    public function boot()
    {
        $this->registerPolicies();

        Passport::routes();

        Passport::tokensCan([
            'place-orders' => 'Place orders',
            'check-status' => 'Check order status',
        ]);
    }

<a name="default-scope"></a>
### Default Scope

If a client does not request any specific scopes, you may configure your Passport server to attach default scope(s) to the token using the `setDefaultScope` method. Typically, you should call this method from the `boot` method of your application's `App\Providers\AuthServiceProvider` class:

    use Laravel\Passport\Passport;

    Passport::tokensCan([
        'place-orders' => 'Place orders',
        'check-status' => 'Check order status',
    ]);

    Passport::setDefaultScope([
        'check-status',
        'place-orders',
    ]);

<a name="assigning-scopes-to-tokens"></a>
### Assigning Scopes To Tokens

<a name="when-requesting-authorization-codes"></a>
#### When Requesting Authorization Codes

When requesting an access token using the authorization code grant, consumers should specify their desired scopes as the `scope` query string parameter. The `scope` parameter should be a space-delimited list of scopes:

    Route::get('/redirect', function () {
        $query = http_build_query([
            'client_id' => 'client-id',
            'redirect_uri' => 'http://example.com/callback',
            'response_type' => 'code',
            'scope' => 'place-orders check-status',
        ]);

        return redirect('http://passport-app.com/oauth/authorize?'.$query);
    });

<a name="when-issuing-personal-access-tokens"></a>
#### When Issuing Personal Access Tokens

If you are issuing personal access tokens using the `App\Models\User` model's `createToken` method, you may pass the array of desired scopes as the second argument to the method:

    $token = $user->createToken('My Token', ['place-orders'])->accessToken;

<a name="checking-scopes"></a>
### Checking Scopes

Passport includes two middleware that may be used to verify that an incoming request is authenticated with a token that has been granted a given scope. To get started, add the following middleware to the `$routeMiddleware` property of your `app/Http/Kernel.php` file:

    'scopes' => \Laravel\Passport\Http\Middleware\CheckScopes::class,
    'scope' => \Laravel\Passport\Http\Middleware\CheckForAnyScope::class,

<a name="check-for-all-scopes"></a>
#### Check For All Scopes

The `scopes` middleware may be assigned to a route to verify that the incoming request's access token has all of the listed scopes:

    Route::get('/orders', function () {
        // Access token has both "check-status" and "place-orders" scopes...
    })->middleware(['auth:api', 'scopes:check-status,place-orders']);

<a name="check-for-any-scopes"></a>
#### Check For Any Scopes

The `scope` middleware may be assigned to a route to verify that the incoming request's access token has *at least one* of the listed scopes:

    Route::get('/orders', function () {
        // Access token has either "check-status" or "place-orders" scope...
    })->middleware(['auth:api', 'scope:check-status,place-orders']);

<a name="checking-scopes-on-a-token-instance"></a>
#### Checking Scopes On A Token Instance

Once an access token authenticated request has entered your application, you may still check if the token has a given scope using the `tokenCan` method on the authenticated `App\Models\User` instance:

    use Illuminate\Http\Request;

    Route::get('/orders', function (Request $request) {
        if ($request->user()->tokenCan('place-orders')) {
            //
        }
    });

<a name="additional-scope-methods"></a>
#### Additional Scope Methods

The `scopeIds` method will return an array of all defined IDs / names:

    use Laravel\Passport\Passport;

    Passport::scopeIds();

The `scopes` method will return an array of all defined scopes as instances of `Laravel\Passport\Scope`:

    Passport::scopes();

The `scopesFor` method will return an array of `Laravel\Passport\Scope` instances matching the given IDs / names:

    Passport::scopesFor(['place-orders', 'check-status']);

You may determine if a given scope has been defined using the `hasScope` method:

    Passport::hasScope('place-orders');

<a name="consuming-your-api-with-javascript"></a>
## Consuming Your API With JavaScript

When building an API, it can be extremely useful to be able to consume your own API from your JavaScript application. This approach to API development allows your own application to consume the same API that you are sharing with the world. The same API may be consumed by your web application, mobile applications, third-party applications, and any SDKs that you may publish on various package managers.

Typically, if you want to consume your API from your JavaScript application, you would need to manually send an access token to the application and pass it with each request to your application. However, Passport includes a middleware that can handle this for you. All you need to do is add the `CreateFreshApiToken` middleware to your `web` middleware group in your `app/Http/Kernel.php` file:

    'web' => [
        // Other middleware...
        \Laravel\Passport\Http\Middleware\CreateFreshApiToken::class,
    ],

> {note} You should ensure that the `CreateFreshApiToken` middleware is the last middleware listed in your middleware stack.

This middleware will attach a `laravel_token` cookie to your outgoing responses. This cookie contains an encrypted JWT that Passport will use to authenticate API requests from your JavaScript application. The JWT has a lifetime equal to your `session.lifetime` configuration value. Now, since the browser will automatically send the cookie with all subsequent requests, you may make requests to your application's API without explicitly passing an access token:

    axios.get('/api/user')
        .then(response => {
            console.log(response.data);
        });

<a name="customizing-the-cookie-name"></a>
#### Customizing The Cookie Name

If needed, you can customize the `laravel_token` cookie's name using the `Passport::cookie` method. Typically, this method should be called from the `boot` method of your application's `App\Providers\AuthServiceProvider` class:

    /**
     * Регистрация сервисов аутентификации и авторизации.
     *
     * @return void
     */
    public function boot()
    {
        $this->registerPolicies();

        Passport::routes();

        Passport::cookie('custom_name');
    }

<a name="csrf-protection"></a>
#### CSRF Protection

When using this method of authentication, you will need to ensure a valid CSRF token header is included in your requests. The default Laravel JavaScript scaffolding includes an Axios instance, which will automatically use the encrypted `XSRF-TOKEN` cookie value to send an `X-XSRF-TOKEN` header on same-origin requests.

> {tip} If you choose to send the `X-CSRF-TOKEN` header instead of `X-XSRF-TOKEN`, you will need to use the unencrypted token provided by `csrf_token()`.

<a name="events"></a>
## Events

Passport raises events when issuing access tokens and refresh tokens. You may use these events to prune or revoke other access tokens in your database. If you would like, you may attach listeners to these events in your application's `App\Providers\EventServiceProvider` class:

    /**
     * The event listener mappings for the application.
     *
     * @var array
     */
    protected $listen = [
        'Laravel\Passport\Events\AccessTokenCreated' => [
            'App\Listeners\RevokeOldTokens',
        ],

        'Laravel\Passport\Events\RefreshTokenCreated' => [
            'App\Listeners\PruneOldTokens',
        ],
    ];

<a name="testing"></a>
## Testing

Passport's `actingAs` method may be used to specify the currently authenticated user as well as its scopes. The first argument given to the `actingAs` method is the user instance and the second is an array of scopes that should be granted to the user's token:

    use App\Models\User;
    use Laravel\Passport\Passport;

    public function test_servers_can_be_created()
    {
        Passport::actingAs(
            User::factory()->create(),
            ['create-servers']
        );

        $response = $this->post('/api/create-server');

        $response->assertStatus(201);
    }

Passport's `actingAsClient` method may be used to specify the currently authenticated client as well as its scopes. The first argument given to the `actingAsClient` method is the client instance and the second is an array of scopes that should be granted to the client's token:

    use Laravel\Passport\Client;
    use Laravel\Passport\Passport;

    public function test_orders_can_be_retrieved()
    {
        Passport::actingAsClient(
            Client::factory()->create(),
            ['check-status']
        );

        $response = $this->get('/api/orders');

        $response->assertStatus(200);
    }
