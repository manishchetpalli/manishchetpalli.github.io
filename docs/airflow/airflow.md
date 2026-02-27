### **What is Orchestration in BigData?**

Orchestration in big data refers to the automated configuration, coordination, and management of complex big data systems and services. Like an orchestra conductor ensures each section of the orchestra plays its part at the right time and the right way to create harmonious music, orchestration in big data ensures that each component of a big data system interacts in the correct manner at the right time to execute complex, multi-step processes efficiently and reliably.

> --- ***Workflow management***

It defines, schedules, and manages workflows involving multiple tasks across disparate systems. These workflows can be simple linear sequences or complex directed acyclic graphs (DAGs) with branching and merging paths.

> --- ***Task scheduling***

Orchestration tools schedule tasks based on their dependencies. This ensures that tasks are executed in the correct order and that tasks that can run in parallel do so, increasing overall system efficiency.

> --- ***Failure handling***

Orchestration tools handle failures in the system, either by retrying failed tasks, skipping them, or alerting operators to the failure.

**------------------------------------------------------------------------------------------------------------**

### **Need of Workflow management while Designing Data Pipelines**


> --- ***Ordering and Scheduling***

Data processing tasks often have dependencies, meaning one task needs to complete before another can begin. For example, a task that aggregates data may need to wait until the data has been extracted and cleaned. A workflow management system can keep track of these dependencies and ensure tasks are executed in the right order.

> --- ***Parallelization*** 

When tasks don't have dependencies, they can often run in parallel. This can significantly speed up data processing. Workflow management systems can manage parallel execution, maximizing the use of computational resources and reducing overall processing time.

> --- ***Error Handling***

If a task in a data pipeline fails, it can have a knock-on effect on other tasks. Workflow management systems can handle these situations, for instance by retrying failed tasks, skipping them, or stopping the pipeline and alerting operators.

> --- ***Visibility and Monitoring***

Workflow management systems often provide tools for monitoring the progress of data pipelines and visualizing their structure. This can make it easier to spot bottlenecks or failures, and to understand the flow of data through the pipeline.

> --- ***Resource Management***

Workflow management systems can allocate resources (like memory, CPU, etc.) depending on the requirements of different tasks. This helps in efficiently utilizing resources and ensures optimal performance of tasks.

**------------------------------------------------------------------------------------------------------------**


### **What is AirFlow?**
Apache Airflow is an open-source platform that programmatically allows you to author, schedule, and monitor workflows. It was originally developed by Airbnb in 2014 and later became a part of the Apache Software Foundation's project catalog.

Airflow uses directed acyclic graphs (DAGs) to manage workflow orchestration. DAGs are a set of tasks with directional dependencies, where the tasks are the nodes in the graph, and the dependencies are the edges. In other words, each task in the workflow executes based on the completion status of its predecessors.

> --- ***Key features of Apache Airflow include***

1. Dynamic Pipeline Creation - Airflow allows you to create dynamic pipelines using Python. This provides flexibility and can be adapted to complex dependencies and operations.
2. Easy Scheduling - Apache Airflow includes a scheduler to execute tasks at defined intervals. The scheduling syntax is quite flexible, allowing for complex scheduling.
3. Robust Monitoring and Logging - Airflow provides detailed status and logging information about each task, facilitating debugging and monitoring. It also offers a user-friendly UI to monitor and manage the workflow.
4. Scalability - Airflow can distribute tasks across a cluster of workers, meaning it can scale to handle large workloads.
5. Extensibility - Airflow supports custom plugins and can integrate with several big data technologies. You can define your own operators and executors, extend the library, and even use the user interface API.
6. Failure Management - In case of a task failure, Airflow sends alerts and allows for retries and catchup of past runs in a robust way

**------------------------------------------------------------------------------------------------------------**

### ***Airflow Architecture***

![Steps](airflowarc.svg)

> --- ***Scheduler***

The scheduler is a critical component of Airflow. Its primary function is to continuously scan the DAGs (Directed Acyclic Graphs) directory to identify and schedule tasks based on their dependencies and specified time intervals. The scheduler is responsible for determining which tasks to execute and when. It interacts with the metadata database to store and retrieve task state and execution information.

> --- ***Metadata Database***

Airflow leverages a metadata database, such as PostgreSQL or MySQL, to store all the configuration details, task states, and execution metadata. The metadata database provides persistence and ensures that Airflow can recover from failures and resume tasks from their last known state. It also serves as a central repository for managing and monitoring task execution.

