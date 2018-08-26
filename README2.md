在学习爬虫时候，scrapyd中文网 上遇到这个文章，可以看看：

http://www.scrapyd.cn/example/176.html

编写代码


接下来开始编写代码，首先创建项目：
```
scrapy startproject Aoisolas
```

item.py编写


主要是定义我们要爬取的字段，其实这里就两个文件夹名称，图片url，代码如下：
```
import scrapy
class AoisolasItem(scrapy.Item):
    # define the fields for your item here like:
    name = scrapy.Field()
    ImgUrl = scrapy.Field()
    pass
```

至于项目名称，我是这样命名的 Aoisola + s，Aoisola是神马意思？不知道？我也不知道……想一探究竟不如百度一下；接下来在spider目录下创建蜘蛛文件：AoiSolaSpider.py，看一下目录结构：

scrapy项目实战


接下来，编写蜘蛛文件：
```
# -*- coding: utf-8 -*-
import scrapy
from AoiSolas.items import AoisolasItem

class AoisolaspiderSpider(scrapy.Spider):
    name = "AoiSola"
    allowed_domains = ["www.mm131.com"]
    start_urls = ['http://www.mm131.com/xinggan/',
                  'http://www.mm131.com/qingchun/',
                  'http://www.mm131.com/xiaohua/',
                  'http://www.mm131.com/chemo/',
                  'http://www.mm131.com/qipao/',
                  'http://www.mm131.com/mingxing/'
                  ]

    def parse(self, response):
        list = response.css(".list-left dd:not(.page)")
        for img in list:
            imgname = img.css("a::text").extract_first()
            imgurl = img.css("a::attr(href)").extract_first()
            imgurl2 = str(imgurl)
            print(imgurl2)
            next_url = response.css(".page-en:nth-last-child(2)::attr(href)").extract_first()
            if next_url is not None:
                # 下一页
                yield response.follow(next_url, callback=self.parse)

            yield scrapy.Request(imgurl2, callback=self.content)

    def content(self, response):
        item = AoisolasItem()
        item['name'] = response.css(".content h5::text").extract_first()
        item['ImgUrl'] = response.css(".content-pic img::attr(src)").extract()
        yield item
        # 提取图片,存入文件夹
        # print(item['ImgUrl'])
        next_url = response.css(".page-ch:last-child::attr(href)").extract_first()

        if next_url is not None:
            # 下一页
            yield response.follow(next_url, callback=self.content)
```

图片下载中间件：pipelines.py编写：


这是scrapy 图片下载的核心，不熟悉scrapy 图片下载的小伙伴，可以查看文章：《scrapy图片下载》、《scrapy图片重命名放入不同目录》他们已经把scrapy图片下载的一切都扒光了，若不清楚scrapy图片下载、不妨去参观哈！还是直接上代码：

```
from scrapy.pipelines.images import ImagesPipeline
from scrapy.exceptions import DropItem
from scrapy.http import Request
import re


class MyImagesPipeline(ImagesPipeline):

    #
    # def file_path(self, request, response=None, info=None):
    #     """
    #     :param request: 每一个图片下载管道请求
    #     :param response:
    #     :param info:
    #     :param strip :清洗Windows系统的文件夹非法字符，避免无法创建目录
    #     :return: 每套图的分类目录
    #     """
    #     item = request.meta['item']
    #     folder = item['name']
    #
    #     folder_strip = re.sub(r'[？\\*|“<>:/]', '', str(folder))
    #     image_guid = request.url.split('/')[-1]
    #     filename = u'full/{0}/{1}'.format(folder_strip, image_guid)
    #     return filename

    def get_media_requests(self, item, info):
        for image_url in item['ImgUrl']:
            yield Request(image_url,meta={'item':item['name']})

    def file_path(self, request, response=None, info=None):
        name = request.meta['item']
        # name = filter(lambda x: x not in '()0123456789', name)
        name = re.sub(r'[？\\*|“<>:/()0123456789]', '', name)
        image_guid = request.url.split('/')[-1]
        # name2 = request.url.split('/')[-2]
        filename = u'full/{0}/{1}'.format(name, image_guid)
        return filename
        # return 'full/%s' % (image_guid)

    def item_completed(self, results, item, info):
        image_path = [x['path'] for ok, x in results if ok]
        if not image_path:
            raise DropItem('Item contains no images')
        item['image_paths'] = image_path
        return item


```

setting.py设置：


爬虫写好、中间件也写好、别忘了设置启动中间件，还有图片需要下载了放那，都是setting的事，接下来设置哈：
```
# 设置图片存储路径
IMAGES_STORE = 'D:\meizi2'
#启动pipeline中间件
ITEM_PIPELINES = {
   'AoiSolas.pipelines.MyImagesPipeline': 300,
}
```



啰嗦几句，中间件的MyImages……你需要设置为你命名的，若你命名的不是和我的一样而设置的一样，那恭喜你肯定出错，这是很多小伙伴没注意到的地方，希望能引起你的重视！


经过这么几个步骤，按理说启动爬虫我们的图片就能乖乖躺在我们设置的目录里面的，爱你胃，其实并没有，我们惊讶的发现，里面没有妹子，只有腾讯的logo，神马鬼？接下来才是正真展示技术的时候，这也是本scrapy 实战最值得看的看点！若你的爬虫只能欺负一些没有反爬虫限制的网站，那你就像用枪指着手无寸铁的平民一样，算神马英雄？要欺负就欺负反爬虫网站，好了接下来就开始欺负她！


要想反反爬虫，你必须明白反爬虫技术实现的要点，如现在遇到的诡异情况，我们爬A站的图片，为神马会得到B站的logo，莫非冥冥之中我们练就了隔山打牛？其实不然，如果你真正运营过优质的网站，你就会明白这其实只是一种防盗链技术，简单的说防盗链就是为了防止A站把B站的资源直接拿去用，而做的一种阻止技术；就用这妹子网站来解释，本来这些妹子图片都是在妹子网站的服务器上，另一个网站看到做的不错，也想做一个，他就想把妹子网站的图片链接直接拿过来就可以了，这样的话图片是显示在自己网站，但处理图片的服务器却是妹子网站的，典型的损人利己！防盗链就是为了防止这么一种情况，让你盗链的链接不显示，呵呵……


那肿么破解呢？这里我们就需要知道防盗链的本质，其实很简单，防盗链的核心是判断你请求的地址是不是来自本服务器，若是，则给你图片，不是则不给，知道了这么一个原则我们就可以破解了，我们每下载一张图片都先伪造一个妹子服务器的请求，然后在下载，那它肯定会给你返回图片，于是我们就能顺利拿到图片！那问题的重点就归结为如何伪造请求地址，scrapy实现起来灰常简单，就是写一个Middleware，代码如下：
```
class AoisolasSpiderMiddleware(object):

    def process_request(self, request, spider):
        referer = request.url
        if referer:
            request.headers['referer'] = referer
```
就这么几行代码，就能轻松破解她的防盗链，然后别忘了在settings里面启动middleware，如下：

```
DOWNLOADER_MIDDLEWARES = {
   'AoiSolas.middlewares.AoisolasSpiderMiddleware': 1,
}
```

最后，运行蜘蛛：
```
scrapy crawl AoiSola
```

来看一下结果：

scrapy采花大盗小爬虫


最后送诸君半首诗：小撸怡情、大撸伤身、樯橹灰飞烟灭，据说是苏轼写的，看人家文采多好……


作者是这位大神：http://www.scrapyd.cn/example/176.html


