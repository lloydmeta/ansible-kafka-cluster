# Kafka CentOS virtual cluster

Easily create a local Kafka cluster w/ Zookeeper quorum via Vagrant + Ansible. Or just use the Ansible playbooks.

The main differences between this and [Wirbelsturm](https://github.com/miguno/wirbelsturm) are:

- Focus on Ansible and playbooks that can be used to provision Zk + Kafka w/o Vagrant
- No Storm provisioning as of writing

## Usage

Depending on your hardware and network connection, the script execution might take between 10-20 minutes.

### 1. Install Vagrant, VirtualBox and Ansible on your machine

For Mac, this can be done with Homebrew:
```
brew install caskroom/cask/brew-cask
brew cask install virtualbox
brew install vagrant
brew install ansible
```

Make sure you are running Ansible v1.7.2 or higher with `ansible --version`.

For other systems, checkout the installation pages of [Vagrant](https://docs.vagrantup.com/v2/installation/), [VirtualBox](https://www.virtualbox.org/wiki/Downloads) and [Ansible](http://docs.ansible.com/intro_installation.html).

### 2. Clone this repo

```
git clone git@github.com:lloydmeta/ansible-kafka-cluster.git
cd ansible-kafka-cluster
```


### 3. Start the cluster with Vagrant

```
vagrant up
```

### 4. SSH into the first and second Kafka nodes

In separate terminals, run:

```
vagrant ssh kafka-node-1
```

```
vagrant ssh kafka-node-2
```

### 5. Create a replicated topic in kafka-node-1 and play with it

```
export LOG_DIR=/tmp # otherwise we get a warning
cd /etc/kafka_2.11-0.8.2.1

bin/kafka-topics.sh --create --zookeeper zk-node-1:2181 --replication-factor 3 --partitions 1 --topic my-replicated-topic

# Look at the topic description to make sure it is indeed replicated
bin/kafka-topics.sh --describe --zookeeper zk-node-1:2181 --topic my-replicated-topic

# Send a few messages
bin/kafka-console-producer.sh --broker-list localhost:9092 --topic my-replicated-topic
...
my test message 1
my test message 2
^C
```

### 6. In kafka-node-2, consume a few messages
```
export LOG_DIR=/tmp # otherwise we get a warning
cd /etc/kafka_2.11-0.8.2.1

bin/kafka-console-consumer.sh --zookeeper zk-node-1:2181 --from-beginning --topic my-replicated-topic
...
my test message 1
my test message 2
^C
```

### 7. Suspend a Kafka node

From  your host, take down a Kafka node

```
vagrant suspend kafka-node-3
```

Wait 6 seconds; this is the default heartbeat frequency for Kafka that tells Zookeeper that a node has gone down.

Then verify on any other Kafka node that your replicated topic is now missing one member under isr (in-sync replica), but otherwise still works. If this takes some time,

```
bin/kafka-topics.sh --describe --zookeeper zk-node-1:2181 --topic my-replicated-topic
```

Feel free to send and consume messages while the node is down. Note though, that you may have to restart some of the shell-based consumer/producers that are already running because they don't work well in disaster situations..

### 8. Bring the Kafka node back up, put it to sleep, etc.

