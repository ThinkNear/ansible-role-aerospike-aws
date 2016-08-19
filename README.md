Ansible Role: Aerospike AWS
=========

[![Apache 2.0](https://img.shields.io/badge/license-Apache%202-blue.svg)](https://raw.githubusercontent.com/DigitalSlideArchive/ansible-role-vips/master/LICENSE)
[![Build Status](https://travis-ci.org/ThinkNear/ansible-role-aerospike-aws.svg?branch=master)](https://travis-ci.org/ThinkNear/ansible-role-aerospike-aws)

An Ansible role that launches a auto-healing and auto-discovering Aerospike Cloudformation stack on AWS.

Requirements
------------

This role makes use of AWS cloud modules, which require python >= 2.6 and `boto`.

This role requires Ansible v2.

This role launches a Cloudformation stack on AWS. 
This requires and AWS account and IAM permissions that allows the ansible user to manage resources.

Using this role on AWS will incur charges to your account.

This role is designed for an existing VPC and subnets.

This role expects a IAM role `aerospike` to exist for the EC2 instance profile.
This role must have permission to get files from s3.
If `aerospike_aws_use_librato` is true then the role `aerospike` must allow querying DynamoDB.

This role expects the role `ThinkNear.aerospike` to be installed. `
ThinkNear.aerospike` is not listed as a dependency because it should not be run before `ThinkNear.aerospike-aws`
This role uses the configuration template and variables from `ThinkNear.aerospike` to configure a stack.

Role Variables
--------------

### Defaults

Available variables are listed below, along with default values (see defaults/main.yml):

    aerospike_aws_temp_dir: "{{ lookup('env', 'TMPDIR') | default('/tmp', true) }}/ansible_role_aerospike_aws.temp"
    
Controls location for generated template files on the localhost.

    aerospike_aws_thinknear_aerospike_role_path: "{{ role_path }}/../ThinkNear.aerospike"
    
Controls the lookup location for `ThinkNear.aerospike`. 
This role uses the configuration template and variables from `Thinknear.aerospike` to configure a stack.

    aerospike_aws_base_vars_file: "{{ role_path}}/vars/main.yml"
    
The path of the base vars file. Each stack managed by `Thinknear.aerospike-aws` may have variables in common.
Define the shared variables in this location.

    aerospike_aws_stacks_directory: "{{ role_path }}/vars"
    
The path to find each stack var file.

    aerospike_aws_use_librato: true

Enables Librato metric reporting for attach eni events.
Set to `false` if you do not want Librato metrics when a ENI fails or succeeds to attach.

    aerospike_aws_dynamodb_table_name: libratoCredentials

When `aerospike_aws_use_librato` is enabled, provides the DynamoDB table to find Librato credentials.
This table is expected to have partition keys `config_key` and `config_value` for `config_key_ values of `LIBRATO_USER` and `LIBRATO_TOKEN`.

### Dependent variables

This role uses the configuration template and variables from `ThinkNear.aerospike` to configure a stack. 

Because this role runs on host `localhost`, the inventory variables must be loaded by a task.

Use the default variables `aerospike_aws_base_vars_file` and `aerospike_aws_stacks_directory` to define locations to load inventory variables.

### Required variables

The user must define the variable `aerospike_aws_stacks`.
This role will include vars from `{{ aerospike_aws_stacks_directory }}/{{ aerospike_aws_stacks }}.yml` to find a variable named `aerospike_aws_included_stacks`.

This role will manage each stack named in `aerospike_aws_included_stacks`, loading **its** vars from `{{ aerospike_aws_stacks_directory }}/{{ each_stack }}.yml`

Each stack must define the following variables (with example values):

    aerospike_aws_s3_bucket: aerospike-configs

The S3 bucket where configuration and boot scripts are staged.

    aerospike_aws_s3_tags:
        Name: aerospike-configs
    
Tags for the S3 bucket

    aerospike_aws_region: us-east-1
    
The region to launch the stack.

    aerospike_aws_template: roles/aerospike-aws/files/cf_stack_ip_5.json
    
The path the the CF template.

    aerospike_aws_ami: ami-b6cbc0dc
    
The AMI to use. See [Aerospike 3 Database on AWS Marketplace](https://aws.amazon.com/marketplace/pp/B00LW9382A/ref=mkt_wir_aerospike3_hvm) for more choices.

    aerospike_aws_route53_zone_name: mydomain.com.
    
This Route53 hosted zone domain name.

    aerospike_aws_included_stacks: my_stack

Each stack to process. This must contain at least the name of one stack.

    aerospike_aws_stack_name: my-stack
    
The name of the Cloudformation stack. Must be unique.

    aerospike_aws_s3_path_aerospike_conf: aerospike.conf
    
The path on the S3 bucket to put the staged config file.

    aersopike_aws_s3_path_eni_attach
    
The path on the S3 bucket to put the staged attach eni script.

    aerospike_aws_template_parameters
    
A list of hashes of parameters used with the cloudformation template. See below for an example.

    # example aerospike_aws_template_parameters
    aerospike_aws_template_parameters: 
      HostedRecordName: aerospike
      InstanceType: t2.small
      AerospikeAvailabilityZone: us-east-1b
      KeyPair: my_key_pair_name
      AerospikeSubnet: subnet-aaaaaaa
      VpcId: vpc-1234567
      SshCidrIp: 10.1.0.0/16
      AsGroupId: sg-1234567
      NumberOfInstances: "{{ aerospike_cluster_size }}"
      PrivateStaticIPs: "{{ aerospike_mesh_seed_addresses | join(',') }}"
      ConfigS3Location: "{{ aerospike_aws_s3_bucket }}/{{ aerospike_aws_s3_path_aerospike_conf }}"
      AerospikeAMI: "{{ aerospike_aws_ami }}"
      AttachEniS3Location: "{{ aerospike_aws_s3_bucket }}/{{ aersopike_aws_s3_path_eni_attach }}"
      MetricSourceName: "{{ collectd_sourcename }}"
      HostedZoneName: "{{ aerospike_aws_route53_zone_name }}"

`aerospike_aws_tags` is a list of hashes of tags for the stack. See below for an example.

    aerospike_aws_tags:
      Name: my-stack
      Stage: test
      Service: aerospike

Dependencies
------------

You must have `ThinkNear.aerospike` installed.
This is not listed as a dependency in `meta.yml` because the role should not be run before this role.

Example Playbook
----------------
The playbook:

    - name: aerospike stack
      hosts: localhost
      gather_facts: no
      tags:
        - stack
    
      roles:
        - role: ThinkNear.aerospike-aws
          aerospike_aws_base_vars_file: ../../inventory/group_vars/aerospike.yml
          aerospike_aws_stacks_directory: ../../inventory/group_vars

inventory/group_vars/aerospike.yml:

    aerospike_version: 3.7.4
    aerospike_tools_version: 3.7.3
    aerospike_service_threads: 8 
    aerospike_transaction_queues: 8
    aerospike_transaction_threads: 8 
    aerospike_namespaces:
      - name: thinknear
        memory_size: 15G
        storage_engine:
          devices:
            - /dev/sdb
            - /dev/sdc
          scheduler_mode: noop
          write_block_size: 128K
    
    aerospike_aws_s3_bucket: aerospike-configs
    aerospike_aws_s3_tags:
      Name: aerospike-configs
    aerospike_aws_region: us-east-1
    aerospike_aws_template: roles/aerospike-aws/files/cf_stack_ip_5.json
    aerospike_aws_ami: ami-b6cbc0dc
    aerospike_aws_route53_zone_name: mydomain.com.
    aerospike_aws_included_stacks:
      - my_stack
      
inventory/group_vars/my_stack.yml:

    aerospike_mesh_seed_addresses:
      - 10.1.0.1
      - 10.1.0.2
      - 10.1.0.3
      - 10.1.0.4
      - 10.1.0.5
    aerospike_cluster_size: 1
    
    aerospike_aws_included_stacks:
      - aerospike_test_tiles
    aerospike_aws_stack_name: my-stack
    aerospike_aws_s3_path_aerospike_conf: aerospike.conf
    aersopike_aws_s3_path_eni_attach: attach_eni.sh
    aerospike_aws_template_parameters:
      HostedRecordName: my-stack
      InstanceType: c3.2xlarge
      AerospikeAvailabilityZone: us-east-1b
      KeyPair: my_key_pem
      AerospikeSubnet: subnet-aaaaaaa
      VpcId: vpc-1234567
      SshCidrIp: 10.1.0.0/16
      AsGroupId: sg-1234567
      NumberOfInstances: "{{ aerospike_cluster_size }}"
      PrivateStaticIPs: "{{ aerospike_mesh_seed_addresses | join(',') }}"
      ConfigS3Location: "{{ aerospike_aws_s3_bucket }}/{{ aerospike_aws_s3_path_aerospike_conf }}"
      AerospikeAMI: "{{ aerospike_aws_ami }}"
      AttachEniS3Location: "{{ aerospike_aws_s3_bucket }}/{{ aersopike_aws_s3_path_eni_attach }}"
      MetricSourceName: "{{ collectd_sourcename }}"
      HostedZoneName: "{{ aerospike_aws_route53_zone_name }}"
    aerospike_aws_tags:
      Name: my-stack
      Stage: test
      Service: aerospike
      
License
-------

ALv2

Author Information
------------------

This role was created in 2016 by Thinknear. 
http://thinknear.com
