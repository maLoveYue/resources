# 创建djang项目

## 1. 创建项目

```shell
django-admin startproject devops
```

## 2. 创建应用

```shell
python manage.py startapp myapp 
```

## 3.运行项目

```shell
python manage.py  runserver 0.0.0.0 8000
```

# Django 工作流程

<img src="C:\Users\Mark\AppData\Roaming\Typora\typora-user-images\image-20240314164302737.png" alt="image-20240314164302737" style="zoom:50%;" />

# 路由系统

## 1.路由格式

```
urlpatterns = [
    path(regex, views, kwarg=None, name=None )
]
```

urlpatterns： 一个列表，每个path函数是一个元素，对应一个视图

regex: 一个字符串或者正则表达式，匹配URL

view: 对应的视图或者类视图（as_view()）,必须返回一个Httpresponse对象

kwarg: 可选，字典形式的数据传递给对应视图

name: 可选，URL的名称

## 2.路由分发

​	路由分发的好处：url配置解耦，便于管理

**项目根目录的urls.py**

```python
from django.urls import path,include

urlpatterns = [
    # path('admin/', admin.site.urls),
    # path('index', views.index),
    path('myapp/', include('myapp.urls')),
]
```

**应用urls.py**

```python
from django.urls import path,include
from myapp import views
urlpatterns = [
    path('index/', views.index),
]
```



3. url正则表达式匹配

urls.py

```python
from django.urls import path,include,re_path
from myapp import views
urlpatterns = [
    path('index/', views.index),
    re_path('^articles/([0-9]{4})/([0-9]{2})/([0-9]+)/$', views.articles)

]
```

views.py

```python
#接受的位置参数，*args可以使用不定长参数去接收，函数类使用元祖
def articles(request,year, month, id):
    #res = f'{args[0]}年{args[1]}月 文章ID{args[2]}'
    res = f'{year}年{month}月 文章ID{id}'
    return HttpResponse(res)
```

**分组命令**

格式：？P<name> re

urls.py

```python
urlpatterns = [
    re_path('^articles/(?P<year>[0-9]{4})/(?P<month>[0-9]{2})/(?P<id>[0-9]+)/$', views.articles)
]
```

views.py

```python
#参数位置不固定
def articles(request,year,id,month):
    res = f'{year}年{month}月 文章ID{id}'
    return HttpResponse(res)
```

# 视图

## 1.HttpRequest常用属性

| 属性            | 描述                                                 |
| --------------- | :--------------------------------------------------- |
| request.scheme  | 请求协议 http or https                               |
| request.body    | 请求正文                                             |
| request.path    | 完整的url路径，不包含域名/myapp/articles/2023/12/12/ |
| request.method  | http方法 GET                                         |
| request.GET     | QueryDict对象                                        |
| request.POST    | QueryDict对象                                        |
| request.FILES   | 上传的文件                                           |
| request.COOKIES | 字典格式返回cookie                                   |
| request.META    | 所有http请求头                                       |

## 2.HttpRequest常用方法

| 方法                    | 描述                                          |
| ----------------------- | --------------------------------------------- |
| request.get_port()      | 端口8000                                      |
| request.get_host()      | 127.0.0.1:8000                                |
| request.get_full_path() | url路径 /myapp/articles/2023/12/12/?name=mark |
| request.get_raw_uri()   | 完整的url路径                                 |

## 3.HttpRespone函数

语法：HttpRespone（content=响应体，content_type=响应数据类型，status=状态码）

```python
def articles(request,year,id,month):
# def articles(request, *args):
    res = f'{year}年{month}月 文章ID{id}'
    req= HttpResponse(res)
    #添加响应头
    req['name'] = 'mark'
    req.status_code = 302
    return req
```

## 4.render函数

render指定函数，返回渲染后的HtppReponse对象

语法：render(request,'article.html',context= {'res': res}, content_type=None, status=None, using=None)

```

def articles(request,year,id,month):
# def articles(request, *args):
    res = f'{year}年{month}月 文章ID{id}
    return render(request,'article.html',context= {'res': res})
```

## 5. redirect函数

语法：redirect(to, *args, ** kwargs)

参数：

* 视图
* 绝对或相对的URL
* 模型

```python
 return redirect('/myapp/index')
```

## 6.StreamingHttpResponse函数

