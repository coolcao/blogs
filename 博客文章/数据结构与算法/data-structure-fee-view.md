---
title: 前端开发视角看数据结构-记一次项目中数据结构的选择
date: 2017-01-13 17:15:22
tags: [js,数据结构]
categories:
- 博客
- 原创
---

最近在写一个博客的小项目，对接github的钩子，当提交markdown工程至github时，通过设置github的钩子，程序获取提交的markdown源码，包括新增，更新，删除的文件列表，然后将其拉取到数据库。前端解析markdown文本至html页面展示。
中间遇到一个很有意思的问题：目录的解析。

<!--more-->

markdown的目录结构可能如下:`前端/js/you_dont_konw_js`,`前端/js/node_at_scale`,`前端/css`等等。我在解析的时候，将其解析成数组：`[['前端','js','your_dont_konw_js'],['前端','js','node_at_scale'],['前端','css']]`，其中的每一项表示每个文件所在的目录。
在有一个分类的页面中，我想将每个目录中文档个数统计出来，并想展示其目录结构，预期如下：

![目录](https://img001-10042971.cos.ap-shanghai.myqcloud.com/blog/coolcao_mdblog_catalog.png)

从数据库中查出来的原始数组如下：

```json
let catalogs =
[
    [
        "ComputerSystem"
    ],
    [
        "ComputerSystem"
    ],
    [
        "ComputerSystem"
    ],
    [
        "ComputerSystem"
    ],
    [],
    [
        "css"
    ],
    [
        "NetWork"
    ],
    [
        "css"
    ],
    [
        "c語言"
    ],
    [
        "c語言"
    ],
    [
        "c語言"
    ],
    [
        "c語言"
    ],
    [
        "c語言"
    ],
    [
        "c語言"
    ],
    [
        "js"
    ],
    [
        "js"
    ],
    [
        "js"
    ],
    [
        "js"
    ],
    [
        "js"
    ],
    [
        "js"
    ],
    [
        "js"
    ],
    [
        "js",
        "你不知道的js笔记"
    ],
    [
        "js",
        "你不知道的js笔记"
    ],
    [
        "js",
        "你不知道的js笔记"
    ],
    [
        "js"
    ],
    [
        "js"
    ],
    [
        "js"
    ],
    [
        "数据结构与算法笔记"
    ],
    [
        "js",
        "dsa_js"
    ],
    [
        "js",
        "dsa_js"
    ],
    [
        "js",
        "dsa_js"
    ],
    [
        "js"
    ],
    [
        "js",
        "you_dont_konw_js"
    ],
    [
        "js",
        "you_dont_konw_js"
    ],
    [
        "js"
    ],
    [
        "js"
    ],
    [
        "js"
    ],
    [
        "js"
    ],
    [
        "js",
        "nodejs_at_scale"
    ],
    [
        "js",
        "you_dont_konw_js"
    ],
    [
        "js"
    ],
    [
        "js"
    ],
    [
        "js",
        "nodejs_at_scale"
    ],
    [
        "js",
        "you_dont_konw_js"
    ],
    [
        "js",
        "dsa_js"
    ],
    [
        "js"
    ],
    [
        "js"
    ],
    [
        "js"
    ],
    [
        "js",
        "nodejs_at_scale"
    ],
    [
        "js"
    ],
    [
        "js",
        "nodejs_at_scale"
    ],
    [
        "js"
    ],
    [
        "js"
    ],
    [
        "js",
        "dsa_js"
    ],
    [
        "js"
    ],
    [
        "js",
        "you_dont_konw_js"
    ],
    [
        "js",
        "nodejs_at_scale"
    ],
    [
        "js",
        "dsa_js"
    ],
    [
        "js",
        "you_dont_konw_js"
    ],
    [
        "js"
    ],
    [
        "js"
    ],
    [
        "js"
    ],
    [
        "js",
        "you_dont_konw_js"
    ],
    [
        "js"
    ],
    [
        "js",
        "dsa_js"
    ],
    [
        "js",
        "dsa_js"
    ],
    [
        "js",
        "nodejs_at_scale"
    ],
    [
        "js"
    ],
    [
        "js"
    ],
    [
        "js",
        "you_dont_konw_js"
    ],
    [
        "js"
    ],
    [
        "js"
    ],
    [
        "js",
        "you_dont_konw_js"
    ],
    [
        "js",
        "you_dont_konw_js"
    ],
    [
        "js",
        "dsa_js"
    ],
    [
        "js"
    ],
    [
        "js"
    ],
    [
        "js"
    ],
    [
        "js"
    ],
    [
        "js",
        "you_dont_konw_js"
    ],
    [
        "js",
        "nodejs_at_scale"
    ],
    [
        "js",
        "you_dont_konw_js"
    ]
]
```

其实统计文档数目很简单：

```js
let obj = {};
catalogs.reduce((pre,current) => {
    return pre.concat(current);
},[]).forEach(item => {
    let keys = Object.keys(obj);
    if(keys.includes(item)){
        obj[item] ++;
    }else{
        obj[item] = 1;
    }
});

console.log(obj);
```

先将二维数组扁平化到一维，然后进行统计即可，输出的结果：

```
{ ComputerSystem: 4,
  css: 2,
  NetWork: 1,
  'c語言': 6,
  js: 67,
  '你不知道的js笔记': 3,
  '数据结构与算法笔记': 1,
  dsa_js: 9,
  you_dont_konw_js: 12,
  nodejs_at_scale: 7 }
```

这里统计好了每个目录里总文档的个数，但是有个问题是，目录之间没有层级关系，在页面上展示时，不好展示。

这里抽象一下的话，目录应该是一个树形的结构，因此，很容易想到使用一个树结构。

第一次，写了一个函数，由上面这个数组，生成一个树型结构对象：

```js
/**
 * 构建目录树结构
 * @param  {Array} catalogs [目录树数组，结构如下：]
 * [['博客','mongo'],['博客','js'],['随笔','音乐'],['随笔','电影','喜剧电影']]
 * @return {obj}          [树结构]
 */
let createCatalogTree = function createCatalogTree(catalogs) {
    let obj = {
        key: 'catalog',
        count: 0,
        child: []
    };



    let insert = (item, parent) => {
        let key = obj.key;
        let child = obj.child;
        let o = obj; //当前parent所在的对象
        let has = false; //标识item是否已经存在
        let existed = null; //已经存在item的对象

        while (key != parent) {
            for (let c of child) {
                key = c.key;
                child = c.child;
                o = c;
                if(key == parent){
                    break;
                }
            }
        }
        // console.log(JSON.stringify(obj));
        if (o.child && Array.isArray(o.child)) {
            for (let ch of o.child) {
                if (ch.key == item) {
                    existed = ch;
                    has = true;
                    break;
                }
            }
        }

        if (existed) {
            existed.count++;
        } else {
            o.child.push({
                key: item,
                count: 1,
                child: []
            });
            // o.count ++;
        }

    };
    catalogs.forEach(catalog => {
        for (let i = 0; i < catalog.length; i++) {
            // console.log(catalog[i]);
            if (i === 0) {
                insert(catalog[i], 'catalog');
            } else {
                insert(catalog[i], catalog[i - 1]);
            }
        }
    });
    //重置顶层目录计数统计
    obj.child.forEach(item => {
        obj.count += item.count;
    });
    return obj;
}
```

跑一下，扔出来的结构：

```JSON
{
  "key": "catalog",
  "count": 81,
  "child": [
    {
      "key": "ComputerSystem",
      "count": 4,
      "child": []
    },
    {
      "key": "css",
      "count": 2,
      "child": []
    },
    {
      "key": "NetWork",
      "count": 1,
      "child": []
    },
    {
      "key": "c語言",
      "count": 6,
      "child": []
    },
    {
      "key": "js",
      "count": 67,
      "child": [
        {
          "key": "你不知道的js笔记",
          "count": 3,
          "child": []
        },
        {
          "key": "dsa_js",
          "count": 9,
          "child": []
        },
        {
          "key": "you_dont_konw_js",
          "count": 12,
          "child": []
        },
        {
          "key": "nodejs_at_scale",
          "count": 7,
          "child": []
        }
      ]
    },
    {
      "key": "数据结构与算法笔记",
      "count": 1,
      "child": []
    }
  ]
}
```

这个函数虽然实现了功能，但是逻辑混乱，另外效率也不高。

每个元素的插入要指定其父元素，每次都得从头查找父元素，所以效率不高。而且每次查找逻辑都冗余在插入逻辑中，混乱。

可以将这个树抽象成一个树对象，将查找，插入等等抽象成其方法。

但是，由于每个目录有多个子目录，因此，不能用简单的二叉树。

改完的树类：

```js
let find = (item, tree) => {
    let q = [];
    let r = null;
    q.push(tree);
    while (q.length) {
        let t = q.shift();
        if (t.key == item) {
            r = t;
            break;
        }
        q = q.concat(t.child);
    }
    return r;
}
class CatalogTree {
    constructor(catalogs) {
        this.key = 'catalog';
        this.count = 0;
        this.child = [];
        if(catalogs && Array.isArray(catalogs)){
            catalogs.forEach(catalog => {
                if(Array.isArray(catalog) && catalog.length > 0){
                    // this.count ++;
                    this.insert(catalog);
                }
            });
        }
    }

    find(item) {
        return find(item, this);
    }

    /**
     * 插入目录
     * @param  {Array} catalog 目录数组，型如：['技术','前端','nodejs']
     */
    insert(catalog = required()) {
        if (!Array.isArray(catalog)) {
            throw new Error('catalog类型错误，只能是数组');
        }

        //如果目录数组为空，则直接返回
        if(catalog.length <= 0){
            return this;
        }

        //每进行一次正常插入，计数加1
        this.count ++;

        let tree = this;
        let last_found = this;

        while (catalog.length > 0) {
            let item = catalog.shift();
            tree = find(item, tree);
            if (tree) {
                tree.count++;
                last_found = tree;
            } else {
                let new_node = {
                    key: item,
                    count: 1,
                    child: []
                };
                last_found.child.push(new_node);
                while (catalog.length > 0) {
                    new_node.child.push(new_node = {
                        key: catalog.shift(),
                        count: 1,
                        child: []
                    });
                }
                break;
            }

        }

        return this;
    }
}
```

其中，查找方法使用广度遍历，使用队列。

插入的时候，每次查找不再从树根开始查找，而是从找到的父目录的子树开始查找，效率比第一个函数方式要稍微高点。

虽然在这种简单的项目上，目录也不是很多，效率什么的并不明显，但是能考虑的地方就要多考虑考虑。

刚才说过了，不能使用二叉树，那么多叉树，也能用图进行抽象。但是图有个问题是，不好处理层级之间的关系，也就作罢。

就先这样吧，仓促之余先写这么多，等有时间再想想有没有其他更好的方法。
