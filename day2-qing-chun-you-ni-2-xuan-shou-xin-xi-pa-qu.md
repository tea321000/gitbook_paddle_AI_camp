# Day2-《青春有你2》选手信息爬取

#### 1.请在下方提示位置，补充代码，完成《青春有你2》选手图片爬取，将爬取图片进行保存，保证代码正常运行

#### 2.打印爬取的所有图片的绝对路径，以及爬取的图片总数，此部分已经给出代码。请在提交前，一定要保证有打印结果，如下图所示：

![](.gitbook/assets/image%20%281%29.png)

**深度学习一般过程:**  
  
![](https://ai-studio-static-online.cdn.bcebos.com/b372f0b8277a40759b91920972951f1184bca6c33fc74881bd4f93f7388ed32a)

**收集数据，尤其是有标签、高质量的数据是一件昂贵的工作。**

**爬虫**的过程，就是模仿浏览器的行为，往目标站点发送请求，接收服务器的响应数据，提取需要的信息，并进行保存的过程。

**Python**为爬虫的实现提供了工具:requests模块、BeautifulSoup库



**上网的全过程:**

普通用户:

打开浏览器 --&gt; 往目标站点发送请求 --&gt; 接收响应数据 --&gt; 渲染到页面上。

爬虫程序:

模拟浏览器 --&gt; 往目标站点发送请求 --&gt; 接收响应数据 --&gt; 提取有用的数据 --&gt; 保存到本地/数据库。

**爬虫的过程**：

1. 发送请求（requests模块）
2. 获取响应数据（服务器返回）
3. 解析并提取数据（BeautifulSoup查找或者re正则）
4. 保存数据

**本实践中将会使用以下两个模块，首先对这两个模块简单了解以下：**  


**request模块：**

* requests是python实现的简单易用的HTTP库，官网地址：[http://cn.python-requests.org/zh\_CN/latest/](http://cn.python-requests.org/zh_CN/latest/)
* requests.get\(url\)可以发送一个http get请求，返回服务器响应内容。

**BeautifulSoup库：**

* BeautifulSoup 是一个可以从HTML或XML文件中提取数据的Python库。网址：[https://beautifulsoup.readthedocs.io/zh\_CN/v4.4.0/](https://beautifulsoup.readthedocs.io/zh_CN/v4.4.0/)
* BeautifulSoup支持Python标准库中的HTML解析器,还支持一些第三方的解析器,其中一个是 lxml。
* BeautifulSoup\(markup, "html.parser"\)或者BeautifulSoup\(markup, "lxml"\)，推荐使用lxml作为解析器,因为效率更高。



这个题目可以说对新人算是较为友好，爬虫程序的绝大部分已经给出，但其中类选择器以及爬取图片的第三部分需要自行完成：

### 一、爬取百度百科中《青春有你2》中所有参赛选手信息，返回页面数据

```python
import json
import re
import requests
import datetime
from bs4 import BeautifulSoup
import os

#获取当天的日期,并进行格式化,用于后面文件命名，格式:20200420
today = datetime.date.today().strftime('%Y%m%d')    

def crawl_wiki_data():
    """
    爬取百度百科中《青春有你2》中参赛选手信息，返回html
    """
    headers = { 
        'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/67.0.3396.99 Safari/537.36'
    }
    url='https://baike.baidu.com/item/青春有你第二季'                         

    try:
        response = requests.get(url,headers=headers)
        print(response.status_code)

        #将一段文档传入BeautifulSoup的构造方法,就能得到一个文档的对象, 可以传入一段字符串
        soup = BeautifulSoup(response.text,'lxml')
        
        #返回的是class为table-view log-set-param的<table>所有标签
        tables = soup.find_all('table',{'class':'table-view log-set-param'})

        crawl_table_title = "参赛学员"

        for table in  tables:           
            #对当前节点前面的标签和字符串进行查找
            table_titles = table.find_previous('div').find_all('h3')
            for title in table_titles:
                if(crawl_table_title in title):
                    return table       
    except Exception as e:
        print(e)
```

这一段先使用requests请求了百度百科的http静态页面，然后对静态页面使用BeautifulSoup（**BeautifulSoup通过CSS选择器的形式来爬取相应元素 不支持xPath**）通过类选择器选择类名为`'table-view log-set-param'`的元素下的表格`table`元素并进行返回：`tables = soup.find_all('table',{'class':'table-view log-set-param'})`

注意此处爬取的是静态页面中的元素，request不像selenium这种使用浏览器内核来进行爬虫的库，是一个http的请求库，**本身不带有渲染Javascript的功能**，也就是说假如网页的某些链接不是通过静态页面返回而是通过Javascript在浏览器对静态页面解释后通过JS进行动态加载的就会无法获取。这里会跟该题的坑相关，下面再进行讲解。



### 二、对爬取的页面数据进行解析，并保存为JSON文件

```python
def parse_wiki_data(table_html):
    '''
    从百度百科返回的html中解析得到选手信息，以当前日期作为文件名，存JSON文件,保存到work目录下
    '''
    bs = BeautifulSoup(str(table_html),'lxml')
    all_trs = bs.find_all('tr')

    error_list = ['\'','\"']

    stars = []

    for tr in all_trs[1:]:
         all_tds = tr.find_all('td')

         star = {}

         #姓名
         star["name"]=all_tds[0].text
         #个人百度百科链接
         star["link"]= 'https://baike.baidu.com' + all_tds[0].find('a').get('href')
         #籍贯
         star["zone"]=all_tds[1].text
         #星座
         star["constellation"]=all_tds[2].text
         #身高
         star["height"]=all_tds[3].text
         #体重
         star["weight"]= all_tds[4].text

         #花语,去除掉花语中的单引号或双引号
         flower_word = all_tds[5].text
         for c in flower_word:
             if  c in error_list:
                 flower_word=flower_word.replace(c,'')
         star["flower_word"]=flower_word 
         
         #公司
         if not all_tds[6].find('a') is  None:
             star["company"]= all_tds[6].find('a').text
         else:
             star["company"]= all_tds[6].text  

         stars.append(star)

    json_data = json.loads(str(stars).replace("\'","\""))   
    with open('work/' + today + '.json', 'w', encoding='UTF-8') as f:
        json.dump(json_data, f, ensure_ascii=False)
```

这一段就是单纯的数据处理了，将表格元素的行和列`tr,td`各种拼接最后保存成json文件。

###  三、爬取每个选手的百度百科图片，并进行保存

### ！！！请在以下代码块中补充代码，爬取每个选手的百度百科图片，并保存 ！！！

```python
def crawl_pic_urls():
    '''
    爬取每个选手的百度百科图片，并保存
    ''' 
    with open('work/'+ today + '.json', 'r', encoding='UTF-8') as file:
         json_array = json.loads(file.read())
    
    headers = { 
        'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/67.0.3396.99 Safari/537.36' 
     }

    for star in json_array:

        name = star['name']
        link = star['link']
        

        #！！！请在以下完成对每个选手图片的爬取，将所有图片url存储在一个列表pic_urls中！！！

        # 百度百科图册只静态显示前30张图片的link 后面的link通过JS动态加载
        # 假如图册不止30张图 还需要对30张图后的再进行JS动态爬取（王姝慧有38张图片）
        # 但由于我们的request库是HTTP静态解析库 无法渲染JS 因此没有办法获取王姝慧30张之后的图片 
        
        try:
            response = requests.get(link,headers=headers)
            assert response.status_code==200, '选手页面返回状态码异常'

            # 爬取百科页面
            bs = BeautifulSoup(response.text,'lxml')
            # 使用类选择器选择summary类下的a标签
            summary_tag = bs.select('.summary-pic a')
            # 因为获取的是路由而没有域名 因此需要拼接原link的域名
            domain=link[:link.find('.com')+len('.com')]
            pic_urls=[]
            response = requests.get(domain+summary_tag[0]['href'],headers=headers)
            assert response.status_code==200, '图册页面返回状态码异常'
            # 爬取图册页面
            bs = BeautifulSoup(response.text,'lxml')

            # 假如图册不止30张图 还需要对30张图后的再进行JS动态爬取（王姝慧有38张图片）
            # 但由于我们的request库是HTTP静态解析库 无法渲染JS 因此没有办法获取王姝慧30张图之后的动态链接
            href_list=[a for a in bs.select('.pic-list a')]
            if len(href_list)<=30:
                # 去掉?后面对图片的大小变换操作 用列表生成式获得所有缩略图对应大图的link
                pic_urls = pic_urls + [img['src'][:img['src'].find('?')] for img in bs.select('.pic-list img')]
            else:
                pic_urls = pic_urls + [img['src'][:img['src'].find('?')] for img in bs.select('.pic-list img')]
                # 在本地环境下用类似Selenium的库渲染JS才能抓取王姝慧第30张之后的图片
                
            #！！！根据图片链接列表pic_urls, 下载所有图片，保存在以name命名的文件夹中！！！
            down_pic(name,pic_urls)
            
        except Exception as e:
            print(e)

```

```python
def down_pic(name,pic_urls):
    '''
    根据图片链接列表pic_urls, 下载所有图片，保存在以name命名的文件夹中,
    '''
    path = 'work/'+'pics/'+name+'/'

    if not os.path.exists(path):
      os.makedirs(path)

    for i, pic_url in enumerate(pic_urls):
        try:
            pic = requests.get(pic_url, timeout=15)
            string = str(i + 1) + '.jpg'
            with open(path+string, 'wb') as f:
                f.write(pic.content)
                print('成功下载第%s张图片: %s' % (str(i + 1), str(pic_url)))
        except Exception as e:
            print('下载第%s张图片时失败: %s' % (str(i + 1), str(pic_url)))
            print(e)
            continue
```

此处

