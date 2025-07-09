This Ansible playbook automates the process of launching an EC2 instance on AWS. It is designed to run locally (on localhost) and does not gather system facts, which speeds up execution since no information about the local machine is needed. The playbook uses variables to store key configuration values such as the AWS region, EC2 instance type, AMI ID, security group name, and key pair name, making the playbook flexible and easy to update for different environments.

The playbook's tasks begin by creating a security group using the amazon.aws.ec2_group module. This security group allows inbound SSH access (port 22) from any IP address and permits all outbound traffic. The details of the created security group are stored in the sg variable for later use. Immediately after, a debug task outputs the security group ID to provide feedback and aid in troubleshooting.

Next, the playbook launches an EC2 instance using the amazon.aws.ec2 module. It references the previously defined variables for configuration, waits for the instance to be ready, and associates it with the created security group. The instance is tagged for identification, and its details are saved in the ec2 variable.

Once the instance is running, the playbook saves its public IP address to a local inventory.ini file. This file can be used for subsequent Ansible runs to connect to the new instance. The playbook then adds the new instance to an in-memory host group called launched_ec2 using the add_host module, making it available for further tasks within the same playbook run. Finally, a debug task outputs the instance ID and public IP for each launched instance, providing clear feedback on the resources created. This structure makes the playbook a practical tool for automating EC2 provisioning and initial setup.

```
- name: Launch EC2 instance
  hosts: localhost
  connection: local
  gather_facts: no
```
* Defines a play named "Launch EC2 instance" that runs on the local machine, does not collect system facts, and uses a local connection.

```
  vars:
    key_name: web-app-key
    region: eu-west-1
    instance_type: t2.micro
    image_id: ami-0fab1b527ffa9b942
    security_group: my-sg-group
    instance_name: ansible-ec2-demo
```
* Sets variables for the AWS key pair, region, instance type, AMI ID, security group, and instance name.

```
  tasks: 
    - name: Create security group
      amazon.aws.ec2_group:
        name: "{{ security_group }}"
        description: Security group for Ansible ec2 ansible-ec2-demo
        region: "{{ region }}"
        rules:
          - proto: tcp
            ports: [22]
            cidr_ip: 0.0.0.0/0
        rules_egress:
          - proto: -1
            cidr_ip: 0.0.0.0/0
        state: present
        register: sg
```
* Creates a security group in AWS with SSH (port 22) open to the world and all outbound traffic allowed. The result is registered as sg.

```
    - name: Output security group details
      ansible.builtin.debug:
        msg: "Security group created with ID: {{ sg.group_id }}"
```
* Prints the security group ID to the console for reference.

```    - name: Launch EC2 instance
      amazon.aws.ec2:
        name: "{{ instance_name }}"
        key_name: "{{ key_name: }}"
        instance_type: "{{ instance_type: }}"
        image_id: "{{ image_id }}"
        wait: yes
        region: "{{ region }}"
        vpc_subnet_id: "{{ lookup('amazon.aws.aws_subnet', 'subnet-062baa97588e6ef8e') | default(omit) }}"
        security_group: "{{ security_group }}"
        count: 1
        tags: 
          Enviroment: Demo
      register: ec2
```
* Launches an EC2 instance with the specified parameters, waits for it to be ready, and tags it. The result is registered as ec2.

```
    - name: Save public ip to inventory
      copy:
        content: |
          [web]
          {{ ec2.instances[0].public_ip }} ansible_user=ubuntu ansible_ssh_private_key_file=/Users/user/Downloads/web-app-key.pem 
        dest: inventory.ini
        when: ec2.instances | length > 0
```
* Writes the public IP of the new instance to an ```inventory.ini``` file for later use, only if an instance was created.

```
    - name: Add new instance to host group
      ansible.builtin.add_host:
        hostname: "{{ item.public_ip }}"
        groupname: launched_ec2
      loop: "{{ ec2.instances }}"
```
* Dynamically adds the new instance(s) to an in-memory Ansible group called ```launched_ec2```.

```
    - name: Output instance details
      ansible.builtin.debug:
        msg: "Launched EC2 instance with ID: {{ item.id }}, Public IP: {{ item.public_ip }}"
      loop: "{{ ec2.instances }}"
```
* Prints the instance ID and public IP for each launched instance.


3. After completion, the public IP of the new instance will be saved in `inventory.ini`.

## What This Playbook Does

- Creates a security group with SSH access (port 22) open to the world.
- Launches an EC2 instance in the specified region and subnet.
- Saves the instance’s public IP to `inventory.ini` for future use.
- Adds the instance to an in-memory Ansible group for further automation.
- Outputs the instance ID and public IP.

## Notes

- Update the subnet ID in the playbook to match your AWS environment.
- The playbook assumes the default user is `ubuntu` (for Ubuntu AMIs). Change as needed.
- Ensure your key pair file path is correct and accessible.
