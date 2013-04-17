---
title: Автотесты Netzke в браузере
created_at: 2011-02-11 19:17:22.524516 +05:00
layout: post
tags: ['post']
---
Создание начальной оболочки для тестирования формы входа. Написана с высоты двухнедельного изучения ruby и Netzke. Сдалано топорно.
<img src="/images/2011-02-16-1.png" />

По мотивам
* "Testing Ext JS/Rails components with Cucumber and WebDriver in Netzke":http://blog.writelesscode.com/blog/2011/01/27/testing-extjs-rails-components-with-cucumber-and-webdriver-in-netzke/
* "Rails: Хватит отмазываться, начинаем BDD-ить!":http://habrahabr.ru/blogs/webdev/111480

Весь код примера доступен на "GitHub":https://github.com/arrrght/netzke-test-1

Создаем новый проект

<% coderay :lang => "bash" do %>
$ rails new netzke-test-1
    create  
    create  README
    create  Rakefile
    create  config.ru
    create  .gitignore
    ...
$ cd netzke-test-1
<% end %>

Добавляем gems в файл Gemfile:

<% coderay :lang => "ruby" do -%>
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
<% end -%>

... и устанавливаем их:

<% coderay :lang => "bash" do %>
$ bundle install
Updating git://github.com/skozlov/netzke-core.git
Updating git://github.com/skozlov/netzke-basepack.git
Fetching source index for http://rubygems.org/
Using rake (0.8.7) 
Using abstract (1.0.0) 
Using activesupport (3.0.3) 
Using builder (2.1.2) 
...
<% end %>
	
Делаем линки на скаченные библиотеки "ExtJS":http://www.sencha.com/products/extjs/download/ и "иконки":http://www.famfamfam.com/lab/icons/silk/:

<% coderay :lang => "bash" do %>
$ ln -s ~/Library/mylibs/ext-3.3.1 public/extjs
$ ln -s ~/Library/mylibs/famfamfam_silk_icons_v013/icons public/images/
<% end %>

Добавляем в <tt>/app/views/layout/application.rb</tt>

<% coderay :lang => "ruby" do -%>
<head>
  <title>NetzkeTest1</title>
  <%%= netzke_init %>
  <%%= csrf_meta_tag %>
</head>
<% end -%>


Генерим оболочку:

<% coderay :lang => "bash" do %>
$ rails g rspec:install
$ rails g cucumber:install --rspec --capybara
<% end %>

Делаем чтобы тесты выполнялись красиво:

<% coderay :lang => "bash" do %>
$ echo '--colour --format documentation' > .rspec
$ sed -i~ 's/progress/pretty/g' config/cucumber.yml
<% end %>

Прописываем драйвер в тесты, для этого вписываем в <tt>features/support/env.rb</tt> следующие строки:

<% coderay :lang => "ruby" do -%>
Capybara.register_driver :selenium do |app|
  Capybara::Driver::Selenium.new app, :browser => :chrome
end
<% end -%>

Пишем фичу в файле <tt>features/login.feature</tt>

<% coderay :lang => "ruby", :line_numbers => "inline" do -%>
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
<% end -%>

Что мы делаем:
В первом сценарии мы проверяем окно, которое должно появится, два поля и отключенная кнопка "Enter", которая должна включится только если в первом и во втором поле что-либо есть.
И второй сценарий - при верных параметрах мы попадаем на страницу приветствия пользователя, а при неверных - сообщение об ошибке.
PS: Тест написано криво и его надо причесать, но для примера пойдет.
PPS: Понаставил пауз потому как не знаю как же дожидаться выполнения работы JS.

Запускаем тест <tt>cucumber features</tt> и видим кучу ошибок. Самая первая из них:

<% coderay :lang => 'bash' do %>
Can't find mapping from "the login page" to a path.
<% end %>

Генерируем контроллер и добавляем action index

<% coderay :lang => 'bash' do %>
$ rails g controller login index
    create  app/controllers/login_controller.rb
    invoke  erb
    create    app/views/login
    ...