> --- ***Web Server***

The web server component provides a user interface for interacting with Airflow. It enables users to monitor task execution, view the status of DAGs, and access logs and other operational information. The web server communicates with the metadata database to fetch relevant information and presents it in a user-friendly manner. Users can trigger manual task runs, monitor task progress, and configure Airflow settings through the web server interface.

> --- ***Executors***

Airflow supports different executor types to execute tasks. The executor is responsible for allocating resources and running tasks on the specified worker nodes.

!!! example

    the local executor executes tasks on the same node as the scheduler, while the Kubernetes executor executes tasks in containers.

> --- ***Worker Nodes***

Worker nodes are responsible for executing the tasks assigned to them by the executor. They retrieve the task details, dependencies, and code from the metadata database and execute the tasks accordingly. The number of worker nodes can be scaled up or down based on the workload and resource requirements.

> --- ***Message Queue***

Airflow relies on a message queue system, such as RabbitMQ, Apache Kafka, or Redis, to enable communication between the scheduler and the worker nodes. The scheduler places task execution requests in the message queue, and the worker nodes pick up these requests, execute the tasks, and update their status back to the metadata database. The message queue acts as a communication channel, ensuring reliable task distribution and coordination.

> --- ***The DAG Processor***

It processes DAG files and stores the metadata in the database. It scans the DAG folder every 30 seconds (default) to look for new or updated files. It uses DagFileProcessorManager to detect changes and DagFileProcessorProcess to parse the actual code. To speed up parsing, import statements and variables should be placed inside methods rather than at the top of the script.In newer versions, the DAG Processor is separated from the Scheduler to allow independent scaling and prevent CPU bottlenecks

> --- ***Triggerer***

Introduced to handle long-running tasks. Normally, long tasks block the Scheduler's task slots.
The Triggerer allows a task to enter a "Deferred" state. It uses a small asynchronous Python code to "watch" the task while the Worker does the heavy lifting, freeing up the Scheduler's pool for other tasks

---

> --- ***Airflow 3.0: The New Client-Server Architecture***

The most significant architectural shift in Airflow 3.0 is the removal of direct database access for several components.

1. Server Side - Only the Scheduler and the API Server can communicate directly with the Database.
2. Client Side - The DAG Processor, Worker, and Triggerer are now "clients." They cannot talk to the DB directly.
3. Communication Flow-
    1. The DAG Processor parses a DAG and sends the info to an Internal API Server.
    2. The API Server writes that info into the DB.
    3. The Scheduler reads from the DB and triggers the task.
    4. If a task is deferred, the Triggerer updates the status via the API Server to the DB.

> --- ***Worker Execution Flow***

1. Task Submission - The Scheduler identifies a task that needs to run and hands it to the Executor. The Executor then submits a message containing the execution command to a Broker/Queue (typically Redis).
2. Message Pulling - The Worker Process constantly monitors the Redis queue. When a message appears, the Worker pulls the message to begin the job.
3. Internal Hierarchy - Once a message is pulled, the execution follows an internal hierarchy within the worker machine to isolate the task.
    
    1. Worker Process: The main process that communicates with the queue.
    2. Worker Child Process: A sub-process spawned for a specific task.
    3. Local Task Process: An internal process dedicated to that specific instance.
    4. Task Runner: The final layer that executes the actual code (e.g., echo copying file).

4. Result Reporting - After the Task Runner completes the work, it writes the task status (Success/Failure) to the Result Backend.
5. Database Update - The Scheduler reads the status from the Result Backend and updates the primary Postgres Metadata Database so the status can be reflected in the Airflow UI

**------------------------------------------------------------------------------------------------------------**

### **DAGs and Tasks**

DAGs are at the core of Airflow’s architecture. A DAG is a directed graph consisting of interconnected tasks. Each task represents a unit of work within the data pipeline. Tasks can have dependencies on other tasks, defining the order in which they should be executed. Airflow uses the DAG structure to determine task dependencies, schedule task execution, and track their progress.

Each task within a DAG is associated with an operator, which defines the type of work to be performed. Airflow provides a rich set of built-in operators for common tasks like file operations, data processing, and database interactions. Additionally, custom operators can be created to cater to specific requirements.
Tasks within a DAG can be triggered based on various events, such as time-based schedules, the completion of other tasks, or the availability of specific data.

In Airflow, there's a task which perform 1 unique job across the DAG, the task is usually calling Airflow Operator.

