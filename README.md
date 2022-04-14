# alluxio-trino-iceberg

Env for local tests of alluxio-trino-iceberg stack <br> Steps to spin-up alluxio-trino iceberg stack:


1. download test data (~150 MB; ~1M rows) and put it in alluxio UFS in `/taxi` prefix <br> ```mkdir -p /tmp/alluxio_ufs/taxi && wget https://s3.amazonaws.com/nyc-tlc/trip+data/green_tripdata_2014-02.csv -O /tmp/alluxio_ufs/taxi/data.csv ```
2. spin up env<br> `docker-compose up --build`
3. Create hive table over downloaded CSV test data <br>  `docker exec -it trino trino --catalog hive` <br> 
   ```
   CREATE SCHEMA test_hive;
   USE test_hive;
   CREATE TABLE tlc_green_trips_201402 (
         vendorid VARCHAR,
         tpep_pickup_datetime VARCHAR,
         tpep_dropoff_datetime VARCHAR,
         store_and_fwd_flag VARCHAR,
         rate_code VARCHAR,
         pickup_longitude VARCHAR,
         pickup_latitude VARCHAR,
         dropoff_longitude VARCHAR,
         dropoff_latitude VARCHAR,
         passenger_count VARCHAR,
         trip_distance VARCHAR,
         fare_amount VARCHAR,
         extra VARCHAR,
         mTA_tax VARCHAR,
         tip_amount VARCHAR,
         tolls_amount VARCHAR,
         ehail_fee VARCHAR,
         total_amount VARCHAR,
         payment_type VARCHAR,
         trip_type VARCHAR)
       WITH (FORMAT = 'CSV', skip_header_line_count = 3, EXTERNAL_LOCATION = 'alluxio://alluxio-master:19998/taxi');
   ```
   load csv into alluxio cache<br>
   `select count(*) from tlc_green_trips_201402;`
   
   quit trino console <br>
   `exit;`
   

4. Mount s3 bucket/prefix in alluxio  `/s3storage` path <br>`docker exec  alluxio-master  alluxio fs mount --option 
   aws.accessKeyId=AKIA5EG3WLYZDRVWTWMQ --option aws.secretKey=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 --shared /s3storage s3://dchomaalluxio/test`
5. Create & populate iceberg table over s3 via trino & alluxio <br> `docker exec -it trino trino --catalog iceberg`<br>
   ``` 
   CREATE SCHEMA test_iceberg;
   USE test_iceberg;
   CREATE TABLE tlc_green_trips_201402 
   WITH (location='alluxio://alluxio-master:19998/s3storage/trips', format = 'parquet')
   AS
   SELECT 
       cast(vendorid as INTEGER) as vendorid,
       tpep_pickup_datetime,
       tpep_dropoff_datetime,
       cast(store_and_fwd_flag as VARCHAR) as store_and_fwd_flag,
       cast(rate_code as INTEGER) as rate_code,
       cast(pickup_longitude as DECIMAL(9,6)) as pickup_longitude,
       cast(pickup_latitude as DECIMAL(9,6)) as pickup_latitude,
       cast(dropoff_longitude as DECIMAL(9,6)) as dropoff_longitude,
       cast(dropoff_latitude as DECIMAL(9,6)) as dropoff_latitude,
       cast(passenger_count as INTEGER) as passenger_count,
       cast(trip_distance as DECIMAL(8, 2)) as trip_distance,
       cast(fare_amount as DECIMAL(8, 2)) as fare_amount,
       cast(extra as DECIMAL(8, 2)) as extra,
       cast(mta_tax as DECIMAL(8, 2)) as improvmta_taxement_surcharge,
       cast(tip_amount as DECIMAL(8, 2)) as tip_amount,
       cast(ehail_fee as VARCHAR) as ehail_fee,
       cast(total_amount as DECIMAL(8, 2)) as total_amount,
       cast(payment_type as INTEGER) as payment_type,
       cast(trip_type as VARCHAR) as trip_type
   FROM hive.test_hive.tlc_green_trips_201402;
6. After a while iceberg data files should be synced with external s3 storage mount 

7. Query the data
   ```
    trino:iceberg_test> select max(total_amount), max(tip_amount) from tlc_green_trips_201402 where trip_distance>5;
     _col0  | _col1  
    --------+--------
     883.50 | 110.03 
    (1 row)
    
    Query 20220414_072800_00027_h2yu4, FINISHED, 1 node
    Splits: 18 total, 18 done (100.00%)
    0.78 [1.01M rows, 4.32MB] [1.29M rows/s, 5.53MB/s]

    ```
   
