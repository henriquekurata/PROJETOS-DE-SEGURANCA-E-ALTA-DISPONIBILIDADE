# ***Configuração de Data Lake com HDFS para trabalhar com alta disponibilidade***

## Ferramentas: 

Hadoop.

## Passos: 
* Fazer o download do JDK, Hadoop e Zookeeper;
* Editar as variáveis de ambientes dos 3 arquivos;
* Fazer o clone da primeira máquina para as demais, pois as configurações até aqui serão as mesmas;
* Editar variáveis de ambiente .Bashrc;
* Criar chaves SSH para conexão entre máquinas do cluster Hadoop sem senha;
* Configurar os arquivos do Hadoop;
* Inicializar o cluster de alta disponibilidade.

## Comandos: 

### Criar VM com Red Hat

### Verificar lista usuários prioritários: 

sudo vi /etc/sudoers >  adicionar o usuário "aluno" na lista junto com o root

### Download JDK/Hadoop/ZOOKEEPER

Fazer o Download JDK 8, descompactar e mover para a pasta: sudo mv jdk1.8 /opt/jdk

Fazer o Download HADOOP, descompactar e mover para a pasta: sudo mv hadoop /opt/hadoop

Fazer o Download Zookeeper, descompactar e mover para a pasta: sudo mv zookeeper /opt/zookeeper

### Ir para a pasta home (~) e editar o arquivo .bash_profile com as variáveis de ambiente:
```
#Java JDK 1.8
export JAVA_HOME=/opt/jdk
export JRE_HOME=/opt/jdk/jre
export PATH=$PATH:$JAVA_HOME/bin:$JRE_HOME/bin

#Hadoop
export HADOOP_HOME=/opt/hadoop
export PATH=$PATH:$HADOOP_HOME/bin

#Zookeeper
export ZOOKEEPER_HOME=opt/zookeeper
export PATH=$PATH:$ZOOKEEPER_HOME/bin
```

### Clone do namenode / Edição de configurações das VMs

#Criar clone do Namenode para o Secondary e o datanode

Obs: Como estamos clonando, é importante fazer o refresh do MAC adress das duas máquinas clonadas. Além disso, nas configurações da VM a rede precisa estar no modo "Bridge adapter"

#Alterar o nome das máquinas:  

sudo vi /etc/hostname (nn1.dsa.com / nn2.dsa.com / dn1.dsa.com)

hostnamectl set-hostname dn1.dsa.com / nn2.dsa.com

#Ajustar o arquivo de rede: 

sudo vi /etc/hosts com o endereço e o hostname de cada máquina

Testar com o comando "ping"

### Variaveis para arquivo .bashrc

#Incluir as linhas abaixo no arquivo ~/.bashrc em todos os nodes do cluster
```
export HADOOP_HOME=/opt/hadoop
export HADOOP_MAPRED_HOME=$HADOOP_HOME
export HADOOP_COMMON_HOME=$HADOOP_HOME
export HADOOP_HDFS_HOME=$HADOOP_HOME
export YARN_HOME=$HADOOP_HOME
export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
export YARN_CONF_DIR=$HADOOP_HOME/etc/hadoop
export JAVA_HOME=/opt/jdk
export ZOOKEEPER_HOME=/opt/zookeeper
export PATH=$PATH:$JAVA_HOME/bin:$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$ZOOKEEPER_HOME/bin
```

### Configurandos as chaves ssh sem senha

ssh-keygen -t rsa

Após esse comando é só dar enter nas configurações, pois queremos a conexão sem senha

Precisamos copiar a chaver pública para o diretório authorized_keys em todos os nodes, portanto execute os comando abaixo no namenode:

#Para o namenode2:

cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys

ssh-copy-id -i .ssh/id_rsa.pub aluno@nn2.dsa.com

#Para o datanode:

cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys

ssh-copy-id -i .ssh/id_rsa.pub aluno@dn1.dsa.com


#Testando a conectividade do ssh:

ssh aluno@nn2.dsa.com

ssh aluno@dn1.dsa.com



### Configurando o namenode

Acesse /opt/hadoop/etc/hadoop

#vi core-site.xml
```
<property>
 <name>fs.defaultFS</name>
 <value>hdfs://ha-cluster</value>
</property>

<property>
 <name>dfs.journalnode.edits.dir</name>
 <value>/home/hadoop/HA/data/jn</value>
</property>

 <property>
  <name>ha.zookeeper.quorum</name>
  <value>nn1.dsa.com:2181,nn2.dsa.com:2181,dn1.dsa.com:2181</value>
 </property>
```