An operator is essentially a Python class that defines what a task should do and how it should be executed within the context of the DAG.

- Operators can perform a wide range of tasks, such as running a Bash script or Python script, executing SQL, sending an email, or transferring files between systems.
- Operators provide a high-level abstraction of the tasks, letting us focus on the logic and flow of the workflow without getting bogged down in the implementation details.
- Apache Airflow has several types of operators that allow you to perform different types of tasks. 

---

> --- **Why use Airflow/DAGs instead of Simple Scripts**

A common question is why one shouldn't just use a Python for loop or sequential method calls. 

1. Parallel Task Execution - If you need to fetch data from 50 different APIs, a sequential script might take 4 minutes (5 seconds each). Airflow can run these tasks in parallel, reducing the total time to just 5 seconds.
2. Granular Re-runs (Error Handling) - In a sequential script, if the "Clean" step fails after a 2-hour "Fetch" step, you might have to restart everything. In an Airflow DAG, you can re-trigger only the failed task (the "Clean" step) without re-running the successful "Fetch" step, saving time and resources

---

> --- ***Create a DAG***

Different Styles of Writing DAGs

1. Classic Way: Defining the DAG object and passing it to tasks (shown above).
2. Context Manager (with): Automatically associates tasks with the DAG without passing dag=dag to every operator.
3. TaskFlow API: Uses decorators like @dag and @task

A DAG file starts with a `dag` object. We can create a `dag` object using a context manager or a decorator.

```python
from airflow.decorators import dag
from airflow import DAG
import pendulum
from airflow.providers.standard.operators.bash import BashOperator 
from airflow.providers.standard.operators.python import PythonOperator

# dag1 - using @dag decorator
@dag(
    schedule="30 4 * * *",
    start_date=pendulum.datetime(2023, 1, 1, tz="UTC"),
    catchup=False,
    tags=["random"]
)
def random_dag1():
    pass

random_dag1()

# dag2 - using context manager
with DAG(
    dag_id="random_dag2",
    schedule="@daily",
    start_date=pendulum.datetime(2023, 1, 1, tz="UTC"),
    catchup=False,
    tags=["random"]
) as dag2:
    pass

# 1. Define a function for the PythonOperator
def print_context(**kwargs): [29]
    print(kwargs)
    print("Job completed")

# 2. Define the tasks
copy_file = BashOperator(
    task_id="copy_file",
    bash_command="echo copying file",
    dag=dag
)

task_2 = PythonOperator(
    task_id="task_2",
    python_callable=print_context,
    dag=dag
)

# 3. Set dependencies
# Using the right-shift operator
copy_file >> task_2  
# Alternatively: copy_file.set_downstream(task_2) 
```

Either way, we need to define a few parameters to control how a DAG is supposed to run.

Some of the most-used parameters are:

1. start_date - If it's a future date, it's the timestamp when the scheduler starts to run. If it's a past date, it's the timestamp from which the scheduler will attempt to backfill.

2. catch_up - Whether to perform scheduler catch-up. If set to true, the scheduler will backfill runs from the start date.

3. schedule - Scheduling rules. Currently, it accepts a cron string, time delta object, timetable, or list of dataset objects.

4. tags - List of tags helping us search DAGs in the UI.

---

## Scheduling in airflow

> --- **Core Scheduling Parameters**

To schedule a pipeline, you must define specific parameters within the `DAG` object.

`dag_id`: A unique identifier for the DAG.

`start_date`: The date from which the DAG starts its schedule. It requires a `datetime` object.

`schedule`: (Previously `schedule_interval` in Airflow 2.0) Defines how often the DAG runs.

`catchup`: A boolean (default is often `True` but recommended as `False` in many cases) that determines if Airflow should run all "missed" instances of the DAG from the `start_date` to the current time.

> --- **Scheduling Methods**

-  Macros (Shortcuts)
These are easy-to-read strings for common intervals.

`@daily`: Runs once a day at midnight (00:00).

`@hourly`: Runs at the start of every hour.

`@weekly`: Runs at midnight on Sunday.

`@monthly`: Runs at midnight on the first day of the month.

- Cron Expressions
This is the most flexible and widely used method. It uses five fields: `minute`, `hour`, `day`, `month`, and `weekday`.

- Timedelta
Used when you need exact intervals that Cron cannot easily handle, such as running a pipeline every 10 days regardless of month transitions.

