
																						## O RKE É UM BINARIO GO QUE FAZ A CRIAÇÃO FACILITADA DE UM CLUSTER KUBERNETES																			 
## PARA QUE O CLUSTER SEJA CRIADO PELO RKE É NECESSÁRIO RODAR COM UM USUÁRIO COMUN, MAS QUE SEJA LIBERADO NO SUDOERS E ESTEJA NO GRUPO WHEEL											 
## SERÁ NECESSÁRIO ALGUNS AJUSTES NO ANSIBLE PARA USÁ-LO.EX: CRIAR OS USUARIOS NOS HOSTS, GERAR E COPIAR CHAVE RSA PARA OS NODES ENFIM ALGUNS PEQUENOS AJUSTES PARA FUNCIONAR.   
################################################################################################################################################################################


##################################
AMBIENTE: 3 SERVIDORES
192.168.1.20 -- > MASTER - ETCD
192.168.1.21 -- > WORKER
192.168.1.22 -- > WORKER
##################################

##############################################
CASO NAO TENHA UM DNS USE O ARQUIVO HOSTS - 
COPIE ESSE ARQUIVO PARA TODOS OS SERVIDORES
vim /etc/hosts
192.168.1.20 master.homelab.lab.local
192.168.1.20 worker1.homelab.lab.local
192.168.1.20 worker2.homelab.lab.local
##############################################


##############################################
             AJUSTES
##############################################
LOGUE NO SERVIDOR DO ANSIBLE 


 1 -   mkdir /tmp/ajustes
       touch /tmp/ajustes/lista_nodes.txt
	   touch /tmp/ajustes/ajustes.sh

 2 -   CRIE O ARQUIVO ABAIXO ESSE ARQUIVO SERÁ A LISTA DOS SEUS NODES
        # vim /tmp/ajustes/lista_nodes.txt
	     192.168.1.19
	     192.168.1.20
	     192.168.1.21
	     192.168.1.22
		 
		 CRIE O SCRIPT ABAIXO E INSIRA O CONTEÚDO PARA AJUSTAR O SISTEMA (SERÁ CRIADO O USUARIO COMUN "admincluster" - O USUÁRIO SERÁ ADICIONADO AO GRUPO WHELL E LIBERADO NO SUDOERS)
         INSIRA NO ARQUIVO /tmp/ajustes/ajustes.sh  OS COMANDOS PARA CRIAR E AJUSTAR AS PERMISSÕES DO USUARIO COMUN (admincluster). (ALTERE NA LINHA echo "102010" POR UMA SENHA DA SUA PREFERÊNCIA)
	   
  #!/bin/bash
  for i in $(cat /tmp/ajustes/lista_nodes.txt);do
  ssh root@$i 'echo "Criando o usuario comun admincluster...";
  echo "###############################################";
  sleep 3 ;
  useradd admincluster ;
  echo "Configura senha do usuario admincluster...";
  echo "###############################################";
  sleep 3 ;
  echo "102010" | passwd --stdin admincluster;
  echo "Configurando sudoers do usuario admincluster...";
  echo "###############################################";
  sleep 3;
  echo "admincluster ALL=(ALL:ALL) NOPASSWD: ALL" >> /etc/sudoers;
  echo "Adicionando o usuario admincluster ao grupo Wheel";
  echo "###############################################";
  usermod -aG wheel admincluster;
  sleep 2 ;
  echo "Ajustes realizados com sucesso "';
done
    - EXECUTE O SCRIPT
	 sh /tmp/ajustes/ajustes.sh

     - Faça um clone do projeto
	   cd /tmp/ajustes
	   git clone https://github.com/leandrojpg/ansible-cluster-rke.git
	   
	 - Copie o arquivo play-cria-cluster-rancher.yml para dentro da raiz de execução do ansible (Assumo que esteja em /etc/ansible - Altera se necessário)
	  cp ansible-cluster-rke/play-cria-cluster-rancher.yml /etc/ansible
	   
	  
 3 - Logue com o usuario admincluster no servidor do ANSIBLE aqui representado pelo 192.168.1.19
	 # su - admincluster
	 # Gere a chave rsa do usuario admincluster
	 # ssh-keygen (Pressione enter em todas as perguntas)
	 # Copie a chave para todos os servidores incluindo o bastion
	 # ssh-copy-id 192.168.1.19(Sera solicitada a senha de root de todos os nodes)
	 
4 - A criação do cluster é dependente do binário do rke mediante o carregamento do arquivo cluster.yml(Nome Opcional) que deve ser gerado, a partir desse arquivo é que se dará a instalação. (Vamos deixar pronto)
     cd /etc/ansible 
	 wget https://github.com/rancher/rke/releases/download/v1.2.1/rke_linux-amd64
     sudo mv rke_linux-amd64 rke
     sudo mv rke /usr/local/bin/
     sudo chmod +x /usr/local/bin/rke
	 
