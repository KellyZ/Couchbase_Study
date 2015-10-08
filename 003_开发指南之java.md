# Couchbase

## 开发指南——java

### 下载SDK

截止2015-10-08，Couchbase-java-client的版本是2.2.0，下载地址： [这里](http://developer.couchbase.com/documentation/server/4.0/sdks/java-2.2/download-links.html)

配置好项目jar包依赖

### 示例

* 连接本地Coubase Server

		Cluster cluster = CouchbaseCluster.create();

	或连接特定服务：

		Cluster cluster = CouchbaseCluster.create("192.168.56.101", "192.168.56.102");

* 连接数据仓库

		Bucket bucket = cluster.openBucket("bucket", "password");

* 释放资源 （应用退出后必须释放资源）

		cluster.disconnect();

* 插入JsonDocument
	* 创建JsonObject
	
			JsonObject user = JsonObject.empty()
		    .put("firstname", "Walter")
		    .put("lastname", "White")
		    .put("job", "chemistry teacher")
		    .put("age", 50);

	* 创建JsonDocument，每个在bucket中的JsonDocument中必须有唯一DocumentID

			JsonDocument doc = JsonDocument.create("walter", user);
			JsonDocument response = bucket.upsert(doc);

* 获取JsonDocument

		JsonDocument walter = bucket.get("walter");
		System.out.println("Found: " + walter);

* 查找并更新示例：
	* 同步示例
	
			JsonDocument loaded = bucket.get("walter");
			if (loaded == null) {
			    System.err.println("Document not found!");
			} else {
			    loaded.content().put("age", 52);
			    JsonDocument updated = bucket.replace(loaded);
			    System.out.println("Updated: " + updated.id());
			}

	* 异步示例

			bucket
		    .async()
		    .get("walter")
		    .flatMap(new Func1<JsonDocument, Observable<JsonDocument>>() {
		        @Override
		        public Observable<JsonDocument> call(final JsonDocument loaded) {
		            loaded.content().put("age", 52);
		            return bucket.async().replace(loaded);
		        }
		    })
		    .subscribe(new Action1<JsonDocument>() {
		        @Override
		        public void call(final JsonDocument updated) {
		            System.out.println("Updated: " + updated.id());
		        }
		    });

	* 执行异步，主线程同步等待示例

			final CountDownLatch latch = new CountDownLatch(1);
			bucket
			    .async()
			    .get("walter")
			    .flatMap(loaded -> {
			        loaded.content().put("age", 52);
			        return bucket.async().replace(loaded);
			    })
			    .subscribe(
			        System.out::println,
			        err -> {
			            err.printStackTrace();
			            latch.countDown();
			        },
			        latch::countDown
			    );
			
			latch.await();

	* 上面示例更优雅的处理方式：

			JsonDocument result = bucket
			    .async()
			    .get("walter")
			    .flatMap(loaded -> {
			        loaded.content().put("age", 52);
			        return bucket.async().replace(loaded);
			    })
			    .toBlocking()
			    .single();

* 使用N1QL方式

	* 使用N1QL:

			String statement = "SELECT firstname FROM `default` WHERE age BETWEEN 49 AND 59";
			N1qlQuery query = N1qlQuery.simple(statement);
			N1qlQueryResult result = bucket.query(query);
			System.out.println("Hello, users in their fifties: ");
			for (N1qlQueryRow row : result) {
			    row.value().getString("firstname");
			}
			//prints "Hello, users in their fifties: Walter!

	* 使用N1QL DSL：
		
			Statement statement = Select.select(x("firstname")).from(i("default"))
				.where(x("age").between(49).and(59));
			N1qlQuery query = N1qlQuery.simple(statement);

### Web应用示例

