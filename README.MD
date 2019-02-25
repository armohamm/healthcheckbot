# Healthcheck Bot

Healthcheck Bot is a standalone highly configurable and extendable application for verifying application status.
User is free to configure targets to be tested as well as the way output needs to be delivered. Application also supports custom
assertions written in Python for the cases when check scenario is more complicated.

## Usage

Package provides executable `healthcheckbot` which is the main entry point. You must pass a configuration file containing definitions of
your watchers and other options. In order to start application run:

```
./healthcheckbot -c ../../examples/config.yaml server run
```


## Concepts

Consider the following configuration example:

```yaml
outputs:
  console:
    provider: healthcheckbot.outputs.ConsoleOutput

triggers:
  each_1_minute:
    provider: healthcheckbot.triggers.SimpleTimer
    interval: 60

watchers:
  google_home:
    provider: healthcheckbot.watchers.HttpRequest
    url: http://google.com
    assert_status: 200
    triggers:
      - each_1_minute
```

In this example we define one single watcher which will send HTTP request to `http://google.com` each minute.
Healthcheck will be treated as failed when response status will be not `200`.
Result of the watcher evaluation will be printed to STDOUT.

Generally there are 4 types of entities(module types) healthcheckbot works with: Outputs, Triggers, Watchers, WatcherAsserts.
Sections below describe each one. Please also note that user has an ability to implement own module
to extend or override default behaviour and connect it without modifying core code. See [Customization](#customization) section below.
Regardless of the module type you define there is one mandatory component called `provider`. It defines fully qualified name
of the class implementing corresponding module. The rest of options are parameters for module instance.

### Outputs

Output define the way how watcher evaluation result will be delivered to the end user. It might be as simple as just console output
or more real life write to database or centralized metric collection system like CloudWatch or Graylog2.

The are a couple of implementations of outputs built-in the package.

#### Console output

Just prints serialized json output to the stdout. There are no any configuration parameters.

Usage Example:

```yaml
outputs:
  console:
    provider: healthcheckbot.outputs.ConsoleOutput
```

#### Logger output

This one is very similar to console output but serialized result will be passed to the logger.

Parameters

| Parameter  | Description                              | Default Value | Required |
| ---------- | ---------------------------------------- | ------------- | -------- |
| log_level  | Log level to be used when output result. | INFO          | No       |
| loger_name | Name of the logger to use                | OUT           | No       |


### Triggers

Triggers responsible for initiation of worker execution. The most common use case is periodic run but other scenarios are possible as well e.g. execution after HTTP call.

#### Simple Timer

This implementation of the trigger is pretty self explanatory - all it does is periodic watchers execution with constant interval specified as parameter.

| Parameter         | Description                              | Default Value | Required |
| ----------------- | ---------------------------------------- | ------------- | -------- |
| interval          | Time interval in seconds between iterations | 300           | No       |
| start_immediately | If set to True the first iteration will be triggered immediately after application start, otherwise in `interval` seconds | True          | No       |

Example

```yaml
triggers:
  each_1_minute:
    provider: healthcheckbot.triggers.SimpleTimer
    interval: 60
  each_5_minutes:
    provider: healthcheckbot.triggers.SimpleTimer
    interval: 300
```

### Watchers

Watchers are modules who actually read the state and could optionally run some assertions over the state. Parameters mostly depend on implementation but there are a couple of options common for all watchers.

* `triggers` - a list of trigger names which will invoke given watcher. It is important to list at list one trigger otherwise watcher will be never invoked.
* `custom_assertions` - a dictionary containing assertions to be applied as a part of state verification after regular module assertions. See section [Watcher Asserts](#watcher-asserts) for details.

### Watcher Asserts

TBD

## Customization

An ability to extend behavior of any module is a key feature of HealthcheckBot. In order to make it easier load modules from the outside user could extend classpath (folders to be scanned for classes) with the simple configuration option. Consider the following example:

```yaml
app:  
  classpath:
    - /tmp
outputs:
  console:
    provider: healthcheckbot.outputs.ConsoleOutput
triggers:
  each_1_minute:
    provider: healthcheckbot.triggers.SimpleTimer
    interval: 60
watchers:
  system_time:
    provider: logicify.watchers.SystemTimeWatcher
    triggers:
      - each_1_minute
```

Our `/tmp/logicify` folder looks as follows:

```
/tmp/logicify/
├── watchers.py
└── __init__.py
```

File `watchers.py` contains class `SystemTimeWatcher` which implements `WatcherModule`:

```python
class SystemTimeWatcher(WatcherModule):

    def __init__(self, application):
        super().__init__(application)
        self.error_when_midnight = False

    def obtain_state(self, trigger) -> object:
        current_time = datetime.now()
        return current_time

    def serialize_state(self, state: datetime) -> [dict, None]:
        return {
            "time": state.isoformat()
        }

    def do_assertions(self, state: datetime, reporter: ValidationReporter):
        if self.error_when_midnight:
            if state.time() == time(0, 0):
                reporter.error('its_midnight', 'Must be any time except of 00:00')

    PARAMS = (
        ParameterDef('error_when_midnight', validators=(validators.boolean,)),
    )
```

This implementation illustrates how you could create own watchers. While this example shows only watcher module many concepts apply to the Triggers, Outputs and Asserts.

`PARAMS` tuple gives you a way to configure arguments for your module. During application bootstrap parameters from yaml will be sanitized, validated and assigned to the module instance according to definition configured with ParameterDef. 

Method `obtain_state` will be invoked by trigger. You should implement your state gathering logic here. Result could be any object.

`do_assertions` will be invoked on state verification stage. `state` parameter here is what was returned from `obtain_state` and `reporter` instance must be used to report assertion errors if any.

And finally `serialize_state` will be called at the very and before passing result to output. It should convert state object to simple types (dictionaries, lists, primitives).

##  Contribution

Initial dev environment configuration:

1. `virtualenv -p python3 venv`
1. `source ./venv/bin/activate`
1. `pip install -r ./requirements.txt`

## Credits

Dmitry Berezovsky, Logicify (http://logicify.com/)

## License

This plug-in is licensed under GPLv3. This allows you free non-commercial use.
Also note there is no warranty for this free software. Please see the included [LICENSE](LICENSE]) file for details.