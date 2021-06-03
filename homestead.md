git 85466c3e68c6a575dc6cb7851e8f044f6c54d733

---

# Laravel Homestead

- [Введение](#introduction)
- [Установка и настройка](#installation-and-setup)
    - [Первые шаги](#first-steps)
    - [Настройка Homestead](#configuring-homestead)
    - [Настройка Nginx](#configuring-nginx-sites)
    - [Настройка сервисов](#configuring-services)
    - [Запуск Vagrant Box](#launching-the-vagrant-box)
    - [Подготовка к установке](#per-project-installation)
    - [Установка дополнительных пакетов](#installing-optional-features)
    - [Алиасы](#aliases)
- [Обновление Homestead](#updating-homestead)
- [Ежедневное использование](#daily-usage)
    - [Подключение через SSH](#connecting-via-ssh)
    - [Добавление сайтов](#adding-additional-sites)
    - [Настройка окружения](#environment-variables)
    - [Порты](#ports)
    - [Версии PHP](#php-versions)
    - [Соединение с базой данных](#connecting-to-databases)
    - [Резервные копии базы данных](#database-backups)
    - [Снимки базы данных](#database-snapshots)
    - [Настройка расписания Cron](#configuring-cron-schedules)
    - [Настройка MailHog](#configuring-mailhog)
    - [Настройка Minio](#configuring-minio)
    - [Laravel Dusk](#laravel-dusk)
    - [Совместное использование](#sharing-your-environment)
- [Отладка и профилирование](#debugging-and-profiling)
    - [Отладка запросов через Xdebug](#debugging-web-requests)
    - [Отладка через CLI](#debugging-cli-applications)
    - [Профилирование приложения через Blackfire](#profiling-applications-with-blackfire)
- [Сетевые интерфейсы](#network-interfaces)
- [Продление Homestead](#extending-homestead)
- [Специфичные настройки](#provider-specific-settings)
    - [VirtualBox](#provider-specific-virtualbox)

<a name="introduction"></a>
## Введение

Laravel стремится сделать весь процесс разработки PHP приятным, включая вашу локальную среду разработки. Laravel Homestead - это официальный предварительно упакованный пакет Vagrant, который предоставляет вам прекрасную среду разработки, не требуя установки PHP, веб-сервера и любого другого серверного программного обеспечения на вашем локальном компьютере.

[Vagrant](https://www.vagrantup.com) предоставляет простой и элегантный способ управления виртуальными машинами и их подготовки. Vagrant-контейнеры полностью одноразовые. Если что-то пойдет не так, вы можете уничтожить и воссоздать контейнер за считанные минуты!

Homestead работает в любой системе Windows, macOS или Linux и включает Nginx, PHP, MySQL, PostgreSQL, Redis, Memcached, Node и все другое программное обеспечение, необходимое для разработки потрясающих приложений Laravel.

> {Примечание} Если вы используете Windows, вам может потребоваться включить аппаратную виртуализацию (VT-x). Обычно его можно включить в BIOS. Если вы используете Hyper-V в системе UEFI, вам может дополнительно потребоваться отключить Hyper-V, чтобы получить доступ к VT-x.

<a name="included-software"></a>
### Включенное в набор программное обеспечение

<style>
    #software-list > ul {
        column-count: 2; -moz-column-count: 2; -webkit-column-count: 2;
        column-gap: 5em; -moz-column-gap: 5em; -webkit-column-gap: 5em;
        line-height: 1.9;
    }
</style>

<div id="software-list" markdown="1">
- Ubuntu 20.04
- Git
- PHP 8.0
- PHP 7.4
- PHP 7.3
- PHP 7.2
- PHP 7.1
- PHP 7.0
- PHP 5.6
- Nginx
- MySQL (8.0)
- lmm
- Sqlite3
- PostgreSQL (9.6, 10, 11, 12, 13)
- Composer
- Node (With Yarn, Bower, Grunt, and Gulp)
- Redis
- Memcached
- Beanstalkd
- Mailhog
- avahi
- ngrok
- Xdebug
- XHProf / Tideways / XHGui
- wp-cli
</div>

<a name="optional-software"></a>
### Дополнительное программное обеспечение

<style>
    #software-list > ul {
        column-count: 2; -moz-column-count: 2; -webkit-column-count: 2;
        column-gap: 5em; -moz-column-gap: 5em; -webkit-column-gap: 5em;
        line-height: 1.9;
    }
</style>

<div id="software-list" markdown="1">
- Apache
- Blackfire
- Cassandra
- Chronograf
- CouchDB
- Crystal & Lucky Framework
- Docker
- Elasticsearch
- EventStoreDB
- Gearman
- Go
- Grafana
- InfluxDB
- MariaDB
- Meilisearch
- MinIO
- MongoDB
- Neo4j
- Oh My Zsh
- Open Resty
- PM2
- Python
- R
- RabbitMQ
- RVM (Ruby Version Manager)
- Solr
- TimescaleDB
- Trader <small>(PHP extension)</small>
- Webdriver & Laravel Dusk Utilities
</div>

<a name="installation-and-setup"></a>
## Установка и настройка

<a name="first-steps"></a>
### Первые шаги

Перед запуском среды Homestead необходимо установить [Vagrant](https://www.vagrantup.com/downloads.html), а также одного из следующих поддерживаемых провайдеров:

- [VirtualBox 6.1.x](https://www.virtualbox.org/wiki/Downloads)
- [Parallels](https://www.parallels.com/products/desktop/)

Все эти программные пакеты предоставляют простые в использовании визуальные установщики для всех популярных операционных систем.

Чтобы использовать провайдер Parallels, вам необходимо установить бесплатный плагин [Parallels Vagrant](https://github.com/Parallels/vagrant-parallels).

<a name="installing-homestead"></a>
#### Установка Homestead

Вы можете установить Homestead, клонировав репозиторий Homestead на свой компьютер. Рассмотрите возможность клонирования репозитория в папку `Homestead` в вашем домашнем каталоге, поскольку виртуальная машина Homestead будет служить хостом для всех ваших приложений Laravel. В этой документации мы будем называть этот каталог - «каталогом Homestead»:

```bash
git clone https://github.com/laravel/homestead.git ~/Homestead
```

После клонирования репозитория Laravel Homestead вы должны проверить ветку `release`. Эта ветка всегда содержит последний стабильный выпуск Homestead:

    cd ~/Homestead

    git checkout release

Затем выполните команду `bash init.sh` из каталога Homestead, чтобы создать файл конфигурации `Homestead.yaml`. Файл `Homestead.yaml` - это то место, где вы настраиваете все параметры установки Homestead. Этот файл будет помещен в каталог Homestead:

    // macOS / Linux...
    bash init.sh

    // Windows...
    init.bat

<a name="configuring-homestead"></a>
### Настройка Homestead

<a name="setting-your-provider"></a>
#### Настройка провайдера

Ключ `provider` в файле `Homestead.yaml` указывает, какой провайдер Vagrant следует использовать: `virtualbox` или `parallels`:

    provider: virtualbox

<a name="configuring-shared-folders"></a>
#### Настройка общих папок

Параметр `folder` файла `Homestead.yaml` перечисляет все директории, которыми вы хотите поделиться со своей виртуальной средой Homestead. При изменении файлов в этих папках они будут синхронизироваться между вашим локальным компьютером и средой Homestead. Вы можете настроить столько общих директорий, сколько необходимо:

```yaml
folders:
    - map: ~/code/project1
      to: /home/vagrant/project1
```

> {Примечание} Пользователи Windows, при указании пути не должны использовать синтаксис `~/`, а вместо этого должны указать полный путь к своему проекту от корня диска, например `C:\Users\user\Code\project1`.

Вы всегда должны сопоставлять каждое ваше приложение с его собственной отдельной директорией вместо назначения одного большого каталога, содержащего все ваши приложения. При назначении папки приложению виртуальная машина должна отслеживать все операции ввода-вывода на диске для *каждого* файла в папке. Поэтому у вас может снизиться производительность среды, если в папке много файлов:

```yaml
folders:
    - map: ~/code/project1
      to: /home/vagrant/project1
    - map: ~/code/project2
      to: /home/vagrant/project2
```

> {Примечание} Вы никогда не должны монтировать `.` (текущий каталог) при использовании Homestead. Это приводит к тому, что Vagrant не отображает текущую папку в `/vagrant`, что нарушает работу дополнительных функций и приводит к неожиданным результатам при подготовке.

Чтобы включить [NFS](https://www.vagrantup.com/docs/synced-folders/nfs.html), вы можете добавить параметр `type` при сопоставлении папок:

    folders:
        - map: ~/code/project1
          to: /home/vagrant/project1
          type: "nfs"

> {Примечание} При использовании NFS в Windows вам следует рассмотреть возможность установки подключаемого модуля [vagrant-winnfsd](https://github.com/winnfsd/vagrant-winnfsd). Этот плагин будет поддерживать правильные разрешения пользователя / группы для файлов и каталогов на виртуальной машине Homestead.

Вы также можете передать любые параметры, поддерживаемые [общими папками Vagrant](https://www.vagrantup.com/docs/synced-folders/basic_usage.html), указав их под ключом options:

    folders:
        - map: ~/code/project1
          to: /home/vagrant/project1
          type: "rsync"
          options:
              rsync__args: ["--verbose", "--archive", "--delete", "-zz"]
              rsync__exclude: ["node_modules"]

<a name="configuring-nginx-sites"></a>
### Настройка Nginx

Не знаком с Nginx? Нет проблем! Свойство `sites` файла `Homestead.yaml` позволяет легко сопоставить "домен" с папкой в среде Homestead. Пример конфигурации сайта включен в файл `Homestead.yaml`. Опять же, вы можете добавить столько сайтов в среду Homestead, сколько необходимо. Homestead может служить удобной виртуальной средой для каждого приложения Laravel, над которым вы работаете:

    sites:
        - map: homestead.test
          to: /home/vagrant/project1/public

Если вы измените свойство `sites` после подготовки виртуальной машины Homestead, вы должны выполнить команду `vagrant reload --provision` в своем терминале, чтобы обновить конфигурацию Nginx на виртуальной машине.

> {Примечание} Скрипты Homestead созданы максимально [идемпотентными](https://ru.wikipedia.org/wiki/Идемпотентность). Однако, если у вас возникли проблемы во время подготовки, вам следует удалить и повторно запустить виртуальную машину, выполнив команду `vagrant destroy && vagrant up`.

<a name="hostname-resolution"></a>
#### Определение имени хоста

Homestead публикует имена хостов, используя `mDNS` для автоматического определения хостов. Если вы установите `hostname: homestead` в вашем файле `Homestead.yaml`, хост будет доступен по адресу `homestead.local`. Настольные дистрибутивы macOS, iOS и Linux по умолчанию включают поддержку `mDNS`. Если вы используете Windows, вы должны установить [Bonjour Print Services для Windows](https://support.apple.com/kb/DL999?viewlocale=en_US&locale=en_US).

Настройку имен хостов лучше всего проводить при [подготовке к установке](#per-project-installation) Homestead. Если вы размещаете несколько сайтов на одном экземпляре Homestead, вы можете добавить домены для своих веб-сайтов в файл `hosts` на вашем компьютере. Файл `hosts` будет перенаправлять запросы для ваших сайтов Homestead на вашу виртуальную машину Homestead. В macOS и Linux этот файл находится в `/etc/hosts`. В Windows он находится в `C:\Windows\System32\drivers\etc\hosts`. Строки, которые вы добавляете в этот файл, будут выглядеть следующим образом:

    192.168.10.10  homestead.test

Убедитесь, что в списке указан IP-адрес, указанный в вашем файле `Homestead.yaml`. После того как вы добавили домен в файл `hosts` и запустили Vagrant-контейнер, вы сможете получить доступ к сайту через свой веб-браузер:

```bash
http://homestead.test
```

<a name="configuring-services"></a>
### Настройка сервисов

Homestead starts several services by default; however, you may customize which services are enabled or disabled during provisioning. For example, you may enable PostgreSQL and disable MySQL by modifying the `services` option within your `Homestead.yaml` file:

```yaml
services:
    - enabled:
        - "postgresql@12-main"
    - disabled:
        - "mysql"
```

The specified services will be started or stopped based on their order in the `enabled` and `disabled` directives.

<a name="launching-the-vagrant-box"></a>
### Launching The Vagrant Box

Once you have edited the `Homestead.yaml` to your liking, run the `vagrant up` command from your Homestead directory. Vagrant will boot the virtual machine and automatically configure your shared folders and Nginx sites.

To destroy the machine, you may use the `vagrant destroy` command.

<a name="per-project-installation"></a>
### Per Project Installation

Instead of installing Homestead globally and sharing the same Homestead virtual machine across all of your projects, you may instead configure a Homestead instance for each project you manage. Installing Homestead per project may be beneficial if you wish to ship a `Vagrantfile` with your project, allowing others working on the project to `vagrant up` immediately after cloning the project's repository.

You may install Homestead into your project using the Composer package manager:

```bash
composer require laravel/homestead --dev
```

Once Homestead has been installed, invoke Homestead's `make` command to generate the `Vagrantfile` and `Homestead.yaml` file for your project. These files will be placed in the root of your project. The `make` command will automatically configure the `sites` and `folders` directives in the `Homestead.yaml` file:

    // macOS / Linux...
    php vendor/bin/homestead make

    // Windows...
    vendor\\bin\\homestead make

Next, run the `vagrant up` command in your terminal and access your project at `http://homestead.test` in your browser. Remember, you will still need to add an `/etc/hosts` file entry for `homestead.test` or the domain of your choice if you are not using automatic [hostname resolution](#hostname-resolution).

<a name="installing-optional-features"></a>
### Installing Optional Features

Optional software is installed using the `features` option within your `Homestead.yaml` file. Most features can be enabled or disabled with a boolean value, while some features allow multiple configuration options:

    features:
        - blackfire:
            server_id: "server_id"
            server_token: "server_value"
            client_id: "client_id"
            client_token: "client_value"
        - cassandra: true
        - chronograf: true
        - couchdb: true
        - crystal: true
        - docker: true
        - elasticsearch:
            version: 7.9.0
        - eventstore: true
            version: 21.2.0
        - gearman: true
        - golang: true
        - grafana: true
        - influxdb: true
        - mariadb: true
        - meilisearch: true
        - minio: true
        - mongodb: true
        - neo4j: true
        - ohmyzsh: true
        - openresty: true
        - pm2: true
        - python: true
        - r-base: true
        - rabbitmq: true
        - rvm: true
        - solr: true
        - timescaledb: true
        - trader: true
        - webdriver: true

<a name="elasticsearch"></a>
#### Elasticsearch

You may specify a supported version of Elasticsearch, which must be an exact version number (major.minor.patch). The default installation will create a cluster named 'homestead'. You should never give Elasticsearch more than half of the operating system's memory, so make sure your Homestead virtual machine has at least twice the Elasticsearch allocation.

> {tip} Check out the [Elasticsearch documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current) to learn how to customize your configuration.

<a name="mariadb"></a>
#### MariaDB

Enabling MariaDB will remove MySQL and install MariaDB. MariaDB typically serves as a drop-in replacement for MySQL, so you should still use the `mysql` database driver in your application's database configuration.

<a name="mongodb"></a>
#### MongoDB

The default MongoDB installation will set the database username to `homestead` and the corresponding password to `secret`.

<a name="neo4j"></a>
#### Neo4j

The default Neo4j installation will set the database username to `homestead` and the corresponding password to `secret`. To access the Neo4j browser, visit `http://homestead.test:7474` via your web browser. The ports `7687` (Bolt), `7474` (HTTP), and `7473` (HTTPS) are ready to serve requests from the Neo4j client.

<a name="aliases"></a>
### Aliases

You may add Bash aliases to your Homestead virtual machine by modifying the `aliases` file within your Homestead directory:

    alias c='clear'
    alias ..='cd ..'

After you have updated the `aliases` file, you should re-provision the Homestead virtual machine using the `vagrant reload --provision` command. This will ensure that your new aliases are available on the machine.

<a name="updating-homestead"></a>
## Updating Homestead

Before you begin updating Homestead you should ensure you have removed your current virtual machine by running the following command in your Homestead directory:

    vagrant destroy

Next, you need to update the Homestead source code. If you cloned the repository, you can execute the following commands at the location you originally cloned the repository:

    git fetch

    git pull origin release

These commands pull the latest Homestead code from the GitHub repository, fetch the latest tags, and then check out the latest tagged release. You can find the latest stable release version on Homestead's [GitHub releases page](https://github.com/laravel/homestead/releases).

If you have installed Homestead via your project's `composer.json` file, you should ensure your `composer.json` file contains `"laravel/homestead": "^12"` and update your dependencies:

    composer update

Next, you should update the Vagrant box using the `vagrant box update` command:

    vagrant box update

After updating the Vagrant box, you should run the `bash init.sh` command from the Homestead directory in order to update Homestead's additional configuration files. You will be asked whether you wish to overwrite your existing `Homestead.yaml`, `after.sh`, and `aliases` files:

    // macOS / Linux...
    bash init.sh

    // Windows...
    init.bat

Finally, you will need to regenerate your Homestead virtual machine to utilize the latest Vagrant installation:

    vagrant up

<a name="daily-usage"></a>
## Daily Usage

<a name="connecting-via-ssh"></a>
### Connecting Via SSH

You can SSH into your virtual machine by executing the `vagrant ssh` terminal command from your Homestead directory.

<a name="adding-additional-sites"></a>
### Adding Additional Sites

Once your Homestead environment is provisioned and running, you may want to add additional Nginx sites for your other Laravel projects. You can run as many Laravel projects as you wish on a single Homestead environment. To add an additional site, add the site to your `Homestead.yaml` file.

    sites:
        - map: homestead.test
          to: /home/vagrant/project1/public
        - map: another.test
          to: /home/vagrant/project2/public

> {note} You should ensure that you have configured a [folder mapping](#configuring-shared-folders) for the project's directory before adding the site.

If Vagrant is not automatically managing your "hosts" file, you may need to add the new site to that file as well. On macOS and Linux, this file is located at `/etc/hosts`. On Windows, it is located at `C:\Windows\System32\drivers\etc\hosts`:

    192.168.10.10  homestead.test
    192.168.10.10  another.test

Once the site has been added, execute the `vagrant reload --provision` terminal command from your Homestead directory.

<a name="site-types"></a>
#### Site Types

Homestead supports several "types" of sites which allow you to easily run projects that are not based on Laravel. For example, we may easily add a Statamic application to Homestead using the `statamic` site type:

```yaml
sites:
    - map: statamic.test
      to: /home/vagrant/my-symfony-project/web
      type: "statamic"
```

The available site types are: `apache`, `apigility`, `expressive`, `laravel` (the default), `proxy`, `silverstripe`, `statamic`, `symfony2`, `symfony4`, and `zf`.

<a name="site-parameters"></a>
#### Site Parameters

You may add additional Nginx `fastcgi_param` values to your site via the `params` site directive:

    sites:
        - map: homestead.test
          to: /home/vagrant/project1/public
          params:
              - key: FOO
                value: BAR

<a name="environment-variables"></a>
### Environment Variables

You can define global environment variables by adding them to your `Homestead.yaml` file:

    variables:
        - key: APP_ENV
          value: local
        - key: FOO
          value: bar

After updating the `Homestead.yaml` file, be sure to re-provision the machine by executing the `vagrant reload --provision` command. This will update the PHP-FPM configuration for all of the installed PHP versions and also update the environment for the `vagrant` user.

<a name="ports"></a>
### Ports

By default, the following ports are forwarded to your Homestead environment:

<div class="content-list" markdown="1">
- **SSH:** 2222 &rarr; Forwards To 22
- **ngrok UI:** 4040 &rarr; Forwards To 4040
- **HTTP:** 8000 &rarr; Forwards To 80
- **HTTPS:** 44300 &rarr; Forwards To 443
- **MySQL:** 33060 &rarr; Forwards To 3306
- **PostgreSQL:** 54320 &rarr; Forwards To 5432
- **MongoDB:** 27017 &rarr; Forwards To 27017
- **Mailhog:** 8025 &rarr; Forwards To 8025
- **Minio:** 9600 &rarr; Forwards To 9600
</div>

<a name="forwarding-additional-ports"></a>
#### Forwarding Additional Ports

If you wish, you may forward additional ports to the Vagrant box by defining a `ports` configuration entry within your `Homestead.yaml` file. After updating the `Homestead.yaml` file, be sure to re-provision the machine by executing the `vagrant reload --provision` command:

    ports:
        - send: 50000
          to: 5000
        - send: 7777
          to: 777
          protocol: udp

<a name="php-versions"></a>
### PHP Versions

Homestead 6 introduced support for running multiple versions of PHP on the same virtual machine. You may specify which version of PHP to use for a given site within your `Homestead.yaml` file. The available PHP versions are: "5.6", "7.0", "7.1", "7.2", "7.3", "7.4", and "8.0" (the default):

    sites:
        - map: homestead.test
          to: /home/vagrant/project1/public
          php: "7.1"

[Within your Homestead virtual machine](#connecting-via-ssh), you may use any of the supported PHP versions via the CLI:

    php5.6 artisan list
    php7.0 artisan list
    php7.1 artisan list
    php7.2 artisan list
    php7.3 artisan list
    php7.4 artisan list
    php8.0 artisan list

You may change the default version of PHP used by the CLI by issuing the following commands from within your Homestead virtual machine:

    php56
    php70
    php71
    php72
    php73
    php74
    php80

<a name="connecting-to-databases"></a>
### Connecting To Databases

A `homestead` database is configured for both MySQL and PostgreSQL out of the box. To connect to your MySQL or PostgreSQL database from your host machine's database client, you should connect to `127.0.0.1` on port `33060` (MySQL) or `54320` (PostgreSQL). The username and password for both databases is `homestead` / `secret`.

> {note} You should only use these non-standard ports when connecting to the databases from your host machine. You will use the default 3306 and 5432 ports in your Laravel application's `database` configuration file since Laravel is running _within_ the virtual machine.

<a name="database-backups"></a>
### Database Backups

Homestead can automatically backup your database when your Homestead virtual machine is destroyed. To utilize this feature, you must be using Vagrant 2.1.0 or greater. Or, if you are using an older version of Vagrant, you must install the `vagrant-triggers` plug-in. To enable automatic database backups, add the following line to your `Homestead.yaml` file:

    backup: true

Once configured, Homestead will export your databases to `mysql_backup` and `postgres_backup` directories when the `vagrant destroy` command is executed. These directories can be found in the folder where you installed Homestead or in the root of your project if you are using the [per project installation](#per-project-installation) method.

<a name="database-snapshots"></a>
### Database Snapshots

Homestead supports freezing the state of MySQL and MariaDB databases and branching between them using [Logical MySQL Manager](https://github.com/Lullabot/lmm). For example, imagine working on a site with a multi-gigabyte database. You can import the database and take a snapshot. After doing some work and creating some test content locally, you may quickly restore back to the original state.

Under the hood, LMM uses LVM's thin snapshot functionality with copy-on-write support. In practice, this means that changing a single row in a table will only cause the changes you made to be written to disk, saving significant time and disk space during restores.

Since LMM interacts with LVM, it must be run as `root`. To see all available commands, run the `sudo lmm` command within Vagrant box. A common workflow looks like the following:

- Import a database into the default `master` lmm branch.
- Save a snapshot of the unchanged database using `sudo lmm branch prod-YYYY-MM-DD`.
- Modify the database.
- Run `sudo lmm merge prod-YYYY-MM-DD` to undo all changes.
- Run `sudo lmm delete <branch>` to delete unneeded branches.

<a name="configuring-cron-schedules"></a>
### Configuring Cron Schedules

Laravel provides a convenient way to [schedule cron jobs](/docs/{{version}}/scheduling) by scheduling a single `schedule:run` Artisan command to run every minute. The `schedule:run` command will examine the job schedule defined in your `App\Console\Kernel` class to determine which scheduled tasks to run.

If you would like the `schedule:run` command to be run for a Homestead site, you may set the `schedule` option to `true` when defining the site:

```yaml
sites:
    - map: homestead.test
      to: /home/vagrant/project1/public
      schedule: true
```

The cron job for the site will be defined in the `/etc/cron.d` directory of the Homestead virtual machine.

<a name="configuring-mailhog"></a>
### Configuring MailHog

[MailHog](https://github.com/mailhog/MailHog) allows you to intercept your outgoing email and examine it without actually sending the mail to its recipients. To get started, update your application's `.env` file to use the following mail settings:

    MAIL_MAILER=smtp
    MAIL_HOST=localhost
    MAIL_PORT=1025
    MAIL_USERNAME=null
    MAIL_PASSWORD=null
    MAIL_ENCRYPTION=null

Once MailHog has been configured, you may access the MailHog dashboard at `http://localhost:8025`.

<a name="configuring-minio"></a>
### Configuring Minio

[Minio](https://github.com/minio/minio) is an open source object storage server with an Amazon S3 compatible API. To install Minio, update your `Homestead.yaml` file with the following configuration option in the [features](#installing-optional-features) section:

    minio: true

By default, Minio is available on port 9600. You may access the Minio control panel by visiting `http://localhost:9600`. The default access key is `homestead`, while the default secret key is `secretkey`. When accessing Minio, you should always use region `us-east-1`.

In order to use Minio, you will need to adjust the S3 disk configuration in your application's `config/filesystems.php` configuration file. You will need to add the `use_path_style_endpoint` option to the disk configuration as well as change the `url` key to `endpoint`:

    's3' => [
        'driver' => 's3',
        'key' => env('AWS_ACCESS_KEY_ID'),
        'secret' => env('AWS_SECRET_ACCESS_KEY'),
        'region' => env('AWS_DEFAULT_REGION'),
        'bucket' => env('AWS_BUCKET'),
        'endpoint' => env('AWS_URL'),
        'use_path_style_endpoint' => true,
    ]

Finally, ensure your `.env` file has the following options:

```bash
AWS_ACCESS_KEY_ID=homestead
AWS_SECRET_ACCESS_KEY=secretkey
AWS_DEFAULT_REGION=us-east-1
AWS_URL=http://localhost:9600
```

To provision Minio powered "S3" buckets, add a `buckets` directive to your `Homestead.yaml` file. After defining your buckets, you should execute the `vagrant reload --provision` command in your terminal:

```yaml
buckets:
    - name: your-bucket
      policy: public
    - name: your-private-bucket
      policy: none
```

Supported `policy` values include: `none`, `download`, `upload`, and `public`.

<a name="laravel-dusk"></a>
### Laravel Dusk

In order to run [Laravel Dusk](/docs/{{version}}/dusk) tests within Homestead, you should enable the [`webdriver` feature](#installing-optional-features) in your Homestead configuration:

```yaml
features:
    - webdriver: true
```

After enabling the `webdriver` feature, you should execute the `vagrant reload --provision` command in your terminal.

<a name="sharing-your-environment"></a>
### Sharing Your Environment

Sometimes you may wish to share what you're currently working on with coworkers or a client. Vagrant has built-in support for this via the `vagrant share` command; however, this will not work if you have multiple sites configured in your `Homestead.yaml` file.

To solve this problem, Homestead includes its own `share` command. To get started, [SSH into your Homestead virtual machine](#connecting-via-ssh) via `vagrant ssh` and execute the `share homestead.test` command. This command will share the `homestead.test` site from your `Homestead.yaml` configuration file. You may substitute any of your other configured sites for `homestead.test`:

    share homestead.test

After running the command, you will see an Ngrok screen appear which contains the activity log and the publicly accessible URLs for the shared site. If you would like to specify a custom region, subdomain, or other Ngrok runtime option, you may add them to your `share` command:

    share homestead.test -region=eu -subdomain=laravel

> {note} Remember, Vagrant is inherently insecure and you are exposing your virtual machine to the Internet when running the `share` command.

<a name="debugging-and-profiling"></a>
## Debugging & Profiling

<a name="debugging-web-requests"></a>
### Debugging Web Requests With Xdebug

Homestead includes support for step debugging using [Xdebug](https://xdebug.org). For example, you can access a page in your browser and PHP will connect to your IDE to allow inspection and modification of the running code.

By default, Xdebug is already running and ready to accept connections. If you need to enable Xdebug on the CLI, execute the `sudo phpenmod xdebug` command within your Homestead virtual machine. Next, follow your IDE's instructions to enable debugging. Finally, configure your browser to trigger Xdebug with an extension or [bookmarklet](https://www.jetbrains.com/phpstorm/marklets/).

> {note} Xdebug causes PHP to run significantly slower. To disable Xdebug, run `sudo phpdismod xdebug` within your Homestead virtual machine and restart the FPM service.

<a name="autostarting-xdebug"></a>
#### Autostarting Xdebug

When debugging functional tests that make requests to the web server, it is easier to autostart debugging rather than modifying tests to pass through a custom header or cookie to trigger debugging. To force Xdebug to start automatically, modify the `/etc/php/7.x/fpm/conf.d/20-xdebug.ini` file inside your Homestead virtual machine and add the following configuration:

```ini
; If Homestead.yaml contains a different subnet for the IP address, this address may be different...
xdebug.remote_host = 192.168.10.1
xdebug.remote_autostart = 1
```

<a name="debugging-cli-applications"></a>
### Debugging CLI Applications

To debug a PHP CLI application, use the `xphp` shell alias inside your Homestead virtual machine:

    xphp /path/to/script

<a name="profiling-applications-with-blackfire"></a>
### Profiling Applications with Blackfire

[Blackfire](https://blackfire.io/docs/introduction) is a service for profiling web requests and CLI applications. It offers an interactive user interface which displays profile data in call-graphs and timelines. It is built for use in development, staging, and production, with no overhead for end users. In addition, Blackfire provides performance, quality, and security checks on code and `php.ini` configuration settings.

The [Blackfire Player](https://blackfire.io/docs/player/index) is an open-source Web Crawling, Web Testing, and Web Scraping application which can work jointly with Blackfire in order to script profiling scenarios.

To enable Blackfire, use the "features" setting in your Homestead configuration file:

```yaml
features:
    - blackfire:
        server_id: "server_id"
        server_token: "server_value"
        client_id: "client_id"
        client_token: "client_value"
```

Blackfire server credentials and client credentials [require a Blackfire account](https://blackfire.io/signup). Blackfire offers various options to profile an application, including a CLI tool and browser extension. Please [review the Blackfire documentation for more details](https://blackfire.io/docs/cookbooks/index).

<a name="network-interfaces"></a>
## Network Interfaces

The `networks` property of the `Homestead.yaml` file configures network interfaces for your Homestead virtual machine. You may configure as many interfaces as necessary:

```yaml
networks:
    - type: "private_network"
      ip: "192.168.10.20"
```

To enable a [bridged](https://www.vagrantup.com/docs/networking/public_network.html) interface, configure a `bridge` setting for the network and change the network type to `public_network`:

```yaml
networks:
    - type: "public_network"
      ip: "192.168.10.20"
      bridge: "en1: Wi-Fi (AirPort)"
```

To enable [DHCP](https://www.vagrantup.com/docs/networking/public_network.html), just remove the `ip` option from your configuration:

```yaml
networks:
    - type: "public_network"
      bridge: "en1: Wi-Fi (AirPort)"
```

<a name="extending-homestead"></a>
## Продление Homestead

Вы можете расширить Homestead, используя сценарий `after.sh` в корне вашего каталога Homestead. В этот файл вы можете добавить любые команды оболочки, которые необходимы для правильной настройки и настройки вашей виртуальной машины.

При настройке Homestead, Ubuntu может спросить вас, сохранять ли исходную конфигурацию пакета или перезаписать ее новым файлом конфигурации. Чтобы избежать этого, вы должны использовать следующую команду, чтобы избежать перезаписи любой конфигурации, ранее записанной Homestead:

    sudo apt-get -y \
        -o Dpkg::Options::="--force-confdef" \
        -o Dpkg::Options::="--force-confold" \
        install package-name

<a name="user-customizations"></a>
### User Customizations

When using Homestead with your team, you may want to tweak Homestead to better fit your personal development style. To accomplish this, you may create a `user-customizations.sh` file in the root of your Homestead directory (the same directory containing your `Homestead.yaml` file). Within this file, you may make any customization you would like; however, the `user-customizations.sh` should not be version controlled.

<a name="provider-specific-settings"></a>
## Provider Specific Settings

<a name="provider-specific-virtualbox"></a>
### VirtualBox

<a name="natdnshostresolver"></a>
#### `natdnshostresolver`

By default, Homestead configures the `natdnshostresolver` setting to `on`. This allows Homestead to use your host operating system's DNS settings. If you would like to override this behavior, add the following configuration options to your `Homestead.yaml` file:

```yaml
provider: virtualbox
natdnshostresolver: 'off'
```

<a name="symbolic-links-on-windows"></a>
#### Symbolic Links On Windows

If symbolic links are not working properly on your Windows machine, you may need to add the following block to your `Vagrantfile`:

```ruby
config.vm.provider "virtualbox" do |v|
    v.customize ["setextradata", :id, "VBoxInternal2/SharedFoldersEnableSymlinksCreate/v-root", "1"]
end
```
