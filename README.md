# Big-Data


Esse tutorial é sobre a criação de uma imagem do Docker com a configuração local do Hadoop, HBase e Kafka. Nesse procedimento, o Hadoop é configurado no modo pseudo-distribuído com cada serviço rodando em uma instância própria da JVM, mas todas na mesma máquina. O HBase e o Kafka também rodam em modo ‘distribuído’ compartilhando uma instância separada do ZooKeeper. Esse procedimento é muito útil para testar funcionalidades desses serviços e aprendizado, mas não é uma solução completa para uso em produção.

Pré-requisito
Nesse procedimento, é necessário que o Docker esteja instalado e funcionando; também é necessário acesso à Internet.

Originalmente, esse procedimento foi testado no ArchLinux atualizado até final de Agosto/2015.

https://wiki.archlinux.org/index.php/Docker

sudo docker version

> Client:
>  Version:      1.8.1
>  API version:  1.20
>  Go version:   go1.4.2
>  Git commit:   d12ea79
>  Built:        Sat Aug 15 17:29:10 UTC 2015
>  OS/Arch:      linux/amd64
>
> Server:
>  Version:      1.8.1
>  API version:  1.20
>  Go version:   go1.4.2
>  Git commit:   d12ea79
>  Built:        Sat Aug 15 17:29:10 UTC 2015
>  OS/Arch:      linux/amd64
Configuração
Hadoop, ZooKeeper, HBase e Kafka.

Container
Começamos com a criação de um conainer do Docker com a imagem do CentOS6.

Importante: para os endereços com grandesdados-hadoop funcionarem fora do container, direto na máquina host, é necessário colocar no /etc/hosts da máquina host o endereço IP do container do Docker para esse nome.

Ao executar o comando run, o Docker automaticamente fará o download da imagem e a shell será inicializada dentro de um novo container.

sudo docker run -i -t --name=grandesdados-hadoop --hostname=grandesdados-hadoop centos:6 /bin/bash

> Unable to find image 'centos:6' locally
> 6: Pulling from library/centos
>
> f1b10cd84249: Pull complete
> fb9cc58bde0c: Pull complete
> a005304e4e74: Already exists
> library/centos:6: The image you are pulling has been verified. Important: image verification is a tech preview feature and should not be relied on to provide security.
>
> Digest: sha256:25d94c55b37cb7a33ad706d5f440e36376fec20f59e57d16fe02c64698b531c1
> Status: Downloaded newer image for centos:6
> [root@grandesdados-hadoop /]#
Já dentro do container criamos um usuário e local que serão usados para a instalação e execução dos processos.

adduser -m -d /hadoop hadoop
cd hadoop
A versão usada nesse procedimento é o Java 8, atual versão estável da Oracle.

curl -k -L -H "Cookie: oraclelicense=accept-securebackup-cookie" -O http://download.oracle.com/otn-pub/java/jdk/8u60-b27/jdk-8u60-linux-x64.rpm
rpm -i jdk-8u60-linux-x64.rpm

echo 'export JAVA_HOME="/usr/java/jdk1.8.0_60"' > /etc/profile.d/java.sh
source /etc/profile.d/java.sh

echo $JAVA_HOME

> /usr/java/jdk1.8.0_60

java -version

> java version "1.8.0_60"
> Java(TM) SE Runtime Environment (build 1.8.0_60-b27)
> Java HotSpot(TM) 64-Bit Server VM (build 25.60-b23, mixed mode)
Para completar o ambiente de execução, instalamos os serviços e bibliotecas necessárias.

yum install -y tar openssh-clients openssh-server rsync gzip zlib openssl fuse bzip2 snappy

> (...)
(configuração do SSH para acesso sem senha)

service sshd start
chkconfig sshd on

su - hadoop

ssh-keygen -C hadoop -P '' -f ~/.ssh/id_rsa
cp ~/.ssh/{id_rsa.pub,authorized_keys}

ssh-keyscan grandesdados-hadoop >>  ~/.ssh/known_hosts
ssh-keyscan localhost >> ~/.ssh/known_hosts
ssh-keyscan 127.0.0.1 >> ~/.ssh/known_hosts
ssh-keyscan 0.0.0.0 >> ~/.ssh/known_hosts

ssh grandesdados-hadoop

