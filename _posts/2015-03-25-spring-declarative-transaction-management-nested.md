---
layout: post
title: "Spring声明式事务之内嵌事务"
description: ""
category: ""
comments: true
tags: [Java,Spring,Transaction]
---
{% include JB/setup %}

好吧，就直接说问题吧。

有实体`User`和实体`Post`，他们之间的关系为`One To Many`。
有实体`Post`和实体`Comment`，他们之间的关系为`One To Many`。

假设现在我要让某个user的所有comments中的正文字段都带上时间戳并且记录comment的更新日志`(二者事务执行)`，伪代码实现如下：

    @Transactional
    public void updateTextOfComments(User user){
      List<Post> posts = user.posts;
      for(Post post : post){
        for(Comment comment : post.comments)
        // 追加时间戳
        commentService.appendTimeStampToTitle(comment);
        // 写入post更新日志
        logger.debug(comment.getId() + 'updated');
      }
    }

为了让追加标题和写入日志操作事务执行，我们在方法前面加上了`@Transactional`，那么问题来了，这样一来如果其中一个post执行失败，抛出异常，前面所有操作都会被回滚，这不是我们想要的结果，所以我们重构一下。

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void updateTitleAndLog(Post post){
      for(Comment comment : post.comments) {
        // 追加时间戳
        postService.appendTimeStampToTitle(comment);
        // 写入post更新日志
        logger.debug(comment.getId() + 'updated');
      }
    }

    @Transactional
    public void updateTextOfComments(User user){
      List<Post> posts = user.posts;
      for(Post post : posts){
        try{
          updateTitleAndLog(post);
        }catch(Exception e){
          // logger
        }
      }
    }

我们把操作每个post的两个行为提取出一个方法，并且将这个方法也声明为事务，但是这个事务是有所不同的，它必须是独立的事务，也就是声明事务的传播性（propagation属性）为`Propagation.REQUIRES_NEW`，表示新建一个事务执行方法内的代码

代码走一个，发现报错了

    org.hibernate.LazyInitializationException: could not initialize proxy - no Session 

是执行这行代码抛出的异常`for(Comment comment : post.comments)`，意思是说不能加载post的comments，因为session已关闭，上网查了一下资料，是因为`@Transactional(propagation = Propagation.REQUIRES_NEW)`在作怪，由于post和comments之间声明的是惰性加载，即在真正使用comments的时候comments才会被从数据库中读出，并且声明一个新事务后，老的事务会被挂起，所以当执行post.comments时，系统会去找post所在的session去读取comments，由于当前已经在新的事务里面，所以是找不到post所在的session，系统就会抛出以上异常。继续重构以上代码。

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void updateTitleAndLog(Post post, List<Comment> comments){
      for(Comment comment : comments) {
        // 追加时间戳
        postService.appendTimeStampToTitle(comment);
        // 写入post更新日志
        logger.debug(comment.getId() + 'updated');
      }
    }

    @Transactional
    public void updateTextOfComments(User user){
      List<Post> posts = user.posts;
      for(Post post : posts){
        try{
          updateTitleAndLog(post, post.comments);
        }catch(Exception e){
          // logger
        }
      }
    }

让post和post读取comments在一个session中，然后传入`updateTitleAndLog`即可。
