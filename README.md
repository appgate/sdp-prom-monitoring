# AppGate monitoring
This is the docker compose stack for a quick setup monitoring with AppGate SDP. Instrumentation is done via the AppGate API, and all the metrics are established through a translation of gathered information. 
The metrics are a subset of the AppGate appliance's on-board prometheus exporter (snmp-exporter). Despite the simplicity of the information, we have found that these metrics offer a good basis to build a system state view and actionable events.

>Logging, log message, request and message tracing is not part of this work. 

This is currently been worked on.

## Overview

### Achitecture
Docker stack deployed on docker swarm
* Reverse proxy as entry point and certificate resolver
* Grafana with prometheus & Co, time series database
* AppGate SDP remote prometheus exporter connecting to one or many AppGate SDP collectives


![architecture](./agprom-architecture.png)



### Technology used:
* [treaefik](https://docs.traefik.io/), edge router (reverse proxy)
* [conkolla](https://github.com/appgate/conkolla/), prometheus to appgate connector and prom exporter
* [prometheus](https://prometheus.io/), et.al, scraper, database
* [grafana](https://grafana.com/), visualization and alerting front-end

### Documents
``` 
├── docker-appgate-monitor-stack.yml
├── alertmanager
│   └── config.yml
├── conkolla
│   └── connections.yml
├── grafana
│   ├── config.monitoring
│   └── provisioning
│       ├── dashboards
│       │   ├── AppGate\ Appliances.json
│       │   ├── AppGate\ Dashoard.json
│       │   ├── AppGate\ Details.json
│       │   ├── dashboard.yml
│       │   ├── Docker\ Conkolla\ Monitoring.json
│       │   └── Docker\ Prometheus\ Monitoring.json
│       └── datasources
│           └── datasource.yml
├── htapass
├── letsencrypt
├── prometheus
│   ├── alert.rules
│   ├── appgate.rules
│   ├── conkolla.rules
│   └── prometheus.yml
```




## Preparations:
### DNS
The subsystems will be accessible from the host's IP address. Every subsystem is routed via a host name. They are defined in the docker stack file, and by default require the bellow host names. You will later match the DOMAIN name for the deployment with a environment variable. For now, you will need prepare on a domain owned by you the following records:
* Four A records (or alias) pointing to the host IP:
	* agtraefik.`${DOMAIN}`
	* agconkolla.`{DOMAIN}`
	* agprometheus.`${DOMAIN}`
	* aggrafana.`{DOMAIN}`


### TLS certs
Traefik is taking care of the certificate resolving for the subsystems host certs. You will need simply provide an email address which will be used by traefik for let's encrypt account. No email conversations will be required. The email will be set, as described below, before deployment in a environment variable.

### Network requirements
Incoming traffic needs to reach to the host on:
* `80/TCP` for let's encrypt (from any)
* `443/TCP` for entry-point for traefik (from any/restricted)

### Host
You can deploy the stack to an existing swarm. In this guide we setup a dedicated host and docker swarm. The host specs are as the following, assuming collective(s) with up to 60 appliances and using 90days retention of the time series (tsdb). 
The heavier queries prometheus/grafana will perform, the more memory and CPU you will require. For now, the following specs are proven to work fine:
* AWS: t3.standard, EC2 Amazon Linux Type 2
* Azure: Centos 7
* Disk, SSD: 40GB


#### Prepare EC2 Amazon linux Type 2
``` 
sudo amazon-linux-extras install -y docker
sudo systemctl enable docker
sudo systemctl start docker
sudo docker swarm init
```
Usermod
```
sudo usermod -a -G docker ec2-user #sudo gpasswd -a ec2-user docker
sudo setfacl -m user:ec2-user:rw /var/run/docker.sock
```
Git
```
sudo yum -y  install git
``` 

htpasswd
``` 
sudo yum -y install httpd-tools
```

#### Prepare CentOS7 (Azure)
Installing [docker community engine](https://docs.docker.com/install/linux/docker-ce/centos/#install-using-the-repository). Boils down to:

```
sudo yum remove docker \
		docker-client \
                docker-client-latest \
                docker-common \
                docker-latest \
                docker-latest-logrotate \
                docker-logrotate \
                docker-engine

sudo yum install -y yum-utils \
		device-mapper-persistent-data \
		lvm2

sudo yum-config-manager \
		--add-repo \
		https://download.docker.com/linux/centos/docker-ce.repo

sudo yum install -y docker-ce docker-ce-cli containerd.io
sudo systemctl start docker
sudo systemctl enable docker

sudo docker swarm init

```
User mod
```
export USER=<user as docker operator>
sudo usermod -a -G docker ${USER}
sudo setfacl -m user:${USER}:rw /var/run/docker.sock
```
Git
```
sudo yum -y  install git
```
Htpasswd
``` 
yum -y install httpd-tools
```

SELinux changes:
```
sudo semanage port -a -t http_port_t  -p tcp 443 # websecure traefik entrypoint
sudo semanage port -a -t http_port_t  -p tcp 80  # lets encrypt traefik cert resolver
sudo setsebool httpd_can_network_connect 1 -P # allow conns (outgoing docker), add permanently
``` 

## Configure

```shell
$ git clone https://github.com/appgate/appgate-prom-monitoring.git
$ cd agmon
```

### Grafana
Adjust any settings if required in `grafana/config.monitoring`:
* The initial password can be replaced for user `admin`, but must match same one in `htapass/grafana_users`. You can later change and add users through the grafana UI, and you always need to add them in the `grafana_users` as well.
* Set the host name for grafana: `GF_SERVER_ROOT_URL=https://grafana.${DOMAIN}`  


#### Dashboards
We provision a set of dashboards during deployment from the directory: `grafana/provisioning/dashboards`: 
* AppGate Dashboard
* AppGate Appliances info
* AppGate Details on users, sessions, bandwidth etc
* Conkolla monitoring
* Docker resource monitoring

The provisioning configuration is defined in the `dashboard.json` file, defaults are: 

``` 
apiVersion: 1
providers:
- name: 'Prometheus'
  orgId: 1
  folder: ''
  type: file
  disableDeletion: false
  editable: true
  allowUiUpdates: true
  options:
    path: /etc/grafana/provisioning/dashboards
```

[Read grafana documentation regarding use of provisioned dashboard](https://grafana.com/docs/grafana/latest/administration/provisioning/#dashboards)

### Conkolla setup
The conkolla can either connect automatically via:
1. Connection definition from file 
2. Manually through the UI on `agconkolla.${DOMAIN}`
3. By an API call/operator process

Conkolla support [AWS KMS](https://github.com/appgate/conkolla#kms-external-encryptiondecryption-provider) and can be given a base64 encoded AWS KMS blob with the additional parameters to decrytp/encrypt. You must use `auto token renewel` such conkolla always can fetch the data from the controller. Please read in the [conkolla doc](https://github.com/appgate/conkolla#prometheus-metrics) how to setup a conkolla user on the AppGate controller.

In this example we are using a connections.yml file with a plain text password for simplicity. An example will follow on a separate page to set up with an EC2 instance, IAM role and KMS. Now, the configuration file `conkolla/connections.yml` can have an example setup:
```
version: 1
connections:
- controllerURL: skip.packnot.com 
  username: monitorUser
  password: plaintext
  skipVerifySSL: true
  autoTokenRenewal: true
  promCollector: true # enable prometheus collectors
  promTargetName: skip.packnot.com # the name under which this collective can be scraped
```


### Prometheus
In prometheus/prometheus.yml, define job(s) according to the conkolla setup. Define one job as below for every target to be scraped from conkolla. The Configuration needs only 2 changes as indicated. The other settings must be kept as specified. 

```
- job_name: '<job name>' # example skip.packnot.com
     dns_sd_configs:
     - names:
       - 'tasks.conkolla'
       type: 'A'
       port: 4433

     honor_timestamps: true
     honor_labels: true
     metrics_path: /metrics
     scheme: http
     params:
       target: [<target name>] # example skip.packnot.com
     relabel_configs:
       - source_labels: [__param_target]
         target_label: target
```

* create htpasswd users: 
   * `htpasswd -c htapass/grafana_users admin`
   * `htpasswd -c htapass/prometheus_users admin`
   * `htpasswd -c htapass/conkolla_users admin`
   * `htpasswd -c htapass/traefik_users admin`

If you want to use a single htpasswd file, then create symlinks to it with the path and name as listed above.



### Alert rules and alert manager
There are just a few alerts defined in `prometheus/alert.rules`. The alerts are to notify about the system/instances/nodes and are not related to AppGate. Alerts are shown in a dedicated dashboard in grafana, deployed when provisioned. 

You can add more alerts, and in the `alertmanager/config.yml` you can specify external receivers for alerts, example using slack:
```  
route:
receiver: 'slack':
receivers:
  - name: 'slack'
    slack_configs:
      - send_resolved: true
        username: '<username>'
        channel: '#<channel-name>'
        api_url: '<incomming-webhook-url>'%
``` 

## Deploy

``` 
export DOMAIN=<domain>
export LE_EMAILADDRESS=<emaoiladdress>
``` 

``` 
HOSTNAME=$(hostname) docker stack deploy -c docker-appgate-monitor-stack.yml agmon
``` 



