Installing PIP 
sudo yum install python-pip

Installing Ansible
sudo pip install ansible


Installing Jenkins

https://stackoverflow.com/questions/22415977/installing-and-managing-jenkins-on-amazon-linux

1. wget -O /etc/yum.repos.d/jenkins.repo http://pkg.jenkins-ci.org/redhat-stable/jenkins.repo

2. rpm --import http://pkg.jenkins-ci.org/redhat-stable/jenkins-ci.org.key

3. yum install jenkins

4. service jenkins start


Prerequistitue

Install Java:

https://www.digitalocean.com/community/tutorials/how-to-install-java-on-centos-and-fedora

1. sudo yum install java-1.8.0-openjdk-devel



Install Jenkins YAML:

#This script installs Jenkins

- hosts: all

  tasks:
    - name: Ensure Apache is NOT installed on this server.
      yum:
        name: httpd
        state: absent
      become: yes

    - name: Patch up the server
      yum: name=* state=latest
      become: yes

    - name: Install Jdk 1.8
      yum:
        name: java-1.8.0-openjdk-devel
        state: latest
      become: yes

    - name: Add Jenkins Red Hat Repository
      yum_repository:
        name: Jenkins
        description: Jenkins Repo
        baseurl: http://pkg.jenkins.io/redhat-stable
      become: yes

    - name: Add Jenkins Public Key
      rpm_key:
        key: http://pkg.jenkins-ci.org/redhat-stable/jenkins-ci.org.key
      become: yes

    - name: Install Jenkins
      yum:
        name: jenkins
        state: latest
      become: yes

    - name: Start Jenkins and Enable acrooss Reboots
      service:
        name: jenkins
        state: started
        enabled: yes
      become: yes



** To start an EC2 instance we need to install boto(python program) which is required to run AWS CLI.
Install boto:
sudo pip install boto


If we are starting ec2 instance we need to provide authentication via 
1. environment variables
2. properties in ec2 module

aws_access_key
AWS access key. If not set then the value of the AWS_ACCESS_KEY_ID, AWS_ACCESS_KEY or EC2_ACCESS_KEY environment variable is used.

aws_secret_key
AWS secret key. If not set then the value of the AWS_SECRET_ACCESS_KEY, AWS_SECRET_KEY, or EC2_SECRET_KEY environment variable is used.


exact_count and count_tag work hand in hand to maintain those many instances specified in exact_count with tag name specified in count_tag.

This exact_count and count_tag will not start a new instance if the count is satisfied else it will start instances to match up exact_count and count_tag name 
(this count is for both RUNNING and STOPPED instances)


If exact_count value is 0 then it will TERMINATE all instances with count_tag value.
How to store instance keypair(public certificate) in a session
1. ssh-agent bash
2. ssh-add microservices-course-keypair.pem


Provisioning Ec2 Instance YAML
# Provision an EC2 Instance

- hosts: localhost

  tasks:
    - name: Start up an EC2 instance
      ec2:
        key_name: microservices-course-keypair
        group: Jenkins-sg
        instance_type: t2.micro
        image: ami-0889b8a448de4fc44
        region: ap-south-1
        # wait for the server to come up
        wait: yes
        exact_count: 1
        instance_tags:
          Name: Jenkins Server Automated
        count_tag:
          Name: Jenkins Server Automated
      # Register started instances to ec2 variable
      register: ec2

    - name: Gather New IP Address of Ec2 Instances and add it to a inmemory group
      add_host:
        groups: new_ec2_instances
        hostname: "{{ item.public_ip }}"
      with_items: "{{ ec2.instances }}"

      #If we don't wait for SSH port to respond
      #then in the apply patches step we are trying to
      #do ssh to the new ec2-instance which will fail.
    - name: Wait for SSH Port to respond
      wait_for:
        port: 22
        host: "{{ item.public_ip }}"
        state: started
      with_items: "{{ ec2.instances }}"


- hosts: new_ec2_instances

  tasks:
    - name: Apply Patches
      yum: name=* state=latest
      become: yes



