# ***Alta disponibilidade entre dois bancos de dados Oracle: Streaming de dados transacionais com OGG***

## Ferramentas: 

Banco Oracle e Oracle Golden Gate.

## Passos:
* Criar VM com Red Hat Enterprise Linux;
* Instalar o banco de dados Oracle (Oracle delivery) e fazer as configurações;
* Instalar o Oracle Golden Gate e fazer as configurações;
* Criar schemas da tabela nos dois bancos;
* Configurar a comunicação entre os bancos de dados Oracle e rede;
* Inserir e verificar a replicação dos dados.

## Comandos:

### Instalação do Banco Oracle 12c no Red Hat Enterprise Linux

#Atualização do SO

Atualizar a lista de sudoers: vi /etc/sudoers (Adicionar o usuário aluno para permissões no sistema, junto ao Root)

Fazer a subinscrição do sistema Red Hat (Válido por 1 ano,  para que possamos utilizar o sistema como VM)

sudo yum update -y

#Instalação de Pacotes
sudo yum install -y binutils.x86_64 compat-libcap1.x86_64 gcc.x86_64 gcc-c++.x86_64 glibc.i686 glibc.x86_64 glibc-devel.i686 glibc-devel.x86_64 ksh compat-libstdc++-33 libaio.i686 libaio.x86_64 libaio-devel.i686 libaio-devel.x86_64 libgcc.i686 libgcc.x86_64 libstdc++.i686 libstdc++.x86_64 libstdc++-devel.i686 libstdc++-devel.x86_64 libXi.i686 libXi.x86_64 libXtst.i686 libXtst.x86_64 make.x86_64 sysstat.x86_64 zip unzip

#Criação de grupos de usuários para o Oracle

sudo groupadd oinstall

sudo groupadd dba

sudo useradd -g oinstall -G dba oracle

sudo passwd oracle

#Adicionar os parâmetros abaixo ao arquivo /etc/sysctl.conf
```
fs.aio-max-nr = 1048576
fs.file-max = 6815744
kernel.shmall = 2097152
kernel.shmmax = 8329226240
kernel.shmmni = 4096
kernel.sem = 250 32000 100 128
net.ipv4.ip_local_port_range = 9000 65500
net.core.rmem_default = 262144
net.core.rmem_max = 4194304
net.core.wmem_default = 262144
net.core.wmem_max = 1048586
```

#Aplicar os parâmetros sem reiniciar o SO

sudo sysctl -p

sudo sysctl -a

#Definir os limits do Oracle em /etc/security/limits.conf
```
oracle soft nproc 2047
oracle hard nproc 16384
oracle soft nofile 1024
oracle hard nofile 65536
```

#Descompactar o arquivo (Download banco Oracle)

sudo unzip V839960-01.zip -d /tmp/stage/

#Criar os diretórios - OFA

sudo mkdir /u01

sudo mkdir /u02

sudo chown -R oracle:oinstall /u01

sudo chown -R oracle:oinstall /u02

sudo chmod -R 775 /u01

sudo chmod -R 775 /u02

sudo chmod g+s /u01

sudo chmod g+s /u02

#Executar o instalador como usuário Oracle

Antes de instalar é importante alterar o usuário do sistema da máquina para "oracle" criado no item anterior 

Executar: ./tmp/stage/database/runInstaller 

#Configurar o Firewall

sudo firewall-cmd --zone=public --add-port=1521/tcp --add-port=5500/tcp --add-port=5520/tcp --add-port=3938/tcp --permanent

sudo firewall-cmd --zone=public --add-port=7809/tcp --permanent

sudo firewall-cmd --reload

#Se tiver problemas, pare o firewall: 

sudo systemctl stop firewalld (Deixar sem firewall)

#Incluir no arquivo .bash_profile (Para usar na máquina PROD)
```
TMPDIR=$TMP; export TMPDIR
ORACLE_BASE=/u01/app/oracle; export ORACLE_BASE
ORACLE_HOME=$ORACLE_BASE/product/12.2.0/dbhome_2; export ORACLE_HOME
ORACLE_SID=orcl2; export ORACLE_SID
PATH=$ORACLE_HOME/bin:$PATH; export PATH
LD_LIBRARY_PATH=$ORACLE_HOME/lib:/lib:/usr/lib:/usr/lib64; export LD_LIBRARY_PATH
CLASSPATH=$ORACLE_HOME/jlib:$ORACLE_HOME/rdbms/jlib; export CLASSPATH
```

