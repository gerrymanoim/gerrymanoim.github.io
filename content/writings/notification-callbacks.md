+++
title = "Better Notifications via Better Callbacks - Part 1"

[taxonomies]
categories = ["Useful Airflow Patterns"]

+++

In Part 1 we will build a simple generic notification router for Airflow. In Part 2 we will then refactor our generic notification router to be more robust and safer.

If you want to skip the discussion, the full code can be found [on github](https://gist.github.com/gerrymanoim/83b08008b2eefd2a20cb4fc38aa25544).

## Motivations for Better Callbacks

Traditionally in airflow one manages notifications for tasks via the `on_success_callback` and `on_failure_callback` which take a `Callable`. For example, you often see something like:

```python
from . import (
    slack_failure_callback,
    slack_retry_callback,
    slack_success_callback
 ) # as examples callback services you may already have

with DAG(dag_id='my_dag') as dag:

    first_task = DummyOperator(
        task_id='first_task',
        on_success_callback=slack_success_callback,
        on_failure_callback=slack_failure_callback,
        on_retry_callback=slack_retry_callback,
    )

    second_task = DummyOperator(
        task_id='second_task',
        on_success_callback=slack_success_callback,
        on_failure_callback=slack_failure_callback,
        on_retry_callback=slack_retry_callback,
    )

    third_task = DummyOperator(
        task_id='third_task',
        on_success_callback=slack_success_callback,
        on_failure_callback=slack_failure_callback,
        on_retry_callback=slack_retry_callback,
    )

    first_task >> second_task >> third_task
```

This gets rather repetitive quickly. Luckily airflow provides a `default_args` mechanism on the `DAG` level, so we could instead do something like:

```python
from . import (
    slack_failure_callback,
    slack_retry_callback,
    slack_success_callback
) # as examples callback services you may alerady have

default_args = {
    'on_failure_callback': slack_failure_callback,
    'on_retry_callback': slack_retry_callback,
    'on_success_callback': slack_success_callback,
}

with DAG(
    dag_id='my_dag',
    default_args=default_args,
) as dag:

    first_task = DummyOperator(
        task_id='first_task',
    )

    second_task = DummyOperator(
        task_id='second_task',
    )

    third_task = DummyOperator(
        task_id='third_task',
    )

    first_task >> second_task >> third_task
```

However:

- What if we want to send different types of notifications depending on the task? We're back to putting callbacks on each task.
- What if we want to send notifications to multiple places? We're stuck building wrappers around other callbacks.

Because the code for attaching a notification to a task and routing that notification is intermingled, this makes callbacks hard to work with. What we need instead is a universal router we can attach to any task that will have a separate machanism to decide where to send events.

# Building A Callback Router

In order to have a router, we must first define the notion of routes. Routes for a success/failure/retry will be a yaml file of `notifier_type` (email, slack, etc) which have places they `send_to` given any number of `condition`s (which are arbitarry python expressions). More concretely, this will look like:

```
<notifier_type>
    - send_to: <who>
      where:
        - <condition>
        - <condition>
        - <...>
```

For example we could have

```yaml
# on_failure.yaml

slack:
    - send_to: '#dataload_notifiactions'
      where:
        - airflow_environment=="Production"
        - dag_id == "very_loud_dag"
email:
    - send_to: 'staging_notifiactions@company.com'
      where:
        - airflow_environment=="Staging"
```

Here we would send an email for every failure in staging and a slack message for every failure in production and any failure coming from `very_loud_dag`. Having an expressive human readable language for our routes allows us to easily make changes to notifications and understand where notifactions go without wading though all the business logic of the tasks themselves.

Now we need code to represent our routes (a `Route` class), code to dispatch events (a `Router` class), and code to actually deal with sending notifications (a `Notifier` class).

## Routes

A `Route` is a simple wrapper arounnd our yaml rules. A route needs to know:

1. When should a notification be sent?
2. What is actually sending the notification?
3. Who do we send a notification to?

```python
Route = namedtuple("Route", ["condition", "notifer_type", "send_to"])
```

## Router

The `Router` is responsible for:

1. Recieving events and figuring out whether they fullfill the `condition` on any `Route`s
2. Enriching the context of events
3. Passing the enriched context to the correct `Notifier`.

When a `Router` is created it will look for a `routing_file` for its `router_type`. Then it will parse and store all the routes from that file. When a `Router` is called to route a particular event it will first enrich the context on the event it has recieved and then see if the conditions on any route match that context. If so, it will call the appropriate `Notifer` for that `Route` with the enriched context.

Here's how we acomplish that:

```python
class Router:
    """
    Wouldn't it be nice to route some notifications
    """
    # Some example notifiers you may have
    notifiers = {
        "slack": SlackNotifer,
        "email": EmailNotifier,
        "page": PagerDutyNotifier,
        "http": HTTPNotifier,
    }

    callback_base_path = Path(__file__).parent / "routes"

    router_types = {
        "failure": self.callback_base_path / "on_failure.yaml",
        "success": self.callback_base_path / "on_success.yaml",
        "retry": self.callback_base_path / "on_retry.yaml",
        "sla_miss": self.callback_base_path / "on_sla_miss_yaml",
    }

    def __init__(self, router_type: str):
        """
        `router_type` is one of
        - failure
        - retry
        - success
        - sla miss
        It controls which rules file this router will load.
        """
        log.debug("Creating Router for {}".format(router_type))
        routing_file = self.router_types.get(router_type)

        if not routing_file:
            raise KeyError(
                "Couldn't find routing file for type {}".format(router_type)
            )

        self.routes = self.make_routes(routing_file)
        self.router_type = router_type

    def __call__(self, *args):
        """
        Airflow has two types of callbacks:
        1. on_success/failure/retry: Called by passing the context
         `fn(context)`
        2. sla_miss: Called by passing multiple args (via schedule_job.py)

        This is a bit of a hack, but probably better than inspecting the call
        stack
        """
        log.info("__call__")
        if len(args) == 1:
            # we were passed a single var which is the context
            context = args[0]
        if len(args) == 5:
            # passed multiple args that we pack up
            # see https://github.com/apache/airflow/blob/7930234726c5e9cb9745cc7944047ac343ab832a/airflow/jobs/scheduler_job.py#L455
            keys = (
                "dag",
                "task_list",
                "blocking_task_list",
                "slas",
                "blocking_tis",
            )
            context = {key: arg for key, arg in zip(keys, args)}
        else:
            raise NotImplementedError(
                "Passed a number of args we do not know how to deal with. "
                f"Passed {len(args)}"
            )

        context = self.enrich_context(context)
        if self.routes:
            self.route(context)
        else:
            log.info(
                "Note routes defined for dispatcher {}".format(
                    self.router_type
                )
            )

    @property
    def airflow_environment(self) -> str:
        """
        An example of something you might want in `enrich_context`.

        This will never get sent as part of the task instance context so we
        patch it in.
        """
        airflow_url = configuration.get("webserver", "BASE_URL")
        if airflow_url == "your-production-url":
            return "Staging"
        elif airflow_url == "your-staging-url":
            return "Production"
        else:
            raise ValueError("Unknown base_url {}. Abort.".format(airflow_url))

    def enrich_context(self, base_context: dict) -> dict:
        """
        Enrich the context we get from the task with more useful information.
        This makes writing some routing rules a bit easier.

        Args:
            base_context (dict): Generally the task instance context
            See https://airflow.apache.org/code.html#macros for a list
        """
        enriched_context = dict(base_context)
        enriched_context["airflow_environment"] = self.airflow_environment
        enriched_context["airflow_url"] = base_context.get("conf").get(
            "webserver", "BASE_URL"
        )
        enriched_context["dag_id"] = base_context.get("dag").dag_id
        enriched_context["task_id"] = base_context.get("task").task_id

        return enriched_context

    def make_routes(self, routing_file: Path) -> List[Route]:
        """
        Converts route file to a flat structure

        We flatten the heiarchy of the route file to make it easier to use when
        evaluating in a dag context.
        """
        if not routing_file.exists() or routing_file.stat().st_size == 0:
            log.info("Cannot work with routing file {}".format(routing_file))
            return []
        with routing_file.open() as f:
            route_dict = yaml.load(f.read(), Loader=yaml.FullLoader)

        #  <3 list comps
        routes = [
            Route(condition, notifier_type, route["send_to"])
            for notifier_type, routes in route_dict.items()
            for route in routes
            for condition in route["where"]
        ]
        log.debug(
            "Created {} routes for dispatcher {}".format(
                len(routes), self.router_type
            )
        )
        return routes

    def route(self, ti_context: dict):
        """
        Routes a context when called
        """
        log.info("Dispatching message")
        messages_to_send = filter(
            lambda route: eval(route.condition, None, ti_context), self.routes
        )

        for message in messages_to_send:
            log.info("Sending message: {}".format(message))

            notifier = self.get_notifier(message.notifier_type)
            notifier.notify(message.send_to, ti_context)


    @lru_cache(maxsize=5)
    def get_notifier(self, notifier_type: str):
        notifier = self.notifiers.get(notifier_type)
        if not notifier:
            raise KeyError(
                "Could not find a notifier for {}".format(notifier_type)
            )
        return notifier(self.router_type)
```

To use the `Router` we will create a router for every type of event we'd like notifications on and then use them as our `default_args` on the DAG level. For example:

```python
on_failure = Router('failure')
on_retry = Router('retry')
on_success = Router('success')
on_sla_miss = Router('sla_miss')
```

Then in the DAG code:

```python
default_args = {
    'on_failure_callback': slack_failure_callback,
    'on_retry_callback': slack_retry_callback,
    'on_success_callback': slack_success_callback,
}

with DAG(
    dag_id='my_dag',
    default_args=default_args,
    sla_miss_callbac=on_sla_miss,
) as dag:
    ### the rest of your code here
```

## Notifier

Finally we need to create all the classes that will actually send our messages. All we need is for the `__init__` function to take a `notifier_type` argument and for the class to implement a `notify` method, taking `send_to` and `context` arguments.

An example set of notifers would look like:

```python
class GenericNotifier:
    """
    Shamelessly taken from `callback_wrappers` in moneytree
    """

    # Unique TI identifier
    ti_id = "{{ dag.dag_id }}.{{ task.task_id }} [{{ ts }}]"

    ti_url = (
        "{{ airflow_url }}/task?"
        "task_id={{ task.task_id }}"
        "&dag_id={{ dag.dag_id }}"
        "&execution_date={{ ts | urlencode }}"
    )

    log_url = (
        "{{ airflow_url }}/log?"
        "task_id={{ task.task_id }}"
        "&dag_id={{ dag.dag_id }}"
        "&execution_date={{ ts | urlencode }}"
    )

    graph_url = "{{ airflow_url }}/graph?dag_id={{ dag.dag_id }}"

    # Uses magic Slack formatting to create a <pre> section that's also a link!
    prefix = "<{ti_url}|`{ti_id}`>".format(ti_url=ti_url, ti_id=ti_id)

    def get_jinja_env(self, context: dict):
        """
        Given a context, return an appropriate Jinja environment.
        """
        dag = context.get("dag")
        jinja_env = dag.get_template_env() if dag else jinja2.Environment(cache_size=0)
        return jinja_env

    def jinja_template(text, context, jinja_env):
        """
        Return the templated string.
        """
        return jinja_env.from_string(text).render(**context)


class SlackNotifer(GenericNotifier):
    def __init__(self, notifier_type: str):
        self.notifier_type = notifier_type
        self.message_tmpl = self.get_message_tmpl(notifier_type)

    def get_message_tmpl(self, notifier_type: str) -> str:
        if notifier_type == "success":
            tmpl = ":success:" + self.prefix + " is complete."
        elif notifier_type == "failure":
            tmpl = ":x:" + self.prefix + " has failed!"
        elif notifier_type == "retry":
            tmpl = ":warning:" + self.prefix + " has retried!"
        elif notifier_type == "sla_miss":
            tmpl = ":warning:" + self.prefix + " has missed its sla!"
        else:
            raise NotImplementedError(
                "No message template for type {}".format(notifier_type)
            )
        return tmpl

    def get_username(self, dag_id: str, airflow_environment: str, **context) -> str:
        return dag_id + "[{}]".format(airflow_environment)

    @property
    def slack_link(self):
        return "<{ti_url}|`{ti_id}`>".format(ti_url=self.ti_url, ti_id=self.ti_id)

    @property
    def token(self) -> str:
        return Variable.get("slack_token", "foobar")

    def notify(self, send_to: str, context: dict):
        jinja_env = self.get_jinja_env(context)

        slack_kwargs = {}
        slack_kwargs["channel"] = send_to
        slack_kwargs["text"] = self.jinja_template(
            self.message_tmpl, context, jinja_env
        )
        slack_kwargs["username"] = self.get_username(**context)
        slack_kwargs["icon_url"] = self.icon_url
        log.info("Sending message: '%s'", json.dumps(slack_kwargs))

        SlackAPIPostOperator(
            task_id="tmp_slack", token=self.token, **slack_kwargs
        ).execute()


class EmailNotifier(GenericNotifier):
    def __init__(self, notifier_type: str):
        self.notifier_type = notifier_type
        self.message_tmpl = self.get_message_tmpl()

    def get_message_tmpl(self) -> str:
        """Get a template for the message

        Returns:
            str -- Message template that jinja will render
        """
        html_template = """
        <html>
        <body>
            Task Instance: <a href="{ti_url}">Link</a> <br />
            Log: <a href="{log_url}">Link</a> <br />
            Graph: <a href="{graph_url}">Link</a> <br />
        </body>
        <html>
        """
        return html_template.format(
            ti_url=self.ti_url, log_url=self.log_url, graph_url=self.graph_url
        )

    def get_subject(self, context: dict) -> str:
        return "Airflow {} alert: <{} [{}]>".format(
            context.get("airflow_environment"),
            context.get("task_instance_key_str"),
            self.notifier_type,
        )

    def notify(self, send_to: str, context: dict):
        jinja_env = self.get_jinja_env(context)

        text = self.jinja_template(self.message_tmpl, context, jinja_env)
        subject = self.get_subject(context)
        some_function_that_sends_emails(to=send_to, subject=subject, html_content=text)

```

That's it! Now airflow will route all notifcations to our `Router`s, which will evalute them againsts the conditions in our `Route`s and send them using the right `Notifer`s.

# Room for Improvement

Forthcoming post covering:

- making sure yaml conditions are valid
- making router cleaner
- being more efficient with our notifiers
- enforcing the interface on notifiers

# Putting it all together

All the code in one place: [https://gist.github.com/gerrymanoim/83b08008b2eefd2a20cb4fc38aa25544](https://gist.github.com/gerrymanoim/83b08008b2eefd2a20cb4fc38aa25544)
