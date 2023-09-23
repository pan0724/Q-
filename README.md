# Q谈吧
下面是项目后端接口开发的实现逻辑



文章博客论坛

## 用户相关AccountController

------











## 文件相关FileController

------



### 文件上传接口`uploadImage`

该接口在文章编辑中会用到，如下：

![image-20230905164951624](https://github.com/pan0724/Q-/assets/110321804/fe71b4a1-c814-44d8-892b-5ad25441f354)


**实现逻辑：**

1. 先判断用户上传的文件是否是属于图片，不是则抛出异常
2. 根据`MultipartFile`对象拿到完整的文件名，然后截取出后缀
3. 根据字符串生成工具生产长度为30的文件名然后加上上面的后缀。这个就是要存在服务器上的文件名
4. 打开要图片保存到的目录，如果没有就自动创建
5. 根据目录路径加上文件名的路径 由此通过File对象打开在目录下的文件
6. 然后将浏览器的文件写到刚刚创建的文件中`file.transferTo(uploadFile)`
7. 判断是否要进行压缩图片。 下面是图片压缩逻辑：
   1. ==TODO==



**用到的新知识：**

- 图片压缩
- `BigDecimal`类
- ffmpeg
- `FileUtils.forceDelete(sourceFile)` 是一个 Apache Commons IO 库中的方法，用于强制删除指定的文件或目录







### 获取用户头像接口`/getAvatar/{userId}`

描述：在用户表中并没有设置用户头像的字段。由于用户头像的文件名是根据用户id命名的，所以在获取用户头像时，直接根据用户id来拼接成文件路径，然后打开即可。

**实现逻辑：**

1. 根据用户id来拼接成文件路径，然后打开。
2. 如果文件不存在则使用默认的头像
3. 然后用响应流向浏览器输出文件



> 在进行文件的读取和写入时，需要进行以下判断：
>
> 1. 文件是否存在：在进行文件读取或写入操作之前，需要先判断文件是否存在。如果文件不存在，则需要进行相应的处理，例如创建新文件或抛出异常等。
> 2. 文件是否可读/可写：在进行文件读取或写入操作之前，需要判断文件是否可读或可写。如果文件不可读或不可写，则需要进行相应的处理，例如抛出异常或更改文件权限等。
> 3. 文件是否为空：在进行文件读取操作之前，需要判断文件是否为空。如果文件为空，则需要进行相应的处理，例如抛出异常或返回空值等。
> 4. 文件是否过大：在进行文件读取或写入操作时，需要注意文件大小。如果文件过大，可能会导致内存溢出或磁盘空间不足等问题。因此，需要进行相应的处理，例如分块读取或写入，或者使用流式处理等。
> 5. 文件编码格式：在进行文件读取或写入操作时，需要注意文件的编码格式。如果文件的编码格式与程序中使用的编码格式不一致，可能会导致乱码或其他问题。因此，需要进行相应的处理，例如指定编码格式或进行编码转换等。
> 6. 文件锁定状态：在进行文件读取或写入操作时，需要注意文件是否被其他程序或线程锁定。如果文件被锁定，则可能会导致读取或写入失败或出现异常。因此，需要进行相应的处理，例如等待文件解锁或抛出异常等。
> 7. 文件路径格式：在进行文件读取或写入操作时，需要注意文件路径的格式。不同操作系统的文件路径格式可能不同，因此需要进行相应的处理，例如使用 `File.separator` 或 `File.pathSeparator` 等跨平台的方式来表示文件路径。











## 文章相关ForumArticleController【重难点】

------





### 加载板块的文章`/loadArticle`

在主页面的时候用到

![image-20230905200249476](https://github.com/pan0724/Q-/assets/110321804/23be2a72-717c-441e-98ae-566618dfdaec)


**实现逻辑：**

1.  二话不说，先构造查询对象`new ForumArticleQuery()`
2.  将前端传过来的参数set给查询对象的属性
3.  从session获取userId ，判断是否dto是否为空，如果为空，表示用户没有登录，就只查询已审核的文章；如果不为空，则也查询用户未审核的文章。
4.  执行分页查询
5.  有查询出来的对象中的有些字段不需要，所以这里还需要将实体类转为VO类型才行
6.  返回给前端







### 加载发帖中的板块`/loadBoard4Post`

描述：用户在进行编辑帖子时，需要进行选择该帖子时属于哪一种板块的，所以该接口用于加载出所有的板块给用户选择。和主界面的`/loadBoard`接口差不多。

![image-20230905231642331](https://github.com/pan0724/Q-/assets/110321804/90042a4f-a75a-4394-833b-5e4e7213df36)


实现逻辑：

1. 从session中获取用户信息，判断是否为管理员，如果不是则把`postType` 置为`1` 表示所有用户都可以发帖。
2. 接着调用`forumBoardService.getBoardTree(postType )` 获取到树状的板块，也就是嵌套集合。
3. `getBoardTree(postType )`实现逻辑：
   1. 直接去数据库中查询`forumBoardMapper.selectList(forumBoardQuery)` 返回的是list集合，但此时的板块关系还不是树状的，因此要进行转换。
   2. `private List<ForumBoard> convertLine2Tree(List<ForumBoard> dataList, Integer pid)`将查询出来的list集合转换为树状，具体实现如下：
      1. 创建一个新的list集合`List<ForumBoard> children = new ArrayList();`
      2. 遍历参数中的list
      3. 遍历的过程中进行判断，在版块列表中查找所有父版块 ID 为指定值 `pid` 的版块，并将它们的子版块设置为递归调用该方法的返回值。具体来说，当找到一个父版块 ID 为 `pid` 的版块时，会将该版块的子版块设置为递归调用该方法的返回值，然后将该版块添加到一个列表中，最后将该列表作为返回值返回。
4. 在控制器返回给前端

知识点：

- 递归算法









### 发布文章接口`/postArticle`【重难点】

描述：用户在点击保存时，也就是要发帖了，就会请求该接口

![image-20230906182737659](https://github.com/pan0724/Q-/assets/110321804/5de4874e-2ada-4372-945e-122210e63982)


实现逻辑：

1. aop登录校验

2. 构建`ForumArticle forumArticle = new ForumArticle();` 然后设置各种属性

3. 构建附件信息对象`ForumArticleAttachment forumArticleAttachment = new ForumArticleAttachment();`

4. 然后调用业务层的方法`forumArticleService.postArticle(userDto.getAdmin(), forumArticle, forumArticleAttachment, cover, attachment);`

5. 业务层实现逻辑：

   1.先调用`checkArticle`方法进行一些基本的校验，其实现逻辑如下：

   1. 获取编辑器的枚举对象，然后判断，如果为null 则抛出”参数错误异常“
   2. 判断：如果文章摘要不为空且长度大于200，则抛出“参数错误异常”
   3. 接下来调用`resetBoardInfo` 方法，对板块信息重新设置： 具体实现如下：
      1. 根据前端传的boardId去数据库查询板块信息，看前端选的是什么板块
      2. 判断该板块用户是否有权限使用该板块，因为有的板块只有管理员才能发【短路与只有在左侧表达式为 false 时才会短路；短路或只有在左侧表达式为 true 时才会短路】
      3. 如果都没问题，就设置板块名称
      4. 再来查询二级板块，如果判断也没问题，则设置板块名称为二级板块的名称，否则就设置板块id为0名称为空字符

6. 设置最后更新时间为当前时间

7. 判断封面是否为空,不为null 则表示用户上传了文章封面。调用`fileUtils.uploadFile2Local（）`方法进行图片处理，然后通过该方法的返回值设置文章的封面属性

   1. 附件处理：不为null 则表示用户上传了文章附件。
   1. 调用`uploadAttachment` 方法将附件保存到本地并将信息存到数据库。其中保存到本地同样是使用` fileUtils.uploadFile2Local（）`方法完成

8. 文章审核：文章是否需要审核 , 判断是否是管理员,如果是管理员则不用审核;不是管理员则先从系统中获取发帖是否需要审核的设置，然后设置改文章的状态

9. 替换文章内容中的图片标签为本地图片路径（因为文章内容是原html格式的）

   

   


知识点：

- 递归
- 正则匹配，`Pattern` `Matcher`
- `FileUtils.copyFile()` 将源文件复制到目标文件
- ffmpeg压缩图片









### 获取文章详情`/getArticleDetail`

**描述**:该接口用于获取某个文章的详情，也就是当用户点击文章进入详情时，就会请求该接口。

![image-20230908104903723](https://github.com/pan0724/Q-/assets/110321804/f290bae3-9bd2-484a-9a80-1b49a9974d73)

实现逻辑：

1. 从session获取用户信息

2. 将文章阅读数量+1 `forumArticleService.readArticle(articleId);`

3. 判断文章是否为空、是否审核、是否已被删除、如果是未审核的，且当前用户不是该文章的作者也不是管理员身份的话，同样也抛出异常

4. 判断文章是否有附件`if (forumArticle.getAttachmentType() == 1) ` 如果有则进行获取附件。这里只用返回数据库中的附件信息即可，并不需要返回真正的附件文件。只有在用户点击下载附件的时候才调用下载接口来下载附件。

5. 判断用户是否点赞过该文章：如果在用户登录的状态下，去数据库查询文章点赞记录，如果自己已点赞过则设置 detailVO.setHaveLike(true);  该属性的作用主要是用于标记文章点赞的icon为红色的，为红色的就表示点赞过，图片如下

  ![image-20230908161724332](https://github.com/pan0724/Q-/assets/110321804/93523bb5-0d9c-4fc9-8894-70aa4754ab52)


6. 返回给前端数据`getSuccessResponseVO(detailVO);`

知识点：

- 需要考虑到用户登录和未登录的两个场景的不同业务的处理方法
- 需要判读用户是否是管理员身份的场景
- 需要判断文章状态、
- 对象如何转换？







### 点赞接口`/doLike`

描述：当你点击文章详情页中的点赞按钮时，会请求该接口

![image-20230908162735364](https://github.com/pan0724/Q-/assets/110321804/cb414d72-10ed-4053-876d-8b990ccbbc2f)


实现逻辑：

1. 从session获取用户信息

2. 调用服务层的点赞方法`likeRecordService.doLike()`  其实现逻辑如下：

   1. 当我们点赞该文章的时候，系统需要给作者发送消息，告诉他有人点赞了他的这篇文章，所以我们先创建一个消息对象`new UserMessage();`

   2. 二话不说，先设置该消息对象的创建时间属性

   3. 接下来通过`OperRecordOpTypeEnum`枚举结合Switch开关来判断是文章点赞还是评论点赞，从而设置给消息对象设置相应的属性值。以文章点赞为例，下面是进入switch的文章点赞case的实现逻辑：

      1. 先是调用` articleLike(objectId, userId, opTypeEnum);` 方法进行文章点赞记录的操作，也就是用户的点赞或取消点赞的核心逻辑。实现如下：

         先查询数据库中该用户的点赞记录。如果不为空，则表示用户之前已经点赞过该文章了，那此时用户的操作应该时取消点赞，所以这里要把记录删除掉，并且减少文章的点赞数量. 否则为空的话，即表示用户本次是要点赞该文章，那么就先查一下该文章是否存在,接着创建点赞记录对象，用于插入数据库中，记录用户的操作。然后 更新文章点赞数量。

      2. 根据文章id查询出文章，用户给消息对象set相应的值

      3. 然后设置消息对象的各个属性

   4. 设置发送消息的目标用户id、昵称、消息状态为未读

   5. 进行判断，符合条件才添加到数据库：

      ```java
       if (likeRecord == null && !userId.equals(userMessage.getReceivedUserId())) {
                  userMessageMapper.insert(userMessage);
              }
      ```

      如果点赞记录为空（也就是用户没有点赞过该文章），并且点赞的用户不是接受消息的用户，则将消息添加到数据库中记录。 因为如果用户已经点赞过了，那就之前肯定就已经保存过一次记录了，如果还插入，那就是重复了，而且自己点赞自己的文章，不需要发送消息给自己。所以这里主要是起到了一个过滤的作用。

3. 返回`return getSuccessResponseVO(null);`









### 获取用户下载信息`/getUserDownloadInfo`

描述：该接口用于在获取用户对附件的下载信息，比如是否下载过、用户积分有多少。  当用户点击下载附件时，会触发该请求接口，请求中会携带文件id这一个参数.

下面是用户首次下载该附件，则会弹出提示：

![image-20230915172002774](https://github.com/pan0724/Q-/assets/110321804/4fd81251-cfd9-4b83-9719-1c6d5dba045f)


当用户再次下载附件时，由于第一次已经支付过积分了，所以下一次点击下载时就不会弹出提示框了，如下图：

![image-20230915172729422](https://github.com/pan0724/Q-/assets/110321804/746fa897-21e2-42c3-b7d2-1afdbd955350)


**实现逻辑：**

1. 从session中获取用户信息
2. 从数据库查询用户所有字段信息
3. 创建一个HashMap集合，由于该接口只需返回给前端用户积分和是否下载过附件这两个字段，没有必要为这两个字段封装到一个VO类（当然，如果其他地方也用得到这两个字段，那么此时就可以考虑封装成类，但是这里只有这个接口用到这两个字段，所以就不封装了）。因此，在这里我们就将这两个字段放到HashMap集合中，然后把map返回给前端即可。
4. 由刚刚查询出的userInfo对象中获取用户积分，然后put到ma中
5. 根据附件id和用户di查询 出改用户的附件下载记录
6. 将是否有下载过附件的字段添加到map中        `result.put("haveDownload", attachmentDownload != null);`
7. 返回map给前端



**知识点：**

- session的生周期
- HashMap









### 附件下载`/attachmentDownload`

描述：用户点击下载按钮后，就会请求该接口，实际上与此同时他还请求`/getUserDownloadInfo`接。该接口是真正的将文件流返回给前端，而`getUserDownloadInfo`接口只是用于判断用户是否有下载过该附件，从而决定是否显示出弹窗提示。

![image-20230915190409275](https://github.com/pan0724/Q-/assets/110321804/a89e0813-aeb8-4f55-9970-eb91889df0a3)

前端按钮绑定到的点击事件方法如下 ：

```js
//下载附件
const downloadAttachment = async (fileId) => {
  if (!currentUserInfo.value) {
    store.commit("showLogin", true);
    return;
  }


  //获取用户下载信息
  let result = await proxy.Request({
    url: api.getUserDownloadInfo,
    params: {
      fileId: fileId,
    },
  });
  if (!result) {
    return;
  }
  //判断用户是否已下载过
  if (result.data.haveDownload) {
    downloadDo(fileId);
    return;
  }

  //判断用户积分是否够
  if (result.data.userIntegral < attachment.value.integral) {
    proxy.Message.warning("你的积分不够，无法下载");
    return;
  }

  proxy.Confirm(
    `你还有${result.data.userIntegral}积分，当前下载会扣除${attachment.value.integral}积分，确定要下载吗？`,
    () => {
      downloadDo(fileId);
    }
  );
};

const downloadDo = (fileId) => {
  document.location.href = api.attachmentDownload + "?fileId=" + fileId;
  attachment.value.downloadCount = attachment.value.downloadCount + 1;
};
```

可以看到这个方法中有两个请求，也就是当触发该方法时，会向后端请求两个接口：/attachmentDownload 和 `/getUserDownloadInfo`



**接口实现逻辑：**

1. 调用业务层的方法 `ForumArticleAttachment downloadAttachment(String fileId, SessionWebUserDto sessionWebUserDto);`  其实现逻辑如下：

   1. 给该方法添加上事物注解 `@Transational(rollbackFor=Exceptoin.clss)` 
   1. 根据文件id查询附件，然后判读如果附件为null，则抛出异常。
   1. 首先进行当前用户身份的判断，这里有两种情况：【1：如果当前用户与该附件的用户是同一个人，也就是当前登录的用户就是该作者，那么自己下载自己的附件就不需要积分】； 【2：如果不是同一个人，则需要去查询附件记录表，看该用户之前有没有下载过，如果查询结果为null，则表示用户本次的下载是属于首次下载，需要判断积分是否足以支付本次下载所需的积分。然后扣除积分】；
   1. 构建附件下载记录对象 `ForumArticleAttachmentDownload updateDownload =new ForumArticleAttachmentDownload();` ，并设置相应属性，然后添加到数据库中 `this.forumArticleAttachmentDownloadMapper.insertOrUpdate(updateDownload);`
   1. 更新附件表（注意不是附件下载记录表，而是附件表）中该附件的下载次数（也就是这里该附加被下载的总次数，包括所有人的下载）  `this.forumArticleAttachmentMapper.updateDownloadCount(fileId);`
   1. 扣除用户下载的积分 `userInfoService.updateUserIntegral`( )  
   1. 给提供附件的作者添加积分 ``userInfoService.updateUserIntegral`( )  `
   1. 构建消息记录对象并设置属性，然后添加数据库中。这样当用户登录的时候程序初始化时会调用其它接口来获取从数据库用户消息数。让用户就可以看到是谁下载该附件。
   1. 最后返回 附件对象给调用者

2. 业务层走完之后，让我们再次回到控制层继续往下的逻辑 ，其实接下来就是一些文件I/O的操作了...

3. 获取附件的完整路径，然后用File对象打开

4. 创建文件输入流 `new FileInputStream(file)`

5. 通过响应对象 `response` 获取 输出流对象  `response.getOutPutStream()`

6. 设置一些浏览器的编码问题

7. 设置相应头 `resposne.setHeader("Content-Dispositon","attachment;filename\""+downloadFileName+"\"")`

8. 创建字节缓冲区

9. 开始while循环写内容到输出流对象中去

10. 刷新 `out.flush();`

11. 在finall代码块进行关闭流的操作。

    

**知识点：**

- 文件读取
- 事物
- 浏览器响应头设置
- 浏览器编码设置













### 文章详情更新 `/articleDetail4Update`

描述：当用户要进行对自己现有的文章进行编辑时，进入编辑界面需要自动填充原来文章的内容到编辑区中去。因此该接口就是为这个提供文章内容数据用的。

![image-20230916204245259](https://github.com/pan0724/Q-/assets/110321804/cf77e5a8-7f7e-4e1d-9244-d13469909c51)


路由到如下编辑界面：

![image-20230916204556124](https://github.com/pan0724/Q-/assets/110321804/6e9314ef-8874-451d-8cd7-1d5125752396)




**实现逻辑：**

1. 从session 获取用户信息
2. 根据文章id查询文章
3. 判断文章是否存在，以及当前用户是否为文章的id，否则抛出异常
4. 构建文章更新详情VO `FormArticleUpdateDetailVO detailVO = new FormArticleUpdateDetailVO();`  用于返回给前端，主要有文章对象和附件对象两个参数
5. 将文章对象赋值给 `detailVO` 相应的属性
6. 接下来就是要将附件的属性也设置给 `detailVO ` ，但是先要进行判断：
   1. 判断文章是否有附件：`if (forumArticle.getAttachmentType() == 1) ` 等于 1 表示有附件，则进行下面的操作
   2. 构建附件查询对象 `ForumArticleAttachmentQuery articleAttachmentQuery = new ForumArticleAttachmentQuery();`
   3. 只需给附件查询对象设置文章id属性即可 `articleAttachmentQuery.setArticleId(forumArticle.getArticleId());`
   4. 进行分页查询（因为可能会有多个附件）`forumArticleAttachmentService.findListByParam(articleAttachmentQuery);`
   5. 如果查询结果不为空则进行如下操作：  先将查询出来的附件集合转成VO类型的对象（因为有的属性是不需要传给前端的，所以这里需要转换），然后给文章对象的附件属性赋值 `detailVO.setAttachment()`
7. 最后返回ResponseVO给前端









### 搜索 ` /search`

描述：该接口用于搜索文章。通过关键之进行模糊查询

![image-20230916222615137](https://github.com/pan0724/Q-/assets/110321804/22364299-d497-4d80-bb26-957c2bc04d2d)


实现逻辑：

1. 构建文章查询对象 `ForumArticleQuery query = new ForumArticleQuery();`
2. 设置模糊字段属性为前端传来的关键字 `query.setTitleFuzzy(keyword);`
3. 去数据库查询 `forumArticleService.findListByPage(query);`
4. 返回查询结果回前端



知识点：

- 模糊查询
- 前端高亮关键字













## 论坛板块相关ForumBoardController

------



### 加载板块接口 `/loadBoard`

描述：该接口用于在主界面加载顶部的板块信息（也就是文章的分类）

![image-20230917154645629](https://github.com/pan0724/Q-/assets/110321804/515a2310-9510-4097-aeb8-530963708507)


**实现逻辑：**

1. 调用业务层的方法 `forumBoardService.getBoardTree(null)` 获取树状结构的板块信息，下面是实现逻辑：

   1. 创建板块查询对象 `new FrumBoardQuery()`

   2. 设置排序方式

   3. 设置发帖类型（传null表示所有板块都查出来） ，也就是用户在发帖时是否可以发在该板块下 ` 0:只允许管理员发帖在该板块下 1:任何人都可以发帖在该板块下`

   4. 去数据库查询出所以的板块，返回一个List<ForumBoard> , 单此时的父子关系还是线性结构的，所以我们要将其转化成树状结构，因此调用`convertLine2Tree(forumBoardList, 0);` 方进行转换。  其实现逻辑如下：

      1. 对 原来的list集合进行遍历

      2. 如果当前的板块对象的pid等于参数的pid（也就是0），表示当前的板块对象是属于一级板块，那么此时就再调用一次该方法 ``convertLine2Tree(forumBoardList, 当前板块的id);` `  由于该方传入的是当前板块的id，那此时本次递归，该方法返回的是该id的子版块。 然后设置当前板块孩子属性 ：`   

         ```java
          private List<ForumBoard> convertLine2Tree(List<ForumBoard> dataList, Integer pid) {
                 List<ForumBoard> children = new ArrayList();
                 for (ForumBoard m : dataList) {
                     if (m.getpBoardId().equals(pid)) { // 也就是参数pid下有该子版块，将再次递归的返回值设置为该板块的孩子属性，知道递归没有。最后添加到list集合中返回
                         m.setChildren(convertLine2Tree(dataList, m.getBoardId()));// 设置其孩子值为下一次递归的板块集合
                         children.add(m);
                     }
                 }
                 return children;
             }
         ```

2. 返回给前端

   

**知识点：**

- 递归









## 评论相关ForumCommmentController

------



### 加载文章评论 `loadComment`

描述：当用户点进文章的详情页时，会请求该接口来拉取改文章的评论。

![image-20230917195905868](https://github.com/pan0724/Q-/assets/110321804/76e1ed79-120f-4816-a10e-5e95b53f4e1d)


**实现逻辑：**

1. 获取系统设置信息，检测系统是否开放评论功能。如果不开放，还请求到该接口的话，那么就认为这是没有通过正常的前端页面进行请求的接口，存在攻击嫌疑，所以抛出 “请求参数错误" 的异常  
2. 接下里就是构建 评论查询对象 了 `new ForumCommentQuery();`
3. 设置文章id
4. 设置是否加载子评论
5. 设置排序方式
6. 设置查询的页码
7. 从session中获取用户信息，如果不为空则 设置 “查询用户是否点赞“  和  设置当前用户id。否则就设置状态为 “已审核的”   【这里主要是区分请求该接口时用户是否登录，如果登录，那么就多查询用户自己点赞过的哪评论。如果没有登录，则只给看已审核过的评论，】
8. 设置一页的评论数是多少
9. 设置评论的pid
10. 进行分页查询 `forumCommentService.findListByPage(commentQuery)`
11. 将查询结果返回给前端

**总结：**

- 需要考虑到用户登录和 未登录时的两种情况下，分别该加载什么样的评论
- 需要考虑到如果 用户没传 页面大小 和 当前页码，则一个给予一个默认值









### 发表评论 `/postComment`

描述：用户在发布帖子时会请求该接口来完成

![image-20230919185925065](https://github.com/pan0724/Q-/assets/110321804/ee3882d4-28bb-45fd-8748-b81b9fcb7eeb)


**实现逻辑：**

1. 先检查系统是否开启评论功能，若不开启还请求到该接口的话，则抛出异常
2. 判断如果评论和图片都为空的话，则表示发表的评论是个空白，则抛出异常，因为不允许评论空的字符串
3. 
4. 













## 用户主页相关UserCenterController

------




