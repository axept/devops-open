Step By Step Guide to Configure a CoreOS Cluster From Scratch
=============================================================

> Originally this instructions has been written in April of 2016.

This guide describes how to bootstrap new Production Core OS Cluster as High Availability Service in a 15 minutes with using etcd2, Fleet, Flannel, Confd, Nginx Balancer and Docker.

# Content

+ [Introduction](#introduction)
  + [Tools Used](#tools-used)
+ [Basic Configuration](#basic-configuration)
  + [Connect Your Servers as a Cluster](#connect-your-servers-as-a-cluster)
  + [Create Fleet Units](#create-fleet-units)
  + [Configure Firewall Rules](#configure-firewall-rules)
  + [Load Balancers and Service Discovery](#load-balancers-and-service-discovery)
+ [Troubleshooting](#troubleshooting)
+ [Update CoreOS](#update-coreos)
+ [Usage with Deis v1](#usage-with-deis-v1)
+ [Appendix 1 - Info and Tutorials](#appendix-1---info-and-tutorials)
+ [Appendix 2 - Tools and Services](#appendix-2---tools-and-services)

## Introduction

CoreOS is a powerful Linux distribution built to make large, scalable deployments on varied infrastructure simple to manage. 
CoreOS is designed for security, consistency, and reliability. Instead of installing packages via yum or apt, CoreOS uses Linux containers to manage your services at a higher level of abstraction. A single service's code and all dependencies are packaged within a container that can be run on one or many CoreOS machines.

Main building blocks of CoreOS — etcd, Docker and systemd.

See: [7 reasons why you should be using CoreOS with Docker](https://www.airpair.com/coreos/posts/coreos-with-docker).

### Tools Used

+ etcd: key-value store for service registration and discovery
+ fleet: scheduling and failover of Docker containers across CoreOS Cluster
+ flannel: gives each docker container a unique IP that allows you to access the internal port (i.e. port 80 not 32679)
+ confd: watch etcd for nodes arriving/leaving and update (with reload) nginx configuration by using specified template

## Basic Configuration

### Connect your servers as a cluster

1. Find your [Cloud Config file location](https://coreos.com/os/docs/latest/cloud-config-locations.html). For examples below we will use:

  ```bash
  /var/lib/coreos-install/user_data
  ```

2. Open your config to edit:

  ```bash
  sudo vi /var/lib/coreos-install/user_data
  ```

3. Generate new token for your cluster: https://discovery.etcd.io/new?size=X, where X is servers count. 
4. Merge follow lines with your Cloud Config:

  ```bash
  coreos:
    etcd2:
      # Generate a new token for each unique cluster from https://discovery.etcd.io/new
      # discovery: https://discovery.etcd.io/<token>
      discovery: https://discovery.etcd.io/9c19239271bcd6be78d4e8acfb393551
      
      # Multi-region and multi-cloud deployments need to use $public_ipv4
      advertise-client-urls: http://$private_ipv4:2379,http://$private_ipv4:4001
      initial-advertise-peer-urls: http://$private_ipv4:2380
      
      # Listen on both the official ports and the legacy ports
      # Legacy ports can be omitted if your application doesn't depend on them
      listen-client-urls: http://0.0.0.0:2379,http://0.0.0.0:4001
      listen-peer-urls: http://$private_ipv4:2380
    
    fleet:
      public-ip: $private_ipv4
      metadata: region=europe,public_ip=$public_ipv4
    
    flannel:
      interface: $private_ipv4
    
    units:
      - name: etcd2.service
        command: start
        # See issue: https://github.com/coreos/etcd/issues/3600#issuecomment-165266437
        drop-ins:
          - name: "timeout.conf"
            content: |
              [Service]
              TimeoutStartSec=0
              
      - name: fleet.service
        command: start
        
      # Network configuration should be here, e.g:
      # - name: 00-eno1.network
      #   content: "[Match]\nName=eno1\n\n[Network]\nDHCP=yes\n\n[DHCP]\nUseMTU=9000\n"
      # - name: 00-eno2.network
      #   runtime: true
      #   content: "[Match]\nName=eno2\n\n[Network]\nDHCP=yes\n\n[DHCP]\nUseMTU=9000\n"
      
      - name: flanneld.service
        command: start
        drop-ins:
        - name: 50-network-config.conf
          content: |
            [Service]
            ExecStartPre=/usr/bin/etcdctl set /coreos.com/network/config '{ "Network": "10.1.0.0/16" }'
            
      - name: docker.service
        command: start
        drop-ins:
        - name: 60-docker-wait-for-flannel-config.conf
          content: |
            [Unit]
            After=flanneld.service
            Requires=flanneld.service

            [Service]
            Restart=always
            
      - name: docker-tcp.socket
        command: start
        enable: true
        content: |
          [Unit]
          Description=Docker Socket for the API
  
          [Socket]
          ListenStream=2375
          Service=docker.service
          BindIPv6Only=both

          [Install]
          WantedBy=sockets.target
  ```
5. Online.net provide has a specific configuration preset. It requires you to process additional step - add those lines to Cloud Config to get Private Network working:

  ```bash
  units:
    # ...
    - name: 00-eno2.network
      runtime: true
      content: "[Match]\nName=eno2\n\n[Network]\nDHCP=yes\n\n[DHCP]\nUseMTU=9000\n"
  ```

6. Validate your changes:
  
  ```bash
  sudo coreos-cloudinit -validate --from-file /var/lib/coreos-install/user_data
  ```

7. Reboot the system:

  ```bash
  sudo reboot
  ```

8. Check status for etcd2:

  ```bash
  sudo systemctl status -r etcd2
  ```

  Output should contain a follow line:
  
  ```bash
   Active: active (running)
  ```

  Sometimes it takes a time. Don't panic. Just wait for a few minutes.

9. Repeat those steps for each server in your cluster.

10. Check your cluster health and fleet status:
  
  ```bash
  # should be healthy
  sudo etcdctl cluster-health
  # should display all servers
  sudo fleetctl list-machines
  ```

### Create Fleet Units

![Fleet Scheduling](https://coreos.com/assets/images/media/Fleet-Scheduling.png)

See: [Launching Containers with fleet](https://coreos.com/fleet/docs/latest/launching-containers-fleet.html)

#### Application Unit

1. Enter to your home directory:
  
  ```bash
  cd ~
  ```
  
2. Create new Application Template Unit. For example - run `vi test-app@.service` and add follow lines:
  
  ```bash
  [Unit]
  Description=test-app%i
  After=docker.service

  [Service]
  TimeoutStartSec=0
  ExecStartPre=-/usr/bin/docker kill test-app%i
  ExecStartPre=-/usr/bin/docker rm test-app%i
  ExecStartPre=/usr/bin/docker pull willrstern/node-sample
  ExecStart=/usr/bin/docker run -e APPNAME=test-app%i --name test-app%i -P willrstern/node-sample
  ExecStop=/usr/bin/docker stop test-app%i
  ```

3. Submit Application Template Unit to Fleet:

  ```bash
  fleetctl submit test-app@.service
  ```

4. Start new instances from Application Template Unit:

  ```bash
  fleetctl start test-app@1
  fleetctl start test-app@2
  fleetctl start test-app@3
  ```

5. Check that all instances has been started and active. It could take a few minutes. Example command and its output:

  ```bash
  $ fleetctl list-units
  UNIT			MACHINE				ACTIVE	SUB
  test-app@1.service	e1512f34.../10.1.9.17	active	running
  test-app@2.service	a78a3229.../10.1.9.18	active	running
  test-app@3.service	081c8a1e.../10.1.9.19	active	running
  ```
    
### Configure Firewall Rules

Run [custom-firewall.sh](https://raw.githubusercontent.com/deis/deis/master/contrib/util/custom-firewall.sh) from Deis v1 on your local machine:

```bash
curl -O https://raw.githubusercontent.com/deis/deis/master/contrib/util/custom-firewall.sh
# run follow line for each server
ssh core@<host1> 'bash -s' < custom-firewall.sh
```

### Load Balancers and Service Discovery

1. Download [someapp@.service](#file-someapp-service), [someapp-discovery@.service](#file-someapp-discovery-service) and [someapp-lb@.service](#file-someapp-lb-service).
2. Modify those Unit Templates according to your application config.
3. Submit modificated files to your Fleet:

  ```bash
  fleetctl submit someapp@.service
  fleetctl submit someapp-discovery@.service
  fleetctl submit someapp-lb@.service
  ```
  
4. Start Unit instances from templates:
  
  ```bash
  fleetctl start someapp@{1..6}
  fleetctl start someapp-discovery@{1..6}
  fleetctl start someapp-lb@{1..2}
  ```
  
5. Verify all is working good:

  ```bash
  fleetctl list-units
  ```

## Troubleshooting

+ **Something goes wrong and a service doesn't work**
  
  Use those commands to debug:

  ```bash
  # also for fleet, etcd, flanneld
  sudo systemctl start etcd2
  sudo systemctl status etcd2
  sudo journalctl -xe
  sudo journalctl -xep3
  sudo journalctl -ru etcd2
  ```

+ **`fleet list-units` is displaying `failed` state for any units**

  For local units:
  
  ```bash
  sudo fleetctl journal someapp@1
  ```
  
  For remote units:
  
  ```bash
  fleetctl journal someapp@1
  ```
  
+ **`fleetctl` reponds with: Error running remote command: SSH_AUTH_SOCK environment variable is not set. Verify ssh-agent is running. See https://github.com/coreos/fleet/blob/master/Documentation/using-the-client.md for help.**

  1. Check you have connected with `ssh -A`.
  2. Check you are not using `sudo` for remote machines. In this case a process under `sudo` can't access to your `SSH_AUTH_SOCK`.
  
+ **Error response from daemon: Conflict. The name "someapp1" is already in use by container c4acbb70c654. You have to delete (or rename) that container to be able to reuse that name.**

  ```bash
  fleetctl stop someapp@1
  docker rm someapp1
  fleetctl start someapp@1
  ```

+ **fleet `ssh` command doesn't working**
  
  1. Ensure your public key has been added everywhere in `user_data`. On each server.
  2. Connect to your server with SSH agent:
    
    ```bash
    eval `ssh-agent -s`
    ssh-add ~/.ssh/id_rsa
    ssh -A <your-host>
    ```

## Update CoreOS

```
sudo update_engine_client -update
sudo reboot
```

See more details [here.](https://coreos.com/os/docs/latest/update-strategies.html)

## Install Deis v1

Attention! It seems that doesn't work correctly with Online.net and other bare metal setups because `ceph` which is using for v1 works unstable and unpredictable. But if you would like to make an experiment, let's go:

1. Create backup copy of your original config:

  ```bash
  sudo vi cp /var/lib/coreos-install/user_data /var/lib/coreos-install/user_data.without-deis1
  ```
  
2. Merge your Cloud Config with [Deis Cloud Config example]( https://raw.githubusercontent.com/deis/deis/master/contrib/coreos/user-data.example).

3. You can configure Deis Platform from your workstation by following [this instruction](http://docs.deis.io/en/latest/installing_deis/install-platform/). The next steps adopted for server environment.

4. Download `deisctl`:

  ```bash
  curl -sSL http://deis.io/deisctl/install.sh | sudo sh -s 1.12.3
  ```
  
5. Set your configuration:

  ```bash
  deisctl config platform set domain=<your-domain>
  ```

6. Run platform installation:

  ```bash
  deisctl install platform
  ```
  
7. Boot up Deis:

  ```bash
  deisctl start platform
  ```
  
  If you get problems try to check Docker containers:
  
  ```bash
  docker ps -a
  ```
  
  Also you could use `journal` and `status` commands for `deisctl` to debug.
  
8. Once you see “Deis started.”, your Deis platform is running on a cluster. Verify that all Deis units are loaded by run:

  ```bash
  deisctl list
  ```
  
  All Deis units should be active. Otherwise you could destroy that all and don't forget to [remove unused Docker volumes](https://github.com/chadoe/docker-cleanup-volumes).

## Appendix 1 - Info and Tutorials

+ [Building Microservices with CoreOS & etcd](https://www.youtube.com/watch?v=5BPSnDSDnOc) <sup>[video talk]<sup>
+ [How To Set Up a CoreOS Cluster on DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-coreos-cluster-on-digitalocean)
+ [How To Secure Your CoreOS Cluster with TLS/SSL and Firewall Rules on DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-secure-your-coreos-cluster-with-tls-ssl-and-firewall-rules)
<sup>[article]</sup>
+ [Automated Nginx Reverse Proxy for Docker](http://jasonwilder.com/blog/2014/03/25/automated-nginx-reverse-proxy-for-docker/)<sup>[article]</sup>
+ [High Availability Apps via Fleet & CoreOS – Start to Finish: Provisioning on Azure](http://blog.stevenedouard.com/high-availability-apps-via-fleet-coreos-from-start-to-finish-provisioning-coreos-using-azure-resource-manager/)<sup>[article]</sup>
+ [Tips for Deploying NGINX (Official Image) with Docker](https://blog.docker.com/2015/04/tips-for-deploying-nginx-official-image-with-docker/)<sup>[article]</sup>
+ [CoreOS Continued: Fleet and Docker](http://blog.scottlowe.org/2014/08/20/coreos-continued-fleet-and-docker/)<sup>[article]</sup>
+ [Nginx Load Balancer Service For Core OS](https://gist.github.com/mssio/1b949d0952aaf69c2543)
+ [ServerFault: Nginx proxy to many container running on different CoreOS nodes](http://serverfault.com/questions/678695/nginx-proxy-to-many-container-running-on-different-coreos-nodes)
+ [Gist: Running a High Availability Service on CoreOS using Docker, Fleet, Flannel, Etcd, Confd & Nginx](https://gist.github.com/learncodeacademy/2a11bd906c87cf2301a1)
+ [Docker Fleet Starter](https://github.com/sedouard/fleet-bootstrapper)<sup>[github repo]</sup>
+ [Do Not Use Public Discovery Service For Runtime Reconfiguration](https://github.com/coreos/etcd/blob/master/Documentation/runtime-reconf-design.md#do-not-use-public-discovery-service-for-runtime-reconfiguration)<sup>[tips]</sup>
  
## Appendix 2 - Tools and Services

+ [docker-nginx-https-redirect](https://github.com/coreos/docker-nginx-https-redirect)
+ [Monitor CoreOS at scale with DataDog](https://www.datadoghq.com/blog/monitor-coreos-scale-datadog/)
+ [CoreGI - WebUI for monitoring CoreOS clusters including fleet and etcd](https://github.com/yodlr/CoreGI)
