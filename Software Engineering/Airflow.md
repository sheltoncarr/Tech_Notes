*Orchestration* of tasks is critical in software, especially in data engineering and [Machine Learning](../Machine%20Learning/ML%20Engineering/Preprocessing.md). *Airflow* is the most popular open source orchestration software used largely to orchestrate data transformations and other actions in a [Directed Acyclic Graph (DAG)](../Data%20Structures%20&%20Algorithms/Data%20Structures/Graphs.md). By running everything in a DAG, we can ensure there are no cycles and tasks are completed in the order we want, preventing race conditions.

## DAGs

A [Directed Acyclic Graph (DAG)](../Data%20Structures%20&%20Algorithms/Data%20Structures/Graphs.md) is the core concept of Airflow, collecting Tasks together, organized with dependencies and relationships to say how they should run.

Here’s a basic example DAG:

![](../Attachments/Screenshot%2024-08-17%at%5.42.55%PM.png)

It defines four Tasks - A, B, C, and D - and dictates the order in which they have to run, and which tasks depend on what others. It will also say how often to run the DAG - maybe “every 5 minutes starting tomorrow”, or “every day since January 1st, 2020”.

The DAG itself doesn’t care about what is happening inside the tasks; it is merely concerned with how to execute them - the order to run them in, how many times to retry them, if they have timeouts, and so on.

#### Declaring a DAG

There are three ways to declare a DAG - either you can use `with` statement (context manager), which will add anything inside it to the DAG implicitly:

```
import datetime

from airflow import DAG
from airflow.operators.empty import EmptyOperator

with DAG(
    dag_id="my_dag_name",
    start_date=datetime.datetime(2021, 1, 1),
    schedule="@daily",
):
    EmptyOperator(task_id="task")
```

Or, you can use a standard constructor, passing the DAG into any operators you use:

```
import datetime

from airflow import DAG
from airflow.operators.empty import EmptyOperator

my_dag = DAG(
    dag_id="my_dag_name",
    start_date=datetime.datetime(2021, 1, 1),
    schedule="@daily",
)
EmptyOperator(task_id="task", dag=my_dag)
```

Or, you can use the `@dag` decorator to turn a function into a DAG generator:

```
import datetime

from airflow.decorators import dag
from airflow.operators.empty import EmptyOperator


@dag(start_date=datetime.datetime(2021, 1, 1), schedule="@daily")
def generate_dag():
    EmptyOperator(task_id="task")

generate_dag()
```

DAGs are nothing without Tasks to run, and those will usually come in the form of either Operators, Sensors or TaskFlow.

#### Task Dependencies

A Task/Operator does not usually live alone; it has dependencies on other tasks (those upstream of it), and other tasks depend on it (those downstream of it). Declaring these dependencies between tasks is what makes up the DAG structure (the edges of the directed acyclic graph).

There are two main ways to declare individual task dependencies. The recommended one is to use the `>>` and `<<` operators:

```
first_task >> [second_task, third_task]
third_task << fourth_task
```

Or, you can also use the more explicit `set_upstream` and `set_downstream` methods:

```
first_task.set_downstream([second_task, third_task])
third_task.set_upstream(fourth_task)
```

There are also shortcuts to declaring more complex dependencies. If you want to make a list of tasks depend on another list of tasks, you can’t use either of the approaches above, so you need to use `cross_downstream`:

```
from airflow.models.baseoperator import cross_downstream

# Replaces
# [op1, op2] >> op3
# [op1, op2] >> op4
cross_downstream([op1, op2], [op3, op4])
```

And if you want to chain together dependencies, you can use `chain`:

```
from airflow.models.baseoperator import chain

# Replaces op1 >> op2 >> op3 >> op4
chain(op1, op2, op3, op4)

# You can also do it dynamically
chain(*[EmptyOperator(task_id='op' + i) for i in range(1, 6)])
```

#### Loading DAGs

Airflow loads DAGs from Python source files, which it looks for inside its configured `DAG_FOLDER`. It will take each file, execute it, and then load any DAG objects from that file.

This means you can define multiple DAGs per Python file, or even spread one very complex DAG across multiple Python files using imports.