5 - Gerando o arquivo de criação do cluster
	rke config cluster.yml ( O nome do arquivo fica a seu critério) Será gerada uma saída semelhante a demonstrada abaixo
	[+] Cluster Level SSH Private Key Path [~/.ssh/id_rsa]: -- > /home/admincluster/.ssh/id_rsa
	[+] Number of Hosts [1]:  -- > 3 (Aqui deverá ser quantos hosts terá no cluster no nosso caso 3 pois o bastion (162.168.1.19) na entra)
	[+] SSH Address of host (1) [none]: 192.168.1.20 -- > IP do primeiro servidor
	[+] SSH Port of host (1) [22]: -- > Enter se os servidor estiver ouvindo o ssh na Porta 22
	[+] SSH Private Key Path of host (192.168.1.20) [none]: -- > Pressione Enter
	[-] You have entered empty SSH key path, trying fetch from SSH key parameter
	[+] SSH Private Key of host (192.168.1.20) [none]: -- > Pressione Enter
	[-] You have entered empty SSH key, defaulting to cluster level SSH key: /home/admincluster/.ssh/id_rsa
	[+] SSH User of host (192.168.1.20) [ubuntu]: -- > admincluster
	[+] Is host (192.168.1.20) a Control Plane host (y/n)? [y]: y ( Iremos eleger 1 servidor Master )
	[+] Is host (192.168.1.20) a Worker host (y/n)? [n]: n ( Nao )
	[+] Is host (192.168.1.20) an etcd host (y/n)? [n]: y (Esse host será também o banco de dados chave-valor do cluster)
	[+] Override Hostname of host (192.168.1.20) [none]: (Enter)
	[+] Internal IP of host (192.168.1.20) [none]: (Enter)
	[+] Docker socket path on host (192.168.1.20) [/var/run/docker.sock]: (Enter)
	[+] SSH Address of host (2) [none]: 192.168.1.21 -- > IP do segundo servidor 
	[+] SSH Port of host (2) [22]: -- > Enter se o servidor estiver ouvindo o ssh na Porta 22
	[+] SSH User of host (192.168.1.21) [ubuntu]: -- > admincluster
	[+] Is host (192.168.1.21) a Control Plane host (y/n)? [y]:n ( Não iremos definir esse servidor como master )
	[+] Is host (192.168.1.21) a Worker host (y/n)? [n]: y ( Esse servidor será um Worker )
	[+] Is host (192.168.1.21) an etcd host (y/n)? [n]: (Não iremos definir esse servidor como master)
	[+] Override Hostname of host (192.168.1.21) [none]: (Enter)
	[+] Internal IP of host (192.168.1.21) [none]: (Enter)
	[+] Docker socket path on host (192.168.1.21) [/var/run/docker.sock]: (Enter)
	[+] SSH Address of host (3) [none]: 192.168.1.22 -- >  IP do segundo servidor
	[+] SSH Port of host (3) [22]:
	[+] SSH Private Key Path of host (192.168.1.22) [none]: /home/admincluster/.ssh/id_rsa
	[+] SSH User of host (192.168.1.22) [ubuntu]: -- > admincluster
	[+] Is host (192.168.1.22) a Control Plane host (y/n)? [y]:n ( Não iremos definir esse servidor como master )
	[+] Is host (192.168.1.22) a Worker host (y/n)? [n]: y -- > ( Esse servidor será um Worker )
	[+] Is host (192.168.1.22) an etcd host (y/n)? [n]:n ( Não iremos definir esse servidor como master )
	[+] Override Hostname of host (192.168.1.22) [none]: (Enter)
	[+] Internal IP of host (192.168.1.22) [none]: (Enter)
	[+] Docker socket path on host (192.168.1.22) [/var/run/docker.sock]: (Enter)
	[+] Network Plugin Type (flannel, calico, weave, canal) [canal]: (Enter)
	[+] Authentication Strategy [x509]: (Enter)
	[+] Authorization Mode (rbac, none) [rbac]: (Enter)
	[+] Kubernetes Docker image [rancher/hyperkube:v1.19.3-rancher1]: (Enter)
	[+] Cluster domain [cluster.local]: (Enter)
	############################################################################
	Edite o arquivo cluster.yml e modifique a linha addon_job_timeout: 120
	############################################################################
	
## Crie um arquivo hosts que será usado pelo ANSIBLE especificando os grupos como no exemplo abaixo com o nome que voce quiser: .
## sudo vim /etc/ansible/hosts
ansible_user=admincluster
become=true
[nodes]
192.168.1.20
192.168.1.21
192.168.1.22
[admin]
192.168.1.19

 5 - Execute esse playbook como usuario admincluster
 cd /etc/ansible
 ansible-playbook play-cria-cluster-rancher.yml --skip-tag remove
 
 6 - Uma vez o cluster criado para poder testar edite o arquivo hosts da sua máquina e aponte para o worker1 (Geralmente o pod sobe no worker1)
 no hosts da sua máquina:
 
 192.168.1.21 rancher.homelab.lab.local (Altere para o nome que voce escolheu para seu cluster e ou domínio)
 
 Acesse por ex: https://rancher.homelab.lab.local
 
############################################################################
                    DICAS IMPORTANTES
############################################################################
 
CASO QUEIRA REMOVER O CLUSTER RANCHER FAÇA:
# cd /etc/ansible
rke remove
Are you sure you want to remove Kubernetes cluster [y/n]: y

PARA REMOVER O DOCKER
ansible-playbook play-cria-cluster-rancher.yml --tag remove

