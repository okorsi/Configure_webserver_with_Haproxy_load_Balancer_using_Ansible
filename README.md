#  Configure webserver with Haproxy load balancer using Ansible playbook

We have two security groups. One for the load balancer and the second for webservers. 
Create a security group for load balancer allowing ports 80 and 8080 port and 22 port for ssh. 
Create another security group and allow load balancer group in its inbound rule for port 8080 and 22 port for ssh. 
so that outsiders/clients can not directly connect to our webservers.

![Ansible](https://user-images.githubusercontent.com/106388100/187774412-b66bdc88-e4cc-4a47-9820-fd54c09e0bd2.png)


ansible.cfg - contains Ansible's settings\
webserverkey.pem - private key for connecting ansible to EC2\
secret/aws_credentials.yml - credentials to access the AWS account\
inventory floder - contains settings for AWS dynamic inventory module

Creating Security group for webservers
```yaml
      - name: "Creating Security group"
        ec2_group:
             name: "Webserver sg"
             description: "sg created using security group for webserver"
             region: "ap-south-1"
             rules:
                 - proto: tcp
                   from_port: 22
                   to_port: 22
                   cidr_ip: 0.0.0.0/0
                 - proto: tcp
                   from_port: '{{ wb_service_port }}'
                   to_port: "{{ wb_service_port }}"
                   cidr_ip: 0.0.0.0/0
             rules_egress:
                 - proto: all
                   cidr_ip: 0.0.0.0/0
             aws_access_key: "{{ aws_access_key }}"
             aws_secret_key: "{{ aws_secret_key }}"
  ````

Provisioning EC2 instances for webserver
```yaml
      - name: "Provisioning EC2 instances for webserver" 
        ec2:
            key_name: "webserverkey"
            image: "ami-0a9d27a9f4f5c0efc"
            instance_type: "t2.micro"
            region: "ap-south-1"
            wait: yes
            group: "Webserver sg"
            instance_tags:
                   "name" : "webserver"
            count: 3
            aws_access_key: "{{ aws_access_key }}"
            aws_secret_key: "{{ aws_secret_key }}"
````

Creating Security group for Load Balancer
```yaml
      - name: "Creating Security group for Load Balancer"
        ec2_group:
             name: "lb sg"
             description: "sg created using security group for LoadBalancer"
             region: "ap-south-1"
             rules:
                 - proto: tcp
                   from_port: 22
                   to_port: 22
                   cidr_ip: 0.0.0.0/0
                 - proto: tcp
                   from_port: '{{ lb_service_port }}'
                   to_port: '{{ lb_service_port }}'
                   cidr_ip: 0.0.0.0/0
             rules_egress:
                 - proto: all
                   cidr_ip: 0.0.0.0/0 
             aws_access_key: "{{ aws_access_key }}"
             aws_secret_key: "{{ aws_secret_key }}"
````

Provisioning EC2 instances for LoadBalancer
```yaml
      - name: "Provisioning EC2 instances for LoadBalancer"
        ec2:
            key_name: "webserverkey"
            image: "ami-0a9d27a9f4f5c0efc"
            instance_type: "t2.micro"
            region: "ap-south-1"
            wait: yes
            group: "lb sg"
            instance_tags:
                   "name" : "LB"
            count: 1
            aws_access_key: "{{ aws_access_key }}"
            aws_secret_key: "{{ aws_secret_key }}"
      - name: "refresh invntory"
        meta: refresh_inventory

      - pause:
            minutes: 1
````

configure web servers
```yaml
- hosts: tag_name_webserver
  tasks:
     - name: "Installing Apache Webserver"    
       package:
           name: "httpd"
           state: present


     - name: "Installing php software"
       package:
           name: "php"
           state: present

     - name: "Copy webpage"
       copy:
           dest: "/var/www/html/index.php"
           content: |
                   <pre>
                   <?php
                   
                   print `/usr/sbin/ifconfig`;
                   ?>
                   </pre>
     - name: "start http server"
       service:
             name: "httpd"
             state: started

````

Configure LoadBalancer
```yaml
- hosts: tag_name_LB
  vars:
      - service_port: 8080
  tasks:
      - name: Installing Reverse proxy software""
        package:
            name: "haproxy"
            state: present

      - name: "Copying configuration file"
        template:
            src: "haproxy.cfg.j2"
            dest: "/etc/haproxy/haproxy.cfg"
        notify: "restart LB"
        

  handlers:
      - name: "restart LB"
        service:
            name: "haproxy"
            state: restarted
            enabled: yes
````