Airflow defaults to UTC. If you are working in a specific region (like India or the US), it is recommended to create timezone-aware DAGs using the `pendulum` library.

Why Pendulum? It handles Daylight Saving Time (DST) adjustments automatically, which standard `datetime` might miss.

Real-world issue: If you run a pipeline at 5:00 AM India time but it defaults to UTC, it might technically be running on the previous calendar day in UTC, causing errors in incremental data loading.


---

## Task level Operator

!!! tip

    While Airflow provides a wide variety of operators out of the box, we may still need to create custom operators to address our specific use cases.

    All operators are extended from `BaseOperator` and we need to override two methods: `__init__` and `execute`.

    The `execute` method is invoked when the runner calls the operator.

The following example creates a custom operator `DataAnalysisOperator` that performs the requested type of analysis on an input file and saves the results to an output file.

!!! warning

    It's advised not to put expensive operations in `__init__` because it will be instantiated once per scheduler cycle.

```python
from airflow.models import BaseOperator
from airflow.utils.decorators import apply_defaults
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt

class DataAnalysisOperator(BaseOperator):
    @apply_defaults
    def __init__(self, dataset_path, output_path, analysis_type, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.dataset_path = dataset_path
        self.output_path = output_path
        self.analysis_type = analysis_type

    def execute(self, context):
        # Load the input dataset into a pandas DataFrame.
        data = pd.read_csv(self.dataset_path)

        # Perform the requested analysis.
        if self.analysis_type == 'mean':
            result = np.mean(data)
        elif self.analysis_type == 'std':
            result = np.std(data)
        else:
            raise ValueError(f"Invalid analysis type '{self.analysis_type}'")

        # Write the result to a file.
        with open(self.output_path, 'w') as f:
            f.write(str(result))
```

The extensibility of the operator is one of many reasons why Airflow is so powerful and popular.


!!! Tip "Tip"

        BashOperator: Executes a bash command.
        PythonOperator: Calls a Python function.
        EmailOperator: Sends an email.
        SimpleHttpOperator: Sends an HTTP request.
        MySqlOperator, SqliteOperator, PostgresOperator, MsSqlOperator, OracleOperator, etc.: Executes a SQL command.
        DummyOperator: A placeholder operator that does nothing.
        Sensor: Waits for a certain time, file, database row, S3 key, etc. There are many types of sensors, like HttpSensor, SqlSensor, S3KeySensor, TimeDeltaSensor, ExternalTaskSensor, etc.
        SSHOperator: Executes commands on a remote server using SSH.
        DockerOperator: Runs a Docker container.
        SparkSubmitOperator: Submits a Spark job.
        Operators in Airflow 
        S3FileTransformOperator: Copies data from a source S3 location to a temporary location on the local filesystem, transforms the data, and then uploads it to a destination S3 location.
        S3ToRedshiftTransfer: Transfers data from S3 to Redshift.
        EmrAddStepsOperator: Adds steps to an existing EMR (Elastic Map Reduce) job flow.
        EmrCreateJobFlowOperator: Creates an EMR JobFlow, i.e., a cluster.
        AthenaOperator: Executes a query on AWS Athena.
        AwsGlueJobOperator: Runs an AWS Glue Job.
        S3DeleteObjectsOperator: Deletes objects from an S3 bucket.
        BigQueryOperator: Executes a BigQuery SQL query.
        BigQueryToBigQueryOperator: Copies data from one BigQuery table to another.
        DataprocClusterCreateOperator: This operator is used to create a new cluster of machines on GCP's Dataproc service.
        DataProcPySparkOperator: This operator is used to submit PySpark jobs to a running Dataproc cluster.
        DataProcSparkOperator: This operator is used to submit Spark jobs written in Scala or Java to a running Dataproc cluster.
        DataprocClusterDeleteOperator: This operator is used to delete an existing cluster.

## Sensor Operator

A special type of operator is called a sensor operator.
It's designed to wait until something happens and then succeed so their downstream tasks can run.
The DAG has a few sensors that are dependent on external files, time, etc.

Common sensor types are:

- **TimeSensor**: Wait for a certain amount of time to pass before executing a task.

- **FileSensor**: Wait for a file to be in a location before executing a task.

- **HttpSensor**: Wait for a web server to become available or return an expected result before executing a task.

- **ExternalTaskSensor**: Wait for an external task in another DAG to complete before executing a task.

We can create a custom sensor operator by extending the `BaseSensorOperator` class and overriding two methods: `__init__` and `poke`.