#vi hdfs-site.xml
```
<property>
 <name>dfs.namenode.name.dir</name>
 <value>/home/hadoop/HA/data/namenode</value>
 </property>
 <property>
 <name>dfs.replication</name>
 <value>1</value>
 </property>
 <property>
 <name>dfs.permissions</name>
 <value>false</value>
 </property>
 <property>
 <name>dfs.nameservices</name>
 <value>ha-cluster</value>
 </property>
 <property>
 <name>dfs.ha.namenodes.ha-cluster</name>
 <value>nn1,nn2</value>
 </property>
 <property>
 <name>dfs.namenode.rpc-address.ha-cluster.nn1</name>
 <value>nn1.dsa.com:9000</value>
 </property>
 <property>
 <name>dfs.namenode.rpc-address.ha-cluster.nn2</name>
 <value>nn2.dsa.com:9000</value>
 </property>
 <property>
 <name>dfs.namenode.http-address.ha-cluster.nn1</name>
 <value>nn1.dsa.com:50070</value>
 </property>
 <property>
 <name>dfs.namenode.http-address.ha-cluster.nn2</name>
 <value>nn2.dsa.com:50070</value>
 </property>
 <property>
 <name>dfs.namenode.shared.edits.dir</name>
 <value>qjournal://nn1.dsa.com:8485;nn2.dsa.com:8485;dn1.dsa.com:8485/ha-cluster</value>
 </property>
 <property>
 <name>dfs.client.failover.proxy.provider.ha-cluster</name>
 <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
 </property>
 <property>
 <name>dfs.ha.automatic-failover.enabled</name>
 <value>true</value>
 </property>
 <property>
 <name>dfs.ha.fencing.methods</name>
 <value>sshfence</value>
 </property>
 <property>
 <name>dfs.ha.fencing.ssh.private-key-files</name>
 <value>/home/hadoop/.ssh/id_rsa</value>
 </property>
```

#Configurando o zookeeper

Acesse: /opt/zookeeper/conf

Faça a cópia do diretório zoo_smple.cfg com outro nome: cp 

zoo_smple.cfg zoo.cfg

Edite:
```
dataDir=/home/hadoop/HA/data/zookeeper
Server.1=nn1.dsa.com:2888:3888
Server.2=nn2.dsa.com:2888:3888
Server.3=dn1.dsa.com:2888:3888
```

#Criar a pasta /home/hadoop/HA/data/jn em todas as máquinas que foram configuradas o Jornalnode (nesse caso as 3 máquinas)

#Criar a pasta /home/hadoop/HA/data/namenode nos dois namenodes para gravação de metadados do namenode (namenode e 
secondary namenode)

#Criar a pasta /home/hadoop/HA/data/datanode no datanode para gravação de metadados do datanode (Apenas no datanode)

#Adicionar o parâmetro abaixo no arquuivo hdfs-site.xml para o datanode:
 ```
    <property>
 <name>dfs.datanode.name.dir</name>
 <value>home/hadoop/HA/data/datanode</value>
 </property>
```

#Arquivo Worker

Em todas as máquinas do cluster, editar o arquivo workers e adicionar a lista de datanodes: 

1-Editar o arquivo: /opt/hadoop/etc/hadoop/workers 

2-Remover a opção de localhost 

3-Incluir a lista de datanodes, um em cada linha (deve ser usado o mesmo nome configurado no arquivo /etc/hosts

#Configuração da ordem no zookeeper
Crie o diretório /home/hadoop/HA/data/zookeeper

Crie e edite o aquivo myid com a indentação da máquina dentro do cluster (Namenode = 1, secondary = 2 e datanode = 3)
Deve ser feito para todas as máquinas



### Inicialização do Cluster HA

OBS: Antes de iniciar o cluster é importante limpar todos os diretórios de metadados 

Para o Journalnode precisamos configurar o jdk nas máquinas que rodarão esse daemon, portanto acesse: cd /opt/hadoop/etc/hadoop e edite o arquivo hadoop-env.sh:
export JAVA_HOME=/opt/jdk

#1- Inicialização do Journal Node nas 3 máquinas do cluster

hdfs --daemon start journalnode

#2- Format do Namenode 1 (somente na primeira inicialização do cluster)

hdfs namenode -format

#3- Verifica se o firewall está ativo

sudo firewall-cmd --state

Firewall precisa estar desativado em todas as máqunas:

sudo firewall-cmd --state

sudo systemctl stop firewalld

sudo systemctl disable firewalld

#4- Inicialização do NameNode no NameNode Ativo

hdfs --daemon start namenode

#5- Copiar os metadados do NomeNode Ativo para o StandBy (apenas na primeira inicialização, executar no NameNode Standby)

hdfs namenode -bootstrapStandby

#6- Inicialização do NameNode no NameNode Standby

hdfs --daemon start namenode

#7- Inicialização do Zookeeper em todas as máquinas do cluster

zkServer.sh start

#8- Inicialização do DataNode 

hdfs --daemon start datanode

#9- Formatar o HA State (apenas na primeira inicialização)

hdfs zkfc -formatZK

#10- Inicializa o Zookeeper HA Failover Controller (nos dois NameNodes)

hdfs --daemon start zkfc

#11 - Checar se os NameNodes estão configurados em HA

hdfs haadmin -getServiceState nn1

Acesse  o endereço ip para verificar o estado do cluster HDFS:
Endereço IP da máquina:50070 no navegador web
