Пишем свой первый plugin для Rails 4
====================

# Задача

Разработать плагин (**dagraph**), который реализует структуру ориентированного ацикличного графа (Directed Acyclic Graph, см. [wiki](https://ru.wikipedia.org/wiki/Направленный_ациклический_граф)) для произвольных моделей `ActiveRecord`. В качестве веса ребер графов также необходимо использовать произвольные модели `ActiveRecord`.

Пример графа: 

![Пример графа](https://upload.wikimedia.org/wikipedia/commons/0/08/Directed_acyclic_graph.png)

Модель, наследуемая от `ActiveRecord`, должна расширяться методом `acts_as_dagraph` и должна реализовывать следующие методы (для модели экземпляра *n* класса `Node < ActiveRecord`):
Наименование|Назначение
-----------------------|------------
`n.add_child(node, weight)`|Назначить узел `node` *дочерним* с весом `weight`. При попытке добавить узел, реализующего циклическую связь выдается *исключение*.
`n.remove_child(node)`|Удалить узел `node` из *дочерних* (сам экземпляр при этом не удаляется)
`n.add_parent(node, weight)` | Назначить узел `node` *родительским* с весом `weight`.  При попытке добавить узел, реализующего циклическую связь выдается *исключение*.
`n.remove_parent(node)` | Удалить узел `node` из *родительских* (сам экземпляр при этом не удаляется)
`n.parents()` | Получение массива *родительских* узлов (непосредственных родителей). На рисунке для узла **11** - это **7** и **5**, а для узла **9** - это [**11**, **8**].
`n.children()` | Получение массива *дочерних* узлов (для которых узел `n` является непосредственным родителем). На рисунке для узла **3** - это [**8**, **10**], а для узла **11** - это [**2**, **9**, **10**].
`n.routing()` | Получение хэша машрутов, проходящих черех этот узел. На рисунке для узла **8** - это {**rid1** => [**7**, **8**, **9**], **rid2** => [**3**, **8**, **9**]}, где rid1, rid2 - идентификаторы маршрутов.
`n.roots()` | Получение массива корневых *родительских* узлов относительно данного узла. На рисунке для узла **2** - это [**7**, **5**], а для узла **9** - это [**7**, **5**, **3**].
`n.leafs()` | Получение массива "листев" для данного узла. На рисунке для **7** - это [**2**, **9**, **10**], а для **3** - это [**9**, **10**].
`n.ancestors()` | Получения массива маршрутов *предков*. На рисунке для **2** - это [[**7**, **11**], [**5**, **11**]], в для **9** - это [[**7**, **11**], [**5**, **11**], [**7**, **8**], [**3**, **8**]]. 
`n.descendants()` | Получения массива маршрутов *потомков*. На рисунке для **7** - это [[**11**, **2**], [**11**, **9**], [**11**, **10**], [**8**, **9**]]. 

Методы расширенного класса `Node < ActiveRecord`
Наименование|Назначение
--------------------|-----------------------
`Node.roots()` | Получение массива *корневых* узлов данного класса. На рисунке - это [**7**, **5**, **3**].
`Node.leafs()` | Получение массива *листев* данного класса. На рисунке - это [**2**, **9**, **10**]


Сценарий использования гема:
1. Добавление гема в Gemfile ( `gem 'dagraph'` )
2. Установка гема ( `bundle install` )
3. Добавление метода в модель ( `acts_as_dagraph` )
4. Генерация необходимых миграций ( `rails generate dagraph:migration` )
5. Применение миграций ( `bundle exec rake db:migrate` )

Тестирование осуществлять средствами фреймворка `RSpec`.

Конец задачи.


# Решение

Для решения данной задачи помимо расширения произвольного класса методом `acts_as_dagraph`, нам понадобится еще три модели (в модуле `Dagraph`): `Edge` - ребра графа, `Route` - идентификаторы маршрутов графа и `RouteNodes` -  вершины на маршруте.
Модель|Описание
-----------|--------
Dagraph::Edge|Модель будет содержать ребра графа и веса ребер в виде полиморфных полей `dag_parent` - ссылка на объект "родитель", `dag_child` - ссылка на объект "непосредственный потомок" и `weight` - ссылка на объект веса.
Dagraph::Route| Модель будет идентифицировать уникальный маршрут.
Dagraph::RouteNodes| Модель содержит принадлежность узлов графа маршрутам. Содержит поля: `route` - внешний ключ на маршрут (`Route`); `node` - полиморфное поле, указывающее на объект на который ссылает `dag_parent` или `dag_child`;  `level` - уровень (или позиция) узла на маршруте.

При вызове методов редактирования графа (`add_child()`, `remove_child()`, `add_parent()` и т.д.) мы будем синхронно вносить изменения во все модели. Это позволит нам, например, оперативно определять попытки зациклить граф. Кроме того, будет проще выполнять выборки (обходы) по всем потомкам узла графа путем частичного объединения маршрутов, в которых учавствует данный узел.

# Кодинг

## Шаг 1 - подготавливаем проект

Создаем новый плагин
```bash
rails plugin new dagraph -T --dummy-path=spec/test_app
```
>Опции:
>-T - пропускаем тесты, т.к. планируем использовать RSpec
>--dummy-path - путь установки тестового приложения

Корректируем файл `dagraph.gemspec` - задаем homepage и убираем все TODO из других параметров, а также добавляем гемы для тестирования и разработки

```ruby
$:.push File.expand_path("../lib", __FILE__)

# Maintain your gem's version:
require "dagraph/version"
# Describe your gem and declare its dependencies:
Gem::Specification.new do |s|
  s.name        = "dagraph"
  s.version     = Dagraph::VERSION
  s.authors     = ["John Ivanoff"]
  s.email       = ["foo@example.com"]
  s.homepage    = "http://example.com"
  s.summary     = "Description"
  s.description = "Description"
  s.license     = "MIT"

  s.files = Dir["{app,config,db,lib}/**/*", "MIT-LICENSE", "Rakefile", "README.rdoc"]

  s.add_dependency "rails", "~> 4.2.6"
  s.add_development_dependency "sqlite3"
  s.add_development_dependency "rspec-rails" 
  s.add_development_dependency "factory_girl_rails" 
  s.add_development_dependency "faker" 
  s.add_development_dependency "hirb"
end
```

Устанавливаем гемы

```bash
bundle install
```

Инициализируем `rspec`

```bash
rspec --init
```

В соответствии с [этой](https://www.viget.com/articles/rails-engine-testing-with-rspec-capybara-and-factorygirl) статьей заменяем файлы `Rakefile` и `spec/spec_helper.rb` следующим содержимым соответственно:

Rakefile
```ruby
#!/usr/bin/env rake

begin
 require 'bundler/setup'
rescue LoadError
 puts 'You must `gem install bundler` and `bundle install` to run rake tasks'
end

APP_RAKEFILE = File.expand_path("../spec/test_app/Rakefile", __FILE__)

load 'rails/tasks/engine.rake'

Bundler::GemHelper.install_tasks

Dir[File.join(File.dirname(__FILE__), 'tasks/**/*.rake')].each {|f| load f }

require 'rspec/core'
require 'rspec/core/rake_task'

desc "Run all specs in spec directory (excluding plugin specs)"

RSpec::Core::RakeTask.new(:spec => 'app:db:test:prepare')

task :default => :spec
```

spec/spec_helper.rb

```ruby
ENV['RAILS_ENV'] ||= 'test'

require File.expand_path("../test_app/config/environment.rb", __FILE__)
require 'rspec/rails'
require 'factory_girl_rails'

Rails.backtrace_cleaner.remove_silencers!

# Load support files
Dir["#{File.dirname(__FILE__)}/support/**/*.rb"].each { |f| require f }

RSpec.configure do |config|
 config.mock_with :rspec
 config.use_transactional_fixtures = true
 config.infer_base_class_for_anonymous_controllers = false
 config.order = "random"
end
```
## Шаг 2 - создаем элементарные тесты для add_child() и add_parent()

Напишем простые тесты для методов `add_child()` и  `add_parent()`, которые позволят нам описать предварительную структуру приложения.
В качестве макетной модели будем использовать некий класс `Unit` (*сборочная единица*), включающая в себя два строковых поля: `code` - обозначение, `name` - наименование. Мы расширяем этот класс в макете методом `acts_as_dagraph` и используя методы  `add_child()` и  `add_parent()` создаем связи между объектами этой модели. Также запретим добавлять отношение узла к самому себе.
В тестах мактеную модель будем генерировать средствами `FactoryGirl`.

Создаем файл `spec/acts_as_dagraph_spec.br` в котором описываем тесты для методов `add_child()` и `add_parent()`:

```ruby

RSpec.describe "ActsAsDagraph" do
  let(:unit) { create(:unit) }
  let(:unit_2) { create(:unit) }

  it "has a valid Unit factory" do
    expect(unit).to be_valid
  end

  describe "#add_child" do
    it "creates an instance of Edge" do
      expect {
        unit.add_child(unit_2)
      }.to change{ Dagraph::Edge.count }.by(1)
    end

    it "raises exception if add self node" do
      expect {
        unit.add_child(unit)
      }.to raise_error(SelfCyclicError)
    end
  end

  describe "#add_parent" do
    it "creates an instance of Edge" do
      expect {
        unit.add_parent(unit_2)
      }.to change{ Dagraph::Edge.count }.by(1)
    end

    it "raises exception if add self node" do
      expect {
        unit.add_parent(unit)
      }.to raise_error(SelfCyclicError)
    end
  end
end
```

Запускаем тесты и получаем ожидаемые ошибки.
```bash
rspec spec/
```

## Шаг 3 - реализация методов add_parent() и add_child()

Порядок действий:
1. Создаем миграцию для Edge
2. Создаем генератор для миграции
3. Подготавливаем макетное приложение
4. Создаем *фабрику* для класса `Unit`
5. Собственно пишем код класса Edge и реализацию методов

#### 1.  Создаем миграцию для Edge
Для начала создадим шаблон миграции для модели `Edge`

```ruby
# lib/generators/dagraph/templates/create_dagraph.rb

class CreateDagraph < ActiveRecord::Migration 
  def change 
    create_table :dagraph_edges do |t| 
      t.references :dag_parent, polymorphic: true, index: true
      t.references :dag_child, polymorphic: true, index: true
      t.references :weight, polymorphic: true, index: true
      t.timestamps null: false
    end 
  end 
end
```
####2. Создаем генератор для миграции
Создаем файл-генератор
```ruby
# lib/generators/dagraph/migration_generator.rb

require 'rails/generators'
require 'rails/generators/active_record'
require 'rails/generators/base'

module Dagraph 
  class MigrationGenerator < Rails::Generators::Base 
    source_root File.expand_path("../templates", __FILE__) 
    include Rails::Generators::Migration 

    def create_migration_file 
      migration_template "create_dagraph.rb", "db/migrate/create_dagraph.rb" 
    end

    def self.next_migration_number(dirname) 
      ActiveRecord::Generators::Base.next_migration_number(dirname) 
    end 
  end 
end
```

Добавляем в файл `lib/dagraph.rb` строку.

```ruby
require "generators/dagraph/migration_generator.rb"
```


####3. Подготавливаем макетное приложение

Проверяем "доступность"  генератора в макете

```bash
cd spec/test_app
bin/rails generate
```

```bash
...
Dagraph:
  dagraph:migrate
...
```

Пока отключим генерирование тестов в макете  следующую строку:

```ruby
# config/application.rb
config.generators.test_framework false
```

Создаем модель `Unit`. 

```bash
bin/rails generate model Unit code name
```

Запускаем вновь созданный генератор

```bash
bin/rails generate dagraph:migration
```

Проверяем наличие двух миграций (одна - для таблицы `units`, вторая - для `dagraph_edges`).

```bash
bin/rake db:migrate:status
```

Запускаем миграции и, заодно, подготавливаем базу для тестирования
```bash
bin/rake db:migrate
bin/rake db:test:prepare
```

В модель `Unit`  добавляем строку `acts_as_dagraph`

#### 4. Создаем *фабрику* для класса `Unit`

В тестах мы используем вот такое выражение `crate(:unit)` - это не что иное, как сокращенная запись метода `FactoryGirl.create(:unit)` который, как предполагается в тесте, создает объект класса Unit. 

Для того, чтобы можно было использовать сокращенную запись добавим методы `FactoryGirl` в область видимости `RSpec`

```ruby
# spec/support/factory_girl.rb
require 'factory_girl_rails'

RSpec.configure do |config|
  config.include FactoryGirl::Syntax::Methods
end
```

Самое время описать нашу фабрику для объектов класса `Unit`

```ruby
#spec/factories/units.rb
require 'faker'

FactoryGirl.define do
  factory :unit, :class => 'Unit' do
    code { Faker::Business.credit_card_number }
    name { Faker::Commerce.product_name }
  end
end
```
#### 5. Пишем код класса Edge и реализацию методов

Опишем класс `Dagraph::Edge` 

```ruby
# lib/dagraph/edge.rb
module Dagraph
  class Edge < ActiveRecord::Base 
    self.table_name = "dagraph_edges"
    belongs_to :dag_parent, polymorphic: true
    belongs_to :dag_child, polymorphic: true
    belongs_to :weight, polymorphic: true
  end
end
```
Обозначим собственные классы для генерации исключений
```ruby
#lib/dagraph/errors.rb
class CyclicError < ArgumentError; end
class SelfCyclicError < CyclicError; end
```
Ну и наконец описываем наши методы `add_child()` и `add_parent()`
```ruby
#lib/dagraph/acts_as_dagraph.rb

module Dagraph
  module ActsAsDagraph
    extend ActiveSupport::Concern

    included do
    end

    module ClassMethods 
      def acts_as_dagraph(options = {}) 

        has_many :parent_edges,
          as: :dag_child,
          class_name:  "Dagraph::Edge",
          dependent: :destroy

        has_many :child_edges,
          as: :dag_parent,
          class_name:  "Dagraph::Edge",
          dependent: :destroy

        include Dagraph::ActsAsDagraph::LocalInstanceMethods
      end
    end

    module LocalInstanceMethods
      def add_child(node) 
        raise SelfCyclicError.exception("Must not add node to itself") if node == self
        Edge.create(dag_parent: self, dag_child: node)
      end

      def add_parent(node)
        raise SelfCyclicError.exception("Must not add node to itself") if node == self
        Edge.create(dag_parent: node, dag_child: self)
      end
    end
  end
end
ActiveRecord::Base.include(Dagraph::ActsAsDagraph)

```
Не забываем добавить эти файлы в `lib/dagraph.rb`
```ruby
#lib/dagraph.rb
require "dagraph/acts_as_dagraph.rb"
require "dagraph/edge.rb"
require "dagraph/errors.rb"
```

Повторно запускаем тесты и ожидаем, что все пройдут удачно
```bash
rspec spec/

.....

Finished in 0.59612 seconds (files took 7.43 seconds to load)
5 examples, 0 failures
```

## Шаг 4 - расширение методов add_parent() и add_child()

Порядок действий:
1. Проверка создания экземпляров модели `Dagraph::Route`
2. Реализация  модели `Dagraph::Route`
3. Проверка создания и реализация модели `Dagraph::RouteNodes`

#### 1. Проверка создания экземпляров модели `Dagraph::Route`

Прежде чем описывать соответствующие тесты добавим контексты в файл c тестами: 
`context "when graph is empty"` (1)
`context "when graph is arbitrary"` (2)
Это делается для того, чтобы отделить простую логику приложения в первом контексте от сложной во втором. В первом случае мы предполагаем, что структура связей еще не создана, что позволяет нам пока опустить ряд проверок и сконцентироваться на описании структуры приложения, а не алгоритма его работы. Во втором контексте будет описываться и тестироваться произвольный граф, в т.ч. и на генерацию соответствующих исключений.

Теперь файл `spec/acts_as_dagraph_spec.rb` будет выглядеть следующим образом:
```ruby
# spec/acts_as_dagraph_spec.rb

RSpec.describe "ActsAsDagraph" do

  let(:unit) { create(:unit) }
  let(:unit_2) { create(:unit) }

  it "has a valid Unit factory" do
    expect(unit).to be_valid
  end

  context "when graph is empty" do
    describe "#add_child" do
      subject { unit.add_child(unit_2) }
      it "creates an instance of Edge" do
        expect { subject }.to change{ Dagraph::Edge.count }.by(1)
      end
    end

    describe "#add_parent" do
      subject { unit.add_parent(unit_2) }
      it "creates an instance of Edge" do
        expect{ subject }.to change{ Dagraph::Edge.count }.by(1)
      end
    end
  end

  context "when graph is arbitrary" do
    describe "#add_child" do
      it "raises exception if add self node" do
        expect {
          unit.add_child(unit)
        }.to raise_error(SelfCyclicError)
      end
    end
    describe "#add_parent" do
      it "raises exception if add self node" do
        expect {
          unit.add_parent(unit)
        }.to raise_error(SelfCyclicError)
      end
    end
  end
end
```
Обновленный файл не должен нарушить прохождение тестов, но лучше проверить:
```bash
rspec spec

Randomized with seed 3999
.....

Finished in 0.3884 seconds (files took 1.35 seconds to load)
5 examples, 0 failures

Randomized with seed 3999
```

Добавляем проверку создания экземпляра класса `Dagraph::Route` в `#add_child` и `#add_parent` первого контекста (`context "when graph is empty"`):

```ruby
it "creates an instance of Route" do
  expect{ subject }.to change{ Dagraph::Route.count }.by(1)
end
```
Пусть пока нас не смущает дублирование кода этого теста в проверках методов `add_parent()` и `add_child()`. Позже мы приведем наши код в соответствие с рекомендациям DRY.

Запускаем проверку и получаем ожидаемые ошибки
```bash
rspec spec

Randomized with seed 56199
..FF...

Failures:

  1) ActsAsDagraph when graph is empty #add_parent creates an instance of Route
     Failure/Error: expect{ subject }.to change{ Dagraph::Route.count }.by(1)
     
     NameError:
       uninitialized constant Dagraph::Route
     # ./spec/acts_as_dagraph_spec.rb:32:in `block (5 levels) in <top (required)>'
     # ./spec/acts_as_dagraph_spec.rb:32:in `block (4 levels) in <top (required)>'

  2) ActsAsDagraph when graph is empty #add_child creates an instance of Route
     Failure/Error: expect{ subject }.to change{ Dagraph::Route.count }.by(1)
     
     NameError:
       uninitialized constant Dagraph::Route
     # ./spec/acts_as_dagraph_spec.rb:20:in `block (5 levels) in <top (required)>'
     # ./spec/acts_as_dagraph_spec.rb:20:in `block (4 levels) in <top (required)>'

Finished in 0.40784 seconds (files took 1.27 seconds to load)
7 examples, 2 failures

Failed examples:

rspec ./spec/acts_as_dagraph_spec.rb:31 # ActsAsDagraph when graph is empty #add_parent creates an instance of Route
rspec ./spec/acts_as_dagraph_spec.rb:19 # ActsAsDagraph when graph is empty #add_child creates an instance of Route

Randomized with seed 56199
```

#### 2. Реализация  модели `Dagraph::Route`

Добавляем таблицу `dagraph_routes` в шаблон для миграции
```ruby
# lib/generators/dagraph/templates/create_dagraph.rb
class CreateDagraph < ActiveRecord::Migration 
  def change 
    create_table :dagraph_edges do |t| 
      ...
    end 
    create_table :dagraph_routes do |t|
      t.timestamps null: false
    end
  end 
end
```
Теперь необходимо заново загрузить миграцию `create_dagraph` в наше тестовое приложение. 
```bash
cd spec/test_app

bin/rake db:rollback # откатываем текущую миграцию create_dagraph
== 20160727064728 CreateDagraph: reverting ====================================
-- drop_table(:dagraph_edges)
   -> 0.0002s
== 20160727064728 CreateDagraph: reverted (0.0023s) ===========================

bin/rails generate dagraph:migration --force # заменяем файл миграции новым
      remove  db/migrate/20160727064728_create_dagraph.rb
      create  db/migrate/20160727065210_create_dagraph.rb

bin/rake db:migrate # применяем новую миграцию
== 20160727065210 CreateDagraph: migrating ====================================
-- create_table(:dagraph_edges)
   -> 0.0033s
-- create_table(:dagraph_routes)
   -> 0.0003s
== 20160727065210 CreateDagraph: migrated (0.0037s) ==========================

bin/rake db:test:prepare # подготавливаем тестовую БД
```
В выводе предпоследней команды видим, что создаются 2 таблицы: `dagraph_edges` и `dagraph_routes`

Описываем модель `Dagraph::Route`
```ruby
# lib/dagraph/route.rb
module Dagraph
  class Route < ActiveRecord::Base 
    self.table_name = "dagraph_routes"
  end
end
```
Корректируем методы `add_child()` и `add_parent()` модуля `ActsAsDagraph`:
```ruby
# lib/dagraph/acts_as_dagraph.rb
...
  def add_child(node)
    ...
    Route.create
  end
  
  def add_parent(node)
    ...
    Route.create
  end
```

Добавляем ссылку на `Dagraph::Route` 
```ruby
# lib/dagraph.rb
...
require "dagraph/route.rb"
```
Повторно запускаем тесты и убеждаемся что всё отрабатывает корректно
```bash
rspec spec

Randomized with seed 14775
.......

Finished in 0.38683 seconds (files took 1.27 seconds to load)
7 examples, 0 failures

Randomized with seed 14775
```

#### 3. Проверка создания и реализация модели `Dagraph::RouteNodes`

Добавляем новые тесты в `add_parent()` и `add_child()`

```ruby
# spec/acts_as_dagraph_spec.rb
...
	it "creates 2 instances of RouteNodes" do
	 expect{ subject }.to change{ Dagraph::RouteNode.count }.by(2)
	end
```
Запускаем проверку и получаем ошибки
```bash
rspec spec/
```
Корректируем файл миграции
```ruby
# lib/generators/dagraph/templates/create_dagraph.rb
...
create_table :dagraph_route_nodes do |t|
  t.integer :route_id, index: true, null: false
  t.references :node, polymorphic: true, index: true
  t.integer :level
end
```
Перезагружаем миграцию в тестовом приложении

```bash
cd spec/test_app
bin/rake db:rollback
bin/rails generate dagraph:migration --force
bin/rake db:migrate
bin/rake db:test:prepare
```
Создаем файл для модели `Dagraph::RouteNode`

```ruby
# lib/dagraph/route_node.rb
module Dagraph
  class RouteNode < ActiveRecord::Base
    self.table_name = "dagraph_route_nodes"
    belongs_to :node, polymorphic: true
    belongs_to :route, class_name:  "Dagraph::Route"
  end
end
```
Корректируем файл `Dagraph::Route`

```ruby

module Dagraph
  class Route < ActiveRecord::Base 
    self.table_name = "dagraph_routes"
    has_many :route_nodes, class_name:  "Dagraph::RouteNode", 
	    foreign_key: "route_id", 
	    dependent: :destroy
    alias nodes route_nodes
  end
end
```

Расширяем поведение методов `add_parent()` и `add_child()`

```ruby
# lib/dagraph/acts_as_dagaph.rb
...
module LocalInstanceMethods
  def add_child(node) 
    raise SelfCyclicError.exception("Must not add node to itself") if node == self
    Edge.create(dag_parent: self, dag_child: node)
    route = Route.create
    route.nodes.create(node: self, level: 0)
    route.nodes.create(node: node, level: 1)
  end

  def add_parent(node)
    raise SelfCyclicError.exception("Must not add node to itself") if node == self
    Edge.create(dag_parent: node, dag_child: self)
    route = Route.create
    route.nodes.create(node: node, level: 0)
    route.nodes.create(node: self, level: 1)
  end
end
```
Добавляем ссылку на `Dagraph::Route` 
```ruby
# lib/dagraph.rb
...
require "dagraph/route_node.rb"
```
Повторно запускаем тесты
```bash
rspec spec/
...
Finished in 0.51824 seconds (files took 1.64 seconds to load)
9 examples, 0 failures
```
