# Web-Server-Log-Analysis

## Project Overview
This activity analyzes web server logs using Apache Hive. It sets up a Hive environment, loads log data, and executes various queries to extract meaningful insights. Partitioning is used to optimize query performance, and the results are exported for further analysis.

## Implementation Approach
1. **Setup Hive Environment** - Create a Hive database and external table.
2. **Load Data** - Upload web server logs (CSV) to Hive.
3. **Execute Queries** - Perform analysis using HiveQL.
4. **Implement Partitioning** - Optimize query performance.
5. **Generate Output** - Export query results.

## Setup and Execution

### 1. **Start the Hadoop Cluster and Hive Server**
```bash
docker compose up -d
```

### 2. **Access the Hive Server Container**
```bash
docker exec -it hive-server /bin/bash
```
```bash
hive
```


### 3. **Create Hive Database and Table**
```sql
CREATE DATABASE web_logs_db;
USE web_logs_db;
```
Now, create an external table to store the logs:
```sql
CREATE EXTERNAL TABLE IF NOT EXISTS web_logs (
    ip STRING,
    `timestamp` STRING,
    url STRING,
    status INT,
    user_agent STRING
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE
LOCATION '/user/hive/web_logs/';
```

### 4. **Load Data into Hive Table**
Upload the CSV file to HDFS:
```bash
hdfs dfs -mkdir -p /user/hive/web_logs
hdfs dfs -put web_server_logs.csv /user/hive/web_logs/
```
Load data into Hive:
```sql
LOAD DATA INPATH '/user/hive/web_logs/web_server_logs.csv' INTO TABLE web_logs;
```

### 5. **Run Hive Queries for Analysis**
#### a) Count Total Web Requests
```sql
SELECT COUNT(*) AS total_requests FROM web_logs;
```
- This query retrieves the total number of requests made to the web server by counting all entries in the web_logs table.

#### b) Analyze Status Codes
```sql
SELECT status, COUNT(*) AS count FROM web_logs GROUP BY status;
```
- This query groups web requests by HTTP status codes and counts the number of occurrences of each status code, helping to understand server responses.

#### c) Identify Most Visited Pages
```sql
SELECT url, COUNT(*) AS visits 
FROM web_logs 
GROUP BY url 
ORDER BY visits DESC 
LIMIT 3;
```
- This query finds the most visited pages by counting requests for each url. It sorts the results in descending order and returns the top 3 most accessed pages.

#### d) Traffic Source Analysis
```sql
SELECT user_agent, COUNT(*) AS count 
FROM web_logs 
GROUP BY user_agent 
ORDER BY count DESC 
LIMIT 3;
```
- This query counts the number of requests from different user agents (browsers/devices) and identifies the top 3 sources of traffic.

#### e) Detect Suspicious Activity
```sql
SELECT ip, COUNT(*) AS failed_requests
FROM web_logs
WHERE status IN (404, 500)
GROUP BY ip
HAVING failed_requests > 3;
```
- This query detects potential malicious activity by identifying IPs that have encountered multiple failed requests (HTTP 404 or 500 errors). It filters IPs with more than 3 failed requests.

#### f) Analyze Traffic Trends
```sql
SELECT substr(`timestamp`, 0, 16) AS minute, COUNT(*) AS request_count
FROM web_logs
GROUP BY substr(`timestamp`, 0, 16)
ORDER BY minute;
```
- This query groups web requests by minute-level timestamps to analyze traffic patterns over time.

### 6. **Implement Partitioning for Performance Optimization**
Create a partitioned table:
```sql
CREATE TABLE web_logs_partitioned (
    ip STRING,
    `timestamp` STRING,
    url STRING,
    user_agent STRING
)
PARTITIONED BY (status INT)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE;
```
Load data into partitions:
```sql
INSERT OVERWRITE TABLE web_logs_partitioned PARTITION (status)
SELECT ip, timestamp, url, user_agent, status FROM web_logs;
```

### 7. **Export Results**
Save query results into a file:
```sql
INSERT OVERWRITE DIRECTORY '/user/hive/output' ROW FORMAT DELIMITED FIELDS TERMINATED BY ',' 
SELECT * FROM web_logs_partitioned;
```
Do this similarly for the other result sets.
Fetch results:
```bash
hdfs dfs -cat /user/hive/output/*
```

### 8. **Copy Output from HDFS to Local Machine**
First, copy the output folder from HDFS to the local file system inside the Hive container:
```bash
hdfs dfs -get /user/hive/output /tmp/output
```
Since you're inside the hive-server container, your Codespace workspace is outside the container. To transfer files, first, exit the container:
```bash
exit
```
Now, check the Codespaces workspace path:
```bash
pwd
```
Copy Files from Container to Codespace:
```bash
docker cp hive-server:/tmp/output /workspaces/webserver-log-analysis-hive-Scheruk1701/
```

Commit changes to GitHub Repo.

## Challenges Faced & Solutions

### 1. Path Errors in Hive
- **Issue:** Error while loading CSV due to wrong path format (`java.net.URISyntaxException`).
- **Solution:** Ensure the file path starts with `file:///` when using local files or use `hdfs:///` for HDFS paths.

### 2. Dynamic Partitioning Strict Mode Error
- **Issue:** `FAILED: SemanticException [Error 10096]: Dynamic partition strict mode requires at least one static partition column.`
- **Solution:** Disabled strict mode using:

```sql
SET hive.exec.dynamic.partition.mode = nonstrict;
```
### 3. Output Not Visible in Codespaces
- **Issue:** Could see output in HDFS but not in Codespaces.
- **Solution:** Used the following commands to bring results into Codespaces:
```sql
hdfs dfs -get /user/hive/output /opt/hive-output/
docker cp hive-server:/opt/hive-output/ .
```

## Sample Input and Output
### Sample Input
```
192.168.1.1	2/1/2024 10:35	/about	200	Opera/74.0
192.168.1.25	2/1/2024 10:35	/products	200	Chrome/90.0
192.168.1.20	2/1/2024 10:43	/contact	200	Opera/74.0
192.168.1.3	2/1/2024 10:26	/checkout	500	Chrome/90.0
```

### Expected Output 
```
Chrome/90.0  23
Edge/88.0    23
Opera/74.0   21
```

## Conclusion
This activity successfully loads and analyzes web server logs using Hive, implementing partitioning for performance improvement and exporting results for further analysis. The provided commands allow easy execution and reproduction of the workflow.
