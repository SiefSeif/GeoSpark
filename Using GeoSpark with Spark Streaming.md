## Using GeoSpark with Spark Streaming

Here's a model for using GeoSpark with Spark Streaming, hope it helps.

### Prerequisites

* Streams from Apache Kafka 10, as direct stream (no receiver), You could use any streaming source else.
* Spark 2.0
* Scala 2.11.8
* GeoSpark 0.8.2

### Use Case

* Streaming of CSV separated coordinates (Longitude, Latitude), each line represents a point, you could change that and parse it to Points the way you like.
* I'm using JoinQuery of Polygons and Points.
* Polygons are read from a local CSV file.

### Code Example

```
import com.vividsolutions.jts.geom.{Coordinate, GeometryFactory, Point}
import java.lang.Double

import org.apache.kafka.clients.consumer.ConsumerRecord
import org.apache.spark.storage.StorageLevel
import org.apache.spark.{SparkConf, SparkContext}
import org.apache.spark.streaming.{Seconds, StreamingContext}
import org.apache.spark.streaming.kafka010.{ConsumerStrategies, KafkaUtils, LocationStrategies}
import org.apache.kafka.common.TopicPartition
import org.datasyslab.geospark.enums.{FileDataSplitter, GridType, IndexType}
import org.datasyslab.geospark.spatialOperator.JoinQuery
import org.datasyslab.geospark.spatialRDD.{PointRDD, PolygonRDD}
import org.apache.kafka.common.serialization.StringDeserializer
import org.apache.spark.rdd.RDD

object StreamingKafkaGeoSpark2 {

  def main(args: Array[String]) {
  
    val conf = new SparkConf().setAppName("Application").setMaster("local[*]")
    val sc = new SparkContext(conf)
    val ssc = new StreamingContext(sc, Seconds(1))

    val kafkaParams = Map(
      "bootstrap.servers" -> "localhost:9092",
      "key.deserializer" -> classOf[StringDeserializer],
      "value.deserializer" -> classOf[StringDeserializer],
      "group.id" -> "points-group",
      "auto.offset.reset" -> "latest")
    val kafkaTopics = List("points")
    val offsets = Map(new TopicPartition("points", 0) -> 2L)
	
	// the DStream
    val lines = KafkaUtils.createDirectStream[String, String](ssc, LocationStrategies.PreferConsistent,
      ConsumerStrategies.Subscribe[String, String](kafkaTopics, kafkaParams, offsets))

    val polygonRDD = new PolygonRDD(sc,"/home/cloudera/Desktop/SmallTest/polygon1k.csv",FileDataSplitter.CSV, true,10,StorageLevel.MEMORY_ONLY);

    val geoFactory = new GeometryFactory()

// for each RDD perform your Query

    lines.foreachRDD({
      rdd => {
		
        // create RDD of points out of RDD of strings.
        // here, you could parse whatever format the streaming is.
          
		  val points = rdd.map(record =>
          geoFactory.createPoint(new Coordinate(
		  Double.parseDouble(record.value().split(",")(0)),
		  Double.parseDouble(record.value().split(",")(1)))))

     // if rdd not empty, perform query
        if(!rdd.isEmpty()) {

          val pointsRDD = new PointRDD(points,StorageLevel.NONE)

          
		  //  set partitioning type, and sample type if required
          //  pointsRDD.setSampleNumber(200000L)
          pointsRDD.spatialPartitioning(GridType.QUADTREE)
          polygonRDD.spatialPartitioning(pointsRDD.partitionTree)

          // build index
          pointsRDD.buildIndex(IndexType.RTREE, true)

          pointRDD.indexedRDD.persist(StorageLevel.MEMORY_ONLY)
          polygonRDD.spatialPartitionedRDD.persist(StorageLevel.MEMORY_ONLY)

          // SpatialJoinQuery returns RDD of Pairs <Polygon, HashSet<Point>>
          val joinResultRDD = JoinQuery.SpatialJoinQuery(pointsRDD, polygonRDD, true, false)
 
             }
        }
    })

    ssc.start()
    ssc.awaitTermination()
  }

}```
