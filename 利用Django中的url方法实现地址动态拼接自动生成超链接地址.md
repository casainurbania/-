建立一个图书列表页面，显示图书名列表，并实现点击书名跳转到图书详细页面，显示图书详细信息。

- URL方法简介

   

   

  - 功能：返回一个绝对路径的引用(不包含域名的URL)；该引用匹配一个给定的视图函数和 
    一些可选的参数。
  - 语法：`{% url 'some-url-name' value1 value2 %}`
  - 参数’some-url-name’表示在urls.py文件中的路由地址；
  - 参数value1和value2表示拼接的值，可选。
  - 例如，urls.py: `url(r'^bookinfo/(\d+)/$', polls_views.bookinfo, name='book')` 
    html代码中：`{% url 'book' 3 %}`; 
    拼接后返回地址为：bookinfo/3/

#### **数据库djangodemo中存有信息：**

#### **表polls_book**

+—-+————–+———–+ 
| id | name | person_id | 
+—-+————–+———–+ 
| 1 | 围城 | 1 | 
| 2 | 蝴蝶梦 | 2 | 
| 3 | 鲁滨逊漂流记 | 3 | 
| 4 | 小王子 | 4 | 
+—-+————–+———–+

#### **表polls_person**

+—-+——+—–+ 
| id | name | age | 
+—-+——+—–+ 
| 1 | Joe | 12 | 
| 2 | walt | 18 | 
| 3 | walt | 17 | 
| 4 | Jany | 20 | 
| 5 | John | 29 | 
+—-+——+—–+

- **先写出图书列表页面**
- **实现超链接自动拼接**
- **编写图书详情页面**

- **项目目录信息**

![这里写图片描述](https://img-blog.csdn.net/20171221235756514?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY2hlbmdjaGVuZzk1NTg4/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

- **建立页面路由** 
  在urls.py文件中添加

```
# 导入路由，支持正则表达式
from django.conf.urls import url
# 在路由匹配模式中添加图书列表页面的路由
urlpatterns = [
    url(r'^booklist/$', polls_views.booklist),
 # 定义拼接地址，获取书籍信息
    url(r'^bookinfo/(\d+)/$', polls_views.bookinfo, name='bookinfo')
]12345678
```

- **1. 在views.py文件中添加**

```
# 图书列表页面控制器
def booklist(request):
    # 导入图书类
    from polls.models import Book
    # 实例化一个图书对象
    books = Book.objects.all()
    # 建立空字典存储booklist
    dict_book = {}
    dict_book['booklist'] = books
    # 向bookList.html页面传入数据dict_book
    return render(request, 'bookList.html', dict_book)1234567891011
```

- **2. 在templates文件夹下新建bookList.html文件，并添加**

```
{# 在bookList.html文件的body下添加如下代码 #}
<body>
    <h2>图书架</h2>
    <ul>
        {% for book in booklist %}
            {# 使用每本书的book.id作为获取详情的查询条件，生成链接 #}
            <li><a href="{% url 'bookinfo'  book.id  %}">{{ book.name }}</a></li>
        {% endfor %}
    </ul>
</body>12345678910
```

- **3. 在view.py文件中定义获取书籍信息详细信息的控制方法**

```
# 获取书籍信息
def bookinfo(request, id):
    # 导入图书类
    from polls.models import Book
    # 实例化一个图书对象，使用book.id查询该书籍数据
    book = Book.objects.get(id=id)
    # 建立空字典存储booklist
    dict_book = {}
    # 存储book书名
    dict_book['book'] = book.name
    # 存储book作者
    dict_book['author'] = book.person.name
    # 存储book作者年龄
    dict_book['author_age'] = book.person.age
    # 向bookInfo.html页面传入数据dict_book
    return render(request, 'bookInfo.html', dict_book)12345678910111213141516
```

- **4. 在templates文件夹下新建bookInfo.html文件，并添加**

```
{# 在bookInfo.html文件的body下添加如下代码 #}
<body>
    <h2>{{ book }}</h2>
    <ul>
        <li>作者：{{ author }}</li>
        <li>年龄：{{ author_age }}</li>
    </ul>
</body>12345678
```

- **在浏览器中访问http://127.0.0.1:8000/booklist/**

![这里写图片描述](https://img-blog.csdn.net/20171221235120651?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY2hlbmdjaGVuZzk1NTg4/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

- **点击“鲁滨逊漂流记**

![这里写图片描述](https://img-blog.csdn.net/20171221235204767?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY2hlbmdjaGVuZzk1NTg4/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

可以看出，地址栏里的127.0.0.1:8000/bookInfo/3中”3”是根据书籍“鲁滨逊漂流记”的id获取的，“鲁滨逊漂流记”在数据库表polls_book中对应的id是3。

表polls_book 
+—-+————–+———–+ 
| id | name | person_id | 
+—-+————–+———–+ 
| 1 | 围城 | 1 | 
| 2 | 蝴蝶梦 | 2 | 
| 3 | 鲁滨逊漂流记 | 3 | 
| 4 | 小王子 | 4 | 
+—-+————–+———–+

1. 以上工作的条件是你已经完成了Django的正常配置，并正常开启了server；
2. 数据库中的数据是预先添加好的，这里只是查询数据库中的数据。
3. 能力有限，欢迎指错纠正。