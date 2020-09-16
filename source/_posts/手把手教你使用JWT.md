---
title: 手把手教你使用JWT
date: 2018-08-29 19:01:17
tags:
- PHP
- JWT
- Laravel
---

实习期间比较忙，好久没有写博客了，今天做完了项目来写一写。

这篇博客介绍的东西是JWT（json web token的简称）。用户通过JWT，就可以登录到我们的系统。OK，接下来让菜鸡我来给小伙伴们演示一下，以PHP的Laravel框架为例子，并且通过jwt-auth这个库来进行演示。这个库的官方文档[这这里](https://jwt-auth.readthedocs.io/en/develop/)。

## 准备环境

为了能够跑我们的项目，我们需要一个可以跑Laravel项目的环境，这里我就直接使用一个可以跑Laravel项目的Docker镜像，小伙伴们可以自己去找比较流行的镜像。

我这里Laravel框架的版本是5.6的，大家也尽量使用5.6的版本，和我的保持一致。

（注：如果小伙伴们自己的电脑已经可以跑Laravel项目了，**准备环境**这一步就可以省略了）

## 准备一个空的Laravel项目

准备一个空的Laravel项目，项目名字叫做**jwt-test**：

```shell
sudo composer create-project --prefer-dist laravel/laravel jwt-test
```

## 准备jwt-auth库

安装jwt-auth这个库：

```shell
sudo composer require tymon/jwt-auth
```

然后，我们打开jwt-test项目下的composer.json文件，找到如下地方：

![](http://oklbfi1yj.bkt.clouddn.com/%E6%89%8B%E6%8A%8A%E6%89%8B%E6%95%99%E4%BD%A0%E4%BD%BF%E7%94%A8JWT/1.png)

然后把

```json
"tymon/jwt-auth": "^0.5.12"
```

换成

```json
"tymon/jwt-auth": "^1.0.0-rc.2"
```

（注意，一定要记得改这里的版本，要不然后面的发布配置会出问题）

然后在命令行中执行命令：

```shell
sudo composer update
```

然后再发布配置：

```shell
php artisan vendor:publish --provider="Tymon\JWTAuth\Providers\LaravelServiceProvider"
```

![](http://oklbfi1yj.bkt.clouddn.com/%E6%89%8B%E6%8A%8A%E6%89%8B%E6%95%99%E4%BD%A0%E4%BD%BF%E7%94%A8JWT/2.png)

此时，会在`jwt-test/config`目录下多出一个文件`jwt.php`（这个文件保存了一些和jwt有关的默认配置信息），并且，我们执行`php artisan`命令的时候，也会多出一个命令：

![](http://oklbfi1yj.bkt.clouddn.com/%E6%89%8B%E6%8A%8A%E6%89%8B%E6%95%99%E4%BD%A0%E4%BD%BF%E7%94%A8JWT/3.png)

然后生成密钥：

```shell
sudo php artisan jwt:secret
```

![](http://oklbfi1yj.bkt.clouddn.com/%E6%89%8B%E6%8A%8A%E6%89%8B%E6%95%99%E4%BD%A0%E4%BD%BF%E7%94%A8JWT/5.png)

此时，在`jwt-test/.env`文件中会有这个密钥：

![](http://oklbfi1yj.bkt.clouddn.com/%E6%89%8B%E6%8A%8A%E6%89%8B%E6%95%99%E4%BD%A0%E4%BD%BF%E7%94%A8JWT/6.png)

（注：我后面会解释这个密钥是用来干嘛的）

然后，我们修改一下Laravel框架默认给我们生成的User.php文件的内容：

```php
<?php

namespace App;

use Tymon\JWTAuth\Contracts\JWTSubject;
use Illuminate\Notifications\Notifiable;
use Illuminate\Foundation\Auth\User as Authenticatable;

class User extends Authenticatable implements JWTSubject
{
    use Notifiable;

    /**
     * The attributes that are mass assignable.
     *
     * @var array
     */
    protected $fillable = [
        'username', 'password',
    ];

    /**
     * The attributes that should be hidden for arrays.
     *
     * @var array
     */
    protected $hidden = [
        'password', 'remember_token',
    ];

    /**
     * Get the identifier that will be stored in the subject claim of the JWT.
     *
     * @return mixed
     */
    public function getJWTIdentifier()
    {
        return $this->getKey();
    }

    /**
     * Return a key value array, containing any custom claims to be added to the JWT.
     *
     * @return array
     */
    public function getJWTCustomClaims()
    {
        return [];
    }
}

```

然后，我们再修改一下`jwt-test/config/auth.php`文件里面的内容：

```php
'defaults' => [
    'guard' => 'web',
    'passwords' => 'users',
],
```

改成

```php
'defaults' => [
    'guard' => 'api',
    'passwords' => 'users',
],
```

然后

```php
'api' => [
    'driver' => 'token',
    'provider' => 'users',
],
```

改成

```php
'api' => [
    'driver' => 'jwt',
    'provider' => 'users',
],
```

## 模拟用户数据

因为Laravel框架为我们准备好了一张users表的迁移表，所以我们就不创建新的users迁移表了。我们直接在框架提供的迁移表上进行修改。

```php
public function up()
{
    Schema::create('users', function (Blueprint $table) {
        $table->increments('id');
        $table->string('username');
        $table->string('password');
        $table->timestamps();
    });
}
```

![](http://oklbfi1yj.bkt.clouddn.com/%E6%89%8B%E6%8A%8A%E6%89%8B%E6%95%99%E4%BD%A0%E4%BD%BF%E7%94%A8JWT/7.png)

然后`jwt-test/User.php`文件我们也修改一下：

```php
protected $fillable = [
    'username', 'password',
];
```

![](http://oklbfi1yj.bkt.clouddn.com/%E6%89%8B%E6%8A%8A%E6%89%8B%E6%95%99%E4%BD%A0%E4%BD%BF%E7%94%A8JWT/8.png)

然后`database/UserFactory.php`文件我们也修改一下：

```php
$factory->define(App\User::class, function (Faker $faker) {
    return [
        'username' => $faker->name,
        'password' => bcrypt('123456'), // 123456
    ];
});
```

![](http://oklbfi1yj.bkt.clouddn.com/%E6%89%8B%E6%8A%8A%E6%89%8B%E6%95%99%E4%BD%A0%E4%BD%BF%E7%94%A8JWT/9.png)

然后，我们在`.env`文件中配置一下我们的数据库：

```
DB_DATABASE=jwt
DB_USERNAME=USERNAME
DB_PASSWORD=PASSWORD
```

（注意：这里的DB_USERNAME和DB_PASSWORD根据小伙伴们的自身情况填写）

然后，我们用MySQL客户端连接上MySQL服务器，创建一个数据库，命名为`jwt`：

```
create database jwt;
```

![](http://oklbfi1yj.bkt.clouddn.com/%E6%89%8B%E6%8A%8A%E6%89%8B%E6%95%99%E4%BD%A0%E4%BD%BF%E7%94%A8JWT/10.png)

然后，我们在jwt-test项目的根目录下执行命令：

```shell
php artisan migrate
```

![](http://oklbfi1yj.bkt.clouddn.com/%E6%89%8B%E6%8A%8A%E6%89%8B%E6%95%99%E4%BD%A0%E4%BD%BF%E7%94%A8JWT/11.png)

然后，我们通过tinker来模拟出用户：

```
php artisan tinker
Factory(App\User::class, 3)->create();
```

这样，在数据表users就有3个用户了，他们的明文密码都是`123456`。

![](http://oklbfi1yj.bkt.clouddn.com/%E6%89%8B%E6%8A%8A%E6%89%8B%E6%95%99%E4%BD%A0%E4%BD%BF%E7%94%A8JWT/12.png)

## 编写登录接口

现在用户有了，我们需要开始编写接口了。首先，我们先定义一个登录接口，这个接口在用户登录成功的时候，返回一个token给客户端。

我们在`routes/api.php`中做如下定义：

```php
Route::group(['prefix'=>'user'], function (){
    Route::post('login', 'UserController@login');
});
```

然后，我们创建对应的`UserController`控制器：

```shell
sudo php artisan make:controller UserController
```

我们在这个控制器里面编写`login`方法：

```php
public function login()
{
    $credentials = request(['username', 'password']);

    if (! $token = auth()->attempt($credentials)) {
        return response()->json(['error' => 'Unauthorized'], 401);
    }

    return response()->json([
        'token' => $token,
    ]);
}
```

编写完之后，我们通过postman来测试这个接口：

![](http://oklbfi1yj.bkt.clouddn.com/%E6%89%8B%E6%8A%8A%E6%89%8B%E6%95%99%E4%BD%A0%E4%BD%BF%E7%94%A8JWT/29.png)

（注意：图中的https是因为我制作的这个docker容器里面使用了证书，所以用了https协议，如果大家是http协议的话，那么把https改成http协议即可）

然后，我们在postman中填写用户名和密码：

![](http://oklbfi1yj.bkt.clouddn.com/%E6%89%8B%E6%8A%8A%E6%89%8B%E6%95%99%E4%BD%A0%E4%BD%BF%E7%94%A8JWT/14.png)

（这个是刚才我们模拟出来的第一个用户）

然后，我们点击右边蓝色的按钮`Send`：

![](http://oklbfi1yj.bkt.clouddn.com/%E6%89%8B%E6%8A%8A%E6%89%8B%E6%95%99%E4%BD%A0%E4%BD%BF%E7%94%A8JWT/15.png)

这个字符串就是jwt。

好了，接下来，我们编写新的接口。我们在`routes/api.php`文件中定义一下：

```php
Route::group(['prefix'=>'user'], function (){
    Route::post('login', 'UserController@login');
    Route::post('hello', 'UserController@hello');
});
```

然后，我们编写这个接口对应的hello方法：

```php
public function hello()
{
    return response()->json([
        'info' => 'hello',
    ]); 
}
```

我们增加了一个接口：`/api/user/hello`，但是，我们又不希望没有登录的用户访问这个接口。那么，我们该怎么做呢？

我们可以编写一个中间件，专门用来检测用户是否登录了，这个中间件就叫做jwt吧。以下是中间件编写的过程。

在jwt-test项目的根目录下执行命令：

```shell
php artisan make:middleware Jwt
```

![](http://oklbfi1yj.bkt.clouddn.com/%E6%89%8B%E6%8A%8A%E6%89%8B%E6%95%99%E4%BD%A0%E4%BD%BF%E7%94%A8JWT/16.png)

执行完之后，会在`app/Http/Middleware`目录下创建一个`Jwt.php`文件。

然后，我们在`Jwt.php`文件里面写入如下代码：

```php
<?php

namespace App\Http\Middleware;

use Closure;

class Jwt
{
    /**
     * Handle an incoming request.
     *
     * @param  \Illuminate\Http\Request  $request
     * @param  \Closure  $next
     * @return mixed
     */
    public function handle($request, Closure $next)
    {
        if (auth()->user() == null) {
            return response()->json([
                'code' => 1001,
                'info' => 'Not logged in',
                'body' => []
            ]);
        }

        return $next($request);
    }
}

```

编写完中间件之后，我们需要注册这个中间件。

在`app/Http/Kernel.php`文件中的`$routeMiddleware`变量加入上面所写的中间件：

```php
'jwt' => \App\Http\Middleware\Jwt::class,
```

![](http://oklbfi1yj.bkt.clouddn.com/%E6%89%8B%E6%8A%8A%E6%89%8B%E6%95%99%E4%BD%A0%E4%BD%BF%E7%94%A8JWT/17.png)

注册完jwt中间件之后，我们就可以使用这个jwt中间件了，我们在UserController控制器中使用。

在UserController中的构造函数里面这样写：

```php
public function __construct()
{
    $this->middleware('jwt', ['except' => ['login']]);
}
```

它代表的意思就是除了login方法外，其他的方法都需要经过中间件。OK，我们现在来测试以下我们的那个hello接口。

首先，我们得进行登录，因此，需要先访问login接口得到一个token字符串：

![](http://oklbfi1yj.bkt.clouddn.com/%E6%89%8B%E6%8A%8A%E6%89%8B%E6%95%99%E4%BD%A0%E4%BD%BF%E7%94%A8JWT/18.png)

我们拿着这个token字符串去访问hello接口：

![](http://oklbfi1yj.bkt.clouddn.com/%E6%89%8B%E6%8A%8A%E6%89%8B%E6%95%99%E4%BD%A0%E4%BD%BF%E7%94%A8JWT/19.png)

那么，如果我们的这个token是错误的，会有什么现象呢？我们来试试：

![](http://oklbfi1yj.bkt.clouddn.com/%E6%89%8B%E6%8A%8A%E6%89%8B%E6%95%99%E4%BD%A0%E4%BD%BF%E7%94%A8JWT/20.png)

我们把正确的那个token的最后一个字符Q删除了时候，得到了用户未登录的信息。为什么？因为你的token是错误的，所以说明用户没有登录过。

博客写到这里，大家应该就知道如何使用jwt了吧。

小伙伴们可能会想，这个token到底是个啥呢？别急，我们对这个token串进行base64解码。我们随便找一个可以进行base64解码的网站：

我们把jwt中第一个`.`前面的字符串都删掉（包括这个`.`），然后再把第二个`.`后面的字符串都删掉（包括这个`.`）：

![](http://oklbfi1yj.bkt.clouddn.com/%E6%89%8B%E6%8A%8A%E6%89%8B%E6%95%99%E4%BD%A0%E4%BD%BF%E7%94%A8JWT/21.png)

然后再点击解密：

![](http://oklbfi1yj.bkt.clouddn.com/%E6%89%8B%E6%8A%8A%E6%89%8B%E6%95%99%E4%BD%A0%E4%BD%BF%E7%94%A8JWT/22.png)

```json
{
  "iss": "https://localhost/api/user/login",
  "iat": 1535557634,
  "exp": 1535561234,
  "nbf": 1535557634,
  "jti": "kjNhwJ7KESuX2AGR",
  "sub": 1,
  "prv": "87e0af1ef9fd15812fdec97153a14e0b047546aa"
}
```

其中，sub对应的1就指的是用户的id。这个token默认保存的用户信息就是用户的id值。

那么为什么我要删除掉两个`.`号前面和后面的字符串呢？因为我想重点说一下这两个东西。我们单独解码一下第一个`.`号的内容：

![](http://oklbfi1yj.bkt.clouddn.com/%E6%89%8B%E6%8A%8A%E6%89%8B%E6%95%99%E4%BD%A0%E4%BD%BF%E7%94%A8JWT/23.png)

我们点击解密：

![](http://oklbfi1yj.bkt.clouddn.com/%E6%89%8B%E6%8A%8A%E6%89%8B%E6%95%99%E4%BD%A0%E4%BD%BF%E7%94%A8JWT/24.png)

typ代表这是一个jwt字符串、alg代表使用了哈希算法hash256。那么，这个哈希算法用在了什么地方呢？

我们第二个`.`后面的字符串就用到了。

我们对第二个点后面的内容进行解码：

![](http://oklbfi1yj.bkt.clouddn.com/%E6%89%8B%E6%8A%8A%E6%89%8B%E6%95%99%E4%BD%A0%E4%BD%BF%E7%94%A8JWT/25.png)

我们点击解密：

![](http://oklbfi1yj.bkt.clouddn.com/%E6%89%8B%E6%8A%8A%E6%89%8B%E6%95%99%E4%BD%A0%E4%BD%BF%E7%94%A8JWT/26.png)

这个就是一个签名。

那么这个签名是怎么来的呢？就是通过`hash256`算法得来的。

OK，那么又有小伙伴们会问了。那我是不是就可以伪造出其他用户的token啦！！因为我可以先用自己的账号进行一个登录，登录完之后，我对第一部分字符串进行解码，知道了用的是什么hash算法。然后，我再解码第二部分的字符串，得到了：

```json
{
  "iss": "https://localhost/api/user/login",
  "iat": 1535557634,
  "exp": 1535561234,
  "nbf": 1535557634,
  "jti": "kjNhwJ7KESuX2AGR",
  "sub": 1,
  "prv": "87e0af1ef9fd15812fdec97153a14e0b047546aa"
}
```

我就去修改sub的值，改成其他用户的，去构造一个新的字符串，这样总会构造出一个正确的jwt吧。

如果没有第三部分的字符串（即签名部分），理论上来说是可以的。但是，有了签名，就很难构造了。因为，这个签名的得来用到了我们之前生成的那个密钥。因为密钥是服务器才知道的，所以，构造者除非是得到了这个密钥，要不然想要构造出jwt，是比较难的。

OK，那么这个jwt是存放在哪里的呢？如果小伙伴们阅读jwt-auth的源码就会发现，其实它是存放在Laravel框架的`jwt-test/storage/`里面。

























