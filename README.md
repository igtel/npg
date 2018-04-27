NPG - monitoring instruments bundle.
Netdata, Prometeus, Grafana bundle deployment instruction.

System requirements: 
OS: Linux x64 
Docker: ver >= 1.12
HDD: 5Gb
Free TCP ports: 9090, 8080, 3000, 9093, 9115
For RHEL/Centos: Please turn off SELinux or make new rules for docker to write in installation directory.

1. Unpack project archive to selected directory for installation. For example installation directory will be /opt. Current user must have read-write access to installation directory and docker process user also must have access, if you use non-root docker installation:
	cd /opt
	tar xvfz npg.tgz
2. Change directory to project npg:
	cd npg
3. (optional) You can change default password for grafana admin user. Edit file config.grafana and change password value in line:
GF_SECURITY_ADMIN_PASSWORD=admin
4. Project "npg" can be implemented at 2 different modes: 
- as docker stack services (var. A)
- as docker compose services (var. B)
You can use var. A when your docker infrastructure installed in swarm mode and var. B for standalone docker host.
For stack deployment your installation directory nust be shared between swarm hosts.  

Var. А)
Use file "docker-stack.yml" for deploy services. You can preliminarily edit this file and change configurations for some services. Stack includes: 
- prometheus - monitoring service and database,
- alertmanager - notification servise,
- grafana - web dashboard service,
- cadvisor - (optional) monitoring agent for docker containers,
- blackbox - (optional) url monitoring agent
All services will be deployed on local docker host by default with 1 running replica. If you want also monitor all swarm containers it's necessary deploy service cadvisor to all hosts. Uncomment line:
	mode: global
in section "deploy" for service "cadvisor" and remove section "placement".
Also you can uncomment service "netdata" if you want deploy it as docker container instead installing netdata as direct OS service.
Start all stack services by command:
	docker stack deploy --compose-file docker-stack.yml npg
Check stack services status:
	docker service ls | grep npg_
Output example:
ID                  NAME                      MODE                REPLICAS            IMAGE                                PORTS
5re3frstseem        npg_blackbox       replicated          1/1                 prom/blackbox-exporter:latest        *:9115->9115/tcp
9zlgb1ojah4h        npg_prometheus     replicated          1/1                 prom/prometheus:latest               *:9090->9090/tcp
gzp7pn1bo4pn        npg_cadvisor       replicated          1/1                 google/cadvisor:latest               *:8080->8080/tcp
r33g0ayg3c1u        npg_grafana        replicated          1/1                 grafana/grafana:latest               *:3000->3000/tcp
rqbtfvln7qeg        npg_alertmanager   replicated          1/1                 prom/alertmanager:latest             *:9093->9093/tcp
Every service with prefix "npg_" must have at least one running replica (first number in REPLICAS column).

Var. B) 
Use file "docker-deploy.yml" for deploy services. You can preliminarily edit this file and change configurations for some services. Compose includes: 
- prometheus - monitoring service and database,
- alertmanager - notification servise,
- grafana - web dashboard service,
- cadvisor - (optional) monitoring agent for docker containers,
- blackbox - (optional) url monitoring agent
Start all services by command:
	docker-compose up --build -d
Check current container status:
	docker ps 
5 containers with prefix "npg_" must have status 'Up' (column STATUS). 

5. After start services will accessible by url addresses:
- prometheus - http://<docker_ip_address>:9090
- alertmanager - http://<docker_ip_address>:9093
- grafana - http://<docker_ip_address>:3000
- cadvisor - http://<docker_ip_address>:8080
- blackbox - http://<docker_ip_address>:9115
Login into grafana web interface using default administrator account: admin/admin

6. Install Netdata agent to hosts that you want to monitor. You can install netdata from binary packages for Linux OS. Detail instructions for netdata installation is here: https://github.com/firehol/binary-packages

7. Add netdata hosts to prometheus server. For current prometheus configuration netdata agents must be statically added to file prometheus/targets.json. In case of big installation you can use service discovery methods for auto discovery agents (see details https://prometheus.io/docs/prometheus/latest/configuration/configuration/).
Copy example file to targets.json.
	cp prometheus/targets.json prometheus/targets-example.json
Edit file and change value of <netdata_ip_address_X> and <netdata_hostname_X> to your values. You can copy and paste new target section for every netdata host:
  {
    "targets": [ "<netdata_ip_address_x>:19999" ],
    "labels": {
      "env": "prod",
      "job": "<netdata_hostname_x>"
    }
  },
After saving file prometheus reload target file automatically and will be retrive netdata metrics for new hosts. You can check prometheus graph console: http://<docker_ip_address>:9090. Choose some metric with netdata_ prefix and execute it. Console show elements with job name for every netdata host.

8. Grafana already have preconfigured datasource for prometheus database and dashboard 'Netdata Host' with graphs for main OS metric. In this dashboard you can select host (by job name).

9. For email notification please edit change mail setting for your enviroment in file alertmanager/config-example.yml and copy  
	cp alertmanager/config-example.yml alertmanager/config.yml
Restart container npg_alertmanager. Get CONTAINER ID:
	docker ps |grep alertmanager |cut -d" "  -f1
	and stop container 
	docker stop <ID>
In case stack container will restarted automatically, else you need start container by command:	
	docker start <ID>
	
10. For URL monitoring please edit prometheus/prometheus.yml file and uncomment section "job_name: 'blackbox'". In section "targets" add your web servers url. After saving file restart container npg_prometheus.

