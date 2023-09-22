# kovermonitoring
This is my experience of setting up a monitoring, visualizing and alerting environment between my devices.
The main goal was to monitor my ASUS laptop and receive alerts on the start of particular processes.
## What has been done
### Start prometheus
I began with starting Prometheus in a container.
For basic demo set up visit [Configuring Prometheus to monitor itself](https://prometheus.io/docs/prometheus/latest/getting_started/#configuring-prometheus-to-monitor-itself) section of [Prometheus Getting Started](https://prometheus.io/docs/prometheus/latest/getting_started) page and configure ```prometheus.yml_. This configuration will allow you to have a working Prometheus instance which monitors itself.
I simply started with the following command to tweak and click around:
```
$ docker run --name=prometheus -dp 9090:9090 -v ./prometheus.yml:/etc/prometheus/prometheus.yml prom/prometheus
```
#### Adding a custom monitoring target
I decided to monitor my Windows laptop. Both devices share the same wi-fi net, so they should have been discoverable. If not, some windows tweaks are required like allowing the machine to be discovered and connected.
In order to export metrics from a Windows device, I used [windows_exporter](https://github.com/prometheus-community/windows_exporter). The .exe binary should be downloaded and started on the target machine via a command line which allows for some options. My set up is the following:
```
$ .\windows_exporter.exe --collectors.enabled "cpu,cpu_info,cs,logical_disk,logon,memory,os,thermalzone,process"
```
I also added my laptop to targets in prometheus.yml using it's ip inside my home network and the port number on which ```windows_exporter``` posts metrics:
```
scrape_configs:
  - job_name: 'asus'
    static_configs:
    - targets: [192.168.1.74:9182]
```
After that I could see my laptop in the targets list on ```http://localhost:9090/targets```, access metrics and make graphs using PromQL.
### Start Grafana
The next step was to start Grafana locally, add prometheus datasource and visualize metrics on a dashboard.
The following command starts grafana container:
```
$ docker run -p 3000:3000 --name=grafana grafana/grafana-oss
```
I followed [Grafana fundamentals](https://grafana.com/tutorials/grafana-fundamentals/?utm_source=grafana_gettingstarted) tutorial to get a grip on initial settings and then added my own datasource. 
The problem was that since the containers run separately in their own environment (network or whatever), they could not see each other easily and the Prometeus datasource could not be accessed the same way it accessed in the host machine browser – by localhost:9090, neither prometheus:9090. I had to change the address in DataSource setting to ```host.docker.internal:9090```. 
I then managed to visualize data from Prometheus in several panels and set up a Dashboard.
### Start Alertmanager and let it communicate with Prometheus
While exploring Grafana I came up with an idea (thanks to my wife) to push alert notifications in a messenger when a certain process starts (Genshin Impact 🌚)
[Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/) allows to manage the alerts and push them forward to other targets, such as Telegram.
In order for Prometheus to be able to communicate with the Alertmanager, the following configuration has been added to ```prometheus.yml```:
```
alerting:
  alertmanagers:
  - static_configs:
    - targets: 
      - 'alertmanager:9093'
```
The alerting rules themselves are set up as a part of Prometheus config.
There is a rules.yml file which is linked in ```prometheus.yml``` as follows:
```
global:
...
rule_files: 
  - 'rules.yml'
...
```
To compose rules.yml I simply took the first basic rule example from Prometheus [Alerting rules](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/) page and put the files together to prom-configs folder which then mounted on the Prometheus container startup, so in order for all config the files to be mounted to ```etc/prometheus.the``` the command had to be changed as follows:
```
# docker run --name=prometheus -dp 9090:9090 -v ./prom-configs:/etc/prometheus/ prom/prometheus
```
I then started Alertmanager instance:
```
$ docker run --name alertmanager -d -p 127.0.0.1:9093:9093 quay.io/prometheus/alertmanager
```
The problem was that ```alertmanager:9093``` was not discoverable the same way my Grafana was not discoverable since it was running as a separate container. Since all the containers are interdependent and should be in the same network, I decided to use [Docker Compose](https://docs.docker.com/compose/). You can find the resulting configuration in [docker-compose.yml](https://github.com/vladundercover/kovermonitoring/blob/main/docker-compose.yml).
So, 
- Prometheus should have alerting rules and
- Alertmanager should be able to access and display the alerts whenever they appear on [```http://localhost:9093/#/alerts```](http://localhost:9093/#/alerts), and
- Grafana should be up and the Prometheus datasource should be available with its data visible on the Dashboard
### Configure sending notification via Telegram
I referenced [telegram_config](https://prometheus.io/docs/alerting/latest/configuration/#telegram_config) section of Prometheus Alertmanager Configuration page in order to find what fields should be added which are not set by default so that Alertmanager could send the notifications to the desired chat via a Telegram Bot.
#### Create a bot and set up a channel
I consulted [How to configure Prometheus Alertmanager to send alerts to Telegram
](https://velenux.wordpress.com/2022/09/12/how-to-configure-prometheus-alertmanager-to-send-alerts-to-telegram/)
1. Create a bot via [@BotFather](https://t.me/BotFather)
2. Set up a channel
3. Add the bot to the channel
4. Send messages
5. Go to [```https://api.telegram.org/bot<YOUR_BOT_TOKEN>/getUpdates```](https://api.telegram.org/bot<YOUR_BOT_TOKEN>/getUpdates)
6. Check the chat id from the resulting JSON
7. Add ```bot_token``` and ```chat_id``` to ```alertmanager.yml```:
```
receivers:
  - name: 'telegram'
    telegram_configs:
    - bot_token: <YOUR_BOT_TOKEN>
      chat_id: <CHAT_ID>
      send_resolved: false
```
## Conclusion
Now all should be set. Simply start 
```
docker-compose up -d
```
