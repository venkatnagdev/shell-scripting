#!/bin/bash
### Wriiten by Sivakumar
#### 27-APR-2018

LOG=/tmp/stack.log
MOD_JK_URL=http://www-eu.apache.org/dist/tomcat/tomcat-connectors/jk/tomcat-connectors-1.2.43-src.tar.gz
MOD_JK_TAR_FILE=$(echo $MOD_JK_URL | cut -d / -f8) #echo $MOD_JK_URL | awk -F / '{print $NF}'
MOD_JK_DIR=$(echo $MOD_JK_TAR_FILE | sed -e 's/.tar.gz//')
R="\e[31m"
G="\e[32m"
Y="\e[33m"
N="\e[0m"

ID=$(id -u)

if [ $ID -ne 0 ];then
	echo "You are not root user, you dont' have priveleges to run this script"
	exit 2
fi

VALIDATE(){

if [ $1 -ne 0 ]; then
	echo -e "$2 ... $R FAILED $N"
	exit 2
else
	echo -e "$2... $G SUCCESS $N"
fi
}

SKIP(){
	echo -e "$1 ...$Y SKIPPING $N"
}


yum install httpd gcc httpd-devel java -y &>>$LOG

VALIDATE $? "Installing web server, gcc, httpd-devel"

systemctl start httpd &>>$LOG

VALIDATE $? "Starting the web server"

if [ -f /opt/$MOD_JK_TAR_FILE ];then
	SKIP "Downloading MOD_JK"
else
	wget $MOD_JK_URL -O /opt/$MOD_JK_TAR_FILE &>>$LOG
	VALIDATE $? "Downloading MOD_JK"
fi

cd /opt

if [ -d $MOD_JK_DIR ]; then
	SKIP "Extracting MOD_JK"
else
	tar -xf $MOD_JK_TAR_FILE &>>$LOG
	VALIDATE $? "Extracting MOD_JK"
fi

cd $MOD_JK_DIR/native

if [ -f /etc/httpd/modules/mod_jk.so ]; then
	SKIP "Compiling MOD_JK"
else
	./configure --with-apxs=/bin/apxs &>>$LOG && make &>>$LOG && make install &>>$LOG
	VALIDATE $? "Compiling MOD_JK"
fi

cd /etc/httpd/conf.d

if [ -f modjk.conf ]; then
	SKIP "Creating modjk.conf"
else
	echo 'LoadModule jk_module modules/mod_jk.so
	JkWorkersFile conf.d/workers.properties
	JkLogFile logs/mod_jk.log
	JkLogLevel info
	JkLogStampFormat "[%a %b %d %H:%M:%S %Y]"
	JkOptions +ForwardKeySize +ForwardURICompat -ForwardDirectories
	JkRequestLogFormat "%w %V %T"
	JkMount /student tomcatA
	JkMount /student/* tomcatA' > modjk.conf																																
    VALIDATE $? "creating modjk.conf"
fi

if [ -f workers.properties ];then
	SKIP "Creating workers.properties"
else
	echo '### Define workers
	worker.list=tomcatA
	### Set properties
	worker.tomcatA.type=ajp13
	worker.tomcatA.host=localhost
	worker.tomcatA.port=8009' > workers.properties
    VALIDATE $? "creating workers.properties"
fi

systemctl restart httpd

VALIDATE $? "Restarting WebServer"


TOMCAT_URL=$(curl -s https://tomcat.apache.org/download-90.cgi | grep -A 20 Core: |  grep nofollow | grep tar.gz | cut -d '"' -f2)
TOMCAT_TAR_FILE=$(echo $TOMCAT_URL | awk -F / '{print $NF}') #echo $MOD_JK_URL | awk -F / '{print $NF}'
TOMCAT_DIR=$(echo $TOMCAT_TAR_FILE | sed -e 's/.tar.gz//')
STUDENT_WAR_URL=https://github.com/devops2k18/DevOpsApril/raw/master/student.war
MYSQL_DRIVER_URL=https://github.com/devops2k18/DevOpsApril/raw/master/APPSTACK/mysql-connector-java-5.1.40.jar
MYSQL_DRIVER=$(echo $MYSQL_DRIVER_URL | awk -F / '{print $NF}')

cd /opt

if [ -f $TOMCAT_TAR_FILE ];then
	SKIP "Downloading TOMCAT"
else
	wget $TOMCAT_URL &>>$LOG
	VALIDATE $? "Downloading TOMCAT"
fi

if [ -d $TOMCAT_DIR ]; then
	SKIP "Extracting TOMCAT"
else
	tar -xf $TOMCAT_TAR_FILE &>>$LOG
	VALIDATE $? "Extracting TOMCAT"
fi

cd $TOMCAT_DIR/webapps

rm -rf * ;

VALIDATE $? "Removing old artifacts"

wget $STUDENT_WAR_URL &>>$LOG

VALIDATE $? "Downloading student.war"

cd ../lib

if [ -f $MYSQL_DRIVER ];then
	SKIP "Downloading MYSQL Driver"
else
	wget $MYSQL_DRIVER_URL &>>$LOG
	VALIDATE $? "Downloading MYSQL Driver"
fi

cd ../conf

sed -i -e '/TestDB/d' context.xml

VALIDATE $? "deleting the testdb line"

sed -i -e '$i <Resource name="jdbc/TestDB" auth="Container" type="javax.sql.DataSource" maxTotal="100" maxIdle="30" maxWaitMillis="10000" username="student" password="student@1" driverClassName="com.mysql.jdbc.Driver" url="jdbc:mysql://localhost:3306/studentapp"/>' context.xml

VALIDATE $? "Updating the context.xml"

cd ../bin

sh shutdown.sh &>>$LOG

sh startup.sh &>>$LOG

VALIDATE $? "Starting the tomcat"

yum install mariadb mariadb-server -y &>>$LOG

VALIDATE $? "Installing mariadb"

systemctl start mariadb &>>$LOG

systemctl enable mariadb &>>$LOG

echo "create database if not exists studentapp;
use studentapp;
CREATE TABLE if not exists Students(student_id INT NOT NULL AUTO_INCREMENT,
	student_name VARCHAR(100) NOT NULL,
    student_addr VARCHAR(100) NOT NULL,
	student_age VARCHAR(3) NOT NULL,
	student_qual VARCHAR(20) NOT NULL,
	student_percent VARCHAR(10) NOT NULL,
	student_year_passed VARCHAR(10) NOT NULL,
	PRIMARY KEY (student_id)
);
grant all privileges on studentapp.* to 'student'@'localhost' identified by 'student@1';" > /tmp/student.sql

VALIDATE $? "Creating student.sql"

mysql < /tmp/student.sql

VALIDATE $? "configure mariadb"
