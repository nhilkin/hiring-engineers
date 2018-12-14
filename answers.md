# Datadog Solutions Engineer Technical Exercise

## Prerequisites - Setup the environment
In order to avoid any dependency issues when setting up an environment for Datadog, it is recommended to use a fresh Ubuntu 16.04 virtual machine via Vagrant running on VirtualBox. Setting up the virtual machine with Vagrant is preferable due to its fast and easy setup. You can install Virtual box [from their website](https://www.virtualbox.org/) and [install the latest version of Vagrant](https://www.vagrantup.com/downloads.html) for your host machine. Once these are installed, a Vagrant Ubuntu box can be initialized from [Hashicorps' Vagrant Cloud box catalog](https://app.vagrantup.com/boxes/search). 

Once you have chosen a Vagrant box, the first step is to initilize the Vagrant Ubuntu image in your command line with the following command (the ubuntu/xenial64 box was chosen to be initilized):
```
vagrant init ubuntu/xenial64
```
Then the Vagrantfile configuration file should be checked to confirm it contains the following:
```
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/xenial64"
end
```
Once this is done the environment can be started using the command:
```
vagrant up
```
Now that Vagrant is running, the virtual machine can be access via ssh through your command line using the following command:
```
vagrant ssh
```
Since the virtual machine is being accessed, it's time for the Datadog Agent to be installed on the server. This is done by navigating to the Datadog **Integrations** tab and selecting **Agent**. You then select your VM operating system (in this case Ubuntu), and copy and paste the generated Datadog API key into your command line:
![alt text](https://github.com/nhilkin/hiring-engineers/blob/master/dd_agent.PNG)

```
DD_API_KEY=ea6ffb73b20d5aa5c01a588b9a8a5a4e bash -c "$(curl -L https://raw.githubusercontent.com/DataDog/datadog-agent/master/cmd/agent/install_script.sh)"
```
Once this is done you will see the following message on the command line confirming that the agent was installed:
![alt text](https://github.com/nhilkin/hiring-engineers/blob/master/dd_agentinstalled.PNG)

## Collecting Metrics
### Adding Tags
Datadog allows for tags to be added as a way to add dimensions to metrics so that they can be filtered, aggregated, and compared in  visualizations.

Tags can be added to the agent by editing the configuration file ```datadog.yaml``` which is located in the folder ```/etc/datadog-agent```.
The file can be edited using the following command:
```
sudo nano /etc/datadog-agent/datadog.yaml
```

You can uncomment the following portion of the file and input custom tags to be added to the Datadog agent:
```
# Set the host's tags (optional)
 tags:
   - env:vagrant
   - role:database
```
Here are the updated tags on the Host Map:
![alt text](https://github.com/nhilkin/hiring-engineers/blob/master/dd_TAGS.PNG)

### Installing a Database
A database and the respective Datadog integration for that database can be installed on the virtual machine. I decided to install PostgreSQL onto the virtual machine by running the following command:
```
sudo apt-get install postgresql postgresql-contrib
```
Once it's finished intalling, PostgreSQL can be started with the command:
```
sudo -u postgres psql
```

Next the Datadog Agent integration for PostgreSQL can be installed on the **Integrations** tab. Under the **Integrations** tab, select the database you want and follow the onscreen instructions. Click **Generate Password** to set up a read-only Datadog user with access to the server. While running PostgreSQL, run the given commands:
```
create user datadog with password '<GENERATED PASSWORD>';
grant SELECT ON pg_stat_database to datadog;
```

To verify the permissions are correct, run the following command:
```
\q
psql -h localhost -U datadog postgres -c "select * from pg_stat_database LIMIT(1);" && \
echo -e "\e[0;32mPostgres connection - OK\e[0m" || \
echo -e "\e[0;31mCannot connect to Postgres\e[0m"
```
After this, enter the generated password from the Datadog Agent when prompted to do so on the command line. 

Next the configuration file has to be edited to start collecting your PostgreSQL metrics and logs, which is found with the following command:
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
       password: <GENERATED PASSWORD>
       tags:
            - my_metric
            - check_interval
```
Restart the Agent to start sending PostgreSQL metrics to Datadog, using the command:
```
sudo service datadog-agent restart
```
Once the Agent has been restarted, we should run our first integration check to make sure everything has passed:
```
sudo -u dd-agent -- datadog-agent check postgres
```
The output of the command should contain a section similar to the following:
```
Checks
======

  [...]

  postgres
  --------
      - instance #0 [OK]
      - Collected 8 metrics & 0 events
```

In order to create an Agent check that submits a metric named my_metric with a random value between 0 and 1000, we need to create a configuration file for the custom check called my_metric.yaml and a check file called my_metric.py. These files need to be added to the ```conf.d``` directory and the ```checks.d``` directory, using the following commands:
```
cd /etc/datadog-agent/conf.d
/etc/datadog-agent/conf.d$ sudo touch my_metric.yaml
/etc/datadog-agent/conf.d$ cd /etc/datadog-agent/checks.d
/etc/datadog-agent/checks.d$ sudo touch my_metric.py
```
Now that the files are made, the configuration file for the custom check can be edited first. The configuration file can be accessed with the commands:
```
cd /etc/datadog-agent/conf.d
sudo nano /etc/datadog-agent/conf.d/my_metric.yaml
```
The configuration file can then be edited to include the following:
```
init_config:

instances: [{}]
```

Then the check file must be edited in order to run the custom metric. The check file can be accessed with the commands:
```
cd /etc/datadog-agent/checks.d
sudo nano /etc/datadog-agent/checks.d/my_metric.py
```

The Python file can be edited to include the following script that will provide the custom metric. In the script we need to import the ```AgentCheck``` class from the checks module. Since the metric needs to return a random value between 0 and 1000, we need to import ```randint``` from the random module. The method ```self.gauge()``` is used to send a guage of a random number between 0 and 1000 for my_metric every time it is called:
```
from checks import AgentCheck
from random import randint
from datadog_checks.checks import AgentCheck

__version__ = "1.0.0"

class MyMetric(AgentCheck):
    def check(self, instance):
        self.gauge('my_metric', randint(0, 1000))
```

Once the check file is edited with the script, it can be verified that the check works by running the command:
```
sudo -u dd-agent -- datadog-agent check my_metric
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
 
Datadog provides an API to integrate with any other external system and control its functionality. The first step to do this is to create an API key. This can be done by navigating to the **Integrations** tab and selecting **APIs**. Here you will fine your API key.

The following script will use this API key and will act as a timeboard containing the new metric ```my_metric``` scoped over the host, anomoly graph of cpu usage by the system from the host, and a custom metric with the rollup function applied to sum up all the points for the past hour into one bucket scoped over my host.
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

You can access your new timeboard by going to the **Dashboard** tab and selecting **Dashboard List**. The timeboard generated by the above script will look like the following:
![alt text](https://github.com/nhilkin/hiring-engineers/blob/master/dd_timeboard.PNG)
 
 * **Bonus Question**: What is the Anomaly graph displaying?
 The Anomaly graph represents anything deviating from average values based on past values. 
 
## Monitoring Data
Create a new Metric Monitor that watches the average of your custom metric (my_metric) and will alert if it’s above the following values over the past 5 minutes:
* Warning threshold of 500
* Alerting threshold of 800
* And also ensure that it will notify you if there is No Data for this query over the past 10m.

The following Metric Monitor accomplishes the purpose listed above of watching the average of ```my_metric``` and alerts if it's above the warning threshold of 500 and alerting threshold of 800 over the past 5 minutes. To set this up, navitage to the **Monitors** tab on Datadog and select the **New Monitor** option. From there, select your monitor type, which will be **Metric**. You can then begin to set up your monitor, first you can leave the **Choose the detection method** as Threshold Alert, and under **Define the metric** enter the custom metric my_metric in the ```Select a metric``` box and enter your tags in the ```avg by``` box. 
You can then fill in the necessary information in the **Set alert conditions** portion as shown to create the Metric Monitor:
![alt text](https://github.com/nhilkin/hiring-engineers/blob/master/dd_metric_monitor_config.png)

Please configure the monitor’s message so that it will:
* Send you an email whenever the monitor triggers.
* Create different messages based on whether the monitor is in an Alert, Warning, or No Data state.
* Include the metric value that caused the monitor to trigger and host ip when the Monitor triggers an Alert state.
* When this monitor sends you an email notification, take a screenshot of the email that it sends you.

This monitor will send a message whenever it's triggered. The message it sends can be customized and is based on whether the monitor is in an Alert, Warning, or No Data state. This message includes the metric value that caused the monitor to be triggered and includes the host ip when the Monitor triggers an Alert state. The message can be entered in the **Say what's happening** portion. Team members can also be added to receive the message in the **Notify your team** portion, and the email addresses of those who are added will show up in the message.
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
![alt text](https://github.com/nhilkin/hiring-engineers/blob/master/dd_notify.PNG)

* **Bonus Question**: Set up two scheduled downtimes for this monitor:
* One that silences it from 7pm to 9am daily on M-F,
* And one that silences it all day on Sat-Sun.
* Make sure that your email is notified when you schedule the downtime and take a screenshot of that notification.

Datadog allows for flexibility in its monitoring and provides the option to schedule downtime and silence alarms for times when they are not necessary, like the weekends. This can be done by navigating to the top of the current **Select a metric to monitor** and selecting the **Manage Downtime** tab. From there select the orange **Schedule Downtime** button.

Select a recurring schedule and enter the following to set up downtime from 7pm to 9am daily on M-F:
![alt text](https://github.com/nhilkin/hiring-engineers/blob/master/dd_5days.PNG)

Then enter the following to set up downtime all day on Sat-Sun:
![alt text](https://github.com/nhilkin/hiring-engineers/blob/master/dd_weekends.PNG)

## Collecting APM Data
The first step to start collecting APM data is to enable APM tracing in the main Agent Configuration yaml file, and using the command ```sudo nano /opt/datadog-agent/etc/datadog.yaml``` edit the following:
```
apm_config:
  enabled: true
  env: none
```
Next we must install ddtrace and flask with the commands, since we will be using the ddtrace method and not the manual insert method:
```
pip install ddtrace
pip install flask
```
Then create the app.py file and paste the given Flask code:
```
sudo nano my_app.py
```
Now we can run the APM python file:
```
ddtrace-run python app.py
```
Here is what the dashboard with both APM and Infrastructure Metrics will look like:
![alt text](https://github.com/nhilkin/hiring-engineers/blob/master/dd_both.png)

* **Bonus Question**: What is the difference between a Service and a Resource?
A Service is a set of processes that provide a function. A Resource is a specific action of a service. 

## Final Question: Is there anything creative you would use Datadog for?

There are many ways Datadog could be used to monitor specific data, such as:
* Monitoring weather condition data and providing alerts when certain temperature, wind levels, pressure levels, or humidity exceed a certain point.  
* Stock market data could be monitored, with an alert being set for certain stocks in a person's portfolio to notify when they exceed or fall behind a certain price point which would notify to buy or sell. 
* Road traffic conditions could be monitored and Datadog could analyze patterns in the flow of traffic, sending alerts when traffic conditions are above or below average. 
