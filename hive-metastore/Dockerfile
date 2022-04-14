FROM alluxio/alluxio as alx

FROM bde2020/hive:2.3.2-postgresql-metastore
COPY --from=alx /opt/alluxio/client/alluxio-*-client.jar /opt/hive/lib
