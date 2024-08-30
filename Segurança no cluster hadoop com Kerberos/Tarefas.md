# ***Configuração de Kerberos no cluster Hadoop***

## Ferramentas: 

Hadoop e Kerberos.

## Passos:
* Instalar o Kerberos e editar os seus arquivos de configurações;
* Criar o user principal;
* Iniciar o Kerberos, criar os service principals e Keytab files;
* Configurar os arquivos do Hadoop para o modo seguro;
* Copiar arquivos para outras duas máquinas do cluster;
* Inicializar o cluster em modo seguro;
* Criar diretório no cluster para acesso do user principal.

## Comandos:

### Instalar o Kerberos:

#Fazer o download do Kerberos (KDC) direto da documentação: krb5-1.17.tar.gz

#Descompactar: tar -xvf krb5-1.17.tar.gz   

#Mover: sudo mv krb5-1.17.tar.gz /opt/kerberos

cd /opt/kerberos/src

#Instalar: ./configure

#Compilando:

su (Fazer com o root)

make

make install

#Configurando:

cd ~

yum intall krb5-libs krb5-server krb5-workstation

#Editar arquivos de configuações do Kerberos:

gedit /etc/krb5.conf
```
[logging]
     default = FILE:/var/log/krb5libs.log
     kdc = FILE:/var/log/krb5kdc.log
     admin_server = FILE:/var/log/kadmind.log

[libdefaults]
     default_realm = DSA.COM
     dns_lookup_realm = false
     dns_lookup_kdc = false
     ticket_lifetime = 24h
     renew_lifetime = 7d
     forwardable = true

[realms]
     DSA.COM = {
      kdc = nn1.dsa.com
      admin_server = nn1.dsa.com
     }

[domain_realm]
     .dsa.com = DSA.COM
     dsa.com = DSA.COM
```

#KDC (kdc.conf)

cd /var/kerberos/krb5kdc

gedit kdc.conf
```
[kdcdefaults]
     kdc_ports = 88
     kdc_tcp_ports = 88

[realms]
     DSA.COM = {
      #master_key_type = aes256-cts
      acl_file = /var/kerberos/krb5kdc/kadm5.acl
      dict_file = /usr/share/dict/words
      admin_keytab = /var/kerberos/krb5kdc/kadm5.keytab
      supported_enctypes = aes256-cts:normal aes128-cts:normal des3-hmac-sha1:normal arcfour-hmac:normal des-hmac-sha1:normal des-cbc-md5:normal des-cbc-crc:normal
      max_life = 24h 0m 0s
      max_renewable_life = 7d 0h 0m 0s
     }

```

#ACL

cd /var/kerberos/krb5kdc

gedit kadm5.acl
```
*/admin@DSA.COM *
```

#Banco de dados do KDC

cd ~

su

kdb5_util create -s

Escolha uma senha para criar o banco de dados

cd *usr/local/var/krb5kdc

vi principal


#Criando o principal (Usuário para priviégios de acesso)

su

kadmin.local -q "addprinc aluno/admin"

OBS:
Para os serviços de inicialização funcionarem é impotante copiar o banco de dados (principal) para a pasta do krb5kdc (cd /var/kerberos/krb5kdc):

