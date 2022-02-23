---
title: "Sensors 101"
description: "An introduction to Sensors in Apache Airflow."
date: 2018-05-21T00:00:00.000Z
slug: "what-is-a-sensor"
heroImagePath: "https://assets.astronomer.io/website/img/guides/IntroToDAG_preview.png"
tags: ["Operators", "Tasks", "Basics", "Sensors"]
---

## Overview

[Apache Airflow Sensors](https://airflow.apache.org/docs/apache-airflow/stable/concepts/sensors.html) are a special kind of operator that are designed to wait for something to happen. When sensors run, they check to see if a certain condition is met before they are marked successful and let their downstream tasks execute. When used properly, they can be a great tool for making your DAGs more event driven.

In this guide, we'll cover the basics of using sensors in Airflow, best practices for implementing sensors in production, and new sensor-related features in Airflow 2 like smart sensors and deferrable operators. For more in-depth guidance on these topics, check out our [Mastering Sensors webinar](https://www.astronomer.io/events/webinars/creating-data-pipelines-using-master-sensors/).

## Sensor Basics

Sensors are conceptually simple: they are a type of operator that check if a condition is satisfied on a particular interval. If the condition has been met, the task is marked successful and the DAG can move on to downstream tasks. If the condition hasn't been met, the sensor waits for another interval before checking again. 

All sensors inherit from the [`BaseSensorOperator`](https://github.com/apache/airflow/blob/main/airflow/sensors/base.py) and have the following parameters:

- `mode`: How the sensor operates. There are two types of modes:
    - `poke`: This is the default mode. When using `poke`, the sensor occupies a worker slot for the entire execution time and sleeps between pokes. This mode is best if you expect a short runtime for the sensor.
    - `reschedule`: When using this mode, if the criteria is not met then the sensor releases its worker slot and reschedules the next check for a later time. This mode is best if you expect a long runtime for the sensor, because it is less resource intensive and frees up workers for other tasks.
- `poke_interval`: When using `poke` mode, this is the time in seconds that the sensor waits before checking the condition again. The default is 30 seconds.
- `exponential_backoff`: When set to `True`, this setting creates exponentially longer wait times between pokes in `poke` mode.
- `timeout`: The maximum amount of time in seconds that the sensor should check the condition for. If the condition has not been met when this time is reached, the task fails.
- `soft_fail`: If set to `True`, the task is marked as skipped if the condition is not met by the `timeout`.

Beyond these parameters, different types of sensors have different implementation details. Below we describe some of the most commonly used sensors and how to implement them.

### Commonly Used Sensors

Many Airflow provider packages contain sensors that wait for various criteria in different source systems. There are too many to cover all of them here, but below are some of the most basic and commonly used sensors that we think everybody should know about:

- [`S3KeySensor`](https://registry.astronomer.io/providers/amazon/modules/s3keysensor): Waits for a key (file) to land in an S3 bucket. This sensor is useful if you want your DAG to process files from S3 as they arrive.
- [`DateTimeSensor`](https://registry.astronomer.io/providers/apache-airflow/modules/datetimesensor): Waits for a specified date and time to pass. This sensor is useful if you want different tasks within the same DAG to run at different times.
- [`ExternalTaskSensor`](https://registry.astronomer.io/providers/apache-airflow/modules/externaltasksensor): Waits for an Airflow task to be completed. This sensor is useful if you want to implement [cross-DAG dependencies](https://www.astronomer.io/guides/cross-dag-dependencies) in the same Airflow environment.
- [`HttpSensor`](https://registry.astronomer.io/providers/http/modules/httpsensor): Waits for an API to be available. This sensor is useful if you want to ensure your API requests are successful.
- [`SqlSensor`](https://registry.astronomer.io/providers/apache-airflow/modules/sqlsensor): Waits for data to be present in a SQL table. This sensor is useful if you want your DAG to process data as it arrives in your database.
- [`PythonSensor`](https://registry.astronomer.io/providers/apache-airflow/modules/pythonsensor): Waits for a Python callable to return `True`. This sensor is useful if you want to implement complex conditions in your DAG.

> To browse and search all of the available sensors in Airflow, visit the [Astronomer Registry](https://registry.astronomer.io/modules?types=sensors), the discovery and distribution hub for Apache Airflow integrations created to aggregate and curate the best bits of the ecosystem.

### Example Implementation

To show how you might implement a DAG using the `SqlSensor`, consider the following code: 

```python
from airflow.decorators import task, dag
from airflow.sensors.sql import SqlSensor

from typing import Dict
from datetime import datetime

def _success_criteria(record):
        return record

def _failure_criteria(record):
        return True if not record else False

@dag(description='DAG in charge of processing partner data',
     start_date=datetime(2021, 1, 1), schedule_interval='@daily', catchup=False)
def partner():
    
    waiting_for_partner = SqlSensor(
        task_id='waiting_for_partner',
        conn_id='postgres',
        sql='sql/CHECK_PARTNER.sql',
        parameters={
            'name': 'partner_a'
        },
        success=_success_criteria,
        failure=_failure_criteria,
        fail_on_empty=False,
        poke_interval=20,
        mode='reschedule',
        timeout=60 * 5
    )
    
    @task
    def validation() -> Dict[str, str]:
        return {'partner_name': 'partner_a', 'partner_validation': True}
    
    @task
    def storing():
        print('storing')
    
    waiting_for_partner >> validation() >> storing()
    
dag = partner()
```

This DAG is waiting for data to be available in a Postgres database before running validation and storing tasks. In general, the `SqlSensor` runs a SQL query and is marked successful when that query returns data: specifically, when the result is not in the set (0, '0', '', None). The `SqlSensor` task in this example (`waiting_for_partner`) runs the `CHECK_PARTNER.sql` script every 20 seconds (the `poke_interval`) until the data is returned. The `mode` is set to `reschedule`, meaning between each 20 second interval the task will not take a worker slot. The `timeout` is set to 5 minutes, so if the data hasn't arrived by that time, the task fails. Once the `SqlSensor` criteria is met, the DAG moves on to the downstream tasks. You can find the full code for this example in [this repo](https://github.com/marclamberti/webinar-sensors).

## Sensor Best Practices

Sensors are easy to implement, but there are a few things to keep in mind when using them to ensure you are getting the best possible Airflow experience. The following tips will help you avoid any performance issues when using sensors:

- **Always** define a meaningful `timeout` parameter for your sensor. The default for this parameter is 7 days, which is a long time for your sensor to be running! When you implement a sensor, consider your use case and how long you expect the sensor to be waiting, then define the sensor's timeout accordingly.
- Whenever possible and especially for long-running sensors, use the `reschedule` mode so your sensor is not constantly occupying a worker slot. This helps to avoid deadlocks in Airflow where sensors take all of the available worker slots. The exception to this rule is the following point:
- If your `poke_interval` is very short (less than about 5 minutes), use the `poke` mode. Using `reschedule` mode in this case can overload your scheduler.
- Define a meaningful `poke_interval` based on your use case. There is no need for a task to check a condition every 30 seconds (the default) if you know the total amount of waiting time is likely to be 30 minutes.

## Smart Sensors

> **Note:** Smart sensors have been deprecated as of Airflow 2.2.4. It is generally recommended to use deferrable operators instead.

[Smart sensors](https://airflow.apache.org/docs/apache-airflow/stable/concepts/smart-sensors.html) are a relatively new feature released with Airflow 2.0, where sensors are executed in batches using a centralized process. This eliminates a major drawback of classic sensors, which use one process per sensor and therefore can consume considerable resources at scale for longer running tasks. With smart sensors, sensors in your DAGs are registered to the metadata database, and then a separate DAG fetches those sensors and manages processing them. 

![Smart Sensors](https://assets2.astronomer.io/main/guides/sensors-101/smart_sensors_architecture.png)

Note that smart sensors have been deprecated since Airflow 2.2.4. They are somewhat complex to implement and not very widely used to-date. They are also generally less preferable to deferrable operators (more on these in the section below), which are more flexible.

If you are going to use smart sensors with an applicable Airflow version, you can enable them using the following steps:

1. Update your Airflow config with the following environment variables. If you're using Astronomer, you can add them to your Dockerfile with the code below.

    ```bash
    ENV AIRFLOW__SMART_SENSOR__USE_SMART_SENSOR=True
    ENV AIRFLOW__SMART_SENSOR__SHARD_CODE_UPPER_LIMIT=10000
    ENV AIRFLOW__SMART_SENSOR__SHARDS=1
    ENV AIRFLOW__SMART_SENSOR__SENSORS_ENABLED=SmartExternalTaskSensor
    ```

    The `SHARD_CODE_UPPER_LIMIT` parameter helps determine how your sensors will be distributed amongst your smart sensor jobs. The `SHARDS` parameters determines how many smart sensor jobs will run concurrently in your Airflow environment. This should be scaled up as you have more sensors. Finally, the `SENSORS_ENABLED` parameter should specify the Python class you will create to tell Airflow that certain sensors should be treated as smart sensors (more on this in Step 2).

2. Create your smart sensor class. Create a file in your `plugins/` directory and define a class for each type of sensor you want to make "smart". In this example, we implement smart sensors with the `FileSensor`, and the class looks like this:

    ```python
    from airflow.sensors.filesystem import FileSensor
    from airflow.utils.decorators import apply_defaults
    from typing import Any

    class SmartFileSensor(FileSensor):
        poke_context_fields = ('filepath', 'fs_conn_id') # <- Required

        @apply_defaults
        def __init__(self,  **kwargs: Any):
            super().__init__(**kwargs)

        def is_smart_sensor_compatible(self): # <- Required
            result = (
                not self.soft_fail
                and super().is_smart_sensor_compatible()
            )
            return result
    ```

    The class should inherit from the sensor class you are updating (e.g. `FileSensor`). It must include `poke_context_fields`, which specifies the arguments needed by your sensor, and the `is_smart_sensor_compatible` method, which tells Airflow this type of sensor is a smart sensor. Note when using smart sensors you cannot use soft fail or any callbacks.

    The implementation of this class varies depending on which sensor you are using. For an example with the more complicated `ExternalTaskSensor`, see the [supporting repo](https://github.com/marclamberti/webinar-sensors).

3. Deploy your code to Airflow and turn on the `smart_sensor_group_shard_0` DAG to run your smart sensors. Note you might have more than one smart sensor DAG if you set your `SHARDS` parameter to greater than 1. 

## Deferrable Operators

[Deferrable operators](https://airflow.apache.org/docs/apache-airflow/stable/concepts/deferring.html) (sometimes referred to as asynchronous operators) were released with Airflow 2.2 and were designed to eliminate the problem of *any* operator or sensor taking up a full worker slot for the entire time they are running. In other words, they solve the same problems as smart sensors, but for a much wider range of use cases. 

For DAG authors, using built-in deferrable operators is no different from using regular operators (i.e. using them is much simpler than implementing smart sensors). You just need to ensure you have a `triggerer` process running in addition to your Scheduler. Currently, the `DateTimeSensorAsync` and `TimeDeltaSensorAsync` sensors are available out of the box in OSS Airflow. More deferrable operators will likely be released in future Airflow versions. For more on writing your own deferrable operators, check out the [Apache Airflow documentation](https://airflow.apache.org/docs/apache-airflow/stable/concepts/deferring.html#smart-sensors).
