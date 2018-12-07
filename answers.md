# Datadog Solutions Engineer Technical Exercise

## Prerequisites - Setup the environment
In order to avoid any dependency issues mentioned in the instructions, I decided to use a fresh Ubuntu 16.04 VM via Vagrant running on VirtualBox. I installed Virtual box [from their website](https://www.virtualbox.org/) and initialized a Vagrant Ubuntu box from [Hashicorps' Vagrant Cloud box catalog](https://app.vagrantup.com/boxes/search). 

The first step was to initilize the Vagrant Ubuntu image with the following command:
```
vagrant init ubuntu/xenial64
```
Then the Vagrantfile configuration file had to be checked to confirm it contained the following:
```
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/xenial64"
end
```
Once this is done the environment can be started using the command:
```
vagrant up
```
Now that Vagrant is running, the virtual machine can be access via ssh using the following command:
```
vagrant ssh
```
Since the virtual machine is being accessed, the Datadog Agent can then be installed on the server using the command:
```
DD_API_KEY=ea6ffb73b20d5aa5c01a588b9a8a5a4e bash -c "$(curl -L https://raw.githubusercontent.com/DataDog/datadog-agent/master/cmd/agent/install_script.sh)"
```

## Collecting Metrics
### Adding Tags
Tags can be added to the agent by editing the configuration file ```datadog.yaml``` which is located in the folder ```/etc/datadog-agent```.

The file can be edited using the following command:
```
sudo nano /etc/datadog-agent/datadog.yaml
```

You can uncomment the following portion and input custom tags to be added to the Datadog agent:
```
# Set the host's tags (optional)
 tags:
   - env:vagrant
   - role:database
```
Here are the updated tags on the Host Map:

![alt text](https://github.com/nhilkin/hiring-engineers/blob/master/dd_TAGS.PNG)

### Installing a Database
I decided to install PostgreSQL onto the VM by running the following command:
```
sudo apt install postgresql postgresql-contrib
```
And PostgreSQL can be started with the command:
```
sudo -u postgres psql
```

Next the Datadog Agent integration for PostgreSQL can be installed on the **Integrations** tab.
The Datadog Agent will generate a password to set up a read only Datadog user with access to the server.
This was done with the commands:
```
create user datadog with password 'luaYcv8gVpcYF80IzpygyExR';
grant SELECT ON pg_stat_database to datadog;
```

```
\q
psql -h localhost -U datadog postgres -c "select * from pg_stat_database LIMIT(1);" && \
echo -e "\e[0;32mPostgres connection - OK\e[0m" || \
echo -e "\e[0;31mCannot connect to Postgres\e[0m"
```
After this I entered the generated password from the Datadog Agent

Next the configuration file had to be edited, which is found with the following command:
```
sudo nano /etc/datadog-agent/conf.d/postgres.yaml
```
The file is then edited with the following:
```
init_config:

instances:
   -   host: localhost
       port: 5432
       username: datadog
       password: luaYcv8gVpcYF80IzpygyExR
       tags:
            - my_metric
            - check_interval
```

In order to create an Agent check that submits a metric named my_metric with a random value between 0 and 1000, we need to create a configuration file for the custom check called my_metric.yaml:
```
init_config:

instances: [{}]
```
Then a Python file to run the custom metric:
```
from checks import AgentCheck
from random import randint
from datadog_checks.checks import AgentCheck

__version__ = "1.0.0"

class MyMetric(AgentCheck):
    def check(self, instance):
        self.gauge('my_metric', randint(0, 1000))
```
* **Bonus Question**: Can you change the collection interval without modifying the Python check file you created?
The collection interval can be changed to only submit the metric once every 45 seconds. This can be done without modifying the Python check file and instead modifying your configuration file with the following:
```
instances:
    - min_collection_interval: 45
```

## Visualizing Data
Utilize the Datadog API to create a Timeboard that contains:
 * Your custom metric scoped over your host.
 * Any metric from the Integration on your Database with the anomaly function applied.
 * Your custom metric with the rollup function applied to sum up all the points for the past hour into one bucket
 
 ```
from datadog import initialize, api

options = {'api_key': '<YOUR_API_KEY>',
           'app_key': '<YOUR_APP_KEY>'}

initialize(**options)

title = "Custom Timeboard"
description = "API dashboard"
graphs = [{
    "definition": {
        "events": [],
        "requests": [
            {"q": "my_metric{host:vagrant}"},
        ],
        "viz": "timeseries"
    },
    "title": "my_metric values"
},
{
    "definition": {
        "events": [],
        "requests": [
            {"q": "anomalies(system.cpu.system{*}, 'basic', 3)"},
        ],
        "viz": "timeseries"
    },
    "title": "anomolies of system cpu"
},
{
    "definition": {
        "events": [],
        "requests": [
            {"q": "my_metric{host:vagrant}.rollup(sum, 3600)"}
        ],
        "viz": "timeseries"
    },
    "title": "Roll up of my metric over 1 hour"
},
{
          "definition": {
          "events": [],
          "requests": [
                       {"q": "my_metric{host:vagrant}.rollup(sum, 3600)"}
                       ],
          "viz": "query_value"
          },
          "title": "Roll up of my metric over 1 hour as query value"
}
  ]

read_only = True
response = api.Timeboard.create(title=title,
                      description=description,
                      graphs=graphs,
                      read_only=read_only)
```

Timeboard Generated:

![alt text](https://github.com/nhilkin/hiring-engineers/blob/master/dd_timeboard.PNG)
 
 * **Bonus Question**: What is the Anomaly graph displaying?
 The Anomaly graph represents the anything deviating from average values based on past values. 
 
 ## Monitoring Data
 
Create a new Metric Monitor that watches the average of your custom metric (my_metric) and will alert if it’s above the following values over the past 5 minutes:
* Warning threshold of 500
* Alerting threshold of 800
* And also ensure that it will notify you if there is No Data for this query over the past 10m.

![alt text](https://github.com/nhilkin/hiring-engineers/blob/master/dd_metric_monitor_config.png)

Please configure the monitor’s message so that it will:
* Send you an email whenever the monitor triggers.
* Create different messages based on whether the monitor is in an Alert, Warning, or No Data state.
* Include the metric value that caused the monitor to trigger and host ip when the Monitor triggers an Alert state.
* When this monitor sends you an email notification, take a screenshot of the email that it sends you.

```
{{#is_alert}}
## Alert!
My_Metric has exceeded 800 in 5 minutes. Value = {{value}} 
IP Address = {{host.ip}} 
{{/is_alert}}

{{#is_warning}}
## Warning
My_Metric has exceeded over 500 in 5 minutes
{{/is_warning}}

{{#is_no_data}} There is no data for the last 10 minutes {{/is_no_data_recovery}}

nhilkin@nyit.edu
```

* **Bonus Question**: Set up two scheduled downtimes for this monitor:
* One that silences it from 7pm to 9am daily on M-F,
* And one that silences it all day on Sat-Sun.
* Make sure that your email is notified when you schedule the downtime and take a screenshot of that notification.

![alt text](https://github.com/nhilkin/hiring-engineers/blob/master/dd_5days.PNG)

![alt text](https://github.com/nhilkin/hiring-engineers/blob/master/dd_weekends2days.PNG)

## Collecting APM Data
The following command can be used to run the given APM solution:
```
ddtrace-run python app.py
```

* **Bonus Question**: What is the difference between a Service and a Resource?
A Service is a set of processes that provide a function. A Resource is a specific action of a service. 

Dashboard with both APM and Infrastructure Metrics:

![alt text](https://github.com/nhilkin/hiring-engineers/blob/master/dd_both.png)

## Final Question: Is there anything creative you would use Datadog for?

There are many ways Datadog could be used to monitor specific data, such as:
* Monitoring weather condition data and providing alerts when certain temperature, wind levels, pressure levels, or humidity exceed a certain point.  
* Stock market data could be monitored, with an alert being set for certain stocks in a person's portfolio to notify when they exceed or fall behind a certain price point which would notify to buy or sell. 
* Road traffic conditions could be monitored and Datadog could analyze patterns in the flow of traffic, sending alerts when traffic conditions are above or below average. 
