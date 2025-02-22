# Instructor notes
---

## Pre-reqs

- AWS CLI installed: http://docs.aws.amazon.com/cli/latest/userguide/installing.html
- Configure AWS CLI: `aws configure`
  - Details on creating credentials: http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-set-up.html
  - These AWS IAM permissions are needed: EC2, AutoScaling, CloudFormation, Marketplace

## How this works


Under each class directory (e.g. generic, sql, ...) there are several scripts used to make this work:

For AWS CloudFormation:

- cloudformation-generate-template.py: generates the AWS Cloudformation template
- cloudformation.json: the CloudFormation template which deploys a cluster

For managing multiple clusters (each takes variables for naming & the number of clusters):

- ../bin/clusters-create.sh: calls cloudformation to create clusters
- ../bin/clusters-report.sh: list cluster details
- ../bin/clusters-terminate.sh: calls cloudformation to terminate clusters
- ../bin/cloudformation-status.sh: list all cloudformation stacks in the region

For preparing the host(s):

- setup.sh: This is called by CloudFormation. It shoudl be usable without CloudFormation in any environment

## Deploy, report & terminate clusters on AWS


1. Get this repo:

```
git clone https://github.com/seanorama/masterclass
```

1. Switch to the directory of the class, or use 'generic':

```
cd masterclass/generic
```

1. Check for conflicting/existing stacks (same name as what you plan to deploy):
    - Open the CloudFormat Web UI (ensure you have chosen the correct region in the top right)
    - Or with the command-line: `..bin/cloudformation-status.sh`

1. Set variables to define the naming & number of clusters to deploy:
    - **each class has specific instructions for this**. Below is a generic example.
        - this deploys 2 clusters (mc-generic100, mc-generic101) in AWS region us-west-2
    - update 'lab_count' to the number of clusters you want

```sh
export AWS_DEFAULT_REGION=us-west-2 ## region to deploy in
export lab_prefix=mc-generic        ## template for naming the cloudformation stacks
export lab_first=100                ## number to start at in naming
export lab_count=2                  ## number of clusters to create
```

2. Set parameters which are passed to CloudFormation:
  - **each class has specific instructions for this**. Below is a generic example.
      - KeyName: The key (added on the EC2 page) to access the cluster.
      - AmbariServices: Which HDP services to deploy.
      - AdditionalInstanceCount: How many additional nodes to deploy. (Setting to 2 will deploy 3 nodes total)
      - SubnetId & SecurityGroups: This CloudFormation deploys in an existing Subnet & Security Group. **You must update this to your environment**.

```sh
export cfn_parameters='
[
  {"ParameterKey":"KeyName","ParameterValue":"training-keypair"},
  {"ParameterKey":"AmbariServices","ParameterValue":"HDFS MAPREDUCE2 PIG YARN HIVE ZOOKEEPER"},
  {"ParameterKey":"SubnetId","ParameterValue":"subnet-02edac67"},
  {"ParameterKey":"SecurityGroups","ParameterValue":"sg-a02d17c4"}]
'
```

1. Provision your clusters

```
../bin/clusters-create.sh
```

1. Check the build status
    - From the CloudFormation Web UI
    - Or from the command-line:

```
../bin/cloudformation-status.sh
```

1. Once your clusters are ready, get list of clusters nodes for providing to students:

```
..bin/clusters-report.sh
```

1. Use the clusters:
   - `ssh centos@ipFromReportAbove` ## use the key which was specified during the build

1. Terminate clusters

```
..bin/clusters-terminate.sh
```

1. Verify that all clusters are terminated
    - From the AWS CloudFormation Web UI
    - Or from the CLI

```
../bin/cloudformation-status.sh
```

## Running sessions

It's recommended to use an "etherpad" to share:

- the cluster details (from above)
- instructions to students

You can create your own, or use a hosted version such as TitanPad. You should create this account in advance.

## Issues: Deployment

#### Creation

- Some instances will fail their creation and time out, being rolled back, this is a nature of deploying large volumes of instances
- Those that fail should simply be manually deleted from the cloudformations web ui

#### Deleting cloudformations

- Occasionally cloudformations will fail to delete due to timing issues, in which case, it’s probably the VPC or InternetGateway, just switch to the VPC service window within the AWS site, delete the specific VPC that is being complained about in the cloudformation and then once the cloudformation delete has failed, retry the delete, deletion should complete this time.
- Once you’ve done the VPC deletion you can also do an AWS CLI call instead:
    - `aws cloudformation delete-stack --stack-name <cluster-name>`

#### AWS Website

If you suddenly notice that your instances/cloudformations/etc have vanished from the AWS control panel, you may have to re-login.

## Issues: Other

#### Run commands in bulk on all nodes

* There are several options, such as pdsh, cssh, ...

* Example using cssh, csshX or tmux-cssh (you'll need to install it)

```
## for Ambari nodes only:
../bin/clusters-report.sh | awk '/^AmbariNode:/ {print $2}' | xargs echo tmux-cssh -u student
```

* After executing you will get a terminal with small windows to all of the clusters.
* Anything you type will go to all hosts.

#### Venue Internet blocks Ambari Server (port 8080)

* Change Ambari to port 8081

```
export TERM=xterm
echo "client.api.port=8081" | sudo tee -a /etc/ambari-server/conf/ambari.properties
sudo ambari-server restart
sudo ambari-agent restart
```