StreamingHttpResponse流式响应可迭代对象 ，响应文件，二进制文件等。

urls.py

```python
from django.urls import path,include,re_path
from myapp import views
urlpatterns = [
    path('index/', views.index),
    re_path('^download/$', views.download),
    re_path('^downfile/(?P<filename>.*)/$', views.downfile, name="downfile"),
]

```



views.py

```python
def download(request):
    file_list = os.listdir('upload')
    return render(request,'fileList.html',{'file_list':file_list})

def downfile(request, filename):
    file_path = os.path.join('upload',filename)
    res = StreamingHttpResponse(open(file_path,'rb'))
    res['Content-Type'] = 'application/octet-stream'
    res['Content-Disposition'] = f'attachment;filename={os.path.basename(file_path)}'
    return res
```

## 7. FileRespone函数

文件下载的建议方法，用法参考StreamingHttpRespone

## 8.JsonRespone函数

JsonRespone函数：响应一个JSON对象

views.py

```python
from django.http import StreamingHttpResponse, JsonResponse
def apiview(request):
    d = {'name': 'mark'}
    return JsonResponse(d)
```

# 模版

## 1.变量

变量的定义：可以使用render函数返回context传入，类似字典形式。

在模版中使用**{{  key  }}**

## 2.全局变量

自定义py文件

devops1/contexts.py

```python
def user(request):
    username = 'mark'
    return {'username': username}
```

settings.py文件中配置py路径

![image-20240317195729660](C:\Users\Mark\AppData\Roaming\Typora\typora-user-images\image-20240317195729660.png)

模板中直接引用

![image-20240317195757924](C:\Users\Mark\AppData\Roaming\Typora\typora-user-images\image-20240317195757924.png)

## 3.流程控制

### 3.1 条件判断

```html
{% if 表达式 %}
内容块
{% elif 表达式%}
内容块
{% else %}
内容块
{% endif %}
```

### 3.2 for循环

```html
  {% for k,v in res.items %}
        {% if k == 'lili'  %}
            {{ v.sex }}
        {% endif %}
    {% endfor %}
```

### 3.3 常用过滤器