When Ansible Server SSH into brand new EC2 Instance then it will prompt for authenticity of host so that it will add this to the known hosts, how to avoid this prompt?

1. Execute below commands
wget https://raw.githubusercontent.com/ansible/ansible/devel/examples/ansible.cfg
sudo mv ansible.cfg /etc/ansible

2. In ansible.cfg uncomment below line
# uncomment this to disable SSH key host checking
#host_key_checking = False



Dynamic Inventory:
If you see above provisioning yaml file it will run “Apply Patches” only for brand new Ec2 Instances that are created as part of ec2 module, because what’s happening is if there are no instances with count_tag then ec2 module will spin up instances as per the exact_count value and it will add those instances into ec2 variable which in turn adds to new_ec2_instances group.


But if there are instances specified by count_tag running then ec2 module will not spin any instances so no instances will be added to new_c2_instances group, this will skip all tasks present in that hosts node. 

To solve this problem we need to use inventory so that it will run steps even though instances are not created by ec2 module.


1. Run below commands
  wget https://raw.githubusercontent.com/ansible/ansible/devel/contrib/inventory/ec2.py
wget https://raw.githubusercontent.com/ansible/ansible/devel/contrib/inventory/ec2.ini
sudo mv ec2.* /etc/ansible/
sudo chmod +x /etc/ansible/ec2.py

2. Edit your region otherwise it will make rest calls to all your regions.

3. Disable your cache this is not mandatory, but if you are running scripts continuously it is better to disable so that we don’t need to wait for 5 minutes to refresh the cache.

4. ec2.py —list

5. we need to tell ansible to take the hosts from this ec2.py program 
   export ANSIBLE_HOSTS=/etc/ansible/ec2.py


Now our dynamic inventory is ready and it will run the tasks even though brand new ec2 instances are not created by ec2 module.




After Starting Jenkins Server, to build our github projects we need to install Git in Global Tool Configuration by providing the zip file for Git.
But it is better to automate this step by installing Git through Ansible.


Similarly if we run maven goals then we need to install maven in the Global Tool Configuration by providing zip file for maven
But it is better to automate this step by installing Maven through Ansible.

As maven is not part of yum we need to do all manual steps like
download to a dest folder, unpack, link the software so that we can run that command from anywhere.



*** Any Software Tool which is installed through YUM will be available in PATH variable.


To execute Ansible Playbook option in Jenkins we need to install below plugin
Ansible


Installing this plugin will not install ansible software in jenkins server
so we can install this using ansible playbook.
Above plugin provides us a way to execute playbooks in deploy step.

To start Ec2 we need to provide AWS Credentials
to store this securely we can use below plugin
CloudBees AWS Credentials

To ssh into remote ec2 instance we need to provide our pem file
This is achieved by using below plugin.
SSH Agent




Jenkins Pipeline Script:
node {
   stage('Preparation') { 
      git 'https://github.com/Leela-Prasad/fleetman-position-tracker'
   }
   stage('Build') {
        sh "mvn package"
   }
   stage('Results') {
      junit '**/target/surefire-reports/TEST-*.xml'
      archive 'target/*.jar'
   }
}



Use pipeline syntax link to generate syntaxes

node {
   stage('Preparation') { 
      git 'https://github.com/Leela-Prasad/fleetman-position-tracker'
   }
   stage('Build') {
        sh "mvn package"
   }
   stage('Results') {
      junit '**/target/surefire-reports/TEST-*.xml'
      archive 'target/*.jar'
   }
   stage('Deploy') {
      withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: 'AWS Credentials', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
        ansiblePlaybook credentialsId: 'SSH-Credentails', playbook: 'deploy.yaml' 
      }
   }
}




Installing Spring Boot Jar as a Service.
If we want to install spring boot jar as a service then we need to make the jar file as executable one, means the jar file will contain “startup script” at the start of the jar content and appended with other dependency jar contents so that when machine reboot happens this script will be ran and make this service available automatically. For this we have to add a property to the spring-boot-maven-plugin

<plugin>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-maven-plugin</artifactId>
</plugin>

 |
\ /

<plugin>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-maven-plugin</artifactId>
	<configuration>
		<executable>true</executable>
	</configuration>
