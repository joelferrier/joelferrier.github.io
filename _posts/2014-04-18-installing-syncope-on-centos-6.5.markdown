---
layout: post
title: "Installing Syncope on centos 6.5"
date: 2014-02-24 10:32:00
categories: centos
---

---
### Install Packages/Maven
- Install packages

{% highlight bash %}
sudo yum install -y vim-enhanced unzip wget java-1.7.0-openjdk mysql mysql-server 
{% endhighlight %}

- Download and link Apache Maven

{% highlight bash %}
wget http://apache.cs.utah.edu/maven/maven-3/3.2.1/binaries/apache-maven-3.2.1-bin.tar.gz -P /tmp
sudo tar -xvzf /tmp/apache-maven-3.2.1-bin.tar.gz -C /usr/local
cd /usr/local
sudo ln -s apache-maven-3.2.1/bin/mvn /usr/bin/mvn
{% endhighlight %}

- Create a file "maven.sh" in the folder /etc/profile.d that contains:

{% highlight bash %}
export M2_HOME=/usr/local/maven
export PATH=${M2_HOME}/bin:${PATH}
{% endhighlight %}

- logout and back in again for the exports to take effect

---
### Install Tomcat

- download tomcat

{% highlight bash %}
wget http://apache.mirrors.pair.com/tomcat/tomcat-7/v7.0.53/bin/apache-tomcat-7.0.53.tar.gz -P /tmp
sudo tar -xzvf /tmp/apache-tomcat-7.0.53.tar.gz -C /opt
{% endhighlight %}

- create an init script "tomcat" in /etc/init.d that contains:

{% highlight bash lineno %}
#!/bin/bash
# Description: Tomcat Start Stop Restart
# processname: tomcat
# chkconfig: 234 20 80
JAVA_HOME=/usr/lib/jvm/java-1.7.0
export JAVA_HOME
PATH=${JAVA_HOME}/bin:${PATH}
export PATH
CATALINA_HOME=/opt/apache-tomcat-7.0.53

case $1 in
start)
sh ${CATALINA_HOME}/bin/startup.sh
;;
stop)
sh $CATALINA_HOME}/bin/shutdown.sh
;;
restart)
sh ${CATALINA_HOME}/bin/startup.sh
sh ${CATALINA_HOME}/bin/shutdown.sh
;;
esac
exit 0
{% endhighlight %}


- chmod tomcat startup scripts


{% highlight bash %}
sudo chmod +x /opt/apache-tomcat-7.0.53/bin/*.sh
{% endhighlight %}

- create iptables rules for tomcat

{% highlight bash %}
sudo iptables -A INPUT -p tcp --dport 8080 -j ACCEPT
sudo /sbin/service iptables save
{% endhighlight %}


- create service account, edit conf/tomcat-users.xml and add:

<role rolename="admin-gui" />
<user username="admin" password="${PASSWORD}" roles="manager-gui,admin-gui" />

---
###Configure mysql
- start mysql

{% highlight bash %}
sudo iptables -A INPUT -p tcp --dport 8080 -j ACCEPT
sudo service mysqld start
{% endhighlight %}

- run the mysql secure installation script

{% highlight bash %}
sudo iptables -A INPUT -p tcp --dport 8080 -j ACCEPT
sudo /usr/bin/mysql_secure_installation
{% endhighlight %}

The default root password is blank, change it, and disallow anonymous users and remote root login. Finally remove the test database

- set mysql to run at boot

{% highlight bash %}
sudo iptables -A INPUT -p tcp --dport 8080 -j ACCEPT
sudo chkconfig --levels 235 mysqld on
{% endhighlight %}

- create syncope database

{% highlight bash %}
mysql -u root -p
{% endhighlight %}

{% highlight mysql %}
mysql> create database syncope
mysql> create user 'syncope'@'localhost' identified by '${syncope_password}';
mysql> grant all privileges on syncope.* to 'syncope'@'localhost';
mysql> exit
{% endhighlight %}

---
###Install Syncope
- download syncope project and move it to opt

{% highlight bash %}
cd ~/
sudo mvn archetype:generate \
    -DarchetypeGroupId=org.apache.syncope \
    -DarchetypeArtifactId=syncope-archetype \
    -DarchetypeRepository=http://repo1.maven.org/maven2 \
    -DarchetypeVersion=1.1.6
    mv syncope-enterprise-bus /opt/
{% endhighlight %}

- When prompted supply:

1. a groupId like "org.orshakes" 
2. an artifactId like "syncope-enterprise-bus" 
3. a version number like "SNAPSHOT-0.1" (if still in development)
4. a packageName matching the groupid
5. a random string of 16 characters


- Create two directories in the syncope-enterprise-bus directory, logs and bundles

- download connID bundles and place them in the bundles directory

{% highlight bash %}
sudo wget http://central.maven.org/maven2/org/connid/bundles/org.connid.bundles.ldap/1.3.6/org.connid.bundles.ldap-1.3.6.jar -P /opt/syncope-enterprise-bus/bundles
sudo wget http://central.maven.org/maven2/org/connid/bundles/org.connid.bundles.flatfile/1.2/org.connid.bundles.flatfile-1.2.jar -P /opt/syncope-enterprise-bus/bundles
sudo wget http://central.maven.org/maven2/org/connid/bundles/org.connid.bundles.ad/1.1.2/org.connid.bundles.ad-1.1.2.jar -P /opt/syncope-enterprise-bus/bundles
{% endhighlight %}

- Configure syncope storage edit `${syncope_install_dir}/core/src/main/resources/persistence.properties` Replace the current database configuration with

{% highlight text %}
jpa.driverClassName=com.mysql.jdbc.Driver
jpa.url=jdbc:mysql://localhost:3306/syncope?characterEncoding=UTF-8
jpa.username=syncope
jpa.password=S{syncopepass}
jpa.dialect=org.apache.openjpa.jdbc.sql.MySQLDictionary
quartz.jobstore=org.quartz.impl.jdbcjobstore.StdJDBCDelegate
quartz.sql=tables_mysql.sql
logback.sql=mysql.sql
{% endhighlight %}

ensure that syncope pass has the same value as what you entered into mysql for the syncope user account

- Edit `${syncope_install_dir}/core/src/main/webapp/WEB-INF/web.xml` and uncomment resource-ref section

- Configure internal storage with tomcat: edit the file ${Catalina_Home}/conf/context.xml to contain the following:

{% highlight xml linenos %}
<Resource name="jdbc/syncopeDataSource" auth="Container"
type="javax.sql.DataSource"
factory="org.apache.tomcat.jdbc.pool.DataSourceFactory"
testWhileIdle="true" testOnBorrow="true" testOnReturn="true"
validationQuery="SELECT 1" validationInterval="30000"
maxActive="50" minIdle="2" maxWait="10000" initialSize="2"
removeAbandonedTimeout="20000" removeAbandoned="true"
logAbandoned="true" suspectTimeout="20000"
timeBetweenEvictionRunsMillis="5000" minEvictableIdleTimeMillis="5000"
jdbcInterceptors="org.apache.tomcat.jdbc.pool.interceptor.ConnectionState;
org.apache.tomcat.jdbc.pool.interceptor.StatementFinalizer"
username="syncope" password="${syncopepass}"
driverClassName="com.mysql.jdbc.Driver"
url="jdbc:mysql://localhost:3306/syncope?characterEncoding=UTF-8"/>
{% endhighlight %}

ensure that syncopepass has the same value as what you entered into mysql for the syncope user account

- uncomment the line "<Manager pathname= "" /> in the file above
- download the JDBC driver from the following url and place it in tomcat's lib directory

{% highlight bash %}
wget https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-java-5.1.30.tar.gz
{% endhighlight %}

---
###Build Syncope

- create a complete syncope project with the following command

{% highlight bash %}
sudo mvn clean package -Dbundles.directory=/opt/syncope-enterprise-bus/bundles -Dlog.directory=/opt/syncope-enterprise-bus/logs
{% endhighlight %}

---
###Configure Admin Account

- After successfully building edit core/src/main/security.properties to change administrator credentials. The default username/pass is admin/password
- passwords are stored as an md5sum of the cleartext password, obtain the hash of a new password with the following command

{% highlight bash %}
echo -n "new password" | md5sum
{% endhighlight %}

---
###Deploy Syncope

- copy the files `core/target/syncope.war` and `console/target/syncope-console.war` to tomcat's webapps directory

- tomcat will automagically deploy all of the files required for syncope-core and syncope-console when it is started

{% highlight bash %}
sudo service tomcat start
{% endhighlight %}

- login to the tomcat manager-gui to ensure that both syncope console and syncope core have started
- connect to `http://yourtomcatserver:8080/sycnope-console` to begin using syncope

