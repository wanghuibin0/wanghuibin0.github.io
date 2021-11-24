# zotero与SiYuan联动


## 从zotero到思源

### 原理

修改zotero的engines.json文件，将思源作为zotero的搜索引擎之一，从zotero跳转到思源的对应页面。
<!--more-->

### engines.json

关键就在于搜索引擎跳转使用的url模板，跳转时会根据文献条目的信息，填充到url模板里，然后再跳转到产生的url地址去。因此需要将定位思源的url进行拆分，将不变部分抽取出来作为模板，变化的部分作为变量填到文献条目的信息里，实际上，我们使用的是archive字段。

engines.json添加如下entry:

```json
{
	"_name": "SiYuan",
	"_alias": "SiYuan",
	"_description": "Zotero和思源联动，页面ID存在archive字段",
	"_icon": "https://cdn.jsdelivr.net/gh/wanghuibin0/picbed@main/img/20210312191013.png",
	"_hidden": false,
	"_urlTemplate": "http://127.0.0.1:6806/blocks/{z:archive}",
	"_urlParams": [],
	"_urlNamespaces": {
		"z": "http://www.zotero.org/namespaces/openSearch#",
		"": "http://a9.com/-/spec/opensearch/1.1/"
	},
	"_iconSourceURI": "https://cdn.jsdelivr.net/gh/wanghuibin0/picbed@main/img/20210312191013.png"
},
```

点击下载完整的<a href="/assets/engines-20210315173931-1saihdw.json" download="engines.json">engines.json</a>，可以直接拷贝到zotero的数据文件夹下的locate目录下。

### 将思源笔记页面关联到zotero中

使用中，需要先将思源中对应页面id（在对应笔记处点「复制URL」，截取斜杠号分割之后的最后一段，即为id），填到文献条目archive字段去。这样，产生的url刚好能跳转到对应的笔记去。

## 从思源到zotero

由于zotero支持形式如`zotero://XXX`的链接，所以可以直接在思源笔记需要的地方填入这种链接。利用链接，可以跳转到对应的zotero item，也可以直接打开附件pdf，甚至可以定位到pdf的特定页面。

**查看文献条目**：`zotero://select/library/items/CSUJ2GFP`

**查看附件pdf特定页**：`zotero://open-pdf/library/items/ITBGH4GB?page=2`

## 参考

[zotero 和思源笔记探索 - 链滴 (ld246.com)](https://ld246.com/article/1613467723754)

[重磅｜Zotero + wolai 双向联动教程 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/342780791)

