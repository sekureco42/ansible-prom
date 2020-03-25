# Purpose of this project

I just wanted to have a all in one ansible playbook to setup a complete monitoring and logging platform based on modern tools like:
* Docker - container environment to run all following components, https://www.docker.com/
* Prometheus - Data collector, https://prometheus.io/docs/introduction/overview/
* Various Exporter:
  * node_exporter - delivers system metrics of linux host, https://github.com/prometheus/node_exporter
  * snmp-generator - creates snmp_exporter config from MIBS, https://github.com/prometheus/snmp_exporter/tree/master/generator
  * snmp_exporter - collects old world monitoring data, https://github.com/prometheus/snmp_exporter
  * blackbox_exporter - implements icmp und http(s) checks, https://github.com/prometheus/blackbox_exporter
* Alertmaanger - collect and send alerts according rules, https://github.com/prometheus/alertmanager
* Loki - collects logs, https://github.com/grafana/loki/blob/master/docs/README.md
* promtail - receives logs from old world logging devices by syslog, https://github.com/grafana/loki/tree/master/docs/clients/promtail
* Grafana - front end for the components above, https://github.com/grafana/grafana
* Watchtower - keep your containers automatically up to date, https://github.com/containrrr/watchtower

Another goal was to have a complete example of the Prometheus/Loki stack to play around. 

Note: **this is not meant for productive deployments!!** All containers are accessible from everywhere without authentication. This is intentionally for debugging and testing purposes.

# Prerequisits
1. You need a host which is capable to run ansible
2. You need a target host based on Ubuntu 18.04 (only platform I've tested)
3. Your target host must have python3 installed

# How do I use it?
1. Clone this repo
```bash
git clone https://github.com/<path>/ansible-prom.git
cd ansible-prom
```
2. Edit `inventories\staging\hosts` and modify entry according your target host
3. Edit `inventories\staging\group_vars\all` and adapt variables to your needs (should be self explained):
   * Adapt node_exporter URL if you want to use a newer one (if it is available)
   * Adapt Slack config, https://api.slack.com/messaging/webhooks (also working with Mattermost, https://docs.mattermost.com/developer/webhooks-incoming.html) 
   * Adapt `ping_targets` to include all systems which should monitored by ping (icmp)
   * Adapt `http_urls` to include all systems which should monitored by http(s)
   * Adapt `node_exporter` to include all node exporters which should be scraped by Prometheus.
     Add here your target host and do not forget to include service port 9100. Note that due the architecture of this installation this host must be resolvable by DNS (host entry on target sytem with interface IP works too - but not with any kind of localhost!!!)
   * `snmp_*`: only Fortigate and Synology are supported with SNMPv3, the generic SNMP is using SNMPv1. If you want to change this you have to adapt the snmp generator, details see file `roles\snmp-generator\files\generator.j2` and `roles\snmp-generator\files\generator.yml.sample`
4. Check if you are able to reach your target host with ansible by issueing following command:
```bash
ansible -i inventories/staging all -m ping
```
5. If you got a successful response from your target host (you should see "PONG") go ahead. If not try to fix your ansible connection to target system (right IP? right username? right ssh key?)
6. Finally you are ready to run the playbook:
```bash
ansible-playbook -b -i inventories/staging site.yml
```

# Notes/Remarks
## Loki
Loki is running in monolithic mode so it is using local file system to save index and chunks. Unfortunatly in this mode Loki creates MANY MANY files so you have to tune your system. My advice is to use a dedicated partition where you tune your file system accordingly (especially inode count - can be done only when formatting partition).
* 13 GByte log data created  over 6'000'000 files
* According to issue tracker it makes also sense to adjust some flags (Reference: https://github.com/grafana/loki/issues/1502):
```bash
  sudo tune2fs -O "^dir_index" <partition>
```
* Adjust retention settings:
```yaml
    table_manager:
      retention_deletes_enabled: true
      retention_period: 24h
```
  This setting sets retention to 24h - for serious log data collection this value has to be increased.

## promtail
I've added a simple example to pre process logs to get additional labels in grafana. Have a look at the config file of promtail `roles\promtail\files\promtail.yml`