The `poke` method performs the necessary checks to determine whether the condition has been satisfied.
If so, return `True` to indicate that the sensor has succeeded. Otherwise, return `False` to continue checking in the next interval.

Here is an example of a custom sensor operator that pulls an endpoint until the response matches with `expected_text`.

```python
from airflow.sensors.base_sensor_operator import BaseSensorOperator
from airflow.utils.decorators import apply_defaults
import requests

class EndpointSensorOperator(BaseSensorOperator):

    @apply_defaults
    def __init__(self, url, expected_text, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.url = url
        self.expected_text = expected_text

    def poke(self, context):
        response = requests.get(self.url)
        if response.status_code != 200:
            return False
        return self.expected_text in response.text
```

#### Poke mode vs. Reschedule mode

There are two types of modes in a sensor:

- **poke**: the sensor repeatedly calls its `poke` method at the specified `poke_interval`, checks the condition, and reports it back to Airflow.
  As a consequence, the sensor takes up the worker slot for the entire execution time.

- **reschedule**: Airflow is responsible for scheduling the sensor to run at the specified `poke_interval`.
  But if the condition is not met yet, the sensor will release the worker slot to other tasks between two runs.

!!! tip

    In general, poke mode is more appropriate for sensors that require short run-time and `poke_interval` is less than five minutes.

    Reschedule mode is better for sensors that expect long run-time (e.g., waiting for data to be delivered by an external party) because it is less resource-intensive and frees up workers for other tasks.

### Backfilling

**Backfilling** is an important concept in data processing.
It refers to the process of populating or updating historical data in the system to ensure that the data is complete and up-to-date.

This is typically required in two use cases:

- **Implement a new data pipeline**: If the pipeline uses an incremental load, backfilling is needed to populate historical data that falls outside the reloading window.

- **Modify an existing data pipeline**: When fixing a SQL bug or adding a new column, we also want to backfill the table to update the historical data.

!!! warning

    When backfilling the table, we must ensure that the new changes are compatible with the existing data; otherwise, the table needs to be recreated from scratch.

    Sometimes, the backfilling job can consume significant resources due to the high volume of historical data.

    It's also worth checking any possible downstream failure before executing the backfilling job.

Airflow provides the backfilling process in its cli command.

```bash
airflow backfill [dag name] -s [start date] -e [end date]
```


To create DAGs, we just need basic knowledge of Python.
However, to create efficient and scalable DAGs, it's essential to master Airflow's specific features and nuances.



## **Create a task object**

A DAG object is composed of a series of dependent tasks. A task can be an operator, a sensor, or a custom Python function decorated with `@task`.
Then, we will use `>>` or the opposite way, which denotes the **dependencies** between tasks.

```python
# dag3
@dag(
    schedule="30 4 * * *",
    start_date=pendulum.datetime(2023, 1, 1, tz="UTC"),
    catchup=False,
    tags=["random"]
)
def random_dag3():

    s1 = TimeDeltaSensor(task_id="time_sensor", delta=timedelta(seconds=2))
    o1 = BashOperator(task_id="bash_operator", bash_command="echo run a bash script")
    @task
    def python_operator() -> None:
        logging.info("run a python function")
    o2 = python_operator()

    s1 >> o1 >> o2

random_dag3()
```

A task has a well-defined life cycle, including 14 states, as shown in the following graph:

![task life cycle](task.svg)

## **Pass data between tasks**

One of the best design practices is to split a heavy task into smaller tasks for easy debugging and quick recovery.

!!! example

    we first make an API request and use the response as the input for the second API request.
    To do so, we need to pass a small amount of data between tasks.

**XComs (cross-communications)** is a method for passing data between Airflow tasks.
Data is defined as a key-value pair, and the value must be serializable.

- The `xcom_push()` method pushes data to the Airflow metadata database and is made available for other tasks.

- The `xcom_pull()` method retrieves data from the database using the key.

![example-xcom](xcom.svg)

Every time a task returns a value, the value is automatically pushed to XComs.

We can find them in the Airflow UI `Admin` -> `XComs`. If the task is created using `@task`, we can retrieve XComs by using the object created from the upstream task.

In the following example, the operator `o2` uses the traditional syntax, and the operator `o3` uses the simple syntax:

```python
@dag(
    schedule="30 4 * * *",
    start_date=pendulum.datetime(2023, 1, 1, tz="UTC"),
    catchup=False,
    tags=["random"]
)
def random_dag4():

    @task
    def python_operator() -> None:
        logging.info("run a python function")
        return str(datetime.now()) # return value automatically stores in XCOMs
    o1 = python_operator()
    o2 = BashOperator(task_id="bash_operator1", bash_command='echo "{{ ti.xcom_pull(task_ids="python_operator") }}"') # traditional way to retrieve XCOM value
    o3 = BashOperator(task_id="bash_operator2", bash_command=f'echo {o1}') # make use of @task feature

    o1 >> o2 >> o3

random_dag4()
```

!!! warning

    Although nothing stops us from passing data between tasks, the general advice is to not pass heavy data objects, such as pandas DataFrame and SQL query results because doing so may impact task performance.

---

## **Taskflow API**

The Taskflow API is a higher-level abstraction in Airflow designed to simplify code, especially when working with Python Operators. Its primary goal is to reduce boilerplate code and handle data sharing (XCom) more intuitively.

Key Advantages:
   Simplifies XCom: You no longer need to explicitly write `xcom_push` or `xcom_pull`.
   Reduces Boilerplate: You don't have to define a `PythonOperator` for every task; you can turn standard Python functions into tasks using decorators.
   Cleaner Dependencies: Dependencies can be defined by simply calling functions.

> --- **Classic vs. Taskflow API: Code Comparison**

- The Classic Way (Manual XCom & Operators)

In the older "classic" style, you define a DAG object and use `PythonOperator` to wrap your functions. You must also manually handle the Task Instance (ti) to move data between tasks.

```python
# Conceptual Classic Logic from the Source
def extract(context):
    path = "/path/to/data.csv"
    context['ti'].xcom_push(key='output_path', value=path)

def transform(context):
    # Manually pulling data
    data = context['ti'].xcom_pull(task_ids='extract', key='output_path')
    print(f"Transforming: {data}")

# Defining the Operator
extract_task = PythonOperator(task_id='extract', python_callable=extract, dag=dag)
```

- The Taskflow Way (Decorators)

With Taskflow, you use the `@dag` and `@task` decorators. Returning a value from a function automatically performs an XCom push, and passing that value as an argument to another task function performs an XCom pull.

```python
from airflow.decorators import dag, task

@dag(dag_id='taskflow_example', start_date=datetime(2025, 1, 1))
def my_pipeline():

    @task # Automatically treats this as a Python task
    def extract():
        return {"output_path": "/path/to/data.csv"}

    @task
    def transform(extracted_data): # Argument acts as XCom pull
        path = extracted_data['output_path']
        print(f"Transforming data from: {path}")

    # Setting Dependencies via Function Calls
    data = extract()
    transform(data)

# Call the DAG function to register it
my_pipeline()
```

> --- **Advanced Taskflow Concepts**

- Handling Multiple Outputs
By default, if a task returns a dictionary, Airflow stores the entire dictionary under a single XCom key called `return_value`. However, if you want each dictionary key to be its own independent XCom entry, you can use the `multiple_outputs` parameter.

Logic: Setting `@task(multiple_outputs=True)` ensures that keys like `status` or `location` are stored as individual XCom keys rather than nested inside a single return value.

- Mixing and Matching
You are not limited to using only Taskflow. You can mix Taskflow tasks with classic operators (like `BashOperator` or a standard `PythonOperator`) within the same DAG.

Example: You might have an `@task` for extraction and a classic `PythonOperator` for loading data if you prefer that structure for certain tasks.

- Dependencies: Implicit vs. Explicit

Implicit: Created when you pass the output of one task function directly into another task function.

Explicit: You can still use the standard bitshift operators (`>>`) if you store the task calls in variables.

> --- **Internal Architecture (Deep Dive)**

While Taskflow looks like simple Python functions, it is internally mapped back to the core Airflow infrastructure.

Default Operator: When you use `@task`, Airflow defaults to the Python Operator. To use others, you can specify them (e.g., `@task.bash`).

Internal Classes: The decorators call the `TaskDecoratorCollection` class, which uses `PythonDecoratorOperator`.

Provider Manager: Airflow's `ProvidersManager` initializes these decorators, identifying which operator (Bash, Python, etc.) to use based on the decorator name.

End-to-End Flow: Even with Taskflow, the final execution still relies on the standard `PythonOperator` logic; it just automates the "handling of output" and "context parsing" on your behalf.

> --- **Limitations and Recommendations**

