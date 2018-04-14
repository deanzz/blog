# 如何使用Play框架写出全程异步的服务接口

>Play框架是现在咱们团队在用的一个很优秀的Web框架，提供了写出高效的异步非阻塞服务接口的基础。但是在实际使用中，总是会出现写同步代码地方，
导致服务接口并不是全程异步，性能变差，所以我觉得有必要给大家演示一下如何在Play框架中写出全程异步的服务接口。

## 发现问题
目前elemental系统中datacenter、elemental-rest等服务接口模块中，大致分为三层，自顶向下是<br/>
1.Controller控制层<br/>
2.Service服务层<br/>
3.Dao数据访问层<br/>
即一个服务接口会由上面3个部分组成。<br/>
想要写出全程异步的服务接口，就得保证上面3个层面的代码都要是异步的。

### Dao数据访问层问题
该层中使用的mongodb驱动其实已是异步的，效率很好，但是由于一些历史原因，在Dao层的一些代码需要是同步的，所以为了兼容，在AbstractDaoImpl中会有这么一个隐式转换方法
```scala
protected implicit def await[A](future: Future[A]): A = Await.result(future, queryTimeout)
```
这个邪恶的方法会将异步驱动返回的异步future，悄悄的转换为同步过程（如果你子类中数据访问方法的返回类型不是Future），我承认这个邪恶的方法是我加的+_+。

### Service服务层问题
Service层接到Dao层的同步结果，顺理成章会接着写同步代码去使用这个同步结果，这样导致Service服务层也是同步的。

### Controller控制层问题
Controller层其实没什么问题，因为调用的是Play框架的Action方法，该方法默认就是异步的，<br/>
但是另一种情况，当Service层返回的是一个异步结果Future，由于Play官方推荐的Action方法需要的是一个返回Result的函数，<br/>
所以如果你使用下面的Action方法，你不得不把Service返回的Future执行Await成一个同步结果，最后组成一个Result返回。
```scala
final def apply(block: R[AnyContent] => Result): Action[AnyContent] = apply(BodyParsers.parse.default)(block)
```

## 解决问题
下面就针对上面三个层面的问题，一一解决。

### Dao数据访问层问题
其实这层的问题很简单，人家默认就能返回一个异步的Future，所以在子类中数据访问方法的返回类型写成Future即可，比如<br/>
同步写法，由于返回类型是`Seq[Project]`，所以会默认走await的隐式转换，将返回的future变成同步返回。
```scala
def getByProjectIds(projectIds: Seq[String]): Seq[Project] = {
    ...
    getCollection.find(in("_id", objectIds: _*)).toFuture()
  }
```
异步写法，修改方法返回类型为Future，就这么简单。
```scala
def getByProjectIds(projectIds: Seq[String]): Future[Seq[Project]] = {
    ...
    getCollection.find(in("_id", objectIds: _*)).toFuture()
  }
```

