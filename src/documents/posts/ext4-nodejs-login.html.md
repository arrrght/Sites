---
title: Ext4. NodeJS. Login.
layout: post
created_at: 21 мая 2011
tags: ['post']
urls: ['ext4-nodejs-login.html']
kdpv: '/images/2011-02-16-1.png'
intro: Создаем окно входа используя Ext4, NodeJS, Express, Cucumis, MongoDB. Постараемся уменьшить количество кода. Так же попробуем частично объединить серверную и клиентскую части кода и протестировать их в cucumis.
---
Создаем окно входа используя Ext4, NodeJS, Express, Cucumis, MongoDB. Постараемся уменьшить количество кода.
Так же попробуем частично объединить серверную и клиентскую части кода и протестировать их в cucumis.
<img src="/images/2011-05-23-1.png" />

Готовые исходники лежат на [GitHub](https://github.com/arrrght/h2e4-show-1)

Лучше сразу смотреть там, я буду давать ссылки на эти исходники и комментировать по мере прохождения.

- По мотивам
  - [Test express app using cucumis](http://node-js.ru/14-test-express-app-using-cucumis)
  - [Ext4 MVC](http://docs.sencha.com/ext-js/4-0/#/api/Ext.app.Application)

Создаем каталог и структуру

```
mkdir h2e4-show-1 && cd h2e4-show-1
express -s -t ejs
   create : .
   create : ./app.js
   create : ./pids
   create : ./public/images
   create : ./public/javascripts
   create : ./public/stylesheets
   create : ./public/stylesheets/style.css
   create : ./logs
   ...
```

Создаем структуру в каталоге app -> controller, model, store и view. Убираем лишнее.
```
$ mkdir lib && rm -R ./views
$ mkdir features
$ mkdir features/step_definitions
$ ln -s ~/Library/mylibs/ext-4.0.0 public/ext4
$ mkdir app && cd app
$ mkdir controller model store view
$ cd ..
```

Прибираемся в [app.js](https://github.com/arrrght/h2e4-show-1/blob/master/app.js) и подключаем mongo в качестве базу для хранения сессий:

```
var app = module.exports = express.createServer(),
    mongoStore = require('connect-mongodb');

app.use(express.session({ secret: 'your secret here',
  store: mongoStore({ dbname: 'tst01' }) }));
```

После конфигурации подключаем файл [h2e4.js](https://github.com/arrrght/h2e4-show-1/blob/master/lib/h2e4.js)

Далее описываем как будет проходить процесс авторизации:

```
function restrictUnauthorized(req, res, next){
  console.log('* User: ' + req.session.user);
  req.session.user_id ? next() : res.redirect('/login');
};

app.get('/login', function(req, res){
  res.render('Login.html');
});

app.get('/logout', function(req, res){
  delete req.session.user_id;
  res.redirect('/');
});

app.get('/', restrictUnauthorized, function(req, res){
  res.render('Application.html');
});
```

  - 01-04 Функция restrictUnauthorized ограничивает анонимов только приложением Login
  - 06-08 Показываем приложение Login на путь /login
  - 10-13 При /logout просто чистим сиссию и перенаправляем на /login
  - 15-17 Авторизованные идут в основное приложение

Т.е. на самом деле у нас два Ext4 MVC приложения - одно для логина(Login) и одно основное(App)

Так, начнем мы с Login.

Рассмотрим файл [app/Login.html](https://github.com/arrrght/h2e4-show-1/blob/master/app/Login.html):
```
<html>
  <head>
    <meta http-equiv="content-type" content="text/html; charset=utf-8"/>
    <link rel='stylesheet' href='/ext4/resources/css/ext-all.css'/>
    <script type='text/javascript' src='/ext4/bootstrap.js'></script>
    <script type='text/javascript' src='/direct/api'></script>
    <script type="text/javascript" charset="utf-8">
      Ext.Loader.setConfig({ enabled: true, disableCaching: false,
        paths: { 'App': '/app', 'Ext': '/ext4/src' } });
      Ext.application({ name: 'Login', controllers: [ 'Login' ] });
    </script>
  </head>
  <body></body>
</html>
```

  - 05 Здесь подключаем наш Ext4 Direct API
  - 08-10 Описываем где лежит наше приложение, а где струкура

Далее, пишем [Viewport](https://github.com/arrrght/h2e4-show-1/blob/master/app/view/Viewport.js), он у нас один на двоих, на самом деле 

```
Ext.define('Login.view.Viewport', {
    extend: 'Ext.window.Window',

    initComponent: function() {
      Ext.apply(this, {
        autoShow: true,
        title: 'Login window',
        width: 350, autoHeight: true,
        closable: false, resizable: false, bodyStyle: 'background-color:#DFE8F6; padding: 5px;',
        items: Ext.create('Ext.form.FormPanel', {
          border: false, bodyStyle: 'background-color:#DFE8F6;', 
          api: { submit: Login.login },
          buttons: [{ text: 'Login', action: 'login' }],
          defaultType: 'textfield',
          defaults: { anchor: '100%', allowBlank: false, msgTarget: 'side' },
          items: [
            { name: 'username', fieldLabel: 'User' },
            { name: 'password', fieldLabel: 'Password', inputType: 'password' }
          ]
        })
      });
      this.callParent(arguments);
    }
});

Ext.define('App.view.Viewport', {
    extend: 'Ext.container.Viewport',

    initComponent: function() {
      Ext.apply(this, {
        layout: 'fit',
        items: {
          xtype: 'panel',
          html: 'It\'s a private area, <%%= session.user %>.'
        }
      });
      this.callParent(arguments);
    }
});
```

  - 01-24 Рисуем окно входа - никаких обработок не делаем, ничего интересного, просто окно с двумя полями и кнопка
  - 13 Здесь мы определяем кто будет обрабатывать нашу форму: конкретно - Login.login, сейчас мы опишем его в контроллере
  - 26-39 Пустое окно с приветствием "Private area", здесь вообще ничего нет

Далее пишем контроллер [Login](https://github.com/arrrght/h2e4-show-1/blob/master/app/controller/Login.js):
```
Ext.define('Login.controller.Login', {
  extend: 'Ext.app.Controller',

  models: [ 'User' ],

  init: function(){
    this.control({
      'button[action=login]': {
        click: this.onLoginClick
      }
    })
  },

  onLoginClick: function(btn, e){
    var form = btn.up('form');
    // Call endpoint (defined in view(Viewport/FormPanel)
    // as DirectAPI: "api: { submit: Login.login }" )
    form.submit({
      success: function(){
        window.location = '/';
      }
    });
  }
});

// Server side code
Ext.endpoint('login', function(para){ //:: { "formHandler": "true" }
                                      // ^^^ look at this comment
  var ret = this,
      User = Mongoose.model('User');

  // Call mongoose model 'auth'
  User.auth(para.username, para.password, function(user){
    if (user){
      // If auth success, set session var and return login
      ret.app.req.session.user_id = user.id;
      ret.app.req.session.user = user.login;
      ret.success({ login: user.login, id: user.id });
    }else{
      // If fails - say it
      ret.failure({ username: 'User not found!' });
    }
  });
  
  // Create one user with password if not exist
  User.findOne({ login: 'user'}, function(err, user){
    if (!user){
      // Look at password magic in model
      new User({ login: 'user', password: 'pass' }).save();
    }
  });
});
```

Вот. Здесь интереснее:

  - 01-24  Сам контроллер, клиентская часть 
  - 04 - Подключаем модель "User":https://github.com/arrrght/h2e4-show-1/blob/master/app/model/user.js
  - 18-22 Отлавливаем нажатие кнопки, отправляем серверу и если все хорошо делаем редирект на корень
  - 27-52 Серверная часть
  - 27 Описание точки входа, в комментах пишем что это обработчик формы
  - 33-43 Вызываем функцию auth, которая задается в модели(скоро дойдем и до нее), и если все хорошо, то возвращаем клиенту login и id, если нет - маппим на поле username ошибку. Ext4 сам разберется что с ней делать.
  - 46-51 Создем юзера с логином user и паролем pass (так, на всякий случай, чтобы не лазить в mongo лишний раз)

И, напоследок, модель [User](https://github.com/arrrght/h2e4-show-1/blob/master/app/model/user.js)

```
// Code for ext4
Ext.define('Login.model.User', {
  extend: 'Ext.data.Model',

  fields: [
    { name: 'login', type: 'string' },
    { name: 'password', type: 'string' },
  ]

});

// Code for Mongoose
var Schema = Mongoose.Schema,
    crypto = require('crypto'),
    ObjectId = Schema.ObjectId,
    Query = Mongoose.Query,
    userFields = Ext.getModel('User');

/* We can define whole shema
var User = new Schema({
  login: String,
  pswd: { salt: String , hash: String }
});
*/

// .. or just change some fields
delete userFields.password; // virtual it
userFields.pswd = { salt: String , hash: String };

var User = new Schema(userFields);

// virtual field
User.virtual('password')
  .set(function(pass){
    var salt = (function getSalt(){
      var str = '',
          chars = '0123456789ABCDEFGHIJKLMNOPQRSTUVWXTZabcdefghiklmnopqrstuvwxyz'.split('');

      chars.forEach(function(){ str += chars[Math.floor(Math.random() * chars.length)] });
      return str;
    })();
    this.set('pswd.salt', salt);
    this.set('pswd.hash', getCrypted(salt, pass));
  })
  .get(function(){
    return this.pswd.hash
  });

function getCrypted(salt, pass){
  return crypto.createHmac('sha1', salt).update(pass).digest('hex')
};

User.static({
  auth: function(login, password, callback){
    this.findOne({ login: login }, function(err, user){
      if (user && user.password === getCrypted(user.pswd.salt, password)){
        callback(user);
      }else{
        callback(null);
      }
    });
  }
});

// Register model at last
Mongoose.model('User', User);
```

Здесь еще интереснее, вкусности mongodb в действии:

  - 01-10 Клиентская часть модели
  - 19-30 Описание серверной части модели для mongo. Можно как воспользоваться автоматическим преобразованием полей(простейщим 'string'->String, 'float'->Number), или создать с самомго начала базу для mongo. Скажем так: если модель простейшая, то серверная часть не пишется вообще. (PS: Хотя нет, все-таки придется сделать Mongoose.model('User', new Schema(userFields)); ) - подумаю как сделать даже без этого, не потеряв в гибкости.
  - 32-46 Храним правильно пароль, для этого делаем виртуальное поле password, которое hash'им до потери пульса. Для этого раздваиваем поле с паролем, каждый раз генерим хеш заново и храним вместе с хешированным паролем. Кажется, так правильно.
  - 52-62 Процедура для аутентификации, возвращающая null если не ничего нет и user если все совпало.

Вот, по сути и все описание. Я постарался максимально комментировать исходники. В процессе создания библиотека h2e4.js (Helpers For Ext4) будет прирастать новым функционалом. Очень хочется сделать сделать общую [валидацию](http://mongoosejs.com/docs/validation.html).

Тестирование.

На самом деле, все надо делать еще в процессе создания программы, чем надеюсь Вы дальше и займетесь.

Запускаем 

```
java -jar selenium-server-standalone-2.0b3.jar
```

... Не дописал тесты. Напишу в следующий раз.

*PS: на дворе 2013 год, тесты так и не написал.*