Note, though, that when Airflow comes to load DAGs from a Python file, it will only pull any objects at the top level that are a DAG instance. For example, take this DAG file:

```
dag_1 = DAG('this_dag_will_be_discovered')

def my_function():
    dag_2 = DAG('but_this_dag_will_not')

my_function()
```

While both DAG constructors get called when the file is accessed, only `dag_1` is at the top level (in the `globals()`), and so only it is added to Airflow. `dag_2` is not loaded.

You can also provide an `.airflowignore` file inside your `DAG_FOLDER`, or any of its subfolders, which describes patterns of files for the loader to ignore.

#### Running DAGs

DAGs will run in one of two ways:

* When they are triggered either manually or via the API

* On a defined schedule, which is defined as part of the DAG

DAGs do not require a schedule, but it’s very common to define one. You define it via the `schedule` argument, like this:

```
with DAG("my_daily_dag", schedule="@daily"):
    ...
```

There are various valid values for the schedule argument:

```
with DAG("my_daily_dag", schedule="0 0 * * *"):
    ...

with DAG("my_one_time_dag", schedule="@once"):
    ...

with DAG("my_continuous_dag", schedule="@continuous"):
    ...
```

Every time you run a DAG, you are creating a new instance of that DAG which Airflow calls a DAG Run. DAG Runs can run in parallel for the same DAG, and each has a defined data interval, which identifies the period of data the tasks should operate on.

As an example of why this is useful, consider writing a DAG that processes a daily set of experimental data. It’s been rewritten, and you want to run it on the previous 3 months of data—no problem, since Airflow can *backfill* the DAG and run copies of it for every day in those previous 3 months, all at once.

Those DAG Runs will all have been started on the same actual day, but each DAG run will have one data interval covering a single day in that 3 month period, and that data interval is all the tasks, operators and sensors inside the DAG look at when they run.

In much the same way a DAG instantiates into a DAG Run every time it’s run, Tasks specified inside a DAG are also instantiated into Task Instances along with it.

A DAG run will have a start date when it starts, and end date when it ends. This period describes the time when the DAG actually ‘ran.’ Aside from the DAG run’s start and end date, there is another date called *logical date* (formally known as execution date), which describes the intended time a DAG run is scheduled or triggered. The reason why this is called *logical* is because of the abstract nature of it having multiple meanings, depending on the context of the DAG run itself.

For example, if a DAG run is manually triggered by the user, its logical date would be the date and time of which the DAG run was triggered, and the value should be equal to DAG run’s start date. However, when the DAG is being automatically scheduled, with certain schedule interval put in place, the logical date is going to indicate the time at which it marks the start of the data interval, where the DAG run’s start date would then be the logical date + scheduled interval.


#### DAG Assignment

Note that every single Operator/Task must be assigned to a DAG in order to run. Airflow has several ways of calculating the DAG without you passing it explicitly:

* If you declare your Operator inside a `with DAG` block

* If you declare your Operator inside a `@dag` decorator

* If you put your Operator upstream or downstream of an Operator that has a DAG

Otherwise, you must pass it into each Operator with `dag=`.


#### Control Flow

By default, a DAG will only run a Task when all the Tasks it depends on are successful. There are several ways of modifying this, however:

* Branching - select which Task to move onto based on a condition

* Trigger Rules - set the conditions under which a DAG will run a task

* Setup and Teardown - define setup and teardown relationships

* Latest Only - a special form of branching that only runs on DAGs running against the present

* Depends On Past - tasks can depend on themselves from a previous run

#### DAG Runs

A DAG Run is an object representing an instantiation of the DAG in time. Any time the DAG is executed, a DAG Run is created and all tasks inside it are executed. The status of the DAG Run depends on the tasks states. Each DAG Run is run separately from one another, meaning that you can have many runs of a DAG at the same time.

#### DAG Run Status

A DAG Run status is determined when the execution of the DAG is finished. The execution of the DAG depends on its containing tasks and their dependencies. The status is assigned to the DAG Run when all of the tasks are in the one of the terminal states (i.e. if there is no possible transition to another state) like `success`, `failed` or `skipped`. The DAG Run is having the status assigned based on the so-called “leaf nodes” or simply “leaves”. Leaf nodes are the tasks with no children.