### Service服务层问题
现在Dao层返回的已经是一个异步的Future了，那么只要保证Service层也返回一个Future即可。<br/>
如何根据Dao层的Future再与Service的逻辑合并处理后再返回一个新的Future呢？<br/>
其实这个问题可以简化为，如何将多个Future转换成一个新的Future。<br/>
这个就要依靠Future本身强大的处理转换的函数了，你可以把Future看成一个集合List，你可以map它、flatMap它、zip它等等，基本可以满足任何需求。<br/>
还有一种更优雅的方式就是用for yield，能对Future进行组合处理，其实for yield就是转换成foreach、map、flatMap、filter或withFilter来实现的。<br/>
具体请见：[scala官方文档](http://docs.scala-lang.org/tutorials/FAQ/yield.html)<br/>
下面的具体实例中会详细演示。

### Controller控制层问题
现在Service层也已经返回了一个Future了，但是就像上面提到的问题，如果你使用Play官方推荐的默认Action方法，那么你不得不将Service层结果await后组成一个Result再返回。<br/>
不过Play还提供了另一个很好用的方法，那就是Action.async方法，它提供一种方式，将一个Future返回给框架处理。
```scala
final def async(block: R[AnyContent] => Future[Result]): Action[AnyContent] = async(BodyParsers.parse.default)(block)
```
所以，你现在要做的只是将Service层返回的Future，通过Future强大的处理转换函数或for yield，将几个Future转换成一个新的`Future[Result]`返回。<br/>
下面的具体实例中会详细演示。



### 实例
以本次后台清理需求中，查询用户各数据类型存储使用量的统计接口为例。

#### Dao层
Dao层的查询逻辑很多，所以这里只拿一个查询逻辑为例，具体代码可以看elemental-rest中的elemental.rest.dao包中代码。
```scala
@Singleton
class StorageStatisticsDao @Inject()(implicit injector: Injector) extends AbstractDaoImpl[StorageStatistics] {
  override protected def collectionName: String = "storage_statistics"

  import org.mongodb.scala.bson.codecs.Macros._
  private implicit val codecRegistry: CodecRegistry = fromRegistries(defaultRegistry, fromProviders(classOf[StorageStatistics]))

  def getByUserID(userId: String): Future[Seq[StorageStatistics]] = {
    getCollection.find(equal("userId", userId)).projection(and(include("storageType", "usedStorage"), exclude("_id"))).toFuture()
  }
}

case class StorageStatistics(storageType: String,
                             usedStorage: Option[Long])
```
这里重点是getByUserID方法的返回值是一个Future。

#### Service层
版本一，通过Future提供的转换函数，进行新Future的构建。
```scala
def storageStatistics(userId: String): Future[Seq[UserStorageStatistics]] = {
      val totalSizeFuture = storageStatisticsDao.getByUserID(userId)
      val uploadFileFtpCountFuture = dataCenterDao.countByOwnerAndType(userId, Some(Seq("FILE", "FTP")))
      val intermediateDataCountSizeFuture = projectDao.getIdAndUsedStorageByUserId(userId).flatMap {
        seq =>
          val totalSize = seq.map(_.usedStorage.getOrElse(0L)).sum
          workflowUseSpaceDao.countByProjectIds(seq.map(_._id.toString)).zip(Future(totalSize))
      }

      val finalFuture = dataCenterDao.getDataIdByOwnerAndType(userId).flatMap {
        seq =>
          val actionFileCountFuture = actionFileDao.countByDataIds(seq.map(_.dataId))
          val dynamicFileDataLinkCountFuture = dynamicFileHistoryDao.countByDataIds(seq.filter(h => h.`type` == "DYNAMIC_FILE" || h.`type` == "DATA_LINK").map(_.dataId))
          actionFileCountFuture.zip(dynamicFileDataLinkCountFuture).zip(uploadFileFtpCountFuture).zip(intermediateDataCountSizeFuture).zip(totalSizeFuture)
      }.map {
        case ((((actionFileCount, dynamicFileDataLinkCount), uploadFileFtpCount), (intermediateCount, intermediateTotalSize)), allTotalSizes) =>
          val dynamicFileCount = dynamicFileDataLinkCount.find(_._id == "DYNAMIC_FILE").map(_.count).getOrElse(0L)
          val dataLinkCount = dynamicFileDataLinkCount.find(_._id == "DATA_LINK").map(_.count).getOrElse(0L)
          val uploadFileCount = uploadFileFtpCount.find(_._id == "FILE").map(_.count).getOrElse(0L)
          val FTPCount = uploadFileFtpCount.find(_._id == "FTP").map(_.count).getOrElse(0L)
          val analysisFileCount = uploadFileCount
          val dynamicFileTotalSize = allTotalSizes.find(_.storageType == "DYNAMIC_FILE").map(_.usedStorage.getOrElse(0L)).getOrElse(0L)
          val dataLinkTotalSize = allTotalSizes.find(_.storageType == "DATA_LINK").map(_.usedStorage.getOrElse(0L)).getOrElse(0L)
          val FTPTotalSize = allTotalSizes.find(_.storageType == "FTP").map(_.usedStorage.getOrElse(0L)).getOrElse(0L)
          val fileTotalSize = allTotalSizes.find(_.storageType == "UPLOAD_FILE").map(_.usedStorage.getOrElse(0L)).getOrElse(0L)
          val uploadFileTotalSize = (fileTotalSize * 80) / 100
          val analysisFileTotalSize = (fileTotalSize * 20) / 100
          Seq(
            UserStorageStatistics("UPLOAD_FILE", uploadFileCount, uploadFileTotalSize),
            UserStorageStatistics("ANALYZED_FILE", analysisFileCount, analysisFileTotalSize),
            UserStorageStatistics("DYNAMIC_FILE", dynamicFileCount, dynamicFileTotalSize),
            UserStorageStatistics("DATA_LINK", dataLinkCount, dataLinkTotalSize),
            UserStorageStatistics("ACTION_FILE", actionFileCount.count, 0L),
            UserStorageStatistics("INTERMEDIATE_DATA", intermediateCount.count, intermediateTotalSize),
            UserStorageStatistics("FTP", FTPCount, FTPTotalSize)
          )
      }
      finalFuture
    }
```
版本二，通过for yield，进行新Future的构建，更直观。
```scala
def storageStatistics(userId: String): Future[Seq[UserStorageStatistics]] = {
    val intermediateDataCountSizeFuture = for {
      seq <- projectDao.getIdAndUsedStorageByUserId(userId)
      count <- workflowUseSpaceDao.countByProjectIds(seq.map(_._id.toString))
      totalSize <- Future(seq.map(_.usedStorage.getOrElse(0L)).sum)
    } yield (count, totalSize)

    val finalFuture = for {
      seq <- dataCenterDao.getDataIdByOwnerAndType(userId)
      actionFileCount <- actionFileDao.countByDataIds(seq.map(_.dataId))
      dynamicFileDataLinkCount <- dynamicFileHistoryDao.countByDataIds(seq.filter(h => h.`type` == "DYNAMIC_FILE" || h.`type` == "DATA_LINK").map(_.dataId))
      uploadFileFtpCount <- dataCenterDao.countByOwnerAndType(userId, Some(Seq("FILE", "FTP")))
      (intermediateCount, intermediateTotalSize) <- intermediateDataCountSizeFuture
      allTotalSizes <- storageStatisticsDao.getByUserID(userId)
    } yield {
      val dynamicFileCount = dynamicFileDataLinkCount.find(_._id == "DYNAMIC_FILE").map(_.count).getOrElse(0L)
      val dataLinkCount = dynamicFileDataLinkCount.find(_._id == "DATA_LINK").map(_.count).getOrElse(0L)
      val uploadFileCount = uploadFileFtpCount.find(_._id == "FILE").map(_.count).getOrElse(0L)
      val FTPCount = uploadFileFtpCount.find(_._id == "FTP").map(_.count).getOrElse(0L)
      val analysisFileCount = uploadFileCount
      val dynamicFileTotalSize = allTotalSizes.find(_.storageType == "DYNAMIC_FILE").map(_.usedStorage.getOrElse(0L)).getOrElse(0L)
      val dataLinkTotalSize = allTotalSizes.find(_.storageType == "DATA_LINK").map(_.usedStorage.getOrElse(0L)).getOrElse(0L)
      val FTPTotalSize = allTotalSizes.find(_.storageType == "FTP").map(_.usedStorage.getOrElse(0L)).getOrElse(0L)
      val fileTotalSize = allTotalSizes.find(_.storageType == "UPLOAD_FILE").map(_.usedStorage.getOrElse(0L)).getOrElse(0L)
      val uploadFileTotalSize = (fileTotalSize * 80) / 100
      val analysisFileTotalSize = (fileTotalSize * 20) / 100
      Seq(
        UserStorageStatistics("UPLOAD_FILE", uploadFileCount, uploadFileTotalSize),
        UserStorageStatistics("ANALYZED_FILE", analysisFileCount, analysisFileTotalSize),
        UserStorageStatistics("DYNAMIC_FILE", dynamicFileCount, dynamicFileTotalSize),
        UserStorageStatistics("DATA_LINK", dataLinkCount, dataLinkTotalSize),
        UserStorageStatistics("ACTION_FILE", actionFileCount.count, 0L),
        UserStorageStatistics("INTERMEDIATE_DATA", intermediateCount.count, intermediateTotalSize),
        UserStorageStatistics("FTP", FTPCount, FTPTotalSize)
      )
    }
    finalFuture
  }
```


#### Controller层
这里重点是使用了Play提供的Action.async方法，返回一个Future给框架。
```scala
def storageStatistics(): Action[AnyContent] = Action.async { implicit request =>
    val userId = request.tags("kmUserId")
    val f = storageResourceService.storageStatistics(userId)
    f.map{
      seq =>
        ResponseAssemblyUtil.ok(StorageStatisticsList(userId, seq))
    }.recover{
      case kme: KMException =>
        log.error(s"storageStatistics got KMException, ${kme.getMessage}", kme)
        ResponseAssemblyUtil.error(kme)
      case e: Throwable =>
        log.error(s"storageStatistics got Throwable exception, ${e.getMessage}", e)
        ResponseAssemblyUtil.error(new KMException(RespCode.UNKNOWN_EXCEPTION_CODE, e.getMessage))
    }
  }
```

## 总结
至此，一个全程异步的服务接口就完成了，如果再配合高效的dispatcher的设计，这样写出的服务接口会相当高效。<br/>
异步非阻塞可以说是当代服务接口的标配，所以大家也要与时俱进，写出更高效的服务接口，<br/>
不仅是为了满足elemental系统的高性能要求，而且也为了转变自己设计、编码服务接口的思维方式，这样写出异步非阻塞的服务接口就信手捏来。<br/>
所以最后再次强调一下，异步非阻塞，异步非阻塞，异步非阻塞！<br/>
重要的事情说三遍。