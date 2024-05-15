# MariaDBGaleraCluster
Este artigo e um passo a passo para a configuração do MariaDB  com Galera Galera Cluster

aqui foi utilizado como sistema operacional o Linux Debian 12.
Vamos considerar o Seguinte que temos 3 servidores aqui chamaremos eles de nos.

- ServerA 192.168.1.10
- ServerB 192.168.1.20
- ServerC 192.168.1.30

Sendo ServerA o no principal do cluster.
Os passos a seguir devem ser executados em todos os servidores.
Considerando que você já está logado como root em todos os servidores, vamos definir a mesma senha e todos os servidores.
Vamos considerar que nossa senha é: abc@123 e vamos utilizar ela no nosso exemplo:
```
#: passwd root
#: abc@123
#: abc@123
````
Agora vamos atualizar o sistema para garantir que tudo ocorra bem:
```
#:  apt update
#:  apt upgrade
y
```
 
Vamos instalar o Mariadb, Galera Cluster e o ufw.
```
#: apt-get -y install mariadb-server mariadb-client galera-4 ufw
```
Agora precisamos agora precisamos editar o arquivo de configuração (my.cnf).
Normalmente este arquivo se encontra em /etc/mysql/my.cnf ou /etc/my.cnf. 
Vamos utilizar o nano para editar o arquivo.
```
#: nano /etc/mysql/my.cnf
```
Navegue até o final do arquivo e adicione as seguintes linhas
```
[mysqld]
binlog_format=ROW
default-storage-engine=innodb
innodb_autoinc_lock_mode=2
bind-address=0.0.0.0
# Galera Provider Configuration
wsrep_on=ON
wsrep_provider=/usr/lib/galera/libgalera_smm.so
# Galera Cluster Configuration
wsrep_cluster_name="test_cluster"
wsrep_cluster_address="gcomm://192.168.1.10, 192.168.1.20, 192.168.1.30"
# Galera Synchronization Configuration
wsrep_sst_method=rsync
```
Agora a configuração do galera e do mariadb esta completa mas talvez ainda não funcione devido a configurações de firewall e infrestrutura:

Para que o Galera Cluster funcione é necessário liberar algumas portas:
3306: Esta é a porta padrão para conexões de cliente MySQL e Transferência de Estado de Snapshot usando mysqldump para backups.
4567: Esta porta é reservada para o tráfego de replicação do Galera Cluster. A replicação multicast usa tanto o transporte UDP quanto o TCP nesta porta.
4568: Esta porta é usada para Transferência Incremental de Estado.
4444: Esta porta é usada para todas as outras Transferências de Estado de Snapshot.

Vamos utilizar o ufw (Uncomplicated Firewal) para isso com os seguintes comandos:
```
#: ufw allow 3306/tcp
#: ufw allow 4567/tcp
#: ufw allow 4568/tcp
#: ufw allow 4444/tcp
#: ufw allow 3306/udp
#: ufw allow 4567/udp
#: ufw allow 4568/udp
#: ufw allow 4444/udp
```
Para conferir a configuração de firewall ultilize o seguinte comando.
```
#: ufw status
```
Agora vamos iniciar os serviços:
No primeiro nó temos que executar o seguinte comando:
```
#: galera_new_cluster
```
Aos demais servidores vamos iniciar o mariadb normalmente:
```
#: systemctl start mariadb
```
Vamos verificar se  o status do cluster, se esta tudo ok....
Primeiro acessando o mariadb
```
#: mariadb
```
Agora enviamos a query para a verificação
```
SQL: SHOW STATUS LIKE 'wsrep_%';
```
Extra, caso não esteja conseguindo conectar ao seu banco de dados a partir de uma maquina remota talvez o usuário não tenha permissão para acessar de determinado IP.
Vamos seguir os passos para criar perfis de usuário, ainda no mariadb entre com o seguinte query
```
SQL:  CREATE USER 'root'@'%' IDENTIFIED BY 'minhaSenha@123';
```
Explicação:
Root é o nome de usuário, você pode colocar qualquer um.
% é o IP de onde ele pode ser acessado, quando colocado % quer dizer que pode ser de qualquer IP.
minhaSenha@123 é a senha que você vai utilizar para acessar este perfil de usuário.
Mas isso ainda não é tudo, precisamos garantir que este usuário tenha permissões.
No exemplo a seguir vamos assegurar que o usuário tem privilégios máximos.
Ainda no mariadb
```
SQL: GRANT ALL PRIVILEGES ON *.* TO 'root'@'%';
```
Se em agum momento você precisar alterar a senha de algum perfil:
Insira o comando no mariadb
```
SQL: SET PASSWORD FOR 'root'@'%' = PASSWORD(‘a01b02c03’);
```
Caso as alterações de usuário não estejam sendo aplicadas tente o seguinte comand no mariadb
```
SQL: FLUSH PRIVILEGES;
```
