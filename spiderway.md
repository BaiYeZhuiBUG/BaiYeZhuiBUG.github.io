### 多种方式抓取HTML

#### requests  get请求:      

```python
    reponse = requests.get(url).content
```

#### requests  post请求：        
```python
    url = 'http://36kr.com/   
    cookies = 'fdsjfsfadsfji'   
    data = {
        'name':'mitnick',
        'pwd':'123456'
    }       
    response = requests.post(url=url, data=data, cookies=cookies)    
 ```
 

#### urllib  get请求：
```python
        response = urllib.urlopen(url, data=None, proxies=None)    
```        
      
#### mechanize:      
```python
    def __init_browser():
        try:
            br = mechanize.Browser()
            br.set_handle_equiv(True)
            br.set_handle_gzip(True)
            br.set_handle_redirect(True)
            br.set_handle_referer(True)
            br.set_handle_robots(False)


            br.set_handle_refresh(mechanize._http.HTTPRefreshProcessor(), max_time=1)
            br.set_debug_http(False)
            return br
        except RuntimeError as err:
            self.logger.error('init browser failed : %s'%err)
            raise err
    
    browser = __init_browser()      
    browser.addheaders = [('USER_AGENTS',self.get_user_agents()),
                                       ('referer','http://weixin.sogou.com/weixin?type=2'),
                                       ('cookie', cookie)]
    host = '112.124.401.27:3128'
    self.browser.set_proxies({"https":host})
    response = self.browser.open(url).read()
```    

