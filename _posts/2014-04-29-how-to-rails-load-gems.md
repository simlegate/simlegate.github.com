---
layout: post
title: "Rails加载gems的方式"
description: ""
category: ""
tags: [Rails, Gem, Bundler]
---
{% include JB/setup %}

当我们在Rails项目中添加一个新的gem，我们不需要在Rails代码中使用`require <gem_name>`来引入gem，就可以直接使用，这是为什么呢？原来Rails用[Bundler](https://github.com/bundler/bundler)来管理gem依赖并且自动加载它们。

### Rails加载gems到LOAD_PATH

# app/config/boot.rb

    require 'rubygems'
        
    # Set up gems listed in the Gemfile.
    ENV['BUNDLE_GEMFILE'] ||= File.expand_path('../../Gemfile', __FILE__)
        
    require 'bundler/setup' if File.exists?(ENV['BUNDLE_GEMFILE'])

这个文件将会在Rails启动的时候被执行，主要的功能就是找到项目根目录下的`Gemfile`，把`Gemfile`中的每一个gem的`lib`目录所在的路径加载到`$LOAD_PATH`常量中，为以后引入gems做准备。(**注意：这个时候gems还没有被require**)

这里可以通过`$:`或者`$LOAD_PATH`获得当前环境下Ruby解释器能找到的所有的路径, 当使用`require`时，Ruby解释器就在这些路径下去找对应的文件

### 引入gems
    
    # app/config/application.rb

    require File.expand_path('../boot', __FILE__)
    
    require 'rails/all'
    
    if defined?(Bundler)
      # If you precompile assets before deploying to production, use this line
      Bundler.require(*Rails.groups(:assets => %w(development test)))
      # If you want your assets lazily compiled in production, use this line
      # Bundler.require(:default, :assets, Rails.env)
    end

这个文件通过`Bundler.require`引入gems。`Bundler.require`可以指定分组(group),将指定的分组的gems载入。

这时可以通过`$"`获得当前环境下Ruby解释器已经加载的所有文件(不仅仅是.rb文件，也可能是.so, .o, .dll文件)

`Bundler.require(*Rails.groups(:assets => %w(development test)))`说明Rails只在`development`和`test`环境下加载`assets`组中的gems，在`production`下需要手动预编译。所以在`development`下，Rails只会加载`:default`,`:development`, `:assets`这三个分组下的gems，在`test`下，Rails只会加载`:default`,`:test`, `:assets`这三个分组下的gems，`production`下，Rails会加载`:default`, `:production`这二个分组下的gems。`Rails.group`的详细分析见github[源代码](https://github.com/rails/rails/blob/3096629d297a77a9b64747a0ac2df6b2cbf47a68/railties/lib/rails.rb#L93)。

