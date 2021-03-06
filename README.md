# Overview of project
- Web services in **Golang** to handle posts, seardh and user login, logout deployed to **Google App Engine(GAE flex)**.
- **ElasticSearch** in **GCE** provide storage and geo-location based search for user nearby posts within a distance.The center of the peoject is to search with Elastic Search based on GeoIndex(query optimization) and save post to Elastic Search. 
- Use **Google Cloud Storage(GCS)** as an assistance for Elastic Search by saving images.
- Use **OAuth** 2.0 to support token based authentication. user.go handles login and signup based on Elastic Search. main.go use JWT to protect post and search endpoints. When we do the requests, we need to add token to verify the identity.
- Use **Google Dataflow** to dump posts from **BigTable** to **BigQuery** for offline analysis.

- Here is the high level of the project 
![image](https://github.com/donghai1/Soical-Network-System-/blob/master/demo/project.png)



# Services provided and API design
- **/signup**
  * save to elasticSearch.
- **/login**
  * check login credential in elasticSearch, if correct return token
- **/search** - search nearby posts.
  1. have token-based authentication first.
  2. search in redis cache, if not found then search elasticSearch(lazy-loading).
  3. use  `"type" : "geo_point"` to map (lat, lon) to geo_point, ES will use geo-indexing to search(**KD tree**) nearby posts.
- **/post**
  1. save post image in GCS. 
  2. save post info in ElasticSearch, bigTable(optional).
 - Here is the logic chain 
 ![image](https://github.com/donghai1/Soical-Network-System-/blob/master/demo/log.png)

# Storage
- ElasticSearch(save user and post infos)
  * user info example
  ```json
   {
      "_index" : "around",
      "_type" : "user",
      "_id" : "jack",
      "_score" : 1.0,
      "_source" : {
        "username" : "jack",
        "password" : "jack"
      }
    }
  ```
  * post info example
  ```
  {
      "_index" : "around",
      "_type" : "post",
      "_id" : "b2c32515-c07d-4154-b2b1-6c7ab5e06d42",
      "_score" : 1.0,
      "_source" : {
        "user" : "jack",
        "message" : "Nice star!",
        "location" : {
          "lat" : 44.70415541365263,
          "lon" : -78.12385288120937
        },
        "url" : "https://www.googleapis.com/download/storage/v1/b/.../6c7ab5e06d42?generation=1522902512391320&alt=media"
      }
    }
  ```
  
- Google Cloud Storage
  * elasticserach saves image url, GCS store real image file.
- Redis
  * redis can be simply regarded as key-value store
    * **key**: **lat:lon:range**, range is redius based on (lat,lon) as circle center.
    * **value**: **post info**
- BigTable, BigQuery
  * we can save posts data to BigTable, use Dataflow to pass posts from BigTable to BigQuery for data analysis. See DataFlow code [here](./dataflow/src/main/java/com/around/PostDumpFlow.java).
  * Several cases can be done in BigQuery:
    * get number of posts per user id -- base to detect spam user.
    * find all messages in LA(lat range [33, 34], lon range [-118, -117]) -- base for geo-based filter.
    * find all message with spam words

 # Data Analysis 
 - Google BigTable, DataFlow and BigQuery are combined for use of data analysis 
 - Here is the demonstration 
 ![image](https://github.com/donghai1/Soical-Network-System-/blob/master/demo/data.JPG)
 
 
 # References
 - [QuickStart](https://cloud.google.com/appengine/docs/flexible/go/quickstart) for Go in GAE flex.
 - [Example](https://github.com/olivere/elastic) elastic search in go.
 - [Example](https://github.com/GoogleCloudPlatform/golang-samples/blob/master/storage/objects/main.go) writing object to GCS.
 - [Example](https://cloud.google.com/storage/docs/reference/libraries#client-libraries-install-go) client connection to GCS. 
 - [Example](https://cloud.google.com/dataflow/model/bigquery-io#writing-to-bigquery) writing to BigQuery
 - [Example](https://github.com/golang-samples/http/blob/master/fileupload/main.go) Go parseMultipart form
 - [uuid](https://github.com/pborman/uuid) : each post is unique
 - [encoding](https://golang.org/pkg/encoding/json/) json

