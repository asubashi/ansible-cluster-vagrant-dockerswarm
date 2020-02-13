# Hands-On: Provision a Docker Swarm Cluster with Vagrant and Ansible

### Automatically provision a Docker Swarm cluster composed of two masters and two workers

## Preconditions:

* Linux host with at least 8GB RAM
* VirtualBox - https://www.virtualbox.org/
* Vagrant - https://www.vagrantup.com/

Here will be deploy the Infrastructure as Code (IaC) and we will cover the Docker native container orchestrator, namely **Swarm**. 
The IaC topic will be explored with **Vagrant**, an automation layer above a hypervisor - in our case, VirtualBox. 

Vagrant leverages a declarative configuration file which describes all your software requirements, packages, operating system configuration, users, and more. With this tool, we will be able to provision four Linux nodes, based on a standard template image, with the proper properties in terms of hardware requirement and network configuration.

Once the nodes are provisioned, we need to use **Ansible** for their configuration, turning them from plain Linux nodes to Docker hosts, and ultimately members of a multi-master Docker Swarm cluster. 

In order to do that, we will install Ansible master on one node, which will act as a configuration master for the others.

## Code Overview

Through the Vagrant script (_Vagrantfile_) we provision four identical Linux machines (Ubuntu 16.04 LTS) on the same subnet:

```ruby
nodes = [
  { :hostname => 'swarm-master-1', :ip => '192.168.77.10', :ram => 2048, :cpus => 1 },
  { :hostname => 'swarm-master-2', :ip => '192.168.77.11', :ram => 2048, :cpus => 1 },
  { :hostname => 'swarm-worker-3', :ip => '192.168.77.14', :ram => 2048, :cpus => 1 },
  { :hostname => 'swarm-worker-1', :ip => '192.168.77.12', :ram => 2048, :cpus => 1 },
  { :hostname => 'swarm-worker-2', :ip => '192.168.77.13', :ram => 2048, :cpus => 1 }
]

Vagrant.configure("2") do |config|
  # Always use Vagrant's insecure key
  config.ssh.insert_key = false
  # Forward ssh agent to easily ssh into the different machines
  config.ssh.forward_agent = true
  # Provision nodes
  nodes.each do |node|
    config.vm.define node[:hostname] do |nodeconfig|
      nodeconfig.vm.box = "bento/ubuntu-18.04";
      nodeconfig.vm.hostname = node[:hostname] + ".box"
      nodeconfig.vm.network :private_network, ip: node[:ip]
      memory = node[:ram] ? node[:ram] : 2048;
      cpus = node[:cpus] ? node[:cpus] : 1;
      nodeconfig.vm.provider :virtualbox do |vb|
        vb.customize [
          "modifyvm", :id,
          "--memory", memory.to_s,
          "--cpus", cpus.to_s
        ]
      end
    end
  end
end
```

A Bash script is then injected in node _swarm-worker-2_ to install Ansible via _pip_:

```sh
$ sudo apt-get install -y python-pip sshpass
$ sudo -H pip install --upgrade pip
$ sudo -H pip install ansible
```

The cluster configuration is carried out by means of three Ansible playbooks. The first one, _cluster.yaml_, installs Docker CE on all four hosts, regardless of their target role:

```yaml
---
- hosts: all
  become: true
  vars_files:
  - vars.yml
  strategy: free

  tasks:
    - name: Add the docker signing key
      apt_key: url=https://download.docker.com/linux/ubuntu/gpg state=present

    - name: Add the docker apt repo
      apt_repository: repo='deb [arch=amd64] https://download.docker.com/linux/ubuntu xenial stable' state=present

    - name: Install packages
      apt:
        name: "{{ PACKAGES }}"
        state: present
        update_cache: true
        force: yes

    - name: Add user vagrant to the docker group
      user:
        name: vagrant
        groups: docker
        append: yes
```

The second playbook, _master.yaml_, initializes the Docker Swarm, and configures the host named _swarm-master-1_ as the (leader) Swarm Manager.

```yaml
---
- hosts: swarm-master-1
  become: true

  tasks:
    - name: Initialize the cluster
      shell: sudo apt update; sudo apt install -y docker; sudo systemctl start docker; sudo curl -sSL https://get.docker.com/ |  sh; sudo docker swarm init --advertise-addr 192.168.77.10 >> cluster_initialized.txt
      args:
        chdir: $HOME
        creates: cluster_initialized.txt

      args:
        chdir: $HOME
        creates: cluster_initialized.txt
```

The last playbook, _join.yaml_, sets up the actual Docker Swarm as composed of four hosts - three masters (so that the manager role is redunded) and two workers. In order to achieve that, two _join-tokens_ (one to join the cluster as a manager, and one to join the cluster as a worker) are generated on the host which initialized the swarm, and automatically passed to the remaining three hosts.