#Incluir no arquivo .bash_profile (Para usar na máquina SERV)
```
TMPDIR=$TMP; export TMPDIR
ORACLE_BASE=/u01/app/oracle; export ORACLE_BASE
ORACLE_HOME=$ORACLE_BASE/product/12.2.0/dbhome_1; export ORACLE_HOME
ORACLE_SID=orcl; export ORACLE_SID
PATH=$ORACLE_HOME/bin:$PATH; export PATH
LD_LIBRARY_PATH=$ORACLE_HOME/lib:/lib:/usr/lib:/usr/lib64; export LD_LIBRARY_PATH
CLASSPATH=$ORACLE_HOME/jlib:$ORACLE_HOME/rdbms/jlib; export CLASSPATH
```

#Source no arquivo

source .bash_profile

#Iniciar o Listener

lsnrctl start

#Inicia o banco de dados

sqlplus / as sysdba

startup

shutdown


#Fazer a clonagem da VM, pois usaremos as mesmas configurações para montar a máquina destino.

É necessário alterar o nome da máquina para não ficarem iguais, portante acesse: sudo vi /etc/hostname

Como houve alteração do hostname precisamos atualizar os arquivos de configuração do Oracle Database: 

/u01/app/oracle/12.2.0/dbhome_1/network/admin/tnsnames.ora e /u01/app/oracle/12.2.0/dbhome_1/network/admin/listener.ora

As máquinas também não podem ter o mesmo MAC adress, portanto é preciso acessar o menu inicial da VM, acessar o menu de Network e fazer o refresh do endereço MAC na máquina clonada(Destino)




### Instalando OGG

#Para instalar o OGG basta fazer o download do arquivo e seguir o mesmmo passo a passo do banco de dados Oracle, executando o ./runinstaller dentro do arquivo

#Editar o arquivo .bash_profile:

GGH=/u01/app/oracle/product/ogg_src

PATH=$GGH:$PATH; export PATH

#Acesse o diretório: /u01/app/oracle/product/ogg_src

Linha de comando do OGG: ./ggsci > start mgr > info all (para iniciar o serviço mgr)

OBS: É necessário executar o banco de dados Oracle e o OGG nas duas máquinas (Serv e Prod)

#Configurando a replicação: 

#Utilizar os comandos abaixo para as duas máquinas:

Acessar a linha de comando do OGG: ./ggsci

Executar o comando: create subdirs (para criação dos diretórios)

start mgr

edit params mgr (parametros do OGG)

edit params ./GLOBALS
```
CHECKPOINTTABLE ggadmin5.ggcheckpoint
```

#Definir configurações no banco de dados Oracle:

Inicializando o banco de dados Oracle: Acessar o diretório do ggsci  e iniciar o banco de dados Oracle com sqlplus / as sysdba (É importante usar esse diretório para a execução dos catálogos SQL)

Executar os comando abaixo no banco de dados Oracle. Servem para ativar os logs, replicação de dados, tablespace, usuário, privilégios e os catálogos de comandos SQL.

Source:
```
alter database add supplemental log data;
alter system set enable_goldengate_replication=true scope=both;
create tablespace ggs_data6 datafile '/u01/app/oracle/oradata/orcl2/ggstarget_data06.dbf' size 1024m autoextend on;
create user ggadmin6 identified by ggadmin5 default tablespace ggs_data6 temporary tablespace temp;
grant connect,resource,create session, alter session to ggadmin5;
grant select any dictionary, select any table,create table to ggadmin6;
grant alter any table to ggadmin6;
grant execute on utl_file to ggadmin6;
grant flashback any table to ggadmin6;
grant execute on dbms_flashback to ggadmin6;
@marker_setup.sql
@ddl_setup.sql
@role_setup.sql
@ddl_enable.sql
@sequence.sql
```


Target
```
alter database add supplemental log data;
alter system set enable_goldengate_replication=true scope=both;
create tablespace ggs_data6 datafile '/u01/app/oracle/oradata/orcl/ggstarget_data05.dbf' size 1024m autoextend on;
create user ggadmin6 identified by ggadmin6 default tablespace ggs_data6 temporary tablespace temp;
grant connect,resource,create session, alter session to ggadmin5;
grant select any dictionary, select any table,create table to ggadmin6;
grant alter any table to ggadmin5;
grant execute on utl_file to ggadmin5;
grant flashback any table to ggadmin5;
grant execute on dbms_flashback to ggadmin5;
@marker_setup.sql
@ddl_setup.sql
@role_setup.sql
@ddl_enable.sql
@sequence.sql
```