There are two possible terminal states for the DAG Run:

* `success` if all of the leaf nodes states are either `success` or `skipped`,

* `failed` if any of the leaf nodes state is either `failed` or `upstream_failed`.

## Tasks

A Task is the basic unit of execution in Airflow. Tasks are arranged into DAGs, and then have upstream and downstream dependencies set between them in order to express the order they should run in.

There are three basic kinds of Task:

* Operators, predefined task templates that you can string together quickly to build most parts of your DAGs.

* Sensors, a special subclass of Operators which are entirely about waiting for an external event to happen.

* A TaskFlow-decorated `@task`, which is a custom Python function packaged up as a Task.

Internally, these are all actually subclasses of Airflow’s `BaseOperator`, and the concepts of Task and Operator are somewhat interchangeable, but it’s useful to think of them as separate concepts - essentially, Operators and Sensors are templates, and when you call one in a DAG file, you’re making a Task.

#### Relationships

The key part of using Tasks is defining how they relate to each other - their dependencies, or as said in Airflow, their upstream and downstream tasks. You declare your Tasks first, and then you declare their dependencies second.

There are two ways of declaring dependencies - using the `>>` and `<<` (bitshift) operators:

```
first_task >> second_task >> [third_task, fourth_task]
```

Or the more explicit set_upstream and set_downstream methods:

```
first_task.set_downstream(second_task)
third_task.set_upstream(second_task)
```

These both do exactly the same thing, but in general it is recommended to use the bitshift operators, as they are easier to read in most cases.

By default, a Task will run when all of its upstream (parent) tasks have succeeded, but there are many ways of modifying this behaviour to add branching, to only wait for some upstream tasks, or to change behaviour based on where the current run is in history. For more, see Control Flow.

Tasks don’t pass information to each other by default, and run entirely independently. If you want to pass information from one Task to another, you should use XComs.

#### Task Instances

Much in the same way that a DAG is instantiated into a DAG Run each time it runs, the tasks under a DAG are instantiated into Task Instances.

An instance of a Task is a specific run of that task for a given DAG (and thus for a given data interval). They are also the representation of a Task that has state, representing what stage of the lifecycle it is in.

The possible states for a Task Instance are:

* `none`: The Task has not yet been queued for execution (its dependencies are not yet met)

* `scheduled`: The scheduler has determined the Task’s dependencies are met and it should run

* `queued`: The task has been assigned to an Executor and is awaiting a worker

* `running`: The task is running on a worker (or on a local/synchronous executor)

* `success`: The task finished running without errors

* `restarting`: The task was externally requested to restart when it was running

* `failed`: The task had an error during execution and failed to run

* `skipped`: The task was skipped due to branching, LatestOnly, or similar.

* `upstream_failed`: An upstream task failed and the Trigger Rule says we needed it

* `up_for_retry`: The task failed, but has retry attempts left and will be rescheduled.

* `up_for_reschedule`: The task is a Sensor that is in reschedule mode

* `deferred`: The task has been deferred to a trigger

* `removed`: The task has vanished from the DAG since the run started

![](../Attachments/Screenshot%2024-08-17%at%6.16.51%PM.png)

Ideally, a task should flow from `none`, to `scheduled`, to `queued`, to `running`, and finally to `success`.

When any custom Task (Operator) is running, it will get a copy of the task instance passed to it; as well as being able to inspect task metadata, it also contains methods for things like XComs.

## Operators

An Operator is conceptually a template for a predefined Task, that you can just define declaratively inside your DAG:

```
with DAG("my-dag") as dag:
    ping = HttpOperator(endpoint="http://example.com/update/")
    email = EmailOperator(to="admin@example.com", subject="Update complete")

    ping >> email
```

Airflow has a very extensive set of operators available, with some built-in to the core or pre-installed providers. Some popular operators from core include:

* BashOperator - executes a bash command

* PythonOperator - calls an arbitrary Python function

* EmailOperator - sends an email

* Use the `@task` decorator to execute an arbitrary Python function. It doesn’t support rendering jinja templates passed as arguments.

But there are many, many more - you can see the full list of all community-managed operators, hooks, sensors and transfers in the providers packages documentation.