```yaml
---
- hosts: swarm-master-1
  become: true
  gather_facts: false

  tasks:
    - name: Get master join command
      shell: docker swarm join-token manager
      register: master_join_command_raw

    - name: Set master join command
      set_fact:
        master_join_command: "{{ master_join_command_raw.stdout_lines[2] }}"

    - name: Get worker join command
      shell: docker swarm join-token worker
      register: worker_join_command_raw

    - name: Set worker join command
      set_fact:
        worker_join_command: "{{ worker_join_command_raw.stdout_lines[2] }}"

- hosts: swarm-master-2
  become: true

  tasks:
    - name: Master joins cluster
      shell: "{{ hostvars['swarm-master-1'].master_join_command }} >> node_joined.txt"
      args:
        chdir: $HOME
        creates: node_joined.txt
        
        
- hosts: swarm-master-3
  become: true

  tasks:
    - name: Master joins cluster
      shell: "{{ hostvars['swarm-master-1'].master_join_command }} >> node_joined.txt"
      args:
        chdir: $HOME
        creates: node_joined.txt        

- hosts: workers
  become: true

  tasks:
    - name: Workers join cluster
      shell: "{{ hostvars['swarm-master-1'].worker_join_command }} >> node_joined.txt"
      args:
        chdir: $HOME
        creates: node_joined.txt
```

## Step 1: Pull the Vagrant Box

We use Bento's Ubuntu 18.04 Vagrant Box - https://app.vagrantup.com/bento/boxes/ubuntu-18.04 - as the image template for all hosts. Enter a shell and type:

```sh
$ vagrant box add bento/ubuntu-18.04 --provider virtualbox
```
to pull the requested image from the Vagrant Cloud catalog.

## Step 2: Run the Vagrantfile

Type the following commands:

```sh
$ cd Vagrant
$ vagrant up
```

This command executes the _Vagrantfile_, which in turn - as esplained above - installs Ansible and runs the playbooks. All this will take a few minutes. Eventually, our Docker Swarm Cluster will be set up and ready to be used.

```sh
root@aida-ubuntu:~/vagrant-ansible-dockerswarm-2/Vagrant# vagrant status
Current machine states:
swarm-master-1            running (virtualbox)
swarm-master-2            running (virtualbox)
swarm-master-3            running (virtualbox)
swarm-worker-1            running (virtualbox)
swarm-worker-2            running (virtualbox)
```

## Step 3: Verify the Swarm Cluster

Log in the first node using Vagrant's built-in SSH access as the _vagrant_ user:

```sh
$ vagrant ssh swarm-master-1
```

On this node, which acts as a Swarm Manager, all nodes are now shown in the inventory:

```sh
root@swarm-master-1:~# docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION                             
xed757vbzsmx89csyctahyqr5 *   swarm-master-1      Ready               Active              Leader              19.03.5
m34639hoebsg4q5lw8qtsx8kh     swarm-master-2      Ready               Active              Reachable           19.03.5
wyi9wvjgdytdf872c5k0clmqo     swarm-master-3      Ready               Active              Reachable           19.03.5

```

The swarm manages individual containers on the nodes for us. Now we work on a higher level, with a new concept called **_Services_**. A service is an abstraction, that says: “I want to run this type of container, but I’m not going to start and manage the individual instances – I want the swarm to do that for me”.

So let’s create a very simple service.

## Step 4: Start a Sample Service

Deploy the application which expose the port 8080. The last file added is the **docker-stack.yml**, this one contains a lot of options that illustrates the last features of Docker 1.13 and that enables to deploy the application as a stack (a group of services). I will get this file and run it against  Swarm using the new option of the docker stack deploy command.

```sh
curl -O https://raw.githubusercontent.com/docker/example-voting-app/master/docker-stack.yml
```

The content of the file is the following one:

```sh
version: "3"
services:
  redis:
    image: redis:alpine
    ports:
      - "6379"
    networks:
      - frontend
    deploy:
      replicas: 2
      update_config:
        parallelism: 2
        delay: 10s
      restart_policy:
        condition: on-failure
  db:
    image: postgres:9.4
    volumes:
      - db-data:/var/lib/postgresql/data
    networks:
      - backend
    deploy:
      placement:
        constraints: [node.role == manager]
  vote:
    image: dockersamples/examplevotingapp_vote:before
    ports:
      - 5000:80
    networks:
      - frontend
    depends_on:
      - redis
    deploy:
      replicas: 2
      update_config:
        parallelism: 2
      restart_policy:
        condition: on-failure
  result:
    image: dockersamples/examplevotingapp_result:before
    ports:
      - 5001:80
    networks:
      - backend
    depends_on:
      - db
    deploy:
      replicas: 1
      update_config:
        parallelism: 2
        delay: 10s
      restart_policy:
        condition: on-failure
  worker:
    image: dockersamples/examplevotingapp_worker
    networks:
      - frontend
      - backend
    deploy:
      mode: replicated
      replicas: 1
      labels: [APP=VOTING]
      restart_policy:
        condition: on-failure
        delay: 10s
        max_attempts: 3
        window: 120s
      placement:
        constraints: [node.role == manager]
  visualizer:
    image: dockersamples/visualizer:stable
    ports:
      - "8080:8080"
    stop_grace_period: 1m30s
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
    deploy:
      placement:
        constraints: [node.role == manager]
networks:
  frontend:
  backend:
volumes:
  db-data:
  ```
  
 ## Deployment of the stack
 