<% end %>

Добавляем default route к нашей странице в файле <tt>config/routes.rb</tt>

<% coderay :lang => 'ruby' do %>
netzke
match "login" => "login#index"
<% end %>

Запускаем наш тест и видим, что он запнулся на том, что не видит окно.

<% coderay :lang => 'bash' do %>
Then I should see "Login window"
<% end %>

Это нормально, потому что мы его еще не написали.
Теперь рисуем наше окно для входа в файле <tt>app/views/login/index.html.erb</tt>

<% coderay :lang => 'ruby' do %>
<%%= netzke :login %>	
<% end %>

И сам компонент. Для этого создаем католог <tt>app/components</tt> и файл <tt>app/components/login.rb</tt>:

<% coderay(:lang => "ruby", :line_numbers => "inline") do -%>
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
<% end -%>

Строки 6-29: Описание окна.
Строка 31: Добавление action типа "кнопка" и привязывание к форме
Строки 33-40: Отправка формы на сервер
Строки 42-50: Проверка пароля на строне сервера (так делать нельзя - это только пример)
Строки 52-60: Приемка ответа от сервера и решения что же дальше

Теперь cucumber ругается на непонятный ему шаг:
	
<% coderay :lang => 'bash' do %>
Undefined step: "button "Enter" should be disabled" (Cucumber::Undefined)
<% end %>

Отлично! Так давайте его и напишем в файле <tt>features/step_definitions/my_netzke_steps.rb</tt>:

<% coderay :lang => 'ruby' do %>
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
<% end %>

У нас готов первый полностью пройденный сценарий, на котором мы проверям окно входа и неактивную кнопку. Идем дальше. 

<% coderay :lang => 'bash' do %>
Undefined step: "I have a user named "user1" with password "pass1""
<% end %>

Система не знает как сделать пользователя - так подскажем как это делается. Добавляем описание этого шага в <tt>features/step_definitions/my_netzke_steps.rb</tt>

<% coderay :lang => 'ruby' do %>
Given /^I have a user named "([^"]*)" with password "([^"]*)"$/ do |arg1, arg2|
  User.create!( :login => arg1, :password => arg2 )
end
<% end %>

OMG! У нас нет модели! Cucumber говорит что <tt>uninitialized constant User (NameError)</tt>. Создадим:

<% coderay :lang => 'bash' do %>
$ rails g model User login:string password:string
    invoke  active_record
    create    db/migrate/20110214075142_create_users.rb
    create    app/models/user.rb
    invoke    rspec
    create      spec/models/user_spec.rb
$ rake db:migrate
$ rake db:test:prepare
...
<% end %>

Осталось два последних шага, которых не понимает cucumber: <tt>When I wait for the response from the server</tt> и <tt>I sleep (\d+) seconds</tt>. Напишем их в <tt>features/step_definitions/my_netzke_steps.rb</tt>

<% coderay :lang => 'ruby' do %>
When /^I wait for the response from the server$/ do
  page.wait_until{ page.driver.browser.execute_script("return !Ext.Ajax.isLoading();") }
end

When /I sleep (\d+) seconds?/ do |arg1|
  sleep arg1.to_i
end
<% end %>

И наконец-то cucumber наткнулся <tt>Then I should see "Hello, user1!"</tt> на нашу последнюю недописанную часть - то, что должно появится после входа в систему. Для этого создадим котроллер <tt>users</tt>, в котором приветствуем пользователя:

<% coderay :lang => 'bash' do -%>
$ rails g controller users index
<% end -%>

И default route <tt>config/routes.rb</tt>

<% coderay :lang => 'bash' do -%>
match "users" => "users#index"
<% end -%>

И напишем в <tt>app/views/users/index.html.erb</tt>:

<% coderay :lang => "ruby" do -%>
Hello, <%%= User.find(session[:user_id]).login %>!
<% end -%>

Всё. Тесты пройдены - вход работет.