Implicit Complexity: Taskflow makes dependencies "implicit," which can sometimes lead to confusion in very complex DAGs where it isn't immediately obvious how tasks are chained.

Best Practice: The source suggests using Taskflow when dealing primarily with Python-based logic to keep the code clean. However, if your DAG involves many different types of operators, using the classic "low-level" operator style might offer better clarity.

---

## **Use Jinja templates**

!!! info

    Jinja is a templating language used by many Python libraries, such as Flask and Airflow, to generate dynamic content.

    It allows us to embed variables within the text and then have those variables replaced with actual values during runtime.

Airflow's Jinja templating engine provides built-in functions that we can use between double curly braces, and the expression will be evaluated at runtime.

Users can also create their own macros using user_defined_macros and the macro can be a variable as well as a function.

```python
def days_to_now(starting_date):
    return (datetime.now() - starting_date).days

@dag(
    schedule="30 4 * * *",
    start_date=pendulum.datetime(2023, 1, 1, tz="UTC"),
    catchup=False,
    tags=["random"],
    user_defined_macros={
        "starting_date": datetime(2015, 5, 1),
        "days_to_now": days_to_now,
    })
def random_dag5():

    o1 = BashOperator(task_id="bash_operator1", bash_command="echo Today is {{ execution_date.format('dddd') }}")
    o2 = BashOperator(task_id="bash_operator2", bash_command="echo Days since {{ starting_date }} is {{ days_to_now(starting_date) }}")

    o1 >> o2

random_dag5()
```