> Warning: Permanently added the RSA host key for IP address '172.17.0.12' to the list of known hosts.
> (nova shell, sem login nem confirmação)

# (sair do shell do ssh)
exit
# (sair do shell do su)
exit

whoami

> root
Hadoop
Procedimento para configuração local do Hadoop em modo pseudo-distribuído com uma JVM por serviço.

Esse procedimento é baseado na documentação do Hadoop.

Serviços:

HDFS: NameNode, SecondaryNameNode, DataNode
YARN: ResouceManager, NodeManager
MR: HistoryServer
…

Instalação

O pacote usado nesse procedimento é o Hadoop 2.7.1 para CentOS6 descrito outro artigo.

Primeiramente, colocamos o pacote dentro do container.

# (shell fora do container)
sudo docker cp hadoop-2.7.1.tar.gz grandesdados-hadoop:/hadoop
De volta ao container como usuário root.

tar zxf hadoop-2.7.1.tar.gz -C /opt
chown hadoop:hadoop -R /opt/hadoop-2.7.1

echo 'export PATH=$PATH:/opt/hadoop-2.7.1/bin:/opt/hadoop-2.7.1/sbin' > /etc/profile.d/hadoop.sh
source /etc/profile.d/hadoop.sh

hadoop version

> Hadoop 2.7.1
> Subversion Unknown -r Unknown
> Compiled by hadoop on 2015-09-01T00:30Z
> Compiled with protoc 2.5.0
> From source with checksum fc0a1a23fc1868e4d5ee7fa2b28a58a
> This command was run using /opt/hadoop-2.7.1/share/hadoop/common/hadoop-common-2.7.1.jar

mkdir -p /data/hadoop
chown hadoop:hadoop /data/hadoop
Configuração

(para a configuração, deve ser usado o usuário hadoop: su - hadoop)

Editar /opt/hadoop-2.7.1/etc/hadoop/core-site.xml:

<configuration>
    <property>
        <name>hadoop.tmp.dir</name>
        <value>/data/hadoop</value>
    </property>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://grandesdados-hadoop</value>
    </property>
</configuration>
Editar /opt/hadoop-2.7.1/etc/hadoop/hdfs-site.xml:

<configuration>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
    <property>
        <name>dfs.blocksize</name>
        <value>8M</value>
    </property>
</configuration>
Editar /opt/hadoop-2.7.1/etc/hadoop/yarn-site.xml:

<configuration>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
</configuration>
Editar /opt/hadoop-2.7.1/etc/hadoop/mapred-site.xml:

<configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
    <property>
        <name>mapreduce.jobtracker.staging.root.dir</name>
        <value>/user</value>
    </property>
</configuration>
Setup Inicial (antes da primeira inicialização).

hdfs namenode -format