#### phantomJS:
```python
    #-*- coding:utf-8 -*-
    from selenium import webdriver
    from django.http import HttpResponse
    from selenium.webdriver.common.by import By
    from time import sleep
    from selenium.webdriver.common.proxy import Proxy
    from selenium.webdriver.common.proxy import ProxyType
    from selenium.webdriver.common.desired_capabilities import DesiredCapabilities
    from datetime import datetime
    from log_helper import LogHelper
    import requests
    import json
    
    driver = None
    logger = None
    
    def init_driver():
       """
       初始化PhantomJS插件
       :return:
       """
       global driver, logger
       logger = LogHelper(log_file='loadvideo/log/spider.log', log_name='spider').get_logger()
       try:
           desired_capabilities = DesiredCapabilities.PHANTOMJS.copy()
           desired_capabilities["phantomjs.page.settings.loadImages"] = False
           # headers = {
           #             # 'referer':'http://weixin.sogou.com/weixin?type=2',
           #            'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8'}
           # for key, value in headers.iteritems():
           #     desired_capabilities['phantomjs.page.customHeaders.{}'.format(key)] = value
    
           # proxy = Proxy({'proxyType': ProxyType.MANUAL,'httpProxy': '121.199.42.46:31280'})#动态代理
           # proxy.add_to_capabilities(desired_capabilities)
           driver = webdriver.PhantomJS(desired_capabilities=desired_capabilities)
       except Exception as err:
           logger.error('init driver error: %s'%err)
           raise err
    
    
    def dynamic_proxies(httpProxy):
       """
       动态代理
       :return:
       """
       try:
           desired_capabilities = DesiredCapabilities.PHANTOMJS.copy()
           proxy = Proxy({'proxyType': ProxyType.MANUAL,'httpProxy': httpProxy})#动态代理
           proxy.add_to_capabilities(desired_capabilities)
           driver.start_session(desired_capabilities)
       except Exception as err:
           logger.error('dynamic proxies error: %s'%err)
           pass
    
    
    def gethtml(requst):
       charset = 'utf-8'
       try:
           url = requst.GET.get('url')
           classname = requst.GET.get('class')
           id = requst.GET.get('id')
           name = requst.GET.get('name')
           if not url:
               logger.error('url is None')
               return
    
           #获取代理
           httpProxy = '112.124.37.13:31280'
           html = ''
           #非异步加载
           if not classname and not id and not name:
               proxies = {'http':httpProxy}
               response = requests.get(url=url, proxies=proxies)
               if response.encoding and response.encoding != 'ISO-8859-1':
                   charset = response.encoding
               try:
                   html = response.content.decode(charset)
               except Exception as err:
                   #如果response.encoding编码抛异常，尝试使用response.apparent_encoding
                   if response.apparent_encoding:
                       html = response.content.decode(response.apparent_encoding)
                       return HttpResponse(html)
               return HttpResponse(html)
    
           #driver初始化
           if not driver:
               init_driver()
               logger.info("Restart driver Ok!")
    
           #切换代理
           # dynamic_proxies(httpProxy)
    
           #隐式等待
           driver.implicitly_wait(20)
           driver.get(url)
    
           if classname:
               driver.find_element(By.CLASS_NAME, classname)
           elif id:
               driver.find_element(By.ID, id)
           elif name:
               driver.find_element(By.NAME, name)
    
           html = driver.page_source
           return HttpResponse(html)
       except Exception as err:
           logger.error('dynamic proxies error: %s'%err)
           init_driver()
           logger.info("Restart driver under OK!")
           return ''
```          
  
         
#### pyqt4:      
简单介绍一下pyqt4吧。之所以用到pyqt4，是因为微信搜狗太坑了。。。微信搜狗在我入职当前这家公司后不久就加强了防爬虫措施，除了对cookie和IP地址判断该用户是不是机器人之外，还加了个头文件属性“referer"，大多数爬虫库都不支持这个头文件参数，除了mechanize，但是，如果你用mechanize，因为cookie的缘故，你很快就会被封了，需要输入验证码，于是我使用了pyqt4+模拟点击，实现了稳定的微信搜狗爬虫。
```python
    #-*- coding:utf-8 -*-
    import sys
    from PyQt4.QtWebKit import QWebView
    from time import sleep
    from PyQt4.QtGui import QApplication
    from PyQt4.QtCore import QUrl
    from PyQt4.QtNetwork import *
    import pyautogui
    from pymouse import PyMouse
    from bs4 import BeautifulSoup
    from spiders.wechat_spider_new import WechatNewSpider
    from redis_helper import RedisHelper
    from datetime import datetime
    import random
    
    gl_status = 0
    page_flag = 0
    x_size = 0
    proxy_flag = 0
    load_time = datetime.now()
    
    class Browser(QWebView):
        def __init__(self, url):
            QWebView.__init__(self)
            self.loadFinished.connect(self.auto_click)
            self.loadProgress.connect(self.print_percent)
            st = self.settings()
            st.setAttribute(st.AutoLoadImages, False)
            self.resize(1200, 600)
            self.redis = RedisHelper()
            # self.redis2 = RedisHelper()
    
        def print_percent(self, percent):
            if page_flag and (datetime.now()-load_time).seconds > 2:
                self.stop()
    
        def _result_availiable(self):
            global gl_status, x_size, proxy_flag, page_flag, load_time
            frame = self.page().mainFrame()
            dom = unicode(frame.toHtml()).encode('utf-8')
            soup = BeautifulSoup(dom,'lxml')
            if soup.find('div', {'class':'b404-box'}):
                print 'news_list is none ,so return 0'
                return 0
            if soup.find("input", {'id': 'seccodeInput'}):
                page_flag = 0
                self.nextLink2()
                # url = self.redis.lpop('wechat:url_backups140')
                # self.redis.lpush('wechat:auto_start_urls', url)
                return 0
            else:
                newsList = soup.find('ul', {'class': 'news-list'})
                try:
                    pagebar_container = soup.find('div', attrs={'id':'pagebar_container'})
                    if pagebar_container:
                        num = len(pagebar_container.findAll('a'))
                        pev = len(pagebar_container.findAll('a',{'class':'pev'}))
                        np = len(pagebar_container.findAll('a',{'class':'np'}))
                        b = len(pagebar_container.findAll('b'))
                        x_size = 220 + (num+b+np+pev)*30
                except RuntimeError as err:
                    print '_result_availiable'
                    pass
                if newsList:
                    if gl_status >= 1:
                        self.redis.rpush('wechat:news_list',newsList)
                return soup.find("a", {'id': 'sogou_next'})
    
        def save(self,dom):
            start_time = datetime.now()
            spider = WechatNewSpider()
            spider.crawlNews(dom)
    
    
        def auto_click(self):
            global gl_status, x_size, proxy_flag, load_time, page_flag
            print 'load finish'
            proxy_flag = 1
            m = PyMouse()
            # self.redis.lpush('wechat:insert_html_time140',datetime.now())
            rs = self._result_availiable()
            # if self._result_availiable() and gl_status > 0:#如果为True，直接把URL提交到start_urls_100
            #     url = self.redis.lpop('wechat:url_backups140')
            #     self.redis.lpush('wechat:start_urls_100',url+"&num=100&tsn=1")186
            #     gl_status = 20
            # if not rs:
            #     gl_status = 10
                # self.nextLink()
            gl_status += 1
            if gl_status == 1:
                pyautogui.click(780, 212)
                pyautogui.click(208, 230)
                pyautogui.click(224, 303)
                load_time = datetime.now()
                page_flag = 1
                print 'gl_status = 1'
            elif rs:
                # for cli in range(3):
                    # m.click(1275, 675)
                m.click(1275, 186, 2)
                m.click(1275, 250)
                sleep(0.3)
                pyautogui.click(x_size, 499)#点击下一页
                # sleep(random.randint(1,5))
                load_time = datetime.now()
                page_flag = 1
                print 'rs is false'
            else:
                print 'last join in'
                self.nextLink()
    
        def nextLink(self):
            global gl_status, x_size, proxy_flag, page_flag, load_time
            page_flag = 0
            url = None
            while True:
                url = self.redis.rpop('wechat:auto_start_urls')
                # self.redis.lpush('wechat:url_backups140',url)
                if not url:
                    print 'sleep...'
                    sleep(300)
                else:
                    break
            proxy = Proxy()
            proxy.setAppProxy()
            # proxy.clearAppProxy()
            load_time = datetime.now()
            self.load(QUrl(url.decode('utf-8')))
            gl_status = 0
            print 'join in nextLink'
    
    
        def nextLink2(self):
            global gl_status, x_size, proxy_flag, load_time, page_flag
            url = "http://www.sogou.com/web?ie=utf8&query="
            proxy = Proxy()
            # proxy.setAppProxy()
            proxy.clearAppProxy()
            load_time = datetime.now()
            self.load(QUrl(url.decode('utf-8')))
            # page_flag = 0
            gl_status = 0
    
    class AutoClick(object):
        def __init__(self):
            self.app = QApplication(sys.argv)
            self.dom = None
            self.view = None
    
        def start_browser(self):
            global page_flag
            url = self.get_url()
            self.view = Browser(url)
            proxy = Proxy()
            proxy.setAppProxy()
            # proxy.clearAppProxy()
            # page_flag = 0
            self.view.load(QUrl(url.decode('utf-8')))
            self.view.setWindowTitle('Mitnick\'s Browser')
            self.view.show()
            sys.exit(self.app.exec_())
    
        def get_url(self):
            redis = RedisHelper()
            while True:
                try:
                    url = redis.rpop('wechat:auto_start_urls')
                    if not url:
                        print 'sleep....'
                        sleep(300)
                        continue
                    else:
                        return url
                except Exception as err:
                    print 'get_url error : %s'%err
                    pass
    
    class Proxy(object):
        def setAppProxy(self):
            host_list = ['121.199.20.211:31280','121.119.201.205:31280']
            i =  random.randrange(0, len(host_list), 1)
            host = host_list[i]
            print 'host is %s'%host
            if host == 0:
                return 'local'
            proxy = QNetworkProxy()
            #Http访问代理
            proxy.setType(QNetworkProxy.HttpProxy)
            #proxy.setType(QtNetwork.QNetworkProxy.DefaultProxy)
            proxy.setHostName(u'%s'%host.split(':')[0])
            proxy.setPort(int(host.split(':')[1]))
            QNetworkProxy.setApplicationProxy(proxy)
    
        def clearAppProxy(self):
            proxy = QNetworkProxy()
            QNetworkProxy.setApplicationProxy(proxy)
```     
