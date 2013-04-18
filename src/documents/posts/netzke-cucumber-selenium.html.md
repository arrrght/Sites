---
title: Автотесты Netzke в браузере
created_at: 2011-02-11 19:17:22.524516 +05:00
layout: post
tags: ['post']
---
Создание начальной оболочки для тестирования формы входа. Написана с высоты двухнедельного изучения ruby и Netzke. Сдалано топорно.
<img src="/images/2011-02-16-1.png" />

По мотивам

  - [Testing Ext JS/Rails components with Cucumber and WebDriver in Netzke](http://blog.writelesscode.com/blog/2011/01/27/testing-extjs-rails-components-with-cucumber-and-webdriver-in-netzke/)
  - [Rails - Хватит отмазываться, начинаем BDD-ить!](http://habrahabr.ru/blogs/webdev/111480)

Весь код примера доступен на [GitHub](https://github.com/arrrght/netzke-test-1)

Создаем новый проект

```
$ rails new netzke-test-1
    create  
    create  README
    create  Rakefile
    create  config.ru
    create  .gitignore
    ...
$ cd netzke-test-1
```

Добавляем gems в файл Gemfile:

```
source 'http://rubygems.org'

gem 'rails', '3.0.3'

gem 'sqlite3-ruby', :require => 'sqlite3'
gem 'netzke-core', :git => "git://github.com/skozlov/netzke-core.git"
gem 'netzke-basepack', :git => "git://github.com/skozlov/netzke-basepack.git"
gem 'will_paginate'

group :development, :test do
  gem "cucumber-rails"
  gem "rspec-rails"
end

group :test do
  gem "capybara"
  gem "launchy"
  gem "database_cleaner"
  gem "selenium-webdriver"
end
```

... и устанавливаем их:

```
$ bundle install
Updating git://github.com/skozlov/netzke-core.git
Updating git://github.com/skozlov/netzke-basepack.git
Fetching source index for http://rubygems.org/
Using rake (0.8.7) 
Using abstract (1.0.0) 
Using activesupport (3.0.3) 
Using builder (2.1.2) 
...
```
	
Делаем линки на скаченные библиотеки [ExtJS](http://www.sencha.com/products/extjs/download/) и [иконки](http://www.famfamfam.com/lab/icons/silk/):

```
$ ln -s ~/Library/mylibs/ext-3.3.1 public/extjs
$ ln -s ~/Library/mylibs/famfamfam_silk_icons_v013/icons public/images/
```

Добавляем в ```/app/views/layout/application.rb```
```
<head>
  <title>NetzkeTest1</title>
  <%%= netzke_init %>
  <%%= csrf_meta_tag %>
</head>
```


Генерим оболочку:
```
$ rails g rspec:install
$ rails g cucumber:install --rspec --capybara
```

Делаем чтобы тесты выполнялись красиво:
```
$ echo '--colour --format documentation' > .rspec
$ sed -i~ 's/progress/pretty/g' config/cucumber.yml
```

Прописываем драйвер в тесты, для этого вписываем в ```features/support/env.rb``` следующие строки:
```
Capybara.register_driver :selenium do |app|
  Capybara::Driver::Selenium.new app, :browser => :chrome
end
```

Пишем фичу в файле ```features/login.feature```

```
Feature: Check login window
In order to value
As a role
I want feature

@javascript
Scenario: Check window properties (and button Enter should be disabled by default)
  Given I am on the login page
  Then I should see "Login window"
  And I should see "User"
  And I should see "Password"
  Then button "Enter" should be disabled
  When I fill in "username" with "some"
  And I fill in "password" with "somepass"
  And I sleep 1 seconds
  Then button "Enter" should be enabled

@javascript
Scenario: Check login
  Given I have a user named "user1" with password "pass1"
  And I am on the login page
  When I fill in "username" with "wrong_user"
  And I fill in "password" with "wrong_password"
  And I press "Enter"
  When I wait for the response from the server
  And I sleep 1 seconds
  Then I should see "Wrong!"
  When I fill in "username" with "user1"
  And I fill in "password" with "pass1"
  And I press "Enter"
  When I wait for the response from the server
  And I sleep 1 seconds
  Then I should see "Hello, user1!"
```

Что мы делаем:

В первом сценарии мы проверяем окно, которое должно появится, два поля и отключенная кнопка "Enter", которая должна включится только если в первом и во втором поле что-либо есть.

И второй сценарий - при верных параметрах мы попадаем на страницу приветствия пользователя, а при неверных - сообщение об ошибке.

PS: Тест написано криво и его надо причесать, но для примера пойдет.

PPS: Понаставил пауз потому как не знаю как же дожидаться выполнения работы JS.

Запускаем тест ```cucumber features``` и видим кучу ошибок. Самая первая из них:
```
Can't find mapping from "the login page" to a path.
```

Генерируем контроллер и добавляем ```action index```
```
$ rails g controller login index
    create  app/controllers/login_controller.rb
    invoke  erb
    create    app/views/login
    ...
```

Добавляем default route к нашей странице в файле ```config/routes.rb```
```
netzke
match "login" => "login#index"
```

Запускаем наш тест и видим, что он запнулся на том, что не видит окно.
```
Then I should see "Login window"
```

Это нормально, потому что мы его еще не написали.
Теперь рисуем наше окно для входа в файле ```app/views/login/index.html.erb```
```
<%%= netzke :login %>	
```

И сам компонент. Для этого создаем католог ```app/components``` и файл ```app/components/login.rb```:
```
# coding: utf-8
class Login < Netzke::Base

  js_base_class 'Ext.Window'

  def configuration
    super.merge(
      :title => 'Login window',
      :hidden => false,
      :width => 350, :autoHeight => true,
      :closable => false, :resizable => false,
      :items => [{
        :buttons => [ :login.action ],
        :xtype => :form,
        :frame => true,
        :defaultType => :textfield,
        :monitorValid => true,
        :defaults => { :anchor => '100%', :allowBlank => false },
        :items => [{
          :name => 'username',
          :fieldLabel => 'User'
        },{
          :name => 'password',
          :fieldLabel => 'Password',
          :inputType => :password
        }]
      }]
    )
  end

  action :login, :text => 'Enter', :icon => :accept, :formBind => true
  
  js_method :on_login, <<-JS.l
    function(){
      var form = this.items.first().getForm();
      if (form.isValid()){
        this.login(form.getFieldValues());
      }
    }
  JS

  endpoint :login do |para|
    user = User.find(:first, :conditions => {
      :login => para['username'],
      :password => para['password']
    })
    session[:user_id] = user.id unless user.nil?
    { :get_answer => !user.nil?  }
  end

  js_method :get_answer, <<-JS.l
    function (ret){
      if (ret == false) {
        Ext.Msg.alert('Wrong!','Wrong username or password');
      }else{
        window.location = '/users';
      }
    }
  JS
  end
```

  - Строки 6-29: Описание окна.
  - Строка 31: Добавление action типа "кнопка" и привязывание к форме
  - Строки 33-40: Отправка формы на сервер
  - Строки 42-50: Проверка пароля на строне сервера (так делать нельзя - это только пример)
  - Строки 52-60: Приемка ответа от сервера и решения что же дальше


Теперь cucumber ругается на непонятный ему шаг:
```
Undefined step: "button "Enter" should be disabled" (Cucumber::Undefined)
```

Отлично! Так давайте его и напишем в файле ```features/step_definitions/my_netzke_steps.rb```:
```
Then /^button "([^"]*)" should be enabled$/ do |arg1|
  page.driver.browser.execute_script(<<-JS).should == true
    var btn = Ext.ComponentMgr.all.filter('text', '#{arg1}').filter('type','button').first();
  return typeof(btn)!='undefined' ? !btn.disabled : false
  JS
end

Then /^button "([^"]*)" should be disabled$/ do |arg1|
  page.driver.browser.execute_script(<<-JS).should == true
    var btn = Ext.ComponentMgr.all.filter('text', '#{arg1}').filter('type','button').first();
    return typeof(btn)!='undefined' ? btn.disabled : false
  JS
end
```

У нас готов первый полностью пройденный сценарий, на котором мы проверям окно входа и неактивную кнопку. Идем дальше. 
```
Undefined step: "I have a user named "user1" with password "pass1""
```

Система не знает как сделать пользователя - так подскажем, как это делается. Добавляем описание этого шага в ```features/step_definitions/my_netzke_steps.rb```
```
Given /^I have a user named "([^"]*)" with password "([^"]*)"$/ do |arg1, arg2|
  User.create!( :login => arg1, :password => arg2 )
end
```

OMG! У нас нет модели! Cucumber говорит что ```uninitialized constant User (NameError)```. Создадим:
```
$ rails g model User login:string password:string
    invoke  active_record
    create    db/migrate/20110214075142_create_users.rb
    create    app/models/user.rb
    invoke    rspec
    create      spec/models/user_spec.rb
$ rake db:migrate
$ rake db:test:prepare
...
```

Осталось два последних шага, которых не понимает cucumber: ```When I wait for the response from the server``` и ```I sleep (\d+) seconds```. Напишем их в ```features/step_definitions/my_netzke_steps.rb```
```
When /^I wait for the response from the server$/ do
  page.wait_until{ page.driver.browser.execute_script("return !Ext.Ajax.isLoading();") }
end

When /I sleep (\d+) seconds?/ do |arg1|
  sleep arg1.to_i
end
```

И наконец-то cucumber наткнулся ```Then I should see "Hello, user1!"``` на нашу последнюю недописанную часть - то, что должно появится после входа в систему. Для этого создадим котроллер ```users```, в котором приветствуем пользователя:
```
$ rails g controller users index
```

И default route ```config/routes.rb```
```
match "users" => "users#index"
```

И напишем в ```app/views/users/index.html.erb```:
```
Hello, <%%= User.find(session[:user_id]).login %>!
```

Всё. Тесты пройдены - вход работет.
