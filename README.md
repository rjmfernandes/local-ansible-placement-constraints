# Local Ansible CP Based on Docker Instances (MacOS) - Test Placement Constraints


- [Local Ansible CP Based on Docker Instances (MacOS) - Test Placement Constraints](#local-ansible-cp-based-on-docker-instances-macos---test-placement-constraints)
  - [Disclaimer](#disclaimer)
  - [Build the base image](#build-the-base-image)
  - [CP Ansible](#cp-ansible)
  - [Start Containers](#start-containers)
  - [Deploy with Ansible](#deploy-with-ansible)
  - [Reproduce the placement test](#reproduce-the-placement-test)
  - [Cleanup](#cleanup)

## Disclaimer

The code and/or instructions here available are **NOT** intended for production usage. 
It's only meant to serve as an example or reference and does not replace the need to follow actual and official documentation of referenced products.

## Build the base image

The image used for our instances comes from the work by Jeff Geerling https://github.com/geerlingguy/docker-ubuntu2204-ansible

We have added a couple of packages including Java. Ubuntu 22.04 provides Python 3.10 on the managed nodes, which is compatible with CP Ansible 8.1.4.

First you will need to create the image:

```bash
docker build . -t my-geerlingguy-docker-ubuntu-ansible
```

## CP Ansible

Clone locally the repository:

```shell
git clone --depth 1 --branch v8.1.4 https://github.com/confluentinc/cp-ansible
```

CP-Ansible 8.1.4 supports Ansible 9.x with Python 3.10–3.12, Ansible 10.x with Python 3.10–3.12, or Ansible 11.x with Python 3.11–3.12. On macOS, install Python 3.10 with Homebrew if it is not already installed, then use that interpreter explicitly to create the virtual environment. Do not continue if the version check fails:

```bash
brew install python@3.10
PYTHON310="$(brew --prefix python@3.10)/bin/python3.10"
"$PYTHON310" --version
"$PYTHON310" -m venv ~/venvs/ansible-cp814
source ~/venvs/ansible-cp814/bin/activate
python -m pip install --upgrade pip
python -m pip install 'ansible>=10,<11' bcrypt
ansible --version
```

Verify the output reports Python 3.10 or later and Ansible Core 2.17.x. Python 3.9.6 selects Ansible 8.7 / Ansible Core 2.15, which is too old for CP-Ansible 8.1.4. If `brew` is not installed, install Python 3.10 from python.org and use its `python3.10` executable instead. If venv creation fails, stop there: the later `pip` commands would otherwise keep using whichever environment is currently active.

Inside the repository you need to copy the playbooks to root:

```bash
cd cp-ansible
cp -fr playbooks/* .
```

Copy hosts.yml to cp-ansible.

```bash
cp ../hosts.yml .
```

Edit the variables of hosts.yml as the example here. Pay attention to the following variables:

```yml
    ansible_connection: docker
    ansible_user: root
    ansible_become: true
    ssl_enabled: false
    confluent.platform.ssl_required: false
    ansible_python_interpreter: /usr/bin/python3
    custom_java_path: /usr/lib/jvm/java-1.17.0-openjdk-arm64
```

The `kafka_controller` group in hosts.yml and the `kc1`–`kc3` services in compose.yml provide the three-node KRaft controller quorum. The `kafka_broker` inventory group and `kafka1`–`kafka6` services provide six brokers across two rack labels (`az1` and `az2`). The placement constraints, replication defaults, and rack labels are already configured in hosts.yml. The ZooKeeper group and ZooKeeper containers are not used.

For CP-Ansible 8.1, Control Center must be configured under the `control_center_next_gen` inventory group; the legacy `control_center` group is unsupported.

## Start Containers

Finally run the docker-compose from the root of the project:

```bash
cd ..
docker compose up -d --remove-orphans
```

The `--remove-orphans` option removes containers for services no longer defined in `compose.yml`. KRaft controllers communicate with the brokers over the Compose network, so their ports do not need to be published to the host.

For access from macOS, map the host-published service names in `/etc/hosts` to localhost. Kafka advertises the `kafka1`–`kafka6` names, so clients on the host need these mappings:

```
127.0.0.1 kafka1
127.0.0.1 kafka2
127.0.0.1 kafka3
127.0.0.1 kafka4
127.0.0.1 kafka5
127.0.0.1 kafka6
127.0.0.1 cc
```

Do not add `kc1`–`kc3` here: the KRaft controllers communicate with brokers over the Compose network and their ports are not published to macOS.

## Deploy with Ansible

Finally back to the cp-ansible cloned repository run:

```bash
cd cp-ansible
ansible-galaxy collection install git+https://github.com/confluentinc/cp-ansible.git,v8.1.4
```

The playbooks import local roles from the cloned repository. From `cp-ansible`, expose its roles directory to Ansible before running the playbook:

```bash
export ANSIBLE_ROLES_PATH="$PWD/roles"
ansible-playbook ./all.yml -i hosts.yml
```

The `bcrypt` package must be installed in the same Python environment that runs Ansible. CP-Ansible uses it on the control node to generate Control Center Next Gen dependency authentication configs. If Ansible is already installed in your virtual environment, activate that environment and run `python -m pip install bcrypt`, then rerun the playbook.

## Reproduce the placement test

The inventory applies the following properties in both `kafka_broker_custom_properties` and `kafka_controller_custom_properties`:

```yaml
default.replication.factor: 4
min.insync.replicas: 3
confluent.log.placement.constraints: >-
  {"version":2,"replicas":[{"count":2,"constraints":{"rack":"az1"}},{"count":2,"constraints":{"rack":"az2"}}],"observers":[{"count":1,"constraints":{"rack":"az1"}},{"count":1,"constraints":{"rack":"az2"}}],"observerPromotionPolicy":"under-min-isr"}
```

The policy assigns two in-sync replicas to each rack and one observer to each rack. The six brokers are rack-labeled in `hosts.yml`: `kafka1`–`kafka3` use `az1`, and `kafka4`–`kafka6` use `az2`.

Run both commands from the host after deployment. The first omits the replication factor so Kafka uses the configured broker default and placement policy. Do not pass `-1` explicitly: this version of `kafka-topics` rejects it as an invalid command-line replication factor. The second command explicitly sets a replication factor of 4 as a control for comparison:

```bash
KAFKA_TOPICS=/usr/bin/kafka-topics
TOPIC_DEFAULT="placement-default-$(date +%s)"
TOPIC_EXPLICIT="placement-explicit-$(date +%s)"

docker exec kafka1 "$KAFKA_TOPICS" --bootstrap-server kafka1:9092 \
  --create --topic "$TOPIC_DEFAULT" --partitions 5
docker exec kafka1 "$KAFKA_TOPICS" --bootstrap-server kafka1:9092 \
  --create --topic "$TOPIC_EXPLICIT" --partitions 5 --replication-factor 4

describe_topic() {
  docker exec kafka1 "$KAFKA_TOPICS" --bootstrap-server kafka1:9092 \
    --describe --topic "$1" |
    awk '
      /PartitionCount:/ {
        printf "%s: partitions=%s, assigned_replicas=%s\n", $2, $6, $8
      }
      /Partition:/ {
        partition = leader = replicas = isr = observers = ""
        for (i = 1; i <= NF; i++) {
          if ($i == "Partition:") partition = $(i + 1)
          else if ($i == "Leader:") leader = $(i + 1)
          else if ($i == "Replicas:") replicas = $(i + 1)
          else if ($i == "Isr:") isr = $(i + 1)
          else if ($i == "Observers:") observers = $(i + 1)
        }
        printf "  partition=%s leader=%s replicas=[%s] isr=[%s] observers=[%s]\n", partition, leader, replicas, isr, observers
      }
    '
}

describe_topic "$TOPIC_DEFAULT"
describe_topic "$TOPIC_EXPLICIT"
```

The default-factor topic is the behavior under test. It should report `assigned_replicas=6`: four in-sync replicas (two per rack) and two observers (one per rack). `isr` contains broker IDs that can serve as in-sync replicas; `observers` contains the additional assigned brokers outside the ISR. Broker IDs 1–3 are in `az1`, and IDs 4–6 are in `az2`.

The explicit-factor topic is a control. It should report four assigned replicas and no observers, showing that an explicit replication factor does not use the default placement policy. Partition leaders and broker IDs can vary. `kafka-configs --entity-default --describe` reports dynamic broker overrides and does not by itself verify static properties written to `server.properties`; inspect the generated broker and controller properties files when confirming Ansible applied the settings.

You should be able to access control center on usual port http://localhost:9021/clusters

## Cleanup

```bash
cd ..
rm -fr cp-ansible
docker compose down -v
```