#Criar o schema da tabela
No prod:
```
create user appuser4 identified by appuser4 default tablespace users temporary tablespace temp;

grant connect, resource, create session to appuser4; 
grant unlimited tablespace to appuser4;

 CREATE TABLE appuser4.TB_EMP4
 (EMPNO INT,
  ENAME VARCHAR(10),
  JOB VARCHAR(9),
  MGR INT,
  HIREDATE DATE,
  SAL DECIMAL(7,2),
  COMM DECIMAL(7,2),
  DEPTNO INT,
  LAST_UPDT_DT_TM TIMESTAMP DEFAULT current_timestamp);
```

No Serv:
```
create user appuser4 identified by appuser4 default tablespace users temporary tablespace temp;

grant connect, resource, create session to appuser4; 
grant unlimited tablespace to appuser4;

 CREATE TABLE appuser4.TB_EMP4
 (EMPNO INT,
  ENAME VARCHAR(10),
  JOB VARCHAR(9),
  MGR INT,
  HIREDATE DATE,
  SAL DECIMAL(7,2),
  COMM DECIMAL(7,2),
  DEPTNO INT,
  LAST_UPDT_DT_TM TIMESTAMP DEFAULT current_timestamp);
```

#Criando a tabela de checkpoint:

Deve ser criado no banco de destino (Servidor que irá receber os dados replicados)

Acessar o ggsci e conectar com o usuário do banco de dados criado anteriormente (Usuário ggadmin5 e senha ggadmin5)

Para adicionar a tabela no destino checkpoint, execute: add checkpointtable 
```
ggadmin5.ggcheckpoint
```

Para que as máquinas se comuniquem precisamos configurar o arquivo tnsnames: /

u01/app/oracle/product/12.2.0/dbhome_2/network/admin/tnsnames.ora

#No prod:
```
TARGET =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = devserver)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = orcl2)
    )
  )
```

#No dev:
```
SOURCE  =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = dbserver)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = orcl2)
    )
  )
```

Também é necessário configurar o arquivo: sudo vi /etc/hosts (com o endereço IP da máquina que será conectada)

Dessa maneira as máquinas se conectarão via oracle e rede




#Configurando a extração na fonte:

Source:

dblogin userid ggadmin5

add trandata appuser4.* cols *

Precisa ativar o arhive log par o banco de dados Oracle, para isso execute:

sqlplus / as sysdba

alter database archivelog;

alter database open;


register extract insrc database

add extract insrc, integrated tranlog, begin now

edit params insrc
```
extract insrc
 SETENV(ORACLE_SID="orcl2")
 SETENV(ORACLE_HOME="/u01/app/oracle/product/12.2.0/dbhome_2")
 USERID ggadmin5, PASSWORD ggadmin5
 TRANLOGOPTIONS IntegratedParams (max_sga_size 256)
 EXTTRAIL ./dirdat/in
 LOGALLSUPCOLS
 UPDATERECORDFORMAT COMPACT
 TABLE appuser3.*;
```

 add exttrail ./dirdat/in, extract insrc, megabytes 10

 add extract pumpint, exttrailsource ./dirdat/in

 edit params pumpint
```
 EXTRACT pumpint
 RMTHOST devserver, MGRPORT 7809
 RMTTRAIL ./dirdat/pn
 TABLE appuser3.*;
```

add rmttrail ./dirdat/pn, extract pumpint, megabytes 10

start extract insrc

start extract pumpint


Target:


edit params repintan
```
 replicat repintan
 SETENV(ORACLE_SID='orcl')
 DBOPTIONS INTEGRATEDPARAMS(parallelism 4)
 AssumeTargetDefs
 DiscardFile ./dirrpt/rpdw.dsc, purge
 USERID ggadmin5, PASSWORD ggadmin5
 MAP appuser3.*, target appuser3.*;
```

dblogin userid ggadmin5

add replicat repintan integrated exttrail ./dirdat/pn

start replicat repintan



#Inserindo e verificado a replicação dos dados com SQL

Use comandos SQL para inserir alguns registros na tabela:
```
INSERT INTO appuser3.TB_EMP3 VALUES (1006, 'Pele', 'Analista', NULL, NULL, 80000, 250, 90, sysdate);
commit;

INSERT INTO appuser4.TB_EMP4 VALUES (1006, 'Lucas', 'Analista', NULL, NULL, 80000, 250, 90, sysdate);
commit;

INSERT INTO appuser4.TB_EMP4 VALUES (1006, 'Henrique', 'Analista', NULL, NULL, 80000, 250, 90, sysdate);
commit;
```

Verificar a replicação no prod e serv:
```
SELECT COUNT (*) FROM APPUSER4.TB_EMP4;
```