# 课件下载利器


## 缘起

在学习南大的课程[程序设计语言的形式语义](https://cs.nju.edu.cn/hongjin/teaching/semantics/index.htm)时，要下载课件，一个一个去下太麻烦了，这么机械性的工作，当然交给脚本来做最合适啦，而且这种事情以后肯定还要用到，于是写了一个python脚本，可以从指定的页面，把所有的ppt和pdf文件下载下来。
<!--more-->

## 课件下载脚本
<script src="https://gist.github.com/wanghuibin0/42824b32c6da5aefdfe7ee6d2d0bdf84.js"></script>
<!---
# Run this script with two command line arguments:
#   1. the source url
#   2. the destination folder

from  pprint import pprint
import requests
from bs4 import BeautifulSoup
import sys, os

def get_html(url):
    try:
        html = requests.get(url).text
    except Exception as e:
        print('Web requests url error: {}\nlink: {}'.format(e, url))
    return html

class WebDownloader(object):
    def __init__(self, base_url, dest_folder):
        self.url = base_url
        self.dest_folder = dest_folder
        self.links = set()

    def parse_html(self, verbose=False):
        full_url = str(self.url)
        base_url = full_url.split('/')[0:-1]
        base_url = '/'.join(base_url)
        print(base_url)
        html = get_html(self.url)
        soup = BeautifulSoup(html, 'html.parser')
        for link in soup.findAll('a'):
            if link.has_attr('href'):
                href = str(link.get('href'))
                if href.startswith('http'):
                    self.links.add(href)
                else:
                    self.links.add(base_url + '/' + href)
                if verbose:
                    print(link.get('href'))

    def download(self):
        for link in self.links:
            link = str(link)
            if link.endswith('.pdf') or link.endswith('.ppt') or link.endswith('pptx'):
                file_name = link.split('/')[-1]
                full_name = self.dest_folder + os.sep + file_name
                if os.path.exists(full_name):
                    print('File ' + full_name + ' exists. Skip it!')
                    continue
                try:
                    print('Downloading file ' + link)
                    r = requests.get(link)
                    with open(full_name, 'wb+') as f:
                        f.write(r.content)
                except Exception as e:
                    print('Downloading error: {}\nlink: {}'.format(e, link))

def _main():
    argv = sys.argv
    if len(argv) < 3:
        print('argv < 3, please input the url and dest folder')
        sys.exit()
    url = str(argv[1])
    dest_folder = str(argv[2])
    wd = WebDownloader(url, dest_folder)
    wd.parse_html()
    pprint(wd.links)
    wd.download()
    print('Mission accomplished! Have a nice day~')

# Press the green button in the gutter to run the script.
if __name__ == '__main__':
    _main()
-->

## 用法示例

命令行执行如下命令

```bash
python3 slides-downloader.py https://cs.nju.edu.cn/hongjin/teaching/semantics/index.htm slides
```

正常会出现如下运行界面：![](https://cdn.jsdelivr.net/gh/wanghuibin0/picbed@main/img/20210319150601.png)

如此，便可以愉快的下载课件啦，就可以把更多时间用来学(mo)习(yu)啦~~~

