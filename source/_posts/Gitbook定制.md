---
title: Gitbook定制
date: 2017-03-02 16:42:21
tags: Gitbook 定制
---

# Gitbook website 主题定制化

## Gitbook的文档组织结构的定制

在文档源文件根目录里的 `SUMMARY.md` 作为目录结构的载体,生成左侧的book_summary块,模板会遍历Summary中定义的parts,对于每个part里的articles通过一个递归方法去生成菜单.想要针对菜单不同层级做定制式的处理,可以从此处入手修改.

> 如果只是需要做css定制而不希望改变DOM结构的话,可以借助菜单节点的 `data-level` 属性去实现.


以default-theme为例
```html
    {% for part in summary.parts %}
        {% if part.title %}
        <li class="header">{{ part.title }}</li>
        {% elif not loop.first %}
        <li class="divider"></li>
        {% endif %}
        {{ articles(part.articles, file, config) }}
    {% endfor %}

    {% macro articles(_articles) %}
        {% for article in _articles %}
            <li class="chapter {% if article.path == file.path and not article.anchor %}active{% endif %}" data-level="{{ article.level }}" {% if article.path %}data-path="{{ article.path|resolveFile }}"{% endif %}>
                {% if article.path and getPageByPath(article.path) %}
                    <a href="{{ article.path|resolveFile }}{{ article.anchor }}">
                {% elif article.url %}
                    <a target="_blank" href="{{ article.url }}">
                {% else %}
                    <span>
                {% endif %}
                        {% if article.level != "0" and config.pluginsConfig['theme-default'].showLevel %}
                            <b>{{ article.level }}.</b>
                        {% endif %}
                        {{ article.title }}
                {% if article.path  or article.url %}
                    </a>
                {% else %}
                    </span>
                {% endif %}

                {% if article.articles.length > 0 %}
                <ul class="articles">
                    {{ articles(article.articles, file, config) }}
                </ul>
                {% endif %}
            </li>
        {% endfor %}
    {% endmacro %}
```

## 自定义样式

Gitbook默认的Theme提供了style自定义的功能.

其配置定义如下
```javascript
"gitbook": {
    "properties": {
      "styles": {
        "type": "object",
        "title": "Custom Stylesheets",
        "properties": {
          "website": {
            "title": "Stylesheet for website output",
            "default": "styles/website.css"
          },
          "pdf": {
            "title": "Stylesheet for PDF output",
            "default": "styles/pdf.css"
          },
          "epub": {
            "title": "Stylesheet for ePub output",
            "default": "styles/epub.css"
          },
          "mobi": {
            "title": "Stylesheet for Mobi output",
            "default": "styles/mobi.css"
          },
          "ebook": {
            "title": "Stylesheet for ebook outputs (PDF, ePub, Mobi)",
            "default": "styles/ebook.css"
          },
          "print": {
            "title": "Stylesheet to replace default ebook css",
            "default": "styles/print.css"
          }
        }
      },
      "showLevel": {
        "type": "boolean",
        "title": "Show level indicator in TOC",
        "default": false
      }
    }
  }
```

可以在自己项目中的book.json中做自定义配置,譬如我希望使用项目根目录名为 `websiteStyle.css` 的样式文件覆盖默认主题在生成website时的部分样式,就可以:

```json
{
    "styles":{
        "website":"./websiteStyle.css"
    },
    // ---忽略下面的部分---
    "plugins": [
        "-sharing",
        "-fontsettings",
        "-search",
        "-lunr"
    ]
}
```

如此一来,我们在此配置的css就会作为站点样式在默认的样式之后加载.可以覆盖默认的css样式以达到自定义的目的.

## 插件的形式定制新的Theme主题

> 下面示例使用github repo的形式创建一个gitbook的插件

- 首先,我们需要创建一个新的repo,然后在其中`npm init`初始化`package.json`配置文件.

注意该配置文件中必须在`engines`中指明`gitbook`的版本
```json
    "engines": {
        "gitbook": ">=3.2.2"
    }
```
如果你想要自定义一些配置节点也可以在`package.json`中声明. 
譬如,现在需要一个keywords的配置,是一个字符串值,用来指定页面渲染后的meta中的keywords,那么我们可以写作:
```json
    	"gitbook":{
		"properties":{
			"keywords":{
				"type":"string",
				"default":"",
				"description":"meta keywords config"
			}
		}
	}
```

- 接下来在入口文件中可以使用插件机制提供的hook来获取指定的config:

index.js
```javascript
module.exports = {
    hooks: {
        config: function(config) {
            console.log(`config hook~\nconfig.keywords has a value:${config['keywords']}`)
            return config
        }
    }
}
```

- 现在将这个简单的插件push到github,然后就可以在gitbook项目中使用了.我们只需要在项目的`book.json`中添加如下配置:

```json
"plugins":["name@git+https://github.com/username/repo-name.git"]
```

接下来,在`book.json`当前目录`gitbook install`安装该插件  
插件安装完成后 `gitbook serve`启动服务器就可以按到我们的插件已经正常运行了


## 提高开发时的效率

虽然,上面我们使用了gitbook自带的插件机制将自定义的部分作为一个单独的插件库.  
但是实际上开发的时候还是会感觉非常麻烦.
比如我改动一些样式,就必须要打包上传到github,然后到gitbook的项目中更新插件引用,然后再在本地host查看效果.

这样的一套流程显然是不能够满足我们的开发需求的.  
所以需要在 `plugin` 项目中用gulp创建一个开发时的 watch 任务,监视开发目录的文件变动,然后打包替换到 `gitbook` 项目里的 `_book(默认的host目录)/gitbook/` 目录.

```javascript
var gulp = require('gulp')
var sass = require('gulp-sass')
var postcss = require('gulp-postcss')
var autoprefixer = require('autoprefixer')
var webpack = require('webpack-stream')
var uglify = require('gulp-uglify')
var cssMinify = require('gulp-minify-css')
var through = require('through2')

const PATHS = {
    SASS: ['./src/scss/style.scss'], // scss 入口文件
    SASS_ALL: ['./src/scss/**/*'],
    SCRIPTS: ['./src/js/**/index.js'],
    SCRIPTS_ALL: ['./src/js/**/*'],
    production: './_assets/website/',
    development: '../pdoc/node_modules/gitbook-plugin-polex/_assets/website/',
    debug: '../pdoc/_book/gitbook/', // 只适用在开发调试过程中 不需要重新host gitbook
}

let getPath = function() {
    let ret = PATHS[process.env.NODE_ENV]
    return ret
}

gulp.task('compileSass', () => {
    var plugins = [autoprefixer({
        browsers: [
            "android 4",
            "iOS 6",
            "last 2 versions"
        ]
    })]
    gulp.src(PATHS.SASS)
        .pipe(sass().on('error', sass.logError))
        .pipe(postcss(plugins))
        .pipe(cssMinify())
        .pipe(gulp.dest(getPath()))
})

gulp.task('compileScript', () => {
    gulp.src(PATHS.SCRIPTS)
        .pipe(through.obj(function(file, enc, cb) {
            var __filename = file.path.split('\\').reverse()[1]
            gulp.src(file.path)
                .pipe(webpack({
                    output: {
                        filename: __filename === 'core' ? 'gitbook.js' : 'theme.js'
                    },
                    module: {
                        loaders: [{
                            test: /\.js$/,
                            exclude: /node_modules/,
                            loader: 'babel',
                            query: {
                                presets: ['es2015']
                            }
                        }]
                    }
                }))
                .pipe(gulp.dest(getPath()))
            cb()
        }))
})

gulp.task('build', ['compileSass', 'compileScript'])

gulp.task('watch', ['compileSass', 'compileScript'], () => {
    gulp.watch(PATHS.SASS_ALL, ['compileSass'])
    gulp.watch(PATHS.SCRIPTS_ALL, ['compileScript'])
})
```

在 `src/` 目录创建一个 `build.dev.sh` 脚本来完成打包替换的过程

```bash

#! /bin/bash
ROOT=../pdoc/node_modules/gitbook-plugin-polex/

# Cleanup folder
rm -rf ${ROOT}_assets
rm -rf ${ROOT}_layouts

# Recreate folder
mkdir -p ${ROOT}_assets/website/

# Copy layouts
cp -R _layouts/ ${ROOT}_layouts

# Copy fonts
mkdir -p ${ROOT}_assets/website/fonts
cp -R node_modules/font-awesome/fonts/ ${ROOT}_assets/website/fonts/fontawesome/

# Packaged via Gulp
cross-env NODE_ENV=development gulp build

# Copy images if they exist
mkdir -p ${ROOT}_assets/website/images
imgs=`du -k "src/images" | cut -f 1` 
zero=0
if [ $imgs -gt $zero ]; then
cp src/images/*.* ${ROOT}_assets/website/images/
else
echo "没有图片要复制."
exit 0
fi

```