cp /usr/local/var/krb5kdc/* . (Copiando tudo da pasta)

cp /usr/local/var/krb5kdc/* .K5.DSA>COM . (Copiando o arquivo oculto)

#Inicializando os 2 seviços Kerberos

service krb5kdc start

service kadmin start

#Verificar o status

sudo systemctl status krb5kdc

#Comandos para deixar a ativação da função automática, mesmo após reiniciar a máquina:

chkconfig kadmin on 

chkconfig krb5kdc on

#Acessando

sudo kadmin (Acessar o principal ativado com o comando kinit)

#Estabelecendo segurança no acesso aos dados

### Criando os service principals:

Conectar no kadmin e criar os service principals abaixo com o 

comando addprinc -randkey
```
#Servico Web
HTTP/dn1.dsa.com@DSA.COM
HTTP/nn1.dsa.com@DSA.COM
HTTP/nn2.dsa.com@DSA.COM

#Datanode
dn/dn1.dsa.com@DSA.COM

#HDFS
nn/dn1.dsa.com@DSA.COM
nn/nn1.dsa.com@DSA.COM
nn/nn2.dsa.com@DSA.COM

#YARN
yarn/dn1.dsa.com@DSA.COM
yarn/nn1.dsa.com@DSA.COM
yarn/nn2.dsa.com@DSA.COM
```

### Comando para verificar os principals do cluster:

listprincs

### Testar conexão

kinit aluno/admin

### Criar Keytab files (para autenticação dos service principals criandos no item anterior)

Acesse o comando Kadmin e execute os itens abaixo:

xst -k nn.service.keytab nn/dn1.dsa.com@DSA.COM nn/nn1.dsa.com@DSA.COM nn/nn2.dsa.com@DSA.COM

xst -k dn.service.keytab dn/dn1.dsa.com@DSA.COM

xst -k yarn.service.keytab yarn/dn1.dsa.com@DSA.COM yarn/nn1.dsa.com@DSA.COM yarn/nn2.dsa.com@DSA.COM

xst -k spnego.service.keytab HTTP/dn1.dsa.com@DSA.COM HTTP/nn1.dsa.com@DSA.COM HTTP/nn2.dsa.com@DSA.COM

 ls -la *.keytab

Mover os arquivos: mv *.keytab /opt/hadoop/etc/hadoop
cd /opt/hadoop/etc/hadoop

ls -la .*keytab

Alterar permissão: chmod 400 *.keytab 

### Ativando o service principal:
kinit -kt /opt/hadoop/etc/hadoop/nn.service.keytab nn/nn1.dsa.

com

klist (Verificar o service principal ativo)


### Configurando o cluster Hadoop para Segurança

#Core-site.xml:
```
 <property>
  <name>hadoop.security.authentication</name>
  <value>kerberos</value>
 </property>

 <property>
  <name>hadoop.security.authorization</name>
  <value>true</value>
 </property>

 <property>
  <name>hadoop.rpc.protection</name>
  <value>authentication</value>
 </property>

 <property>
  <name>hadoop.security.auth_to_local</name>
  <value>
    RULE:[2:$1/$2@$0]([nd]n/.*@DSA.COM)s/.*/hdfs/
    RULE:[2:$1/$2@$0](yarn/.*@DSA.COM)s/.*/yarn/
    DEFAULT
  </value>
</property>
```


#yarn-site.xml
```
<property>
  <name>yarn.resourcemanager.principal</name>
  <value>yarn/_HOST@DSA.COM</value>
</property>

<property>
  <name>yarn.resourcemanager.keytab</name>
  <value>/opt/hadoop/etc/hadoop/yarn.service.keytab</value>
  </property>

<property>
  <name>yarn.nodemanager.principal</name>
  <value>yarn/_HOST@DSA.COM</value>
</property>

<property>
  <name>yarn.manager.keytab</name>
  <value>/opt/hadoop/etc/hadoop/yarn.service.keytab</value>
</property>

<property>
  <name>yarn.nodemanager.container-executor.class</name>
  <value>org.apache.hadoop.yarn.server.nodemanager.LinuxContainerExecutor</value>
</property>
```

#hdfs-site.xml:
```
<property>
  <name>dfs.namenode.kerberos.principal</name>
  <value>nn/_HOST@DSA.COM</value>
</property>

<property>
  <name>dfs.secondary.namenode.kerberos.principal</name>
  <value>nn/_HOST@DSA.COM</value>
</property>

<property>
  <name>dfs.datanode.kerberos.principal</name>
  <value>dn/_HOST@DSA.COM</value>
</property>

<property>
  <name>dfs.journalnode.kerberos.principal</name>
  <value>nn/_HOST@DSA.COM</value>
</property>

<property>
  <name>dfs.web.authentication.kerberos.principal</name>
  <value>HTTP/_HOST@DSA.COM</value>
</property>

<property>
  <name>dfs.namenode.kerberos.internal.spnego.principal</name>
  <value>HTTP/_HOST@DSA.COM</value>
</property>

<property>
  <name>dfs.journalnode.kerberos.internal.spnego.principal</name>
  <value>HTTP/_HOST@DSA.COM</value>
</property>

<property>
  <name>dfs.secondary.namenode.kerberos.internal.spnego.principal</name>
  <value>HTTP/_HOST@DSA.COM</value>
</property>

<property>
  <name>dfs.namenode.keytab.file</name>
  <value>/opt/hadoop/etc/hadoop/nn.service.keytab</value>
</property>

<property>
  <name>dfs.journalnode.keytab.file</name>
  <value>/opt/hadoop/etc/hadoop/nn.service.keytab</value>
</property>

<property>
  <name>dfs.secondary.namenode.keytab.file</name>
  <value>/opt/hadoop/etc/hadoop/nn.service.keytab</value>
</property>

<property>
  <name>dfs.datanode.keytab.file</name>
  <value>/opt/hadoop/etc/hadoop/dn.service.keytab</value>
