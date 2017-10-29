# CCA-175-hints
A repo to hold hints and tips and exercises for CCA-175 Hadoop and Spark Cloudera Certification

**Get mysql hostname (sometimes not provided in exam)**
1. login mysql as root
$ mysql -u root -p
 
2. mysql> use mysql;
3. mysql> select host, user from user;
+---------------------+------------+
| host | user |
+---------------------+------------+
… 
| 127.0.0.1 | root |
| localhost | root |
| quickstart.cloudera | root |
+---------------------+------------+

*Use compression on sqoop*
sqoop import-all-tables \
--connect "jdbc:mysql://quickstart.cloudera:3306/retail_db" \
--username retail_dba \
--password cloudera \
--warehouse-dir /user/hive/warehouse/retail_stage.db \
--compress \
--compression-codec snappy \
--as-avrodatafile \
-m 1;

**Import with --query:  necessary $CONDITIONS and --split-by**
sqoop import --connect jdbc:mysql://localhost/retail_db --username root --password cloudera --as-parquetfile   --hive-import --hive-database "test"  --hive-table "stupidqueryParquet"  --hive-overwrite --create-hive-table  --target-dir /user/cloudera/basuritas  --query 'SELECT * FROM orders WHERE order_id < 10 AND  $CONDITIONS' --split-by 'order_id'

**Sqoop Hive import limitations**
Sqoop does have a create-hive-table tool which can create a Hive schema.. but it only works with delimited file formats. 

**Sqoop export: Column position corresponds to source, name to destination**
Sqoop export ...
--columns “product_id, product_description”

Means it only takes columns 0 and 1 from source, and will go to those names in mysql table.


**Use avro on output**
-> import com.databricks.spark.avro._;
-> sqlContext.setConf("spark.sql.avro.compression.codec","snappy")
->dataFrameResult.write.avro("/user/cloudera/problem2/products/result-df");

**Check avro schema**
En el caso de avro, se crea al hacer el import
avro-tools getschema part-m-00000.avro > orders.avsc

**Describe data from a Hive Table**
Describe formatted mytable
**Create table HIVE with AVRO with HDFS schema**
First import to warehouse, when create the table passing it a schema via TBLPROPERTIES
CREATE EXTERNAL table orders_sqoop
STORED AS AVRO
LOCATION '/user/hive/warehouse/retail_stage.db/orders'
TBLPROPERTIES ('avro.schema.url'='/user/hive/schemas/order/orders.avsc')

**HIVE DDL with arrays and maps**
Data:
1000000|suresh,chimmili|ongl,ap|42|sad:false

Table:
CREATE TABLE mytable (
age array<smallint>
feel map<string,boolean>
address struct<city:String,state:String>
) 
ROW FORMAT DELIMITED
FIELD DELIMITED BY “,” 
COLLECTION ITEMS DELIMITED BY “;” 
MAP KEYS DELIMITED BY “:” 
STORED AS TEXTFILE 
See an example of Complex Data Types

**Load data from other table** 
 INSERT OVERWRITE TABLE orders_avro
  PARTITION (order_month)
  SELECT *, substr(from_unixtime(cast(order_date/1000 as int)),1,7)  as order_month FROM orders_sqoop ;

**Create partitioned table**
**1. Giving a preexistent schema**
create table orders_avro
PARTITIONED BY  (order_month STRING)
STORED AS AVRO
TBLPROPERTIES ("avro.schema.url"="/user/hive/schemas/orders.avsc");

**2. providing the schema**
create table orders_avro_prv (
  order_id INT,
  order_date STRING,
  order_customer_id INT,
  order_status STRING
)
PARTITIONED BY  (order_month STRING)
STORED AS AVRO;


TODO review timestamps en Impala, vale más crear la tabla en Hive. Falta de funciones de fecha. Ver si sigue así. 

**Save to parquet: compression settings on sqlContext.setConf**
var dataFile = sqlContext.read.avro("/user/cloudera/problem5/avro");
sqlContext.setConf("spark.sql.parquet.compression.codec","snappy");
dataFile.repartition(1).write.parquet("/user/cloudera/problem5/parquet-snappy-compress");
Save as text file, compressed with Gzip
dataFile.map(x=> x(0)+"\t"+x(1)+"\t"+x(2)+"\t"+x(3)).saveAsTextFile("/user/cloudera/problem5/text-gzip-compress",classOf[org.apache.hadoop.io.compress.GzipCodec]);


