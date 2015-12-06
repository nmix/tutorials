#Rails Gem. Руководство

Данное руководство представляет собой адаптацию [раздела](http://guides.rubyonrails.org/engines.html) официальной документации RailsGuids (или его [перевода](http://rusrails.ru/engines)).

Здесь приводится порядок действий по созданию rails-приложения, которое будет интегрировано в другое rails-приложение в виде сторонней библиотеки (gem) и будет предоставлять этому (другому) приложению некоторый функционал.
Основные отличия от оригинальной документации:
* использование `RSpec` вместо `UnitTest`
* использование `HAML` вместо `ERB`
* использование `Bootstrap`

> Примечание. Это поле примечания. Оно будет встречаться довольно часто по тексту. В свете неопытности автора в Rails в таких полях могут иметь место некорректные формулировки.

##1. Создание engine
> В данном контексте `engine` - это rails-приложение, которое будет встраиваться в другое rails-приложение.

Создаем `engine` с учетом приминения `rspec` ([источник](http://www.andrewhavens.com/posts/27/how-to-create-a-new-rails-engine-which-uses-rspec-and-factorygirl-for-testing/))

    $ rails plugin new blorgh -T --mountable --full --dummy-path=spec/test_app

Добавляем в файл спецификации `blorgh/blorgh.gemspec` следущие строки:

```ruby
s.add_dependency "haml-rails", "~> 0.9"
s.add_dependency "jquery-rails", "~> 4.0", ">= 4.0.5"
s.add_dependency "bootstrap-sass", "~> 3.3", ">= 3.3.5"
s.add_dependency "sass-rails", "~> 5.0", ">= 5.0.4"
s.add_dependency "bootstrap_form", "~> 2.3", ">= 2.3.0"

s.add_development_dependency "rspec-rails", "~> 3.3.3"
s.add_development_dependency "factory_girl_rails", "~> 4.5.0"
s.add_development_dependency "hirb", "~> 0.7.3"
```

Библиотека|Описание
---|--------
[rspec-rails](https://github.com/rspec/rspec-rails)|Фреймворк для тестирования для Rails 3.x/4x
[factory_girl_rails](https://github.com/thoughtbot/factory_girl_rails)| Продвинутая замена испытательных стендов (fixtures)
[haml-rails](https://github.com/indirect/haml-rails)|Haml генератор для Rails 4
[hirb](https://github.com/cldwalker/hirb)|Минифреймворк для Rails консоли
[jquery-rails](https://github.com/rails/jquery-rails)|Поддержка JQuery
[bootstrap-sass](https://github.com/twbs/bootstrap-sass)|Sass версия Bootstrap
[sass-rails](https://github.com/rails/sass-rails)|Поддержка Sass
[bootstrap_form](https://github.com/bootstrap-ruby/rails-bootstrap-forms)|Компоновщик Bootstrap форм

> Метод `add_dependency` добавляет функциональную зависимость - это значит, что без данной библиотеки наш `engine` работать не сможет (например, bootstrap). Эти библиотки будут затребоваться (см. ниже) в коде и, в последствии, будут автоматически загружаться при встраивании `engine` в другое приложение. Метод `add_development_dependency` добавляет "локальную" зависимость, т.е. эти библиотеки будут использоваться только при разработке `engine` и НЕ будут загружаться при встраивании в другое приложение.

Устанавливаем библиотеки

    $ bundle

Корректируем файл `blorgh/lib/blorgh/engine.rb`
```ruby
module Blorgh# blorgh/lib/blorgh/engine.rb
  class Engine < ::Rails::Engine
    isolate_namespace Blorgh
    config.generators do |g|
      g.test_framework :rspec,
        fixtures: true,
        view_specs: false,
        helper_specs: false,
        routing_specs: false,
        controller_specs: true,
        request_specs: true
      g.fixture_replacement :factory_girl, dir: "spec/factories"
      g.template_engine :haml
    end
  end
end
```

Запускаем `rspec` генератор

    $ rails generate rspec:install



> В рамках `engine` создается файл `app/views/layout/blorgh/application.html.erb`. В соответствии с [описанием](https://github.com/indirect/haml-rails) на rails-haml, преобразование данного файла в формат HAML осуществляется командой
> ```
> $ rails generate haml:application_layout convert
> ```
> Но она выдает ошибку. `$ rake haml:erb2haml` также не работает. Поэтому пока будем преобразовыть с использованием команды `html2haml`
> ```
> $ html2haml app/views/layouts/blorgh/application.html.erb app/views/layouts/blorgh/application.html.haml
> $ rm app/views/layouts/blorgh/application.html.erb```

Переименовываем файл `application.css`, находящийся в директории `blorgh/app/assets/stylesheets/blorgh`, в `application.scss`

    $ mv app/assets/stylesheets/blorgh/application.css app/assets/stylesheets/blorgh/application.scss

Добавляем в `blorgh/app/assets/stylesheets/blorgh/application.scss` следующие строки:

```scss
@import "bootstrap-sprockets";
@import "bootstrap";
```
Если верить [руководству](https://github.com/twbs/bootstrap-sass#a-ruby-on-rails), предварительно нобходимо удалить из этого файла строки `*= require_self` и `*= require_tree .`.

Корректируем файл `blorgh/app/assets/javascripts/blorgh/applications.js`
```javascript
//= require jquery
//= require bootstrap-sprockets
//= require_tree .
```

Корректируем файл `blorgh/lib/blorgh.rb`

```ruby
require 'haml-rails'
require 'jquery-rails'
require 'bootstrap-sass'
require 'sass-rails'
require 'bootstrap_form'

require "blorgh/engine"
...
```

## 2. Создание функционала

#### Создаем ресурс Articles

Генерируем `scaffold` для нового ресурса и применяем миграцию

    $ rails generate scaffold article title:string text:string
    $ rake db:migrate

Переходим в директорию `blorgh/spec/test_app` и запускаем встроенный веб-сервер

    $ rails server

В браузере переходим по ссылке `http://localhost:3000/blorgh/articles` и наблюдаем шаблон `index.html.haml` (Listing Articlies...).


>По умолчанию стили `scaffold` не применяются (в нашем случае используются стили из `Bootstrap`). При необходимости можно их включить:
1. Добавить в файл `blorgh/spec/test_app/config/initializers/assets.rb` следующую строку
```ruby
Rails.application.config.assets.precompile += %w( scaffold.css )
```
2. Добавить в файл `blorgh/app/views/layouts/blorgh/application.html.haml` в тэг `<head>` строку:
```haml
= stylesheet_link_tag scaffold
```


Добавим данный ресурс в корень `engine`. В файл `blorgh/config/routes.rb` добавляем строку:

```ruby
root to: 'articles#index'
```

Теперь можем перейти по адресу `http://localhost:3000/blorgh` вместо `http://localhost:3000/blorgh/articles`.

Для добавление боковых отступов на странице откорректируем файл `blorgh/app/views/layouts/blorgh/application.html.haml`:

```haml
...
  %body
    .container
      = yield
```

Откорректируем файл `blorgh/app/views/blorgh/articles/index.html.haml` для примиенния стилей `Bootstrap`
```haml
- if notice
  .alert.alert-success= notice
%h1 Listing articles
%table.table
  %thead
...
```

В браузере переходим по адресу `http://localhost:3000/blorgh` и наблюдаем отображение таблицы из шаблона в стиле `Bootstrap`.

С исползованием компоновщика `bootstrap_form` создаем форму для редактирования ресурса `article`. Для этого редактируем файл `blorg/app/views/blorgh/articles/_form.html.haml`:
```haml
= bootstrap_form_for @article do |f|
  - if @article.errors.any?
    = f.alert_message t('errors.template.header', count: @article.errors.count)
  = f.text_field :title
  = f.text_area :text
  = f.submit
```

#### Создаем ресурс Comments

Создаем модель `Comment` и запускаем миграцию

    $ rails generate model Comment article_id:integer text:text
    $ rake db:migrate

Для отображения комментариев статьи, а также формы для создания нового коментария, добавляем в файл `blorgh/app/views/blorgh/articles/show.html.erb` следующие строки:
```haml
%h3 Comments
= render @article.comments
= render "blorgh/comments/form"
```

Создаем файл партиала `blorgh/app/views/blorgh/comments/_comment.html.haml` для отображения комметария статьи (см. `= render @article.comments`):
```haml
= "#{comment_counter + 1}. #{comment.text}"
```
> Переменная `comment_counter` определяется автоматически и увеличивается с каждой интерацией.

Создаем файл формы комменария `blorgh/app/views/blorgh/comments/_form.html.haml` следующего содержания:
```haml
%h3 New comment
= bootstrap_form_for [@article, @article.comments.build] do |f|
  = f.text_area :text
  = f.submit
```

В модель `Article` (файл `blorgh/app/models/blorgh/article.rb`) добавляем связь `has_many`:
```ruby
has_many :comments
```

Корректируем файл маршрутов для создания вложеного ресурса `comments`:
```ruby
resources :articles do
  resources :comments
end
```
> Подробная информация по вложенным ресурсам приведена [здесь](http://rusrails.ru/rails-routing#vlozhennye-resursy)

Создаем контроллер для ресурса `comments`

    $ rails generate controller comments

В созданный контроллер (файл `blorgh/app/controllers/blorgh/comments_controller.rb`) добавляем следующие строки:

```ruby
def create
  @article = Article.find(params[:article_id])
  @comment = @article.comments.create(comment_params)
  flash[:notice] = "Comment has been created!"
  redirect_to articles_path
end

private
  def comment_params
    params.require(:comment).permit(:text)
  end
```

Создадим файл `blorgh/db/seeds.rb` и добавим туда несколько записей

```ruby
Comment.delete_all
Article.delete_all

article_1 = Article.create!(title: 'First article', text: 'It is my first article')
article_2 = Article.create!(title: 'Second article', text: 'It is my second article')
```

#### Тестирование

В файл `blorgh/spec/spec_helper.rb` добавляем следующие строки:

```ruby
ENV["RAILS_ENV"] ||= 'test'
require File.expand_path("../test_app/config/environment", __FILE__)
```

Создаем тестовую БД:

	$ rake db:setup RAILS_ENV=test

Проверяем корректный запуск тестов:

	$ rspec spec

Отладочный вывод должен содержать набор отложенных (желтых) тестов

> Не смотря на то, что мы явно указали окружение выполнения тестирования и место расположения файла окружения (`environment`) может возникнуть ошибка при запуске `rspec`: `cannot load such file -- <path>/blorgh/config/environment (LoadError)`, т.е. "не найден файл окружения". Как временное решение - создать (пустой) файл `blorgh/config/environment.rb` и повторно запустить `rspec spec`.


> Возможно, будет иметь место набор ошибочных (красных) тестов, например `No route matches {:action=>"new", :controller=>"blorgh/articles"}`, при этом фактически путь существует и переход по данному пути происходит корректно. Вот, что удалось найти: [ссылка](http://blog.pivotal.io/labs/labs/writing-rails-engine-rspec-controller-tests) и [ссылка](https://github.com/rspec/rspec-rails/issues/1304). На момент разработки приложения ошибочный тест был закоментирован.


Все тесты оставим в "отложенном" состоянии. Тема тестирования будет описана в другом руководстве.


## 3. Внедрение в приложение

##### 3.1. Настройка `engine`

Создаем новое Rails-приложение (в которое будем внедрять `blorgh`)

	$ rails new unicorn

В `Gemfile` созданного приложения добавляем строку (файл `unicorn/Gemfile`)

```ruby
...
gem 'blorgh', path: '../blorgh'
```

В примере используется относительный путь. Допускается указывать абсолютный.

> Если из файла `blorgh/blorgh.gemspec` не удалить все "FIXME" и "TODO", то `gem` не будет установлен.
> Образец заполнения полей в `gemspec`:
> s.homepage    = "http://example.com"
  s.summary     = "Summary of Blorgh."
  s.description = "Description of Blorgh."

Выполняем установку новой библиотеки

	$ bundle

В файл `unicorn/config/routes.rb` добавляем следующую строку:

```ruby
mount Blorgh::Engine, at: 'blog'
```

Копируем миграции из `engine`

	$ rake blorgh:install:migrations

и применяем их

	$ rake db:migrate SCOPE=blorgh

Теперь запускаем сервер

	$ rails server

и нам становится доступным адрес `http://localhost:3000/blog`. Т.е. мы корректно интегрировали наш `engine` (приложение `blorgh`) в приложение `unicorn`.

> Если при переходе по адресу `http://localhost:3000/blog` выдается сообщение об ошибке (например, `Template is missing`), то возможно мы забыли затребовать (`require`) какую-то runtime библиотеку в файле `blorgh/lib/blorgh.rb`.

##### 3.2. Использование (в `blorgh`) модели приложения (из `unicorn`)

В приложении `unicorn` создадим модель `User`, которую "привяжем" к `engine`.

В `unicorn` создаем простую модель `User`

	$ rails g model user name:string
    $ rake db:migrate

В `engine` создаем миграцию, которая определит идентификатор "автора" в таблице `articles`.

	$ rails g migration add_author_id_to_blorgh_articles author_id:integer
    $ rake db:migrate

Корректируем файл `blorgh/app/views/blorgh/articles/_form.html.haml`

```haml
...
= f.text_area :text
= f.text_field :author_name
...
```

Корректируем файл `blorgh/app/controllers/blorgh/article_controller.rb`

```ruby
def article_params
  params.require(:article).permit(:title, :text, :author_name)
end
```

Теперь необходимо откорректировать модель `Blorgh::Article` (файл `blorgh/app/models/blorgh/article.rb`)

```ruby
module Blorgh
  class Article < ActiveRecord::Base
    has_many :comments

    attr_accessor :author_name
    belongs_to :author, class_name: "User"

    before_save :set_author

    private
      def set_author
        self.author = User.find_or_create_by(name: author_name)
      end
  end
end
```

Здесь мы (1) с помощью метода `attr_accessor` создаем аттрибут `:author_name` и одноименный метод доступа к нему; (2) создаем связь `:author`, которая будет жестко "привязана" к модели `User` и хранить идентификатор в поле `author_id` (см. миграцию выше); (3) с помощью метода `:before_save` определяем метод, который будет создавать связь с моделью `User` с использованием аттрибута `:author_name` перед сохранением нового объекта `Article`.

Корректируем файл представления `blorgh/app/views/blorgh/articles/show.html.haml`

```haml
%p
  %b Author:
  = @article.author
```

В приложении `unicorn` применяем миграции из `engine` и запускаем сервер

	$ rake blorgh:install:migrations
    $ rake db:migrate
    $ rails server

Переходим по адресу `http://localhost:3000/blog` и нажимаем на ссылочку `New Article`. В окне формы новой статьм отображается вновь созданное поле `Author name`. Заполняем все поля и нажимаем кнопку `Create Article`. Отображается страница статьи, где в поле `Author` отображается идентификатор объекта `#<User:....>`. Для придания этому полю нормальный вид, необходимо в модели `User` (файл `unicorn/app/models/user.rb`) определить метод `to_s`

```ruby
def to_s
  name
end
```

##### 3.3. Конфигурирование `engine`

В предыдущем разделе мы создали жесткую приязку моделей `Blorgh::Article` и `User`. В данном разделе сделаем эту связь настраиваемой.

Добавляем в файл `blorgh/lib/engine.rb` следующую строку:

```ruby
mattr_accessor :author_class
```

> строку нужно добавить внутрь модуля `Blorgh`

Изменяем файл `blorgh/app/models/blorgh/article.rb`

```ruby
module Blorgh

  def self.author_class
    @@author_class.constantize
  end

  class Article < ActiveRecord::Base
    has_many :comments

    attr_accessor :author_name
    belongs_to :author, class_name: Blorgh.author_class.to_s

    before_save :set_author

    private
      def set_author
        self.author = Blorgh.author_class.find_or_create_by(name: author_name)
      end
  end

end

```

Мы создали конфигурационную настройку, которую можно задать в инициализаторе приложения `unicorn`.
> Инициализатор - это файл, который расположен в директории `<appname>/config/initializers/`.

Создаем инициализатор для `blorgh` (файл `unicorn/config/initializers/blorgh.rb`) следующего содержания

```ruby
Blorgh.author_class = "User"
```

Запускаем сервер и контролируем отсутсвие каких-либо изменений при создании и отображении ресурса `Article`.



###### (еще не) Конец