</plugin>


Ansible script to install spring boot as a service
  - name: Build a link to the executable so it becomes a service
    file:
            src: /home/ec2-user/global-config-server/target/fleetman-global-config-0.0.1-SNAPSHOT.jar
            dest: /etc/init.d/config-server
            state: link
    become: true

  - name: Install Config Server as a startup service
    service:
            name: config-server
            state: started
            enabled: yes
    become: true


The jar will have following contents shown in spring-boot-executable-jar.txt file.

Create an Image from the instance that is brought by provision_global_config_server.yaml file, so that we can give this image to the ASG group to create the instances dynamically.

Config Server ELB configuration
Port mapping
elb 8888 instance 8888
security group config-server-sg (select an existing group)
Health can be configured to /env
Healthy threshhold changed to 2 for development purposes.
Don’t add any instances to LB, ASG will take care.
create a tag with 
Name = config-server-elb (which is used by asg in the ansible script)

** If you want to see log of a service that is registered in /etc/init.d
go to
cd /var/log
tail -f {service}.log


Eureka:
Eureka will run in a peer to peer mode means every eureka server knows other eureka server
so if clients registers with eureka instance 2 then this information will be replicated on other eureka instances present in the application.
For this reason in every eureka instance we need to specify other eureka instances Ip addresses. So definitely we need to use Elastic IP’s for all the eureka instances.

Self Preservation Mode in Eureka:
Eureka is registered with many number of instances and will know the health of the instances through heart beats that will happen every 30 sec.

Self Preservation = OFF:
if for some reason there is a network connectivity problem between instances and Eureka then Eureka will de-register the instances from the registry and this information is replicated across Peer Eureka Instances and the same information is pushed from Eureka to other eureka instances. So in this situation even the micro service is available it will not be able to contact because that information is removed from Eureka.

Self Preservation = ON:
if for some reason there is a large number of instances not sending heart beats then Eureka will enter into Self Preservation Mode means Eureka will display Emergency Notification about this incident on the console and will preserve the instance information  in the registry, hoping that the network connectivity will be resolved and instances will be back.


Generally during development self preservation ON will create confusion as we will be terminating instances and because of this mode instance information is not be removed from registry.
In production we need this to be enabled.

We can also make this situation NOT to happen by setting below property.

eureka.server.renewalPercentThreshold=0.1

Above property tells eureka that if you receive 10% of traffic from the instances then we are good and it will not enter into self preservation mode.



Running Eureka Server With AWS Security:
When you run Eureka Server it will do below 2 things
1. Replicates registry information to another eureka servers.
2. Changes its Dynamic IP Address to Elastic IP Address(i.e., static IP) (This is a intelligent step that Eureka Framework will do for us)

Changing this dynamic ip to static ip is called as binding
and this binding will fail if the user doesn’t have a role with following options
eureka-eip-reallocation:
1. Associate Address
2. Describe Addresses
3. Disassociate Address


** To launch an ec2 instance with a role(in this case eureka-eip-reallocation) we need to give user with following permissions in the policy.
1. PassRole
2. ListInstanceProfiles









Eureka:
To deploy on AWS we need change below properties

old properties running on a single machine:

server.port=8010

eureka.client.register-with-eureka=false
eureka.client.fetch-registry=false

logging.level.com.netflix.eureka=OFF
logging.level.com.netflix.discovery=OFF

# For development
eureka.server.enableSelfPreservation=false

New Properties on AWS:
server.port=8010

#This is needed so that Eureka knows that we are running on AWS EC2.
eureka.datacenter=cloud

# Renewal Threshold states if it receives 10% of traffic then 
# Eureka assumes that the instances are healthy, This is for development purposes
eureka.server.renewalPercentThreshold=0.1

# This Self Preservation we are disabling and it for development purposes.
eureka.server.enableSelfPreservation=false

eureka.client.region=ap-south-1