Note: The `@task` decorator is recommended over the classic PythonOperator to execute Python callables with no template rendering in its arguments.

#### Jinja Templating

Airflow leverages the power of Jinja Templating and this can be a powerful tool to use in combination with macros.

For example, say you want to pass the start of the data interval as an environment variable to a Bash script using the `BashOperator`:

```
# The start of the data interval as YYYY-MM-DD
date = "{{ ds }}"
t = BashOperator(
    task_id="test_env",
    bash_command="/tmp/test.sh ",
    dag=dag,
    env={"DATA_INTERVAL_START": date},
)
```

Here, `{{ ds }}` is a templated variable, and because the `env` parameter of the `BashOperator` is templated with Jinja, the data interval’s start date will be available as an environment variable named `DATA_INTERVAL_START` in your Bash script.


## Sensors

Sensors are a special type of Operator that are designed to do exactly one thing - wait for something to occur. It can be time-based, or waiting for a file, or an external event, but all they do is wait until something happens, and then succeed so their downstream tasks can run.

Because they are primarily idle, Sensors have two different modes of running so you can be a bit more efficient about using them:

* `poke` (default): The Sensor takes up a worker slot for its entire runtime

* `reschedule`: The Sensor takes up a worker slot only when it is checking, and sleeps for a set duration between checks

The `poke` and `reschedule` modes can be configured directly when you instantiate the sensor; generally, the trade-off between them is latency. Something that is checking every second should be in `poke` mode, while something that is checking every minute should be in `reschedule` mode.

Much like Operators, Airflow has a large set of pre-built Sensors you can use, both in core Airflow as well as the providers system.

## TaskFlow

If you write most of your DAGs using plain Python code rather than Operators, then the TaskFlow API will make it much easier to author clean DAGs without extra boilerplate, all using the `@task` decorator.

TaskFlow takes care of moving inputs and outputs between your Tasks using XComs for you, as well as automatically calculating dependencies - when you call a TaskFlow function in your DAG file, rather than executing it, you will get an object representing the XCom for the result (an `XComArg`), that you can then use as inputs to downstream tasks or operators. For example:

```
from airflow.decorators import task
from airflow.operators.email import EmailOperator

@task
def get_ip():
    return my_ip_service.get_main_ip()

@task(multiple_outputs=True)
def compose_email(external_ip):
    return {
        'subject':f'Server connected from {external_ip}',
        'body': f'Your server executing Airflow is connected from the external IP {external_ip}<br>'
    }

email_info = compose_email(get_ip())

EmailOperator(
    task_id='send_email_notification',
    to='example@example.com',
    subject=email_info['subject'],
    html_content=email_info['body']
)
```

Here, there are three tasks - `get_ip`, `compose_email`, and `send_email_notification`.

The first two are declared using TaskFlow, and automatically pass the return value of `get_ip` into `compose_email`, not only linking the XCom across, but automatically declaring that `compose_email` is downstream of `get_ip.

`send_email_notification` is a more traditional Operator, but even it can use the return value of `compose_email` to set its parameters, and again, automatically work out that it must be downstream of `compose_email`.

You can also use a plain value or variable to call a TaskFlow function - for example, this will work as you expect (but, of course, won’t run the code inside the task until the DAG is executed - the `name` value is persisted as a task parameter until that time):

```
@task
def hello_name(name: str):
    print(f'Hello {name}!')

hello_name('Airflow users')
```

#### Context

You can access Airflow context variables by adding them as keyword arguments as shown in the following example:

```
from airflow.models.taskinstance import TaskInstance
from airflow.models.dagrun import DagRun


@task
def print_ti_info(task_instance: TaskInstance | None = None, dag_run: DagRun | None = None):
    print(f"Run ID: {task_instance.run_id}")  # Run ID: scheduled__2023-08-09T00:00:00+00:00
    print(f"Duration: {task_instance.duration}")  # Duration: 0.972019
    print(f"DAG Run queued at: {dag_run.queued_at}")  # 2023-08-10 00:00:01+02:20
```

#### Logging

To use logging from your task functions, simply import and use Python’s logging system:

```
logger = logging.getLogger("airflow.task")
```

Every logging line created this way will be recorded in the task log.