> 15/09/16 02:12:03 INFO namenode.NameNode: STARTUP_MSG:
> /************************************************************
> STARTUP_MSG: Starting NameNode
> STARTUP_MSG:   host = grandesdados-hadoop/172.17.0.12
> STARTUP_MSG:   args = [-format]
> STARTUP_MSG:   version = 2.7.1
> (...)
> INFO namenode.NameNode: createNameNode [-format]
> Formatting using clusterid: CID-5daa32a0-3ab6-405e-bfd2-05c0a6e1e7e6
> (...)
> INFO common.Storage: Storage directory /data/hadoop/dfs/name has been successfully formatted.
> (...)
HDFS

(como usuário hadoop su - hadoop)

start-dfs.sh

> Starting namenodes on [grandesdados-hadoop]
> grandesdados-hadoop: starting namenode, logging to /opt/hadoop-2.7.1/logs/hadoop-hadoop-namenode-grandesdados-hadoop.out
> localhost: starting datanode, logging to /opt/hadoop-2.7.1/logs/hadoop-hadoop-datanode-grandesdados-hadoop.out
> Starting secondary namenodes [0.0.0.0]
> 0.0.0.0: starting secondarynamenode, logging to /opt/hadoop-2.7.1/logs/hadoop-hadoop-secondarynamenode-grandesdados-hadoop.out

# criação do diretório do usuário hadoop
hdfs dfs -mkdir -p /user/hadoop
Interface Web do Name Node:

http://grandesdados-hadoop:50070/

Interface Web do Data Node (vazia):

http://grandesdados-hadoop:50075/

Interface Web do Secondary Name Node:

http://grandesdados-hadoop:50090/

Para parar o serviço:

stop-dfs.sh
YARN

(como usuário hadoop su - hadoop)

start-yarn.sh

> starting yarn daemons
> starting resourcemanager, logging to /opt/hadoop-2.7.1/logs/yarn-hadoop-resourcemanager-grandesdados-hadoop.out
> localhost: starting nodemanager, logging to /opt/hadoop-2.7.1/logs/yarn-hadoop-nodemanager-grandesdados-hadoop.out
Interface Web do Resource Manager:

http://grandesdados-hadoop:8088/

Interface Web do Node Manager:

http://grandesdados-hadoop:8042/

Para parar o serviço:

stop-yarn.sh
History Server

(como usuário hadoop su - hadoop)

mr-jobhistory-daemon.sh start historyserver

> starting historyserver, logging to /opt/hadoop-2.7.1/logs/mapred-hadoop-historyserver-grandesdados-hadoop.out
Interface Web do History Server:

http://grandesdados-hadoop:19888/

Para parar o serviço:

mr-jobhistory-daemon.sh stop historyserver
Teste

(para os testes, deve ser usado o usuário hadoop: su - hadoop)

Processos:

ps x

>   PID TTY      STAT   TIME COMMAND
>  5162 ?        S      0:00 -bash
>  5327 ?        Sl     0:08 /usr/java/jdk1.8.0_60/bin/java -Dproc_namenode (...)
>  5423 ?        Sl     0:07 /usr/java/jdk1.8.0_60/bin/java -Dproc_datanode (...)
>  5612 ?        Sl     0:06 /usr/java/jdk1.8.0_60/bin/java -Dproc_secondarynamenode (...)
>  5772 ?        Sl     0:08 /usr/java/jdk1.8.0_60/bin/java -Dproc_resourcemanager (...)
>  5870 ?        Sl     0:07 /usr/java/jdk1.8.0_60/bin/java -Dproc_nodemanager (...)
>  6189 ?        Sl     0:08 /usr/java/jdk1.8.0_60/bin/java -Dproc_historyserver (...)
>  6273 ?        R+     0:00 ps x
Exemplos de MapReduce:

yarn jar /opt/hadoop-2.7.1/share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.1.jar

> An example program must be given as the first argument.
> Valid program names are:
>   aggregatewordcount: An Aggregate based map/reduce program that counts the words in the input files.
>   aggregatewordhist: An Aggregate based map/reduce program that computes the histogram of the words in the input files.
>   bbp: A map/reduce program that uses Bailey-Borwein-Plouffe to compute exact digits of Pi.
>   dbcount: An example job that count the pageview counts from a database.
>   distbbp: A map/reduce program that uses a BBP-type formula to compute exact bits of Pi.
>   grep: A map/reduce program that counts the matches of a regex in the input.
>   join: A job that effects a join over sorted, equally partitioned datasets
>   multifilewc: A job that counts words from several files.
>   pentomino: A map/reduce tile laying program to find solutions to pentomino problems.
>   pi: A map/reduce program that estimates Pi using a quasi-Monte Carlo method.
>   randomtextwriter: A map/reduce program that writes 10GB of random textual data per node.
>   randomwriter: A map/reduce program that writes 10GB of random data per node.
>   secondarysort: An example defining a secondary sort to the reduce.
>   sort: A map/reduce program that sorts the data written by the random writer.
>   sudoku: A sudoku solver.
>   teragen: Generate data for the terasort
>   terasort: Run the terasort
>   teravalidate: Checking results of terasort
>   wordcount: A map/reduce program that counts the words in the input files.
>   wordmean: A map/reduce program that counts the average length of the words in the input files.
>   wordmedian: A map/reduce program that counts the median length of the words in the input files.
>   wordstandarddeviation: A map/reduce program that counts the standard deviation of the length of the words in the input files.
Rodando o Cálculo do Pi com MapReduce:

yarn jar /opt/hadoop-2.7.1/share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.1.jar pi 16 100000

> Number of Maps  = 16
> Samples per Map = 100000
> (...)
> INFO impl.YarnClientImpl: Submitted application application_1442439610364_0001
> INFO mapreduce.Job: The url to track the job: http://grandesdados-hadoop:8088/proxy/application_1442439610364_0001/
> INFO mapreduce.Job: Running job: job_1442439610364_0001
> (...)
> Job Finished in 48.333 seconds
> Estimated value of Pi is 3.14157500000000000000
ZooKeeper
Esse procedimento é baseado na documentação do ZooKeeper.

Dentro do container como usuário root:

curl -L -O http://archive.apache.org/dist/zookeeper/zookeeper-3.4.6/zookeeper-3.4.6.tar.gz
tar zxf zookeeper-3.4.6.tar.gz -C /opt
chown hadoop:hadoop -R /opt/zookeeper-3.4.6

echo 'export PATH=$PATH:/opt/zookeeper-3.4.6/bin' > /etc/profile.d/zookeeper.sh
source /etc/profile.d/zookeeper.sh

mkdir -p /data/zookeeper
chown hadoop:hadoop /data/zookeeper
Editar /opt/zookeeper-3.4.6/conf/zoo.cfg:

tickTime=6000
dataDir=/data/zookeeper
clientPort=2181
Inicializando o serviço:

su - hadoop
zkServer.sh start

> JMX enabled by default
> Using config: /opt/zookeeper-3.4.6/bin/../conf/zoo.cfg
> Starting zookeeper ... STARTED
Para parar o serviço:

zkServer.sh stop
Teste

(para os testes, deve ser usado o usuário hadoop: su - hadoop)

Processo:

ps x | grep zoo

> 8246 ?        Sl     0:00 /usr/java/jdk1.8.0_60/bin/java (...) org.apache.zookeeper.server.quorum.QuorumPeerMain (...)
> 8291 ?        S+     0:00 grep zoo
Telnet:

echo 'ruok' |  curl telnet://grandesdados-hadoop:2181

> imok
Cliente:

zkCli.sh -server grandesdados-hadoop:2181

> Connecting to grandesdados-hadoop:2181
> ...
> [zk: grandesdados-hadoop:2181(CONNECTED) 0] ls /
> [zookeeper]
> [zk: grandesdados-hadoop:2181(CONNECTED) 1] help
> ZooKeeper -server host:port cmd args
> 	stat path [watch]
> 	set path data [version]
> 	ls path [watch]
> 	delquota [-n|-b] path
> 	ls2 path [watch]
> 	setAcl path acl
> 	setquota -n|-b val path
> 	history
> 	redo cmdno
> 	printwatches on|off
> 	delete path [version]
> 	sync path
> 	listquota path
> 	rmr path
> 	get path [watch]
> 	create [-s] [-e] path data acl
> 	addauth scheme auth
> 	quit
> 	getAcl path
> 	close
> 	connect host:port
> [zk: grandesdados-hadoop:2181(CONNECTED) 3] quit
HBase
Esse procedimento é baseado na documentação do HBase.

Dentro do container como usuário root:

curl -L -O http://archive.apache.org/dist/hbase/1.1.2/hbase-1.1.2-bin.tar.gz
tar zxf hbase-1.1.2-bin.tar.gz -C /opt
chown hadoop:hadoop -R /opt/hbase-1.1.2

echo 'export PATH=$PATH:/opt/hbase-1.1.2/bin' > /etc/profile.d/hbase.sh
source /etc/profile.d/hbase.sh

mkdir -p /data/hbase/tmp
chown hadoop:hadoop -R /data/hbase
Editar /opt/hbase-1.1.2/conf/hbase-site.xml:

<configuration>
  <property>
    <name>hbase.cluster.distributed</name>
    <value>true</value>
  </property>
  <property>
    <name>hbase.rootdir</name>
    <value>hdfs:///hbase</value>
  </property>
  <property>
    <name>hbase.tmp.dir</name>
    <value>/data/hbase/tmp</value>
  </property>
  <property>
    <name>hbase.zookeeper.quorum</name>
    <value>grandesdados-hadoop</value>
  </property>
</configuration>
Editar /opt/hbase-1.1.2/conf/hbase-env.sh: 
(manter conteúdo original, só alterar os valores abaixo)

export HBASE_OPTS="-XX:+UseConcMarkSweepGC -Djava.net.preferIPv4Stack=true"
export HBASE_MANAGES_ZK=false
Inicializando o serviço:

su - hadoop
start-hbase.sh

> starting master, logging to /opt/hbase-1.1.2/bin/../logs/hbase-hadoop-master-grandesdados-hadoop.out
> Java HotSpot(TM) 64-Bit Server VM warning: ignoring option PermSize=128m; support was removed in 8.0
> Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=128m; support was removed in 8.0
> starting regionserver, logging to /opt/hbase-1.1.2/bin/../logs/hbase-hadoop-1-regionserver-grandesdados-hadoop.out
Interface Web do Master:

http://grandesdados-hadoop:16010/

Interface Web do Region Server:

http://grandesdados-hadoop:16301/

Para parar o serviço:

stop-hbase.sh
Teste

(para os testes, deve ser usado o usuário hadoop: su - hadoop)

Processo:

ps x | grep hbase

> 8790 ?        S      0:00 bash /opt/hbase-1.1.2/bin/hbase-daemon.sh --config /opt/hbase-1.1.2/bin/../conf foreground_start master
> 8804 ?        Sl     0:14 /usr/java/jdk1.8.0_60/bin/java -Dproc_master (...)
> 8915 ?        S      0:00 bash /opt/hbase-1.1.2/bin/hbase-daemon.sh --config /opt/hbase-1.1.2/bin/../conf foreground_start regionserver -D hbase.regionserver.port=16201 -D hbase.regionserver.info.port=16301
> 8929 ?        Sl     0:14 /usr/java/jdk1.8.0_60/bin/java -Dproc_regionserver (...)
> 9329 ?        S+     0:00 grep hbase
Cliente:

hbase shell

> HBase Shell; enter 'help<RETURN>' for list of supported commands.
> Type "exit<RETURN>" to leave the HBase Shell
> Version 1.1.2, rcc2b70cf03e3378800661ec5cab11eb43fafe0fc, Wed Aug 26 20:11:27 PDT 2015
>
> hbase(main):001:0> status
> 1 servers, 0 dead, 2.0000 average load
>
> hbase(main):002:0> help
> HBase Shell, version 1.1.2, rcc2b70cf03e3378800661ec5cab11eb43fafe0fc, Wed Aug 26 20:11:27 PDT 2015
> Type 'help "COMMAND"', (e.g. 'help "get"' -- the quotes are necessary) for help on a specific command.
> Commands are grouped. Type 'help "COMMAND_GROUP"', (e.g. 'help "general"') for help on a command group.
> (...)
> hbase(main):004:0> exit
Kafka
Esse procedimento é baseado na documentação do Kafka.

Dentro do container como usuário root:

curl -L -O http://archive.apache.org/dist/kafka/0.8.2.1/kafka_2.10-0.8.2.1.tgz
tar zxf kafka_2.10-0.8.2.1.tgz -C /opt
chown hadoop:hadoop -R /opt/kafka_2.10-0.8.2.1

echo 'export PATH=$PATH:/opt/kafka_2.10-0.8.2.1/bin' > /etc/profile.d/kafka.sh
source /etc/profile.d/kafka.sh

mkdir -p /data/kafka
chown hadoop:hadoop /data/kafka
Editar /opt/kafka_2.10-0.8.2.1/config/server.properties: 
(manter conteúdo original, só alterar os valores abaixo)

log.dirs=/data/kafka
zookeeper.connect=grandesdados-hadoop:2181
Inicializando o serviço:

su - hadoop
kafka-server-start.sh /opt/kafka_2.10-0.8.2.1/config/server.properties &> kafka.out &
Para parar o serviço:

kafka-server-stop.sh /opt/kafka_2.10-0.8.2.1/config/server.properties
Teste

(para os testes, deve ser usado o usuário hadoop: su - hadoop)

Processo:

ps x | grep kafka

> 9818 ?        Sl     0:03 /usr/java/jdk1.8.0_60/bin/java (...) kafka.Kafka /opt/kafka_2.10-0.8.2.1/config/server.properties
> 9928 ?        S+     0:00 grep kafka
Enviando e recebendo mensagens:

kafka-topics.sh \
--create \
--zookeeper grandesdados-hadoop:2181 \
--replication-factor 1 \
--partitions 1 \
--topic test

> Created topic "test".

echo 'Primeira mensagem de teste' | kafka-console-producer.sh --broker-list grandesdados-hadoop:9092 --topic test

> (...)

kafka-console-consumer.sh --zookeeper grandesdados-hadoop:2181 --topic test --from-beginning

> Primeira mensagem de teste
> ^CConsumed 1 messages