!!! note

    The full list of built-in variables, macros and filters in Airflow can be found in the [Airflow Documentation](https://airflow.apache.org/docs/apache-airflow/stable/templates-ref.html)

Another feature around templating is the `template_searchpath` parameter in the DAG definition.

It's a list of folders where Jinja will look for templates.

!!! example

    A common use case is invoking an SQL file in a database operator such as BigQueryInsertJobOperator. Instead of hardcoding the SQL query, we can refer to the SQL file, and the content will be automatically rendered during runtime.

```python
@dag(
    schedule="30 4 * * *",
    start_date=pendulum.datetime(2023, 1, 1, tz="UTC"),
    catchup=False,
    tags=["random"],
    template_searchpath=["/usercode/dags/sql"])
def random_dag6():

    BigQueryInsertJobOperator(
        task_id="insert_query_job",
        configuration={
            "query": {
                "query": "{% include 'sample.sql' %}",
                "useLegacySql": False,
            }
        }
    )

random_dag6()
```

## **Manage cross-DAG dependencies**

In principle, every DAG is an independent workflow.
However, sometimes, it's necessary to create **dependencies** between DAGs.

!!! example

    a DAG performs an ETL job that produces a table sales. The sales table is the source of two downstream DAGs, where one generates revenue reports, and the other one uses it to train a machine learning model.

There are several ways to implement cross-DAG dependencies in Airflow.

- `TriggerDagOperator` is an operator that triggers a downstream DAG from any point in the DAG. It's similar to a push mechanism where the producer decides when to notify the consumers.

```python
from airflow.operators.trigger_dagrun import TriggerDagRunOperator

@dag(
    schedule="30 4 * * *",
    start_date=pendulum.datetime(2023, 1, 1, tz="UTC"),
    catchup=False,
    tags=["random"]
)
def random_dag7():

    TriggerDagRunOperator(
        task_id="trigger_dagrun",
        trigger_dag_id="random_dag1",
        conf={},
    )

random_dag7()
```

- `ExternalTaskSensor` is a sensor operator for downstream DAGs to pull states of the upstream DAG, similar to a pull mechanism. The downstream DAG will wait until the task is completed in the upstream DAG.

```python
from airflow.sensors.external_task import ExternalTaskSensor

@dag(
    schedule="30 4 * * *",
    start_date=pendulum.datetime(2023, 1, 1, tz="UTC"),
    catchup=False,
    tags=["random"]
)
def random_dag8():

    ExternalTaskSensor(
        task_id="external_sensor",
        external_dag_id="random_dag3",
        external_task_id="python_operator",
        allowed_states=["success"],
        failed_states=["failed", "skipped"],
    )

random_dag8()
```

Another method introduced in `version 2.4` uses datasets to create data-driven dependencies between DAGs.

An Airflow dataset is a logical grouping of data updated by upstream tasks. The upstream task defines the output dataset via `outlets` parameter. The completion of the task means the successful update of the dataset.

In downstream DAGs, instead of using a time-based schedule, the DAG refers to the corresponding dataset produced by the upstreams.
Therefore, the downstream DAG will be triggered in a data-driven manner rather than a scheduled-based manner.

```python
dag1_dataset = Dataset("s3://dag1/output_1.txt", extra={"hi": "bye"})

@dag(
    schedule="30 4 * * *",
    start_date=pendulum.datetime(2023, 1, 1, tz="UTC"),
    catchup=False,
    tags=["random"]
)
def random_dag9_producer():
    BashOperator(outlets=[dag1_dataset], task_id="producer", bash_command="sleep 5")

random_dag9_producer()

@dag(
    schedule=[dag1_dataset],
    start_date=pendulum.datetime(2023, 1, 1, tz="UTC"),
    catchup=False,
    tags=["random"]
)
def random_dag9_consumer():
    BashOperator(task_id="consumer", bash_command="sleep 5")

random_dag9_consumer()
```

## **Best practices**

When working with Airflow, there are several best practices to keep in mind that help ensure our pipelines run smoothly and efficiently.

### **Idempotency**

**Idempotency** is a fundamental concept for data pipelines. In the context of Airflow, idempotency means running the same DAG Run multiple times has the same effect as running it only once.
When a DAG is designed to be idempotent, it can be executed repeatedly without causing unexpected changes to the pipeline's output.

This is especially necessary when a DAG Run might be rerun due to failures or errors in the processing.

!!! example

    An example to make DAG idempotent is to use templates such as variable `{{ execution_date }}`.

    It's associated with the expected scheduled time of each run, and the date won't be changed even if we rerun the DAG Run a few hours later.

### **Avoid top-level code in the DAG file**

By default, Airflow reads the dag folder every 30 seconds, including the top-level code that is outside of DAG context.

Because of this, having expensive top-level code, such as making requests to external APIs, can cause performance issues because they are called every 30 seconds rather than only when DAG is scheduled.

The general advice is to limit the amount of top-level code in the DAG file and move it within the DAG context or operators.

This can help reduce unnecessary overheads and allow Airflow to focus on executing the right things.

The following example shows both good and bad ways of making an API request:

```python
# Bad example - requests will be made every 30 seconds instead of everyday at 4:30am
res = requests.get("https://api.sampleapis.com/coffee/hot")

@dag(
    schedule="30 4 * * *",
    start_date=pendulum.datetime(2023, 1, 1, tz="UTC"),
    catchup=False,
    tags=["random"]
)
def random_dag7():

    @task
    def python_operator() -> None:
        logging.info(f"API result {res}")
    python_operator()

random_dag7()

# Good example

@dag(
    schedule="30 4 * * *",
    start_date=pendulum.datetime(2023, 1, 1, tz="UTC"),
    catchup=False,
    tags=["random"]
)
def random_dag7():

    @task
    def python_operator() -> None:
        res = requests.get("https://api.sampleapis.com/coffee/hot") # move API request within DAG context
        logging.info(f"API result {res}")
    python_operator()

random_dag7()
```

## **How to execute tasks parallelly?**
    ```bash
    start_task = DummyOperator(task_id='start_task', dag=dag)
    parallel_task_1 = DummyOperator(task_id='parallel_task_1', dag=dag) 
    parallel_task_2 = DummyOperator(task_id='parallel_task_2', dag=dag) 
    parallel_task_3 = DummyOperator(task_id='parallel_task_3', dag=dag) 
    end_task = DummyOperator(task_id='end_task', dag=dag)
    # Setting up the dependencies start_task >> [parallel_task_1, parallel_task_2, parallel_task_3] >> end_task
    ```
In this example, the three parallel tasks (parallel_task_1, parallel_task_2, parallel_task_3) are specified as a list in the dependency chain. The start_task runs first. Once it completes, all three parallel tasks begin. When all three of them complete, the end_task starts.

## **How to overwrite depends_on_past property?**
The depends_on_past attribute in the default_args dictionary of a DAG applies globally to all tasks in the DAG when set. However, if you want to override this behavior for specific tasks, you can specify the depends_on_past attribute directly on those tasks.

For specific tasks where you want to override this behavior, set depends_on_past directly on the task:
    ```bash
    task_with_custom_dep = DummyOperator(     
        task_id='task_with_custom_dep',
        depends_on_past=False, 
        dag=dag
        )
    ```
