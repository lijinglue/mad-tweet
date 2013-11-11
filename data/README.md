Data processing
==================

1. ** Set up environment **
------------------

The data sources are processed with Hive over Hadoop.

You need to get a hive and a hadoop up and running first and put the files in the hdfs. The data provided by twitter are in json format. Hive doesn't support that out of the box. We are going to use a JSONSerDe to help us serialize/de-serialize them.

I would use a proven serde here, you can download and compile it from

[Cloudera Serde] (https://github.com/cloudera/cdh-twitter-example.git)


2. ** Creating table on the data source **
------------------

CREATE EXTERNAL TABLE tweets (
  id BIGINT,
  created_at STRING,
  source STRING,
  favorited BOOLEAN,
  geo STRUCT<
    type:STRING,
    coordinates:ARRAY<STRING>
  >,
  place STRUCT<
    country_code:STRING,
    place_type:STRING,
    name:STRING,
    full_name:STRING,
    bounding_box:STRUCT<
      type:STRING,
      coordinates:ARRAY<ARRAY<ARRAY<STRING>>>
    >
  >,
  retweeted_status STRUCT<
    text:STRING,
    user:STRUCT<screen_name:STRING,name:STRING>,
    retweet_count:INT>,
  entities STRUCT<
    urls:ARRAY<STRUCT<expanded_url:STRING>>,
    user_mentions:ARRAY<STRUCT<screen_name:STRING,name:STRING>>,
    hashtags:ARRAY<STRUCT<text:STRING>>>,
  text STRING,
  user STRUCT<
    screen_name:STRING,
    name:STRING,
    friends_count:INT,
    followers_count:INT,
    statuses_count:INT,
    verified:BOOLEAN,
    utc_offset:INT,
    time_zone:STRING>,
  in_reply_to_screen_name STRING
)
ROW FORMAT SERDE 'com.cloudera.hive.serde.JSONSerDe'
LOCATION '/user/lijinglue/input';



3. ** Run queries **
-----------------

The queries are straight forward. I'm going to take top hastags, users , cities and ngrams for presentation.They will be dlimited by comma and directly output to my local fs.

The results of those queries are save in this directory as top_hash_tag.csv, active_user.csv, active_city.csv and ngrams.csv respectively

SELECT TOP HASHTAGs
-------------------

INSERT OVERWRITE LOCAL DIRECTORY '/Users/lijinglue/Documents/data/hashtags'
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
SELECT
 LOWER(hashtags.text),
 COUNT(*) AS total_count
FROM tweets
LATERAL VIEW EXPLODE(entities.hashtags) t1 AS hashtags
GROUP BY LOWER(hashtags.text)
ORDER BY total_count DESC
LIMIT 20;

SELECT MOST ACTIVE CITY
----------------------

INSERT OVERWRITE LOCAL DIRECTORY '/Users/lijinglue/Documents/data/cities'
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
SELECT
  place.name,place.bounding_box.coordinates[0][0][0],place.bounding_box.coordinates[0][0][1], count(*) as TOTAL_TWEETS FROM tweets
  WHERE place.place_type='city'
  GROUP BY place.name,place.bounding_box.coordinates
  ORDER BY TOTAL_TWEETS DESC
  LIMIT 20;

TOP ACTIVE USER
---------------

INSERT OVERWRITE LOCAL DIRECTORY '/Users/lijinglue/Documents/data/users'
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
SELECT
  user.screen_name, count(*) as TWEET_COUNT FROM tweets
  GROUP BY user.screen_name
  ORDER BY TWEET_COUNT DESC
  LIMIT 20;

n-grams
-----------

INSERT OVERWRITE LOCAL DIRECTORY '/Users/lijinglue/Documents/data/ngram'
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
    SELECT ngrams(sentences(lower(text)),3,10) FROM tweets;

