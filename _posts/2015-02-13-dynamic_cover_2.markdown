---
layout:         post
title:          封面 - 随机一些
date:           15-2-13
categories:     Cover
tags:           SAE Python Cron
image:          /assets/article_images/2014-11-30-mediator_features/night-track.JPG
---

<br/>
15年的生日又是一团糟，Whatever, 喘了几天气，给自己一点 Break ，一切都会过去，俺们接着开心的玩耍 SAE。

回顾一下先前的工作，我利用 SAE 搭建了一个最基本的 RESTFUL API Service。功能非常简单，从 Bing.com 得到每日一图的 JSON 数据后包装为 JavaScript 对象提供给客户端解析图片 URL 并渲染封面图片。虽然现在的封面图片不需要人肉指定了，但问题是我所有博文的封面全都一个模样。对于一个“花心”的男人这怎么可以忍受！

改变总是出现在不断审视和对某样事物提出要求的基础上。下一步自己想要的是随机选取封面图片。

## 目标 Ready!
1. 第一版实现中，对于 image JSON 的采集的触发是基于 API 调用。现在的目标需要改为 Cron job 模式，每天定点自动获取数据。
2. 采用 SAE 的数据库服务 (MySQL) 把每天获取的 image JSON 保存下来。
2. 客户端引用用 bingbg.js 的时候需要随机从库中抓取一条记录并返回 JSON 数据。

目标明确，开搞喽。在实现的过程中或多或少的被折腾了一把。哎，还好这些年被 SAE 坑过的人也不少，暂且记录一下备忘吧。

## 新浪挖的“坑”
关于 SAE Python 开发环境文档真是有点误导我这类“皮糙肉厚滴小鲜肉”。结果就是我完全没有使用他们提供的工具。本身 SAE 所支持的就是源生 Python 环境，除了对第三方模块的支持有一定局限性之外，开发人员完全可以无障碍使用本地环境工作。当然代码层面稍微需要一点点改动。来举个很无语的例子：

{% highlight python linenos %}
    try:
        # Local Env
        from flask.ext.sqlalchemy import SQLAlchemy
    except ImportError:
        # Production
        from flaskext.sqlalchemy import SQLAlchemy
{% endhighlight %}

实在是懒得说，尼玛新浪连包名都别出心裁是为哪般? 另由于新浪使用 wsgi 文件作为应用入口，本地环境中需要给这个文件做一个硬连接 app.py 方便本地运行：

{% highlight python linenos %}
    from ApiController import app

    if __name__ == "__main__":
        app.run(debug=True)
    else:
        import sae
        application = sae.create_wsgi_app(app)
{% endhighlight %}

最后就是 MySQL Gone 的问题，网上也有解决方案，剽窃之。

{% highlight python linenos %}
# Hack for MySql gone issue
class NullpoolSQLAlchemy(SQLAlchemy):
    def apply_driver_hacks(self, app, info, options):
        super(NullpoolSQLAlchemy, self).apply_driver_hacks(app, info, options)
        from sqlalchemy.pool import NullPool
        options['poolclass'] = NullPool
        del options['pool_size']


app = Flask(__name__)
db = NullpoolSQLAlchemy(app)


# Generate mysql connection URI
def _create_db_connect_string():
    # local db - fake data
    host = 'localhost'
    db = 'db'
    port = '3306'
    user = 'user'
    pwd = 'pwd'
    try:
        import sae.const
        db = sae.const.MYSQL_DB
        user = sae.const.MYSQL_USER
        pwd = sae.const.MYSQL_PASS
        host = sae.const.MYSQL_HOST
        port = sae.const.MYSQL_PORT
    except ImportError:
        pass
    return 'mysql://' + user + ':' + pwd + '@' + host + ':' + port + '/' + db


app.config['SQLALCHEMY_DATABASE_URI'] = _create_db_connect_string()
{% endhighlight %}

## Cron Job
Cron Job 在 SAE 的实现还是很容易滴，先配置一下 config.yaml。

