+++
title = "Timezone Aware SLAs"

[taxonomies]
categories = ["Useful Airflow Patterns"]

+++


**The Problem**: You have a DAG that runs on UTC time (the airflow default), but you perform certain actions (for example: moving files) relative to some local time (say before 9:30am EST). Now let's say you want an alert if that process doesn't complete by a certain *local time*.

In airflow SLAs must be set as a `timedelta` to the DAG start time (which in this case is in UTC). (https://airflow.apache.org/docs/stable/concepts.html#slas). To create a local time SLA, we need a helper to create a `timedelta` to a time in a specified timezone:

```python
import pendulum # installed as part of airflow

def make_tz_based_sla(time_str: str, timezone: str ="America/New_York") -> timedelta:
    local_time = pendulum.parse(time_str, tz=timezone)
    midnight_utc = pendulum.parse("0:00", tz="UTC")
    return local_time.diff(midnight_utc)
```

You can customize this depending on the start time, default timezone of your DAG. Use anywhere you want to set an SLA.