**Impala invalidate metadata**
1. Run 'Invalidate metadata'

**Change memory params to spark-shell (important!! Launch it first like this, otherwise you will have to open a new shell and create your computations anew)**
spark-shell --driver-memory 4G --executor-memory 4G --executor-cores 8
… more on spark-shell --help
Crear un HiveContext (si no viene ya de propio, que en la vm 5.12 sí)
var hc = new org.apache.spark.sql.hive.HiveContext(sc);
If no hive-site.xml config is in Spark maybe it’s because there is no hive-site.xml file in Spark config. 
Sudo ln -s /usr/lib/hive/conf/hive-site.xml /usr/lib/spark/conf/hive-site.xml 

**AggregateByKey: Do this query via groupByKey and via aggregateByKey.**
SELECT age, avg(rating) average
FROM users u, ratings r
WHERE u.user_id = r.user_id
GROUP BY age 
ORDER BY average
// Get users with the SELECT. 
val users = sc.textFile("file:///home/cloudera/course-spark/datasets/movielens-1M/users.dat").map(_.split("::")).map (row => User( row(0).toInt, row(1), row(2).toInt, row(3), row(4))  )
val usersKv = users.map(user => (user.id, user));
// Get ratings with the SELECT 
val ratings = sc.textFile("file:///home/cloudera/course-spark/datasets/movielens-1M/ratings.dat").map(_.split("::")).map(row => Rating(row(0).toInt, row(1).toInt, row(2).toInt, row(3).toLong  ) )
val ratingsKv = ratings.map(rating => (rating.userId, rating) );
// join 
val usersJoined = usersKv.join(ratingsKv);
// get the age, rating. 
val couples  =  usersJoined.map(row => ( row._2._1.age, row._2._2.rating ) );
// aggregate on a tuple: (count, sum) -- ojito que la sintaxis es fina. Pero así es más clara.  
val aggregated = couples.aggregateByKey
(	(0,0) ) 
(  	(acc: (Int,Int) , value: Int)  => (acc._1 + 1, acc._2 + value) , 
(acc: (Int, Int), acc2: (Int, Int)) => (acc._1 + acc2._1, acc._2 + acc2._2)   
)
// do the math, swap and sort. Cuidado con el toDouble para las divisiones! 
val result = aggregated.map(a => (a._1, a._2._2.toDouble / a._2._1)) .map(_.swap).sortByKey(false)

// WIthout aggregateByKey, doing the math on the iterator… this is less scalable but much simpler. 
usersByUserId.join(ratingsByUserId).
| map(tuple => (tuple._1, tuple._2._1.age, tuple._2._2.rating)).
| map({case(userId, age, rating) => (age, rating)}).
| groupByKey.   // easier aggregation like that! 
map({case(age, ratings) => (age, ratings.sum.toDouble/ratings.toList.size)})


**Windowing functions: rank, dense_rank, percent_rank (SQL), denseRank(), percentRank() DF API** 
**Windowing (rank and other functions): Python** 
from pyspark.sql.window import Window
import pyspark.sql.functions as func
windowSpec = \
  Window 
    .partitionBy(df['category']) \
    .orderBy(df['revenue'].desc()) \
    .rangeBetween(-sys.maxsize, sys.maxsize)
# also rowsBetween exists, for preceding and following #rows.
dataFrame = sqlContext.table("productRevenue")
revenue_difference = \
  (func.max(dataFrame['revenue']).over(windowSpec) - dataFrame['revenue'])
dataFrame.select(
  dataFrame['product'],
  dataFrame['category'],
  dataFrame['revenue'],
  revenue_difference.alias("revenue_difference"))

**Windowing (rank and other functions): Scala**
val w = Window.orderBy($"value")
// ... 
df.select($"user", rank.over(w).alias("rank")).show
Windowing (rank and other functions): SQL 
OVER (PARTITION BY ... ORDER BY ... frame_type BETWEEN start AND end)
rank() OVER (PARTITION BY d.department_id ORDER BY p.product_price) AS product_price_rank

**Python shell: include autocompletion**
>>> import rlcompleter, readline
>>> readline.parse_and_bind('tab:complete')