</property>

<property>
  <name>dfs.web.authentication.kerberos.keytab</name> 
  <value>/opt/hadoop/etc/hadoop/spnego.service.keytab</value>              
</property>  

<property>
  <name>dfs.block.access.token.enable</name>
  <value>true</value>
</property>

<property>
  <name>dfs.datanode.address</name> 
  <value>0.0.0.0:1019</value>           
</property>  

<property>
  <name>dfs.datanode.http.address</name> 
  <value>0.0.0.0:1022</value>           
</property>  

```

### Copiar os arquivos do nn1 para nn2 e dn1

scp *.keytab aluno@nn2.dsa.com:/opt/hadoop/etc/hadoop

scp *.keytab aluno@dn1.dsa.com:/opt/hadoop/etc/hadoop

scp core-site.xml hdfs-site.xml yarn-site.xml aluno@nn2.dsa.com:/opt/hadoop/etc/hadoop

scp core-site.xml hdfs-site.xml yarn-site.xml aluno@dn1.dsa.com:/opt/hadoop/etc/hadoop

### Camada de autenticação e segurança simples (SASL) e JSVC

#O JSVC vai iniciar o serviço do Datanode como Root, mas vai reduzir ao usuário que estará apontado no arquivo de configurações do Hadoop(Tudo isso para aumentar a segurança dos dados gravados), portanto JSVC é quem vai garantir o acesso a JVM

Comando para instalar o JSVC: sudo yum instlall jsvc

### Alterar o arquivo de configuração hadoop-env.sh:

export JSVC_HOME=/usr/bin

export HADOOP_HOME=/opt/hadoop

#Diretórios para gravação dos arquivos de log:

export HADOOP_SECURE_PID_DIR=${HADOOP_PID_DIR}

export HADOOP_SECURE_LOG=${HADOOP_LOG_DIR}

#Apontando o usuário a ser alterado pelo root com o JSVC:

export HDFS_DATANODE_SECURE_USER=aluno



### Inicialização do Cluster HA com Kerberos

#0- Antes de tudo é necessário a solicitação do ticket pelo comando kinit e a verificação com klist no nemanode 1.
Antes de executar o comando 1 também é importante limpar o diretório /opt/hadoop/logs,  /tmp, /home/hadoop/HA/data/jn (de todas as máquinas) e /opt/hadoop/etc/hadoop/home/hadoop/HA/data/namenode (no namenode ativo).

#1- Inicialização do Journal Node nas 3 máquinas do cluster
hdfs --daemon start journalnode

#2- Format do Namenode 1 (somente na primeira inicialização do cluster)
hdfs namenode -format

#3- Verifica se o firewall está ativo
sudo firewall-cmd --state

#4- Inicialização do NameNode no NameNode Ativo
hdfs --daemon start namenode

#5- Copiar os metadados do NomeNode Ativo para o StandBy (apenas na primeira inicialização, executar no NameNode Standby)
hdfs namenode -bootstrapStandby

#6- Inicialização do NameNode no NameNode Standby
hdfs --daemon start namenode

#7- Inicialização do Zookeeper em todas as máquinas do cluster
zkServer.sh start

#8- Inicialização do DataNode em modo seguro
sudo /opt/hadoop/bin/hdfs --daemon start datanode
ps -ax | datanode (Verificar o status no modo seguro)

#9- Formatar o HA State (apenas na primeira inicialização - nos dois namenodes)
hdfs zkfc -formatZK

#10- Inicializa o Zookeeper HA Failover Controller (nos dois NameNodes)
hdfs --daemon start zkfc

#11 - Checar se os NameNodes estão configurados em HA
hdfs haadmin -getServiceState nn1




### Auditoria

#Habilitando auditoria nos arquivos do hadoop (cd opt/hadoop/etc/hadoop)

hadoop-env.sh

export HADOOP_JAAS_DEBUG=true (Habilitar para saber quem está tentanto acesar)

export HADOOP_OPTS="-Djava.net.preferIPv4Stack=true -Dsun.security.krb5.debug=true -Dsun.security.spnego.debug" (leitua de tudo que foi acessado na JVM)

log4j.properties

hadoop.security.logger=ERROR,NullAppender (Verificar erros no log)

### Criando pasta e arquivo no hdfs:

hdfs dfs -ls /

hdfs dfs -mkdir /user

vi teste.txt

hdfs dfs -put teste.txt /user

Obs: Se o acesso for feito sem o user principal (kinit), não será possível nem acessar o cluster, caso o Kerberos esteja ativo.