# list of Eureka Servers with comma separated so that one eureka server will communicate with another eureka server
# and updates/syncs its registry.
# Here YOU MUST use full DNS names of the server
# http://ec2-xxx-xxx-xxxx-xxx.region.compute.amazonaws.com:8010/eureka/
eureka.client.serviceUrl.defaultZone=http://ec2-13-234-198-221.ap-south-1.compute.amazonaws.com:8010/eureka/


Eureka AWS Aware:
If you are running Eureka on AWS then Eureka instance needs to be configured AWS Aware and this can be done by following code.

@Bean
public EurekaInstanceConfigBean eurekaInstanceConfigBean(InetUtils utils) {
	AmazonInfo info = AmazonInfo.Builder.newBuilder().autoBuild("eureka");
	instance.setHostname(info.get(AmazonInfo.MetaDataKey.publicHostname));
	instance.setIpAddress(info.get(AmazonInfo.MetaDataKey.publicIpv4));
	instance.setDataCenterInfo(info);
	instance.setNonSecurePort(port);

	return instance;
}

*** If Eureka is AWS aware then following things will be happened automatically
1. Binds to one of the Elastic IP defined in “eureka.client.serviceUrl.defaultZone” property.
2. Connects to other Eureka instances defined in “eureka.client.serviceUrl.defaultZone” and replicate registry information.

*** If this AWS aware not done then it will not Bind to elastic ip and will be trying to connect to other instances defined in “eureka.client.serviceUrl.defaultZone” and this connection will timeout as eureka is not binding to elastic IP’s
Below is the Exception we will get

2019-04-09 18:00:23.954 ERROR 16693 --- [nfoReplicator-0] c.n.d.s.t.d.RedirectingEurekaHttpClient  : Request execution error

com.sun.jersey.api.client.ClientHandlerException: org.apache.http.conn.ConnectTimeoutException: Connect to ec2-13-234-198-221.ap-south-1.compute.amazonaws.com:8010 timed out
	at com.sun.jersey.client.apache4.ApacheHttpClient4Handler.handle(ApacheHttpClient4Handler.java:187) ~[jersey-apache-client4-1.19.1.jar!/:1.19.1]
	at com.sun.jersey.api.client.filter.GZIPContentEncodingFilter.handle(GZIPContentEncodingFilter.java:123) ~[jersey-client-1.19.1.jar!/:1.19.1]
	at com.netflix.discovery.EurekaIdentityHeaderFilter.handle(EurekaIdentityHeaderFilter.java:27) ~[eureka-client-1.4.11.jar!/:1.4.11]



*** Even Eureka is AWS Aware it doesn’t have enough permissions to automatically bind to an elastic IP, so we will get below exception.

2019-04-10 16:17:03.702  INFO 4528 --- [      Thread-10] com.netflix.eureka.aws.EIPManager        : No EIP is free to be associated with this instance. Candidate EIPs are: [13.234.198.107]
2019-04-10 16:17:03.706 ERROR 4528 --- [      Thread-10] com.netflix.eureka.aws.EIPManager        : Failed to bind elastic IP: 13.234.198.107 to i-0a33729357c4254e9

com.amazonaws.AmazonClientException: The requested metadata is not found at http://169.254.169.254/latest/meta-data/iam/security-credentials/
	at com.amazonaws.internal.EC2CredentialsUtils.readResource(EC2CredentialsUtils.java:80) ~[aws-java-sdk-core-1.11.18.jar!/:na]
	at com.amazonaws.auth.InstanceProfileCredentialsProvider$InstanceMetadataCredentialsEndpointProvider.getCredentialsEndpoint(InstanceProfileCredentialsProvider.java:117) ~[aws-java-sdk-core-1.11.18.jar!/:na]
	at com.amazonaws.auth.EC2CredentialsFetcher.fetchCredentials(EC2CredentialsFetcher.java:121) ~[aws-java-sdk-core-1.11.18.jar!/:na]


*** To bind to an elastic ip we need to define a role that will be attached to the instance
Role: eureka-eip-allocation 
with following options
1. Associate Address
2. Describe Addresses
3. Disassociate Address

** But to launch an instance with a role the user should have permission to pass IAM roles.
So below are following that user need.
1. PassRole
2. ListInstanceProfiles


