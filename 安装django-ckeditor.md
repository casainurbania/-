## **安装django-ckeditor**

pip install django-ckeditor

##  

## **安装Pillow**

Pillow是python的一个图像处理库，django-ckeditor需要依赖该库。最简单的安装方法，当然是使用pip，假设你装过pip，可以直接运行以下命令安装：

pip install pillow

 

**配置你的django**

要使安装好的django-ckeditor生效，你需要对你的django应用进行一系列配置。

1、在你的settings.py文件中，将ckeditor、ckeditor_uploader添加到INATALLED_APPS中。

2、在你的settings.py文件中，添加CKEDITOR_UPLOAD_PATH配置项。例如，我的是：

```
MEDIA_URL = "/media/"
MEDIA_ROOT = os.path.join(BASE_DIR, "media")
CKEDITOR_UPLOAD_PATH = "article_images"
```

CHEDITOR_UPLOAD_PATH的作用是设定你通过ckeditor所上传的文件的存放目录。需要注意的是，这是一个相对路径，它相对与你设置的的MEDIA_ROOT。django-ckeditor默认使用django的后台文件存储系统，这需要你设置好MEDIA_ROOT和MEDIA_URL，如何设置超出了本文的范围，请自行查看django的官方文档，请务必确保这两个设置项是生效的，否则你将看不到你上传的文件。

比如，我上传一张名为shiguang.gif的小图片，该图片将会被存储到：

```
/my/django/app/root/media/article_images/
```

至此，你的ckeditor已经可以在django中正常使用了。

**需要指出的是**：在开发阶段，这样设置settings.py已经足够了。但是，到了正式部署你的应用时，你需要设置好STATIC_ROOT和STATIC_URL，并运行manage.py collectstatic命令，该命令会将ckeditor相关的静态资源拷贝到你的工程下。

3、在urls.py中增加ck的url配置：url(r'^ckeditor/', include('ckeditor_uploader.urls')),

**如何应用ckeditor**

django-ckeditor提供了两个类：RichTextField和CKEditorWidget，分别用于模型和表单。内容型网站通常在后台会有一个文章发布和编辑的界面，如果你想让该界面拥有一个富文本编辑器，只需按如下方式定义你的django模型：

```python
from django.db import models
from ckeditor.fields import RichTextField

class Article(models.Model):
    content = RichTextField('文章标题')
启动应用看看，这时可以看到文章标题输入框显示了富文本编辑框
但是怎么可以用的工具那么少？？？
别急，在settings目录中增加如下配置即可
CKEDITOR_CONFIGS = {
    'default': {
        'toolbar': (
			['div','Source','-','Save','NewPage','Preview','-','Templates'], 
			['Cut','Copy','Paste','PasteText','PasteFromWord','-','Print','SpellChecker','Scayt'], 
			['Undo','Redo','-','Find','Replace','-','SelectAll','RemoveFormat'], 
			['Form','Checkbox','Radio','TextField','Textarea','Select','Button', 'ImageButton','HiddenField'], 
			['Bold','Italic','Underline','Strike','-','Subscript','Superscript'], 
			['NumberedList','BulletedList','-','Outdent','Indent','Blockquote'], 
			['JustifyLeft','JustifyCenter','JustifyRight','JustifyBlock'], 
			['Link','Unlink','Anchor'], 
			['Image','Flash','Table','HorizontalRule','Smiley','SpecialChar','PageBreak'], 
			['Styles','Format','Font','FontSize'], 
			['TextColor','BGColor'], 
			['Maximize','ShowBlocks','-','About', 'pbckcode'],
		),
	}
}


现在一个完美的富文本输入框就完成了，现在可以在admin页面愉快的输入内容丰富的文章了~

如何在前端显示ck输入的内容
前端要显示ck输入的内容时，在需要使用的页面头部引入：
<script src="{% static 'ckeditor/ckeditor/ckeditor.js' %}"></script>

光这样你会发现，显示出来的时候是原始的包含各种html标签等符号的内容，此时在变量中增加safe过滤即可，如：{{ content | safe }}。
```