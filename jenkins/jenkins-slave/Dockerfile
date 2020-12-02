FROM centos:7

RUN yum install -y java-1.8.0-openjdk maven curl git libtool-ltdl-devel && \ 
    yum clean all && \
    rm -rf /var/cache/yum/* && \
    mkdir -p /usr/share/jenkins

COPY slave.jar /usr/share/jenkins/slave.jar  
COPY jenkins-slave /usr/bin/jenkins-slave
RUN chmod +x /usr/bin/jenkins-slave
COPY helm kubectl /usr/bin/
COPY settings.xml /etc/maven/settings.xml
ENTRYPOINT ["jenkins-slave"]