| 过滤器 | 说明             | 实例                             |
| ------ | ---------------- | -------------------------------- |
| add    | 整数相加         | {{ 11\| add: '2' }} ==>13        |
| cut    | 切除字符         | {{ “word ”\| cut: 'w }} ==>ord   |
| first  | 返回第一个元素   | {{“heelo”\|first }} ==>h         |
| last   | 返回最后一个元素 | {{ “123” \| last}} ==> 3         |
| join   | 字符串拼接       | {{ “123” \| join:'-'}} ==> 1-2-3 |
| length |                  |                                  |
| lower  |                  |                                  |
| upper  |                  |                                  |
| slice  |                  |                                  |
| title  |                  |                                  |

### 3.4 自定义过滤器

定义一个自定义装饰器之前，必须确定应用在settings.py文件中

![image-20240318100032398](C:\Users\Mark\AppData\Roaming\Typora\typora-user-images\image-20240318100032398.png)

在应用目录下创建一个templateages的python包，编写过滤器方法

![image-20240318100145815](C:\Users\Mark\AppData\Roaming\Typora\typora-user-images\image-20240318100145815.png)

模版中加载过滤器文件

![image-20240318100254915](C:\Users\Mark\AppData\Roaming\Typora\typora-user-images\image-20240318100254915.png)

### 3.5 注释

```
{#  内容 #}
```

### 3.6 模版继承

定义一个母版，base.html

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>
        {% block title %}
        
        {% endblock %}
    </title>
</head>
<body>
    <h1>第一导航</h1>
    <h2>第二导航</h2>
    {% block context %}

    {% endblock %}
</body>
</html>
```

在子模板中导入

```html
{%  extends 'base.html' %}

{% block title %}
    继承测试
{% endblock %}

{% block context %}
    <p>这是一个子模板</p>
{% endblock %}
```

### 3.7 引用静态文件

在settings.py配置静态文件的路径

```python
#引用静态文件
STATICFILES_DIRS = (
    os.path.join(BASE_DIR, 'static'),
)

STATIC_URL = '/static/'
```

在模版中应用

```html
<link rel="stylesheet" href="/static/main.css">
或者
<link rel="stylesheet" href="{% static 'main.css' %}">
```

# ORM 数据模型

## 1. Model模型类

### 1.1 models.py

```python
from django.db import models

# Create your models here.

class User(models.Model):
    user = models.CharField(max_length=50)
    name = models.CharField(max_length=50)
    sex =  models.CharField(max_length=10)
    age = models.IntegerField()
    label = models.CharField(max_length=100)
    class Meta:
        app_label = 'myapp' #返回一个app名称
        db_table = 'user' #表名
        verbose_name = '用户表' #容易读的模型名
        verbose_name_plural = '用户表' #不到复数s
        ordering = ['age'] #对象默认顺序

    def __str__(self):
        return self.name
```

生成迁移文件

```
python .\manage.py   makemigrations
```

执行迁移文件生成表

```
 python .\manage.py   migrate
```

### 1.2 admin管理系统

myapp/admin.py

```python
from django.contrib import admin
# Register your models here
from .models import User
admin.site.register(User) #注册User表至后台管理

```

### 1.3 数据库配置

用docker起一个mysql容器

```shell
docker run -d --name db -p 3306:3306 -v mysqldata:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=123456 mysq:l5.7 \
 --character-set-server=utf8  
```

settings.py

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'devops1',
        'USER': 'root',
        'PASSWORD': '123456',
        'HOST': '192.168.0.203',
        'PORT': '3306'
    }
}


LANGUAGE_CODE = 'zh-hans'

TIME_ZONE = 'Asia/Shanghai'

USE_I18N = True

USE_L10N = True

USE_TZ = False
```

### 1.4 模型的增删改查

**增加**

```python
User.objects.create(user='mark', name='马克', sex='男',age=30, label='it工程师')
或者
user=User(user='mark', name='马克', sex='男',age=30, label='it工程师')
user.save
```

**查询**

```python
User.objects.all() #返回所有的对象，QuerySet
User.objects.filter(id=1) #返回符合条件的对象，QuerySet
User.object.get(id=1) #返回一个对象， object
```

**更新**

```python
User.objects.filter(id=1).update(age=18, label='xxx')
或者
obj = User.object.get(id=1)
obj.age = 18
obj.save()
```

**删除**

```python
obj = User.object.get(id=1)
obj.delete()
或者
User.object.filter(id=2).delete()
```

## 2.QuerySet序列化

序列化：将python对象转为可传输的数据格式

反序列化：将传输的数据格式转为python对象

* 内建函数serialize

```python

from django.core import serializers
def get_user(request):
    queryset = User.objects.all()
    data = serializers.serialize('json',queryset)
    return HttpResponse(queryset)
```

* 通过QuerySet对象转化成字典，再通过json编码

```python
obj = User.object.all()
data1 = {}
for i in obj:
	data1['user'] = i.user 
	data1['name'] = i.name 
	data1['sex'] = i.sex 
data  = json.dumps(data1)
```

## 3.django多表操作

![image-20240320092939134](C:\Users\Mark\AppData\Roaming\Typora\typora-user-images\image-20240320092939134.png)

### 3.1 一对一

一对一,就是表的扩展，表和表的对应关系，A表和B表每条数据需要对应，通过OneToOneField来建立对应关系

modes.py	

```PYTHON
class User(models.Model):
    user = models.CharField(max_length=50)
    name = models.CharField(max_length=50)
    sex =  models.CharField(max_length=10)
    age = models.IntegerField()
    label = models.CharField(max_length=100)
    class Meta:
        app_label = 'myapp'
        db_table = 'user'
        verbose_name = '用户表'
        verbose_name_plural = '用户表'
        ordering = ['age']

    def __str__(self):
        return self.name

class IdCard(models.Model):
    number = models.CharField(max_length=15, blank=False, null=False, unique=True)
    address = models.CharField(max_length=100)
    user = models.OneToOneField(User, on_delete=models.CASCADE)

    class Meta:
        db_table = 'idcard'
        verbose_name = '身份证表'
        verbose_name_plural = '身份证表'

    def __str__(self):
        return self.number
```

#### 3.1.1  一对一基本操作

**增加**

```python
obj = User.objects.get(id=1)
IdCard.objects.create(number='111', adress='beijing', user=obj)
```

**查询**

*正向查询(idcard-->user)*

```python
IdCard.objects.get(id=1).user.name
```

*反向查询(user-->idcard)*

```python
User.objects.get(id=1).idcard.number
```

**改**

```python
User.objects.get(id=1).idcard.number = '111'
User.objects.get(id=1).idcard.save() 
```

**删除**

```python
User.objects.filter(id=1).delete() #联级删除，idcard表对应记录也会删除
```

### 3.2 一对多

A表中每条记录对应B表中多个记录，B表中每条记录对应一个A表记录，称为一对多。ForeignKey放在多的那张表

models.py

```python
#项目
class Project(models.Model):
    name = models.CharField(max_length=30, unique=True)
    describe = models.CharField(max_length=100)
    datetime = models.DateTimeField(auto_now_add=True)

    class Meta:
        db_table = 'project'
        verbose_name_plural = '项目'

    def __str__(self):
        return self.name

#应用
class App(models.Model):
    name = models.CharField(max_length=50)
    describe = models.CharField(max_length=100)
    datetime = models.DateTimeField(auto_now_add=True)
    project = models.ForeignKey(Project, on_delete=models.CASCADE)

    class Meta:
        db_table = 'app'
        verbose_name_plural = '应用'

    def __str__(self):
        return self.name
```

#### 3.2.1 一对多基本操作

* 增加

  ```python
  p = Project.objects.get(id=1)
  App.objects.create(name='order', describe='...', project=p)
  #或者
  app = App(name='order', describe='...', project=p)
  app.save()
  ```

* 查询

  ```python
  #正向查询 （app->project）
  App.object.get(id=1).project.name
  #反向查询（project->app）
  Project.objects.get(id=1).app_set.all() 
  ```

* 改

  ```python
  app= App.object.get(id=1)
  p1 = Project.objects.get(id=2)
  app.project = p1 
  app.save()
  ```

  

* 删除

  ```
  App.object.get(id=1).delete() 
  或者
  删除主表如project，app表中对应的记录也会被删除
  Project.objects.get(id=1).delete()
  ```

  

### 3.3 多对多

A表中每条记录对应B表的多条记录，B表中的每条记录对应A表中的多条记录，通过ManyToManyField来关联，会生成一个中间表来关联。

models.py

```python
#应用
class App(models.Model):
    name = models.CharField(max_length=50)
    describe = models.CharField(max_length=100)
    datetime = models.DateTimeField(auto_now_add=True)
    project = models.ForeignKey(Project, on_delete=models.CASCADE)

    class Meta:
        db_table = 'app'
        verbose_name_plural = '应用'

    def __str__(self):
        return self.name

#服务器
class Server(models.Model):
    hostname =  models.CharField(max_length=50, unique=True)
    ip = models.GenericIPAddressField()
    app = models.ManyToManyField(App)

    class Meta:
        db_table = 'server'
        verbose_name_plural = '服务器'

    def __str__(self):
        return self.hostname
```

#### 3.3.1 多对多基本操作

* 增加

  ```python
  app = App.obecjts.get(id=1)
  Server.objects.get(id=1).app.add(app)
  ```

  

* 查询

  ```python
  #正向查询（server->app）
  Server.objects.get(id=1).app.all()
  
  #反向查询(app->server)
  app = App.obecjts.get(id=1)
  app.server_set.all()
  ```

  

* 删除

  ```python
  app = App.obecjts.get(id=1)
  Server.objects.get(id=1).app.remove（app）
  #清空该服务器所有对应关系
  Server.objects.get(id=1).app.clear()
  ```

  

# 用户认证

## 1. auth认证

Django内置的用户系统，使用auth模块实现。

![image-20240323211258323](C:\Users\Mark\AppData\Roaming\Typora\typora-user-images\image-20240323211258323.png)

views.py

```python
from django.contrib.auth.decorators import login_required
from django.contrib import auth

#登录验证装饰器
@login_required
def index(request):
    return render(request, 'index.html')

def login(request):
    if request.method == 'GET':
        return render(request,'login.html')
    elif request.method == 'POST':
        username = request.POST.get('username', None)
        password = request.POST.get('password', None)
		#验证用户名和密码正确性
        user = auth.authenticate(username=username, password=password)
        print(user,type(user))
        if user:
            #允许登录并写入session
            auth.login(request, user)
            return redirect('my-index')
        msg = '用户名或者密码错误'
        return render(request, 'login.html', {'msg': msg})


def logout(request):
    #退出清理session
    auth.logout(request)
    return redirect('login')
```

## 2. 自定义登录案例

* auth.py登录验证装饰

```python
from django.shortcuts import HttpResponse,redirect, render

def my_login_required(func):
    def inner(request):
        if request.session.get('is_logined', False):
            return func(request)
        else:
            return redirect('login')
    return inner
```

* login.py

```python
def login(request):
    if request.method == 'GET':
        return render(request,'login.html')
    elif request.method == 'POST':
        print(request.POST)
        username = request.POST.get('username', None)
        password = request.POST.get('password', None)
        user = User.objects.filter(user=username, password=password)
        if user:
            #登录成功后，配置session数据
            request.session['is_logined'] = True
            request.session['username'] = username
            return redirect('my-index')
        else:
            msg = '用户或密码错误'
            return render(request, 'login.html', {'msg': msg})
```

* logout.py

```python
def logout(request):
    #清空session
    request.session.flush()
    return redirect('login')
```

## 3. cookies和session

### 发展史

1、很久很久以前，Web 基本上就是文档的浏览而已， 既然是浏览，作为服务器， 不需要记录谁在某一段时间里都浏览了什么文档，每次请求都是一个新的HTTP协议， 就是请求加响应， 尤其是我不用记住是谁刚刚发了HTTP请求， 每个请求对我来说都是全新的。这段时间很嗨皮。

2、但是随着交互式Web应用的兴起，像在线购物网站，需要登录的网站等等，马上就面临一个问题，那就是要管理会话，必须记住哪些人登录系统， 哪些人往自己的购物车中放商品， 也就是说我必须把每个人区分开，这就是一个不小的挑战，因为HTTP请求是无状态的，所以想出的办法就是给大家发一个会话标识(session id), 说白了就是一个随机的字串，每个人收到的都不一样， 每次大家向我发起HTTP请求的时候，把这个字符串给一并捎过来， 这样我就能区分开谁是谁了

3、这样大家很嗨皮了，可是服务器就不嗨皮了，每个人只需要保存自己的session id，而服务器要保存所有人的session id ！如果访问服务器多了， 就得由成千上万，甚至几十万个。

这对服务器说是一个巨大的开销 ， 严重的限制了服务器扩展能力， 比如说我用两个机器组成了一个集群， 小F通过机器A登录了系统， 那session id会保存在机器A上， 假设小F的下一次请求被转发到机器B怎么办？机器B可没有小F的 session id啊。

有时候会采用一点小伎俩： **session sticky** ， 就是让小F的请求一直粘连在机器A上， 但是这也不管用， 要是机器A挂掉了， 还得转到机器B去。

那只好做session 的复制了， 把session id 在两个机器之间搬来搬去， 快累死了。

![img](https://ask.qcloudimg.com/http-save/yehe-1346475/hbjcat7gdb.png)

后来有个叫Memcached的支了招：把session id 集中存储到一个地方， 所有的机器都来访问这个地方的数据， 这样一来，就不用复制了， 但是增加了单点失败的可能性， 要是那个负责session 的机器挂了， 所有人都得重新登录一遍， 估计得被人骂死。

![img](https://ask.qcloudimg.com/http-save/yehe-1346475/pwfpbjc3u5.png)

也尝试把这个单点的机器也搞出集群，增加可靠性， 但不管如何， 这小小的session 对我来说是一个沉重的负担

4、于是有人就一直在思考， 我为什么要保存这可恶的session呢， 只让每个客户端去保存该多好？

可是**如果不保存这些session id , 怎么验证客户端发给我的session id 的确是我生成的呢？** 如果不去验证，我们都不知道他们是不是合法登录的用户， 那些不怀好意的家伙们就可以伪造session id , 为所欲为了。

嗯，对了，关键点就是验证 ！

*比如说， 小F已经登录了系统， 我给他发一个令牌(token)， 里边包含了小F的 user id， 下一次小F 再次通过Http 请求访问我的时候， 把这个token 通过Http header 带过来不就可以了。*

不过这和session id没有本质区别啊， 任何人都可以可以伪造， 所以我得想点儿办法， 让别人伪造不了。

那就对数据做一个签名吧， 比如说我用HMAC-SHA256 算法，加上一个只有我才知道的密钥， 对数据做一个签名， 把这个签名和数据一起作为token ， 由于密钥别人不知道， 就无法伪造token了。

![img](https://ask.qcloudimg.com/http-save/yehe-1346475/x5cdhpiifx.png)

这个token 我不保存， 当小F把这个token 给我发过来的时候，我再用同样的HMAC-SHA256 算法和同样的密钥，对数据再计算一次签名， 和token 中的签名做个比较， 如果相同， 我就知道小F已经登录过了，并且可以直接取到小F的user id , 如果不相同， 数据部分肯定被人篡改过， 我就告诉发送者：对不起，没有认证。

![img](https://ask.qcloudimg.com/http-save/yehe-1346475/ac6328y0dg.png)

Token 中的数据是明文保存的（虽然我会用Base64做下编码， 但那不是加密）， 还是可以被别人看到的， 所以我不能在其中保存像密码这样的敏感信息。

当然， 如果一个人的token 被别人偷走了， 那我也没办法， 我也会认为小偷就是合法用户， 这其实和一个人的session id 被别人偷走是一样的。

这样一来， 我就不保存session id 了， 我只是生成token , 然后验证token ， 我用我的CPU计算时间获取了我的session 存储空间 ！

解除了session id这个负担， 可以说是无事一身轻， 我的机器集群现在可以轻松地做水平扩展， 用户访问量增大， 直接加机器就行。这种无状态的感觉实在是太好了！

### Cookie

cookie 是一个非常具体的东西，指的就是浏览器里面能永久存储的一种数据，仅仅是浏览器实现的一种[数据存储](https://cloud.tencent.com/product/cdcs?from_column=20065&from=20065)功能。

cookie由服务器生成，发送给浏览器，浏览器把cookie以kv形式保存到某个目录下的文本文件内，下一次请求同一网站时会把该cookie发送给服务器。由于cookie是存在客户端上的，所以浏览器加入了一些限制确保cookie不会被恶意使用，同时不会占据太多磁盘空间，所以每个域的cookie数量是有限的。

### Session

session 从字面上讲，就是会话。这个就类似于你和一个人交谈，你怎么知道当前和你交谈的是张三而不是李四呢？对方肯定有某种特征（长相等）表明他就是张三。

session 也是类似的道理，服务器要知道当前发请求给自己的是谁。为了做这种区分，服务器就要给每个客户端分配不同的“身份标识”，然后客户端每次向服务器发请求的时候，都带上这个“身份标识”，服务器就知道这个请求来自于谁了。至于客户端怎么保存这个“身份标识”，可以有很多种方式，对于浏览器客户端，大家都默认采用 cookie 的方式。

服务器使用session把用户的信息临时保存在了服务器上，用户离开网站后session会被销毁。这种用户信息存储方式相对cookie来说更安全，可是session有一个缺陷：如果web服务器做了[负载均衡](https://cloud.tencent.com/product/clb?from_column=20065&from=20065)，那么下一个操作请求到了另一台服务器的时候session会丢失。

### Token

在Web领域基于Token的身份验证随处可见。在大多数使用Web API的互联网公司中，tokens 是多用户下处理认证的最佳方式。

以下几点特性会让你在程序中使用基于Token的身份验证

1. 无状态、可扩展
2. 支持移动设备
3. 跨程序调用
4. 安全

那些使用基于Token的身份验证的大佬们

大部分你见到过的API和Web应用都使用tokens。例如Facebook, Twitter, Google+, GitHub等。

#### Token的起源

在介绍基于Token的身份验证的原理与优势之前，不妨先看看之前的认证都是怎么做的。

##### 基于服务器的验证

我们都是知道HTTP协议是无状态的，这种无状态意味着程序需要验证每一次请求，从而辨别客户端的身份。

在这之前，程序都是通过在服务端存储的登录信息来辨别请求的。这种方式一般都是通过存储Session来完成。

随着Web，应用程序，已经移动端的兴起，这种验证的方式逐渐暴露出了问题。尤其是在可扩展性方面。

##### 基于服务器验证方式暴露的一些问题

1. **Seesion：**每次认证用户发起请求时，服务器需要去创建一个记录来存储信息。当越来越多的用户发请求时，内存的开销也会不断增加。
2. **可扩展性：**在服务端的内存中使用Seesion存储登录信息，伴随而来的是可扩展性问题。
3. **CORS(跨域资源共享)：**当我们需要让数据跨多台移动设备上使用时，跨域资源的共享会是一个让人头疼的问题。在使用Ajax抓取另一个域的资源，就可以会出现禁止请求的情况。
4. **CSRF(跨站请求伪造)：**用户在访问银行网站时，他们很容易受到跨站请求伪造的攻击，并且能够被利用其访问其他的网站。

在这些问题中，可扩展行是最突出的。因此我们有必要去寻求一种更有行之有效的方法。

##### 基于Token的验证原理

基于Token的身份验证是无状态的，我们不将用户信息存在服务器或Session中。

这种概念解决了在服务端存储信息时的许多问题

> NoSession意味着你的程序可以根据需要去增减机器，而不用去担心用户是否登录。

基于Token的身份验证的过程如下:

1. 用户通过用户名和密码发送请求。
2. 程序验证。
3. 程序返回一个签名的token 给客户端。
4. 客户端储存token,并且每次用于每次发送请求。
5. 服务端验证token并返回数据。

每一次请求都需要token。token应该在HTTP的头部发送从而保证了Http请求无状态。我们同样通过设置服务器属性Access-Control-Allow-Origin:* ，让服务器能接受到来自所有域的请求。

需要主要的是，在ACAO头部标明(designating)*时，不得带有像HTTP认证，客户端SSL证书和cookies的证书。

实现思路：

![img](https://ask.qcloudimg.com/http-save/yehe-1346475/36apvbo7ts.png)

1. 用户登录校验，校验成功后就返回Token给客户端。
2. 客户端收到数据后保存在客户端
3. 客户端每次访问API是携带Token到服务器端。
4. 服务器端采用filter过滤器校验。校验成功则返回请求数据，校验失败则返回错误码

当我们在程序中认证了信息并取得token之后，我们便能通过这个Token做许多的事情。

我们甚至能基于创建一个基于权限的token传给第三方应用程序，这些第三方程序能够获取到我们的数据（当然只有在我们允许的特定的token）

#### Token的优势

**无状态、可扩展**

在客户端存储的Tokens是无状态的，并且能够被扩展。基于这种无状态和不存储Session信息，负载[负载均衡器](https://cloud.tencent.com/product/clb?from_column=20065&from=20065)能够将用户信息从一个服务传到其他服务器上。

如果我们将已验证的用户的信息保存在Session中，则每次请求都需要用户向已验证的服务器发送验证信息(称为Session亲和性)。用户量大时，可能会造成一些拥堵。

但是不要着急。使用tokens之后这些问题都迎刃而解，因为tokens自己hold住了用户的验证信息。

**安全性**

请求中发送token而不再是发送cookie能够防止CSRF(跨站请求伪造)。即使在客户端使用cookie存储token，cookie也仅仅是一个存储机制而不是用于认证。不将信息存储在Session中，让我们少了对session操作。

token是有时效的，一段时间之后用户需要重新验证。我们也不一定需要等到token自动失效，token有撤回的操作，通过token revocataion可以使一个特定的token或是一组有相同认证的token无效。

**可扩展性**

Tokens能够创建与其它程序共享权限的程序。例如，能将一个随便的社交帐号和自己的大号(Fackbook或是Twitter)联系起来。当通过服务登录Twitter(我们将这个过程Buffer)时，我们可以将这些Buffer附到Twitter的数据流上(we are allowing Buffer to post to our Twitter stream)。

使用tokens时，可以提供可选的权限给第三方应用程序。当用户想让另一个应用程序访问它们的数据，我们可以通过建立自己的API，得出特殊权限的tokens。

**多平台跨域**

我们提前先来谈论一下CORS(跨域资源共享)，对应用程序和服务进行扩展的时候，需要介入各种各种的设备和应用程序。

> Having our API just serve data, we can also make the design choice to serve assets from a CDN. This eliminates the issues that CORS brings up after we set a quick header configuration for our application.

只要用户有一个通过了验证的token，数据和资源就能够在任何域上被请求到。

```javascript
Access-Control-Allow-Origin: *      
```

复制

基于标准创建token的时候，你可以设定一些选项。我们在后续的文章中会进行更加详尽的描述，但是标准的用法会在JSON Web Tokens体现。

最近的程序和文档是供给JSON Web Tokens的。它支持众多的语言。这意味在未来的使用中你可以真正的转换你的认证机制。

 

## 4. CSRF