{% highlight yaml linenos %}
    name: wsandbox
    version: 1
    cron:
    - description: fetch_images
      url: /cron/fetch_images
      schedule: "* 17 * * *"
{% endhighlight %}

关于 Cron 的配置，新浪的文档还是比较详细，[放个传送门在这里](http://sae.sina.com.cn/doc/python/cron.html)，指定 API URL, 设置任务频率，配置部分完成。

接着是 fetch_images 的具体实现：
{% highlight Python linenos %}
    # picBase model
    class PicBase(db.Model):
        __tablename__ = 'picBase'
        id = db.Column(db.Integer, primary_key=True)
        url = db.Column(db.String(250))
        copyright = db.Column(db.String(250))
        title = db.Column(db.String(50))
        text = db.Column(db.String(100))
        type = db.Column(db.String(50))
        fullstartdate = db.Column(db.String(30))
        raw_data = db.Column(db.Text)

        def __init__(self, url, copyright, title, text, type, fullstartdate, raw_data):
            self.url = url
            self.copyright = copyright
            self.title = title
            self.text = text
            self.type = type
            self.fullstartdate = fullstartdate
            self.raw_data = raw_data

        def __repr__(self):
            return '<URL %r>' % self.url


    # New route for the cron job which fetches image json everyday
    @app.route('/cron/fetch_images', methods=['GET'])
    def fetch_images():
        # Get JSON string from target URL
        response = urllib2.urlopen('http://www.bing.com/HPImageArchive.aspx?format=js&idx=0&n=1&mkt=en-US')
        json_string = response.read()
        json_obj = json.loads(json_string)
        image = json_obj['images'][0]
        new_record = PicBase(image["url"], image["copyright"], image['msg'][0]['title'],
                             image['msg'][0]['text'], 'bing.com', image['fullstartdate'], json_string)

        insert_new_record(new_record)
        return '200'


    # Insert new record
    def insert_new_record(new_record):
        if PicBase.query.filter_by(fullstartdate=new_record.fullstartdate).first() is None:
            try:
                db.session.add(new_record)
                db.session.commit()
            except:
                db.session.rollback()
                raise
        return
{% endhighlight %}

上面的代码分为三块，第一块为定义 Data Model, 由于使用了 Flask-SQLAlChemy（ORM）, 这里的 Model 和库中的表保持一致。第二块为 Cron 的触发入口，SAE 会根据用户应用 config 设置使用这个 URL 进行工作。第三快就是数据库操作了，如果库中没有相同的数据，则插入。

## 最后一步
这里需要对先前的实现进行一点点小修改，首先去掉读取远程数据动作，增加随机读取本地数据库记录的功能。


{% highlight Python linenos %}
    # Create new route for request and return wrapped javascript object
    @app.route('/bingbg.js', methods=['GET'])
    def get_json():
        if request.method == 'GET':
            results = db.engine.execute('SELECT * FROM picBase ORDER BY RAND() LIMIT 1;')

            json_result = []
            for result in results:
                d = {'id': result.id,
                     'url': result.url,
                     'copyright': result.copyright,
                     'title': result.title,
                     'text': result.text,
                     'type': result.type,
                     'fullstartdate': result.fullstartdate,
                     'raw_data': result.raw_data}

                json_result.append(d)

        return 'bingBgJSON = {images: ' + json.dumps(json_result) + '}'
{% endhighlight %}

进化阶段性完成! 在做这部分实现的时候，自己还是从侧面了解到一些关于 JSON 安全性的问题，原理不复述，[习惯性传送门（可能被墙，真不理解）](http://flask.pocoo.org/docs/0.10/security/#json-security)，简而言之就是在返回 JSON 数据的时候禁止 top-level 为数组。完成了这一段，下一步自己想玩一下 SAE 的 Storeage 服务，原因很直接，保存链接的形式虽然成本低廉但是风险也很高，猜想会出现死链的情况。所以还是把想要的东西留在身边最好啦...

To be continue ...
