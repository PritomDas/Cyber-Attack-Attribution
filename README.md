![Language](https://img.shields.io/badge/language-python--3.8.2-blue) [![LinkedIn][linkedin-shield]][linkedin-url]

<!-- PROJECT LOGO -->
<br />

<p align="center">
  <img src="./images/1.PNG" alt="Logo" width="100" height="100">
</p>

<p align="center">
  <img src="./images/2.jpg" alt="Logo" width="200" height="150">
</p>
> Cyber Attack Attribution, Machine Learning, Random Forest Classifier, Malware Artifacts, Advanced Persistent Threats



<!-- ABOUT THE PROJECT -->

## About The Project

A startup called Sparkify wants to analyze the data they've been collecting on songs and user activity on their new music streaming application. Sparkify has decided that it is time to introduce more automation and monitoring to their data warehouse ETL pipelines and have come to the conclusion that the best tool to achieve this is Apache Airflow.

They'd like a data engineer to create high grade data pipelines that are dynamic and built from reusable tasks, can be monitored, and allow easy backfills. They have also noted that the data quality plays a big part when analyses are executed on top the data warehouse and want to run data quality tests against their datasets after the ETL steps have been executed to catch any discrepancies in the datasets.

The source data resides in S3 and needs to be processed in Sparkify's data warehouse in Amazon Redshift. The source datasets consist of JSON logs that tell about user activity in the application and JSON metadata about the songs the users listen to.

### Project Description

In this project, we will build data pipelines using Apache Airflow using custom defined operators to perform tasks such as staging the data, filling the data warehouse, and running checks on the data as the final step.

### Built With

* python
* AWS
* Apache Airflow

### Dataset

#### Song Dataset

Songs dataset is a subset of [Million Song Dataset](http://millionsongdataset.com/). Each file in the dataset is in JSON format and contains meta-data about a song and the artist of that song. The dataset is hosted at S3 bucket `s3://udacity-dend/song_data`.

Sample Record :

```
{"num_songs": 1, "artist_id": "ARJIE2Y1187B994AB7", "artist_latitude": null, "artist_longitude": null, "artist_location": "", "artist_name": "Line Renaud", "song_id": "SOUPIRU12A6D4FA1E1", "title": "Der Kleine Dompfaff", "duration": 152.92036, "year": 0}
```

#### Log Dataset

Logs dataset is generated by [Event Simulator](https://github.com/Interana/eventsim). These log files in JSON format simulate activity logs from a music streaming application based on specified configurations. The dataset is hosted at S3 bucket `s3://udacity-dend/log_data`.

Sample Record :

```
{"artist": null, "auth": "Logged In", "firstName": "Walter", "gender": "M", "itemInSession": 0, "lastName": "Frye", "length": null, "level": "free", "location": "San Francisco-Oakland-Hayward, CA", "method": "GET","page": "Home", "registration": 1540919166796.0, "sessionId": 38, "song": null, "status": 200, "ts": 1541105830796, "userAgent": "\"Mozilla\/5.0 (Macintosh; Intel Mac OS X 10_9_4) AppleWebKit\/537.36 (KHTML, like Gecko) Chrome\/36.0.1985.143 Safari\/537.36\"", "userId": "39"}
```



## Database Schema Design

### Data Model ERD

The Star Database Schema (Fact and Dimension Schema) is used for data modeling in this ETL pipeline. There is one fact table containing all the metrics (facts) associated to each event (user actions), and four dimensions tables, containing associated information such as user name, artist name, song meta-data etc. This model enables to search the database schema with the minimum number of *SQL JOIN*s possible and enable fast read queries. 

The data stored on S3 buckets is staged and then inserted to fact and dimensional tables on Redshift. The whole process in orchestrated using Airflow which is set to execute periodically once every hour.

![database](./images/database.png)

## Apache Airflow Orchestration 

### DAG Structure

The DAG parameters are set according to the following :

- The DAG does not have dependencies on past runs
- DAG has schedule interval set to hourly
- On failure, the task are retried 3 times
- Retries happen every 5 minutes
- Catchup is turned off
- Email are not sent on retry

The DAG dependency graph is given below.

![dag](./images/dag.png)

### Operators

Operators create necessary tables, stage the data, transform the data, and run checks on data quality. Connections and Hooks are configured using Airflow's built-in functionalities. All of the operators and task run SQL statements against the Redshift database. 

#### Stage Operator
The stage operator loads any JSON formatted files from S3 to Amazon Redshift. The operator creates and runs a SQL COPY statement based on the parameters provided. The operator's parameters specify where in S3, the file is loaded and what is the target table.

- **Task to stage JSON data is included in the DAG and uses the RedshiftStage operator**: There is a task that to stages data from S3 to Redshift (Runs a Redshift copy statement).

- **Task uses params**: Instead of running a static SQL statement to stage the data, the task uses parameters to generate the copy statement dynamically. It also contains a templated field that allows it to load timestamped files from S3 based on the execution time and run backfills.

- **Logging used**: The operator contains logging in different steps of the execution.

- **The database connection is created by using a hook and a connection**: The SQL statements are executed by using a Airflow hook.

#### Fact and Dimension Operators
The dimension and fact operators make use of the SQL helper class to run data transformations. Operators take as input the SQL statement from the helper class and target the database on which to run the query against. A target table is also defined that contains the results of the transformation.

Dimension loads are done with the truncate-insert pattern where the target table is emptied before the load. There is a parameter that allows switching between insert modes when loading dimensions. Fact tables are massive so they only allow append type functionality.

- **Set of tasks using the dimension load operator is in the DAG**: Dimensions are loaded with on the LoadDimension operator.

- **A task using the fact load operator is in the DAG**: Facts are loaded with on the LoadFact operator.

- **Both operators use params**: Instead of running a static SQL statement to stage the data, the task uses parameters to generate the copy statement dynamically.

- **The dimension task contains a param to allow switch between append and insert-delete functionality**: The DAG allows to switch between append-only and delete-load functionality.

#### Data Quality Operator
The data quality operator is used to run checks on the data itself. The operator's main functionality is to receive one or more SQL based test cases along with the expected results and execute the tests. For each the test, the test result and expected result are checked and if there is no match, the operator raises an exception and the task is retried and fails eventually.

For example one test could be a SQL statement that checks if certain column contains NULL values by counting all the rows that have NULL in the column. We do not want to have any NULLs so expected result would be 0 and the test would compare the SQL statement's outcome to the expected result.

- **A task using the data quality operator is in the DAG and at least one data quality check is done**: Data quality check is done with correct operator.

- **The operator raises an error if the check fails**: The DAG either fails or retries n times.

- **The operator is parametrized**: Operator uses params to get the tests and the results, tests are not hard coded to the operator.


### Airflow UI views of DAG and plugins

The DAG follows the data flow provided in the instructions, all the tasks have a dependency and DAG begins with a start_execution task and ends with a end_execution task.

## Project structure

Files in this repository:

|   File / Folder   |                         Description                          |
| :---------------: | :----------------------------------------------------------: |
|       dags        | Folder at the root of the project, where DAGs and SubDAGS are stored |
|      images       |  Folder at the root of the project, where images are stored  |
|  plugins/helpers  |        Contains a SQL helper class for easy querying         |
| plugins/operators |    Contains the custom operator to perform the DAG tasks     |
| create_tables.sql | Contains SQL commands to create the necessary tables on Redshift |
|      README       |                         Readme file                          |



<!-- GETTING STARTED -->

## Getting Started

Clone the repository into a local machine using

```sh
git clone https://github.com/PritomDas/Mastery-in-Data-Engineering/tree/master/Udacity%20Data%20Engineering%20Nano%20Degree
```

### Prerequisites

These are the prerequisites to run the program.

* python 3.8.2 ( as of my current installation )
* AWS account
* Apache Airflow

### How to run

Follow the steps to extract and load the data into the data model.

1. Set up Apache Airflow to run in local environment.
2. Navigate to `Projects` and then `5. Data Pipelines` folder.
3. Set up `AWS Connection` and `Redshift Connection` to Airflow using necessary values.
4. In Airflow, turn the DAG execution ON.
5. View the Web UI for detailed insights about the operation.


<!-- CONTACT -->

## Contact

Pritom Das Radheshyam - [Portfolio Website](https://pritom.uwu.ai/)



<!-- MARKDOWN LINKS & IMAGES -->
<!-- https://www.markdownguide.org/basic-syntax/#reference-style-links -->

[linkedin-shield]: https://img.shields.io/badge/-LinkedIn-black.svg?style=flat-square&logo=linkedin&colorB=555
[linkedin-url]: https://www.linkedin.com/in/you-found-pritom
[product-screenshot]: images/screenshot.jpg
