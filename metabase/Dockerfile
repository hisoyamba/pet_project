FROM eclipse-temurin:21-jdk
ENV MB_PLUGINS_DIR=/home/plugins/
COPY metabase/metabase.jar /home/
ADD https://github.com/MotherDuck-Open-Source/metabase_duckdb_driver/releases/download/0.2.12-b/duckdb.metabase-driver.jar /home/plugins/
RUN chmod 744 /home/plugins/duckdb.metabase-driver.jar
CMD ["java", "-jar", "/home/metabase.jar"]