After Automatic Binding to Elastic IP 
following lines will be logged in the console

Associated i-02c38f79ed05db964 running in zone: ap-south-1a to elastic IP: 13.234.165.150
2019-04-10 17:31:24.187  INFO 13204 --- [      Thread-10] com.netflix.eureka.aws.EIPManager        : My instance i-02c38f79ed05db964 seems to be already associated with the EIP 13.234.165.150
2019-04-10 17:31:24.423  INFO 13204 --- [      Thread-10] com.netflix.eureka.aws.EIPManager        : My instance i-02c38f79ed05db964 seems to be already associated with the EIP 13.234.165.150


This is small issue in Spring Cloud upto Camden Release and it will not be there from Dalton Release
If you go to Eureka home page the instance will still have the dynamic ip which Amazon allocates not the elastic ip, because this information is not refreshed. If we have the stale data then eureka server will not work properly. To avoid this below is the fix which will run timer at a regular interval to refresh instance information.


final EurekaInstanceConfigBean instance = new EurekaInstanceConfigBean(utils)
{
	@Scheduled(initialDelay = 30000L, fixedRate = 30000L)
	public void refreshInfo() {
	AmazonInfo newInfo = AmazonInfo.Builder.newBuilder().autoBuild("eureka");
	if (!this.getDataCenterInfo().equals(newInfo)) {
		((AmazonInfo) this.getDataCenterInfo()).setMetadata(newInfo.getMetadata());
		this.setHostname(newInfo.get(AmazonInfo.MetaDataKey.publicHostname));
		this.setIpAddress(newInfo.get(AmazonInfo.MetaDataKey.publicIpv4));
		this.setDataCenterInfo(newInfo);
		this.setNonSecurePort(port);
	}
	}         
};



How to stop a service in Linux:
systemctl stop fleetman-registry

sudo service fleetman-registry restart



Active MQ:
yaml is written to download active mq from internet and start an instance.
ASG will take care of adding this instance to ELB.

Position Tracker:
In this course we are launching micro services through Jenkins.
only Config Server, Eureka and Active MQ are launched using ansible playbooks.
To launch micro service using jenkins we need to 2 files.
1. Jenkins file - which will downloads code from github and builds the artefact.
2. deploy.yaml - which will provision a new instance and deploys the artefact.


Position Tracker needs to know info about mq and eureka
and these info is present in config server, so this micro service needs to know where config server is located.

Below file needs to be there in resources folder

bootstrap.properties:    
spring.application.name=fleetman-position-tracker
eureka.instance.instanceId=${spring.application.name}:${random.value}
spring.cloud.config.uri=http://config-server-elb-1787181940.ap-south-1.elb.amazonaws.com:8888


And change the server.port to 8080 as we are checking for 8080 port is up or not.



webapp:
To launch web using jenkins we need to 2 files.
1. Jenkins file - which will downloads code from github and builds the artefact.
2. deploy.yaml - which will provision a new instance and deploys the artefact.


webapp needs to know info about other micro services(position tracker) and this info can be obtained from eureka and this info is present in config server, so this micro service needs to know where config server is located.

Below file needs to be there in resources folder

bootstrap.properties:  
spring.application.name=fleetman-webapp
eureka.instance.instanceId=${spring.application.name}:${random.value}
spring.cloud.config.uri=http://config-server-elb-1787181940.ap-south-1.elb.amazonaws.com:8888

Config Server:
we need to update mq and eureka locations in config server.

spring.activemq.broker-url=tcp://active-mq-elb-276517005.ap-south-1.elb.amazonaws.com:61616
eureka.client.serviceUrl.defaultZone=http://ec2-13-127-31-20.ap-south-1.compute.amazonaws.com:8010/eureka/,http://ec2-13-234-199-167.ap-south-1.compute.amazonaws.com:8010/eureka/
fleetman.position.queue=positionQueueNew






In this course we will launch ASG as follows:
1. we will run an instance using ansible playbook, once everything looks good then we will take a image of that instance.
2. We will mention this Custom AMI instance id in asg ansible playbook so that it will not build instance from sractch.

