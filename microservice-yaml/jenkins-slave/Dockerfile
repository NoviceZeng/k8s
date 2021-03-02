FROM centos:7

RUN yum install -y java-1.8.0-openjdk maven curl git libtool-ltdl-devel && \ 
    yum clean all && \
    rm -rf /var/cache/yum/* && \
    mkdir -p /usr/share/jenkins

COPY slave.jar /usr/share/jenkins/slave.jar  
COPY jenkins-slave /usr/bin/jenkins-slave
RUN chmod +x /usr/bin/jenkins-slave
RUN echo '10.70.128.55   test.viba.club' > /etc/hosts
COPY helm kubectl /usr/bin/

ENTRYPOINT ["jenkins-slave"]
