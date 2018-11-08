# Deploy Nomad & Consult via ansible, on Centos nodes on Vagrant local environment#

Vagrant ENV: Ubuntu 18.04 with latest virtualbox, vagrant, ansible installed

NOTE: vagrant plugin install vagrant-vbguest

vagrant up

Change consul and nomad versions to latest (sep 2018)

$ grep version ansible-role-consul/defaults/main.yml 
consul_version: 1.3.0
consul_webui_version: 1.3.0
$ grep version ansible-role-nomad/defaults/main.yml 
nomad_version: 0.8.6
davar@home ~/LABS/nomad-consult-ansible $ 

ansible-playbook -i ./inventory playbook.yml

for node2-3 : sudo usermod -aG docker vagrant ; exit; 

    nomad run redis.nomad
    nomad run flask.nomad
    nomad run nginx.nomad
    
## Troubleshooting

check nomad status

    nomad status
    nomad node status  
    nomad nomad server members
    nomad status redis  

Examples:

[vagrant@node-2 vagrant]$ nomad server members
Name          Address        Port  Status  Leader  Protocol  Build  Datacenter  Region
node1.global  192.168.50.10  4648  alive   true    2         0.8.6  dc1         global
[vagrant@node-2 vagrant]$ nomad node status
ID        DC   Name   Class   Drain  Eligibility  Status
4ed78222  dc1  node3  <none>  false  eligible     ready
8b9fbcc4  dc1  node2  <none>  false  eligible     ready
    


# Deploy Nomad & Consult via ansible, on Centos nodes on your prefered cloud provider via ansible#

## What you will get? ###

* Setup a 3 nodes (1 server, 2 clients) hashicorp nomad container orchestrator
* Run all on your prefered cloud provider via ansible


## Architecture of the stack

![nomad.PNG](https://github.com/adavarski/Hashicorp-Nomad-Consul-Ansible/raw/master/nomad.PNG)

- **Master/client nomad host**: master will take care of electing cluster leader, planning and rescheduling nomad jobs. While client will report node status to the master and look for jobs to run, and then run container
- **Consul**: service discovery. Nomad will record nodes, services in this database. Consul is replicated and  present on all nodes so nomad agents always have access to this data locally
- **dnsmasq**: a kind of proxy for dns resolution, route *.consul query to consul, and other query to localhost then internet
- **Ansible**: recipe will provide easy deployment of the solution

## Prerequisit

* Ansible
* A cloud infra providing centos hosts

## Deploy nomad cluster

Clone this repo

    git clone https://github.com/adavarski/Hashicorp-Nomad-Consul-Ansible 

Please deploy 3 standard Centos hosts (nomad-server1, nomad-client1, nomad-client2) on your prefered cloud provider.

Edit ansible inventory with your ips and ssh access key:

    nano inventory

**Deploy consul, docker, dnsmasq & nomad**

Run few times until you got no more errors)

    ansible-playbook playbook.yml

**Checks**

Consul: you can view consul UI on http://a_node_ip:8500/
Here are registered nomad clients and consul itself.

Dnsmask will help redirect dns queries to *.consul
Ssh connect to a node and try some dsn query:

    ping node1.node.consul.
    ping node2.node.consul.
    ping consul.service.consul.	

## Run nomad jobs

We will deploy here a very simple application:
- One nginx which redirect 80 --> flask app on 5000
- Flask app, which records in a redis database a number of view
- A redis database, on port 6379

Each of these components exposes a service, registered and avalaible for query in consul.

Copy nomad jobs to a client node and connect to it:

    scp -r -i ~/.ssh/id_rsa_sbexx jobs/* root@client_node_ip:/root/
    ssh -i ~/.ssh/id_rsa_sbexx jobs root@client_node_ip

    nomad run redis.nomad
    nomad run flask.nomad
    nomad run nginx.nomad

**Test services**

Redis

    echo 'PING' | nc global-redis-check.service.consul 6379

flask
  
    curl global-flask-check.service.consul:5000
 
Nginx

    curl global-nginx-check.service.consul
 

## Troubleshooting

check nomad status

    nomad status
    nomad node status  
    nomad nomad server members
    nomad status redis
    nomad alloc-status <alloc-id>

Delete nomad job

    nomad stop redis
    
    If you wish to lower the GC interval permanently for jobs, you can use the job_gc_threshold configuration parameter within the server config stanza.
curl -X PUT http://localhost:4646/v1/system/gc
    