**Hive ordering**  
ORDER BY x: guarantees global ordering, but does this by pushing all data through just one reducer. This is basically unacceptable for large datasets. You end up one sorted file as output.
SORT BY x: orders data at each of N reducers, but each reducer can receive overlapping ranges of data. You end up with N or more sorted files with overlapping ranges.
DISTRIBUTE BY x: ensures each of N reducers gets non-overlapping ranges of x, but doesn't sort the output of each reducer. You end up with N or unsorted files with non-overlapping ranges.
CLUSTER BY x: ensures each of N reducers gets non-overlapping ranges, then sorts by those ranges at the reducers. This gives you global ordering, and is the same as doing (DISTRIBUTE BY x and SORT BY x). You end up with N or more sorted files with non-overlapping ranges.
Make sense? So CLUSTER BY is basically the more scalable version of ORDER BY.

**Scala Imports**
// this is used to implicitly convert an RDD to a DataFrame. import sqlContext.implicits._
============== Hadoop Related ============
import org.apache.hadoop.io._ ==>{IntWritable,NullWritable,FloatWritable,Text,DoubleWritable}
import org.apache.hadoop.io.compress._ =>{SnappyCodec,GzipCodec}
import org.apache.hadoop.mapred._ =>{TextInputFormat,TextOutputFormat,SequenceFileInputFormat,SequenceFileOutputFormat,KeyValueTextInputFormat} OLD API
import org.apache.hadoop.mapreduce.lib.input._
import org.apache.hadoop.mapreduce.lib.output._
=========== Save As New API Hadoop FIle and Retrival ============
import org.apache.hadoop.conf.Configuration
val conf = new Configuration()
conf.set("textinputformat.record.delimeter","\u0001") // required when reading newAPIHadoopFile in classOf[TextInputFormat]
newRDD.saveAsNewAPIHadoopFile("/user/vishvaspatel34/result/deptNewAPIHadoopFile",classOf[IntWritable],classOf[Text],classOf[TextOutputFormat[IntWritable,Text]])
val newAPIRDD = sc.newAPIHadoopFile("/user/vishvaspatel34/result/deptNewAPIHadoopFile",classOf[TextInputFormat],classOf[LongWritable],classOf[Text])
newAPIRDD.map(x=>x._2.toString).collect().foreach(println)
In above I have stored as IntWritable but I have to retrive as LongWritable
============= SaveAsObjectFIle =============
============ Scala Related ===============
import scala.math._
========== DataBricks ==================== com.databricks:spark-csv_2.10:1.5.0 , com.databricks:spark-avro_2.10:2.1.0
import com.databricks.spark.avro._
import com.databricks.spark.csv._
============== Spark Related =============
import org.apache.spark.sql.SQLContext
import org.apache.spark.sql.hive.HiveContext
import org.apache.spark.storage.StorageLevel =>{StorageLevel.DISK_ONLY, MEMORY_ONLY}
import sqlContext.implicits._
import org.apache.spark.sql.SaveMode._ =>{Append,ErrorIfExists,Ignore,Overwrite,musq}
------------ Sql Context ------------------
sqlContext.setConf("spark.sql.shuffle.partitions","2")
sqlContext.setCont("spark.sql.parquet.compression.codec","snappy") =>{snappy,gzip,lzo}
sqlContext.setCont("spark.sql.avro.compression.codec","snappy") =>{snappy,gzip,lzo}

**Exam Delivery**
During the exam, I faced some issues and wasted almost around 30 minutes in my examination and panicked. Some tips for a smooth exam would be
1. Please go through the videos towards the end of the CCA playlist that covers the common issues faced by exam takers, it is very important please don't neglect it. 
2. Try to login into a Ubuntu machine and become familiar with how to use it. 
3. Learn on how to increase the font size of terminal (Go to preferences on the left top corner)
4. Learn on how to run a .sh file, in case if you are not able to run it, you can open the file using vi editor and run each of the commands individually. 
5. Make sure you go through the question completely and then start coding, as the expected output might be different than what you thought.
6. Validate the output, before moving to the next question helps if in the last minute you are not able to verify your answers.
7. Set aside 15 minutes of time in the end and verify all your answers.
8. If the time is over, don't leave any unsaved files and quit the examination, save all files and then end the examination.