I  will deploy the stack on a manager node, running the following commands. 

  ```sh
  docker stack deploy -c docker-stack.yml vote
  
  Creating network vote_backend
  Creating network vote_frontend
  Creating network vote_default
  Creating service vote_result
  Creating service vote_worker
  Creating service vote_visualizer
  Creating service vote_redis
  Creating service vote_db
  Creating service vote_vote

 ```
  
  Then curl to check if it is accessible 
  
  ```sh
  http://192.168.77.10:8080/
  ```
  
  ref : https://docs.docker.com/v17.12/datacenter/ucp/2.1/guides/user/services/deploy-app-cli/#cleanup
  
  
## Docker metrics for Prometheus

Docker exposes Prometheus-compatible metrics on port `9323`. This support is only available as an experimental feature.
To configure the Docker daemon as a Prometheus target, it is need to specify the metrics-address. The best way to do this is via the daemon.json, which is located at one of the following locations by default. If the file does not exist, create it.

```sh
Linux: /etc/docker/daemon.json
```
add be

```sh
{
  "metrics-addr" : "0.0.0.0:9323",
  "experimental" : true
}
```

 # Click on `Apply & Restart` to restart the daemon

Show the complete list of metrics using `curl http://localhost:9323/metrics`
Show the list of engine metrics using `curl http://localhost:9323/metrics | grep engine`

## Start Prometheus

In this section, we'll start Prometheus and use it to scrape it's own health.

. Create a new directory `prometheus` and change to that directory
. Create a text file `prometheus.yml` and use the following content

```sh
# A scrape configuration scraping a Node Exporter and the Prometheus server
# itself.
scrape_configs:
  
  
  Scrape Prometheus itself every 5 seconds.
  - job_name: 'prometheus'
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9090']
```

find below the content of yaml file 

```sh

root@swarm-master-1:~# cat /tmp/prometheus.yml
# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

  # Attach these labels to any time series or alerts when communicating with
  # external systems (federation, remote storage, Alertmanager).
  external_labels:
      monitor: 'codelab-monitor'

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first.rules"
  # - "second.rules"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
      - targets: ['192.168.77.10:9090']

  - job_name: 'docker'
    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
      - targets: ['192.168.77.10:9323']


  - job_name: 'application'
    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
      - targets: ['192.168.77.10:8080']     
   ```

This configuration file scrapes data from the Prometheus container which will be started subsequently on port 9090.
Start a single-replica Prometheus service:

```sh
docker service create \
  --replicas 1 \
  --name metrics \
  --mount type=bind,source=`pwd`/prometheus.yml,destination=/etc/prometheus/prometheus.yml \
  --publish 9090:9090/tcp \
  prom/prometheus
```

This will start the Prometheus container on port 9090.

. Prometheus dashboard is at http://localhost:9090. Check the list of enabled targets at http://localhost:9090/targets (also accessible from `Status` -> `Targets` menu).


It shows that the Prometheus endpoint is available for scraping.


. Click on `Graph` and click on `-insert metric at cursor-` to see the list of metrics available:

These are all the metrics published by the Prometheus endpoint.


. Choose `http_request_total` metrics, click on `Execute`

. Switch from `Console` to `Graph`




## Method solution — Open Docker Swarm Ports Using FirewallD

FirewallD is the default firewall application on Ubuntu VM. In case it is not,let’s enable it and add the network ports necessary for Docker Swarm to function.

Before starting, verify its status:
```sh
systemctl status firewalld
```

It should not be running, so start it:
```sh
systemctl start firewalld
```

Then enable it so that it starts on boot:
```sh
systemctl enable firewalld
```

On the node that will be a Swarm manager, use the following commands to open the necessary ports:

```sh
firewall-cmd --add-port=8080/tcp --permanent
```
Note: If you make a mistake and need to remove an entry, type:

```sh
firewall-cmd --remove-port=port-number/tcp —permanent.
```

Afterwards, reload the firewall:

```sh
firewall-cmd --reload
```
Then restart Docker.
```sh
systemctl restart docker
```

