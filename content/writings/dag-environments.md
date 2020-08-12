+++
title = "DAG Environment"

[taxonomies]
categories = ["Useful Airflow Patterns"]

+++

A common way to run airflow is a stating/production instance where DAGs are tested in staging and then promoted to production. You'd like the same files to run in staging and production, but you don't want staging workflows interfering with production (touching the same files, kicking off the same proceses, etc). Since DAGs are just python files, a common pattern is to disambiguate your deployements by storing a airlfow variable/enviornment variable with the environment and then using it like so:

```python
airflow_environment = os.environ["AIRFLOW_ENVIRONMENT"]
# or alternatively if you've exported the env as `AIRFLOW_VAR_AIRFLOW_ENVIRONMENT`
airflow_environemnt = Variable.get("airflow_environment")
```

Then in your code, when setting enviornment dependent resouces you switch on `airflow_environment`:

```python
if airflow_environment == "production":
    s3_bucket = "my-prod-bucket"
elif airflow_environemnt == "staging":
    s3_bucket = "my-staging-bucket"
else:
    # make sure we don't screw this up
    raise ValueError("Unknown Airflow Environment")
```

Alternatively you might wrap this logic in a function call:

```python
def get_s3_bucket():
    if airflow_environment == "production":
        return "my-prod-bucket"
    elif airflow_environemnt == "staging":
        return "my-staging-bucket"
    else:
        # make sure we don't screw this up
        raise ValueError("Unknown Airflow Environment")

### somewhere later

sensor = S3KeySensor(
    bucket_name=get_s3_bucket(),
    bucket_key="key_i_look_for"
)
```

Using this pattern quickly leads to sprawl and makes refactoring (or adding envs) a pain. A better pattern is to wrap anything environment dependent into a central class that can be used anywhere. For example:

```python
import os
from typing import *

class DAGEnvironment:
    """
    A class to manage any resource that depends on the environment the dag runs in
    """

    ALLOWED_ENVIRONMENTS = ["Staging", "Production"]

    def __init__(self, airlfow_environment: Optional[str] = None):
        if airflow_environment is None:
            airflow_environment = os.eviron.get("AIRFLOW_ENVIRONMENT")
            # or alternatively if you've exported the env as `AIRFLOW_VAR_AIRFLOW_ENVIRONMENT`
            # airflow_environemnt = Variable.get("airflow_environment")

        if not airflow_environment:
            raise ValueError("Unable to detect environment")
        elif airflow_environment not in self.ALLOWED_ENVS:
            raise ValueError(f"Unknown env {airflow_environment}. Abort for safety.")

        self.airflow_environment = airflow_environment

    @property
    def production(self) -> bool:
        """Helper property"""
        return self.airflow_environment == "Production"

    @property
    def staging(self) -> bool:
        """Helper property"""
        return self.airflow_environment == "Staging"

    def _get_environment_bucket(self, staging_bucket: str, production_bucket: str) -> str:
        """
        Helper covering most use cases
        """
        return staging_bucket if self.staging else production_bucket

    # Now we can define all the environment specific variables
    @property
    def my_bucket(self) -> str:
        return _get_environment_bucket("my-staging-bucket", "my-production-bucket")

    # etc
```

Putting all the environment switching logic in a class (and in one place in the class) helps encapsualte the logic and leads to cleaner, more understandable, less brittle code.

We can now use it:

```python
from .dag_environment import DAGEnvironment

env = DAGEnvironment()

with Dag("my_dag") as dag:
    sensor = S3KeySensor(bucket_name=env.my_bucket, bucket_key="key_i_look_for")
# etc
```
