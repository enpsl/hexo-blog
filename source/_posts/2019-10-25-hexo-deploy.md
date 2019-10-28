---
layout:     post
title:      "hexo 部署小记"
subtitle:   "hexo 部署小记"
date:       2019-10-25 13:30:00
author:     "Psl"
catalog:    true
tags:
  - hexo
---

## hexo部署流程

 开启本地web服务:
```bash
hexo s
```
- 生成静态文件:
```bash
hexo g
```
- 文件压缩:
```bash
gulp
```
- 文件部署到远程服务器:
```bash
hexo d
```

## gulp安装：
### 安装 gulp
使用 npm install xxx --save命令分别安装如下工具
```bash
"gulp": "^3.9.1",
"gulp-htmlclean": "^2.7.6",
"gulp-htmlmin": "^1.3.0",
"gulp-imagemin": "^2.4.0",
"gulp-minify-css": "^1.2.4",
"gulp-uglify": "^1.5.3",
```

###建立 gulpfile.js 文件
在 Hexo 的根目录建立 gulpfile.js
```javascript
var gulp = require('gulp');
var minifycss = require('gulp-minify-css');
var uglify = require('gulp-uglify');
var htmlmin = require('gulp-htmlmin');
var htmlclean = require('gulp-htmlclean');
var imagemin = require('gulp-imagemin');

// 压缩html
gulp.task('minify-html', function() {
    return gulp.src('./public/**/*.html')
        .pipe(htmlclean())
        .pipe(htmlmin({
            removeComments: true,
            minifyJS: true,
            minifyCSS: true,
            minifyURLs: true,
        }))
        .pipe(gulp.dest('./public'))
});
// 压缩css
gulp.task('minify-css', function() {
    return gulp.src('./public/**/*.css')
        .pipe(minifycss({
            compatibility: 'ie8'
        }))
        .pipe(gulp.dest('./public'));
});
// 压缩js
gulp.task('minify-js', function() {
    return gulp.src('./public/js/**/*.js')
        .pipe(uglify())
        .pipe(gulp.dest('./public'));
});
// 压缩图片
gulp.task('minify-images', function() {
    return gulp.src('./public/images/**/*.*')
        .pipe(imagemin(
            [imagemin.gifsicle({'optimizationLevel': 3}),
                imagemin.jpegtran({'progressive': true}),
                imagemin.optipng({'optimizationLevel': 7}),
                imagemin.svgo()],
            {'verbose': true}))
        .pipe(gulp.dest('./public/images'))
});
// 默认任务
gulp.task('default', [
    'minify-html','minify-css','minify-js','minify-images'
]);
```