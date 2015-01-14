This document will tell you how to deploy the whole open-hackathon web application environment manually.    
And the whole environment cotains serval components such guacamole , nginx , tomcat and the open-hackathon running Dependencies. 

#install components
```shell
sudo apt-get update && sudo apt-get upgrade
sudo add-apt-repository ppa:nginx/stable

sudo apt-get install build-essential python python-dev python-setuptools libmysqlclient-dev
sudo apt-get install guacamole libguac-dev libcairo-dev libvncserver0 libguac-client-ssh0 libguac-client-vnc0 

sudo apt-get install vim git
sudo easy_install pip

sudo apt-get install tomcat7
sudo apt-get install openjdk-7-jdk
sudo apt-get install nginx
sudo apt-get install mysql-server

sudo pip install virtualenv
sudo pip install uwsgi
```
#Get src from github
```
cd /opt/
git clone https://github.com/msopentechcn/open-hackathon.git
```

#config guacamole
Check the guacamole config file `/etc/guacamole.properties`, and edit the file like this:
```shell
guacd-hostname: localhost
guacd-port:     4822

lib-directory: /var/lib/guacamole
auth-provider: com.openhackathon.guacamole.OpenHackathonAuthenticationProvider
auth-request-url: http://osslab.msopentech.cn/checkguacookies
```
Then copy the auth-provider jar file to the path that was setted in the config file
```
sudo cp deploy/openhackathon-gucamole-authentication-1.0-SNAPSHOT.jar /var/lib/guacamole/
```
Note: the `auth-request-url` value must be setted match the _open-hackathon_ src provides
And every time you change this file , you may need to restart guacd and tomcat7 service

#config tomcat7
After installed tomcat7 we need to make tomcat load the guacamole-UI web application. The guacamole.war was provided after we install guacamole commponent.     
So we can config tomcat7 like these steps:
```
sudo ln -s /var/lib/tomcat7/conf /usr/share/tomcat7/conf
sudo ln -s /var/lib/tomcat7/webapps /usr/share/tomcat7/webapps
sudo cp /var/lib/guacamole/guacamole.war /usr/share/tomcat7/webapps/

sudo mkdir /usr/share/tomcat7/.guacamole
sudo ln -s /etc/guacamole/guacamole.properties /usr/share/tomcat7/.guacamole/guacamole.properties

sudo service guacd restart
sudo service tomcat7 restart
```

#Deploy open-hackathon withn Nginx

Before deploy the web application ,we need to setup all the dependencies and do some pre-execution
```
echo "127.0.0.1    osslab.msopentech.cn" >> /etc/hosts
sudo mkdir /var/log/uwsgi
sudo mkdir /var/log/open-hackathon
cd /opt/open-hackathon
virtualenv venv
source venv/bin/activate
cd /opt/open-hackathon/open-hackathon/
sudo pip install -r requirement.txt
```
####- confi mysql
edit `/etc/mysql/my.conf` make changes like this:
```shell
[client]
default-character-set=utf8

[mysqld]
default-storage-engine=INNODB
character-set-server=utf8
collation-server=utf8_general_ci
```
Then restart the `mysqld` service     

Next logon mysql console with root user(mysql -u root -p) and then:
```mysql
create database hackathon;
create User 'hackathon'@'localhost' IDENTIFIED by 'hackathon';
GRANT ALL on hackathon.* TO 'hackathon'@'localhost';
```
Next update `app/config.py` with your user/password. And don't submit your password to github!!!

Then we should initialize tables and creat test data:
```
sudo python /opt/open-hackathon/open-hackathon/src/setup_db.py
mysql -u root -p
use hackathon;
insert into register (register_name, email, submitted, enabled) values("Your Name", "xxx@abc.com", 0, 1);
```
####- deploy withn nginx
To deploy the web application in nginx service , we need to provide:            
- a conf file to let nginx load it; 
- a uWsgi to hold the python-flask web application;    - 

To make it to be easily operational, we would setup a linux demo to hanld the uwsgi process           

What we need is already putted in the `deploy` folder, So we can do it like this:
```
sudo rm -rf /etc/nginx/sites-enabled/default
sudo cp deploy/nginx_openhackathon.conf /etc/nginx/conf.d/
sudo cp deploy/nginx_openhackathon.uwsgi.ini ../
sudo cp deploy/uwsgi /etc/init.d/
```
Note: Please check the three files carefully , especially about the paths !!!           
Then we would start and restart these services
```
sudo /etc/init.d/guacd restart
sudo /etc/init.d/tomcat7 restart
sudo /etc/init.d/uwsgi start
sudo /etc/init.d/nginx restart
```
# Test the Web-application
After finishsd those steps you could do some tests on local machine                     
check guacd service:[http://localhost:8080/guacamole](http://localhost:8080/guacamole)                     
check nginx service for proxy forwarding:[http://osslab.msopentech.cn/guacamole](http://localhost:8080/guacamole)     
check open-hackathon web application:[http://localhost](http://localhost:8080/guacamole)                   


#Python IDE
Get "PyCharm" from [https://www.jetbrains.com/pycharm/](https://www.jetbrains.com/pycharm/) freely on your localhost     
Extract the download `pycharm-community-4.0.4.tar.gz` , execuse `pycharm-community-4.0.4/bin/pycharm.sh`can open this software.
Then we can develop or debug the open-hackathon

# setup cloudVM service

####download dependencies 
```
sudo apt-get install docker.io
sudo pip install -r /opt/open-hackathon/cloudvm/requestment.txt
```
####download docker images 
```
sudo docker pull 42.159.103.213:5000/rails
sudo docker pull 42.159.103.213:5000/mean
sudo docker pull 42.159.103.213:5000/ubuntu-sshd
sudo docker pull sffamily/ubuntu-gnome-vnc-eclipse
```
####config docker remote api
If you want to use docker remote api to visit docker on your host machine or Azure server, please change the following configure:          
Please edit this file: /etc/init/docker.conf or /etc/default/docker and update the DOCKER_OPTS variable to the following:
```
DOCKER_OPTS = '-H tcp://0.0.0.0:4243 -H unix:///var/run/docker.sock'
```
The daemon process will listen on port '4243', if '4243' port has been occupied on the machine which you want to visit, please change it. 
And add user into docker gourp 
```
sudo groupadd docker
sudo gpasswd -a ${USER} docker
```
Then restart the docker process: `service docker.io restart`

#Debug the web application