# Ansible EC2 Launch Playbook

This Ansible playbook automates the process of launching an EC2 instance on AWS. It is designed to run locally (on `localhost`) and does not gather system facts, which speeds up execution. The playbook uses variables to store key configuration values such as the AWS region, EC2 instance type, AMI ID, security group name, and key pair name, making it flexible and easy to update for different environments.

## How It Works

The playbook follows these steps:

1. **Creates a Security Group**  
   Uses the `amazon.aws.ec2_group` module to create a security group with SSH (port 22) open to the world and all outbound traffic allowed.

2. **Outputs Security Group Details**  
   Prints the security group ID for reference.

3. **Gets Subnet Information**  
   Uses `amazon.aws.ec2_vpc_subnet_info` to look up the subnet where the instance will be launched.

4. **Checks for Existing EC2 Instances**  
   Uses `amazon.aws.ec2_instance_info` to check if an EC2 instance with the specified name tag already exists and is in a `running` or `pending` state.  
   **Reason:** This prevents launching duplicate instances and ensures idempotency.

5. **Conditionally Launches EC2 Instance**  
   Only launches a new EC2 instance if no existing instance with the specified tag is found.  
   This is controlled by a `when` condition:
   ```yaml
   when: existing_instances.instances | length == 0
   ```
   This ensures that if an instance already exists, the playbook will not create another one.

6. **Outputs EC2 Instance Details**  
   Prints the details of the launched instance for visibility and troubleshooting.

7. **Saves Public IP to Inventory**  
   Writes the public IP of the new instance to an `inventory.ini` file for use in future Ansible runs.

8. **Adds Instance to Host Group**  
   Dynamically adds the new instance to an in-memory Ansible group for further automation.

9. **Outputs Instance Details**  
   Prints the instance ID and public IP for each launched instance.

## Example Task: Conditional EC2 Launch

```yaml
- name: Check for existing EC2 instance
  amazon.aws.ec2_instance_info:
    region: "{{ region }}"
    filters:
      "tag:Name": "{{ instance_name }}"
      instance-state-name: [ "running", "pending" ]
  register: existing_instances

- name: Launch EC2 instance if not present
  amazon.aws.ec2_instance:
    name: "{{ instance_name }}"
    key_name: "{{ key_name }}"
    instance_type: "{{ instance_type }}"
    image_id: "{{ image_id }}"
    wait: true
    state: running
    exact_count: 1
    region: "{{ region }}"
    vpc_subnet_id: "{{ subnet_info.subnets[0].subnet_id | default(omit) }}"
    security_group: "{{ security_group }}"
    tags: 
      Enviroment: Demo
  register: ec2
  when: existing_instances.instances | length == 0
```

## The following section explains every task within the playbook

```yaml
- name: Launch EC2 instance
  hosts: localhost
  connection: local
  gather_facts: no
```
* Defines a play named "Launch EC2 instance" that runs on the local machine, does not collect system facts, and uses a local connection.

```yaml
  vars:
    key_name: web-app-key
    region: eu-west-1
    instance_type: t2.micro
    image_id: ami-0fab1b527ffa9b942
    security_group: my-sg-group
    instance_name: ansible-ec2-demo
```
* Sets variables for the AWS key pair, region, instance type, AMI ID, security group, and instance name.

```yaml
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

```yaml
    - name: Output security group details
      ansible.builtin.debug:
        msg: "Security group created with ID: {{ sg.group_id }}"
```
* Prints the security group ID to the console for reference.

```yaml   
        amazon.aws.ec2_instance:
        name: "{{ instance_name }}"
        key_name: "{{ key_name }}"
        instance_type: "{{ instance_type }}"
        image_id: "{{ image_id }}"
        wait: true
        state: running
        exact_count: 1
        region: "{{ region }}"
        vpc_subnet_id: "{{ subnet_info.subnets[0].subnet_id  | default(omit) }}"
        security_group: "{{ security_group }}"
        tags: 
          Enviroment: Demo
      register: ec2
      when: existing_instances.instances | length == 0
```
* Launches an EC2 instance with the specified parameters, waits for it to be ready, and tags it. The result is registered as ec2.

```yaml
 - name: Check for existing ec2 instances
      amazon.aws.ec2_instance_info:
        region: "{{ region }}"
        filters:
          tag:Name: "{{ instance_name }}"
          instance-state-name: ["running", "pending"]
      register: existing_instances
```

* Uses ```amazon.aws.ec2_instance_info``` to check if an EC2 instance with the specified name tag already exists and is in a running or pending state.
```yaml
   - name: Save public ip to inventory
      copy:
        content: |
          [web]
          {{ ec2.instances[0].public_ip_address }} ansible_user=ubuntu ansible_ssh_private_key_file=/Users/user/Downloads/web-app-key.pem 
        dest: inventory.ini
      when: ec2.instances | length > 0
```
* Writes the public IP of the new instance to an ```inventory.ini``` file for later use, only if an instance was created.

```yaml
    - name: Add new instance to host group
      ansible.builtin.add_host:
        hostname: "{{ item.public_ip }}"
        groupname: launched_ec2
      loop: "{{ ec2.instances }}"
```
* Dynamically adds the new instance(s) to an in-memory Ansible group called ```launched_ec2```.

```yaml
    - name: Output instance details
      ansible.builtin.debug:
        msg: "Launched EC2 instance with ID: {{ item.instance_id }}, Private IP: {{ item.private_ip_address }}"
      loop: "{{ ec2.instances }}"
```
* Prints the instance ID and public IP for each launched instance.


3. After completion, the public IP of the new instance will be saved in `inventory.ini`.


## Notes

- Update the subnet ID in the playbook to match your AWS environment.
- The playbook assumes the default user is `ubuntu` (for Ubuntu AMIs). Change as needed.
- Ensure your key pair file path is correct and accessible.
