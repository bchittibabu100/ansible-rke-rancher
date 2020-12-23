# Ansible + RKE - Instruções de Uso  
																																						
Data Criação: 23/12/2020  
  
Qualquer dúvida por favor me envie um e-mail:   
leandrojpg@gmail.com	  																																						
																																											
O RKE É UM BINARIO GO QUE FAZ A CRIAÇÃO FACILITADA DE UM CLUSTER KUBERNETES.
																									
PARA QUE O CLUSTER SEJA CRIADO VIA ANSIBLE UTILIZANDO O RKE É NECESSÁRIO EXECUTAR A PLAYBOOK COM UM USUÁRIO COMUN, AQUI CHAMADO DE ADMINCLUSTER. PORÉM SERÃO NECESSÁRIOS ALGUNS AJUSTES PARA A EXECUÇÃO DESSA PLAY.

# OBS: Não será contemplada a instalação do Ansible
Ambiente de exemplo: 
- 192.168.1.19 --> SERVIDOR ANSIBLE   
- 192.168.1.20 -- > MASTER - ETCD
- 192.168.1.21 -- > WORKER1  
- 192.168.1.22 -- > WORKER2  

Caso não tenha um dns use o arquivo hosts:  
192.168.1.19 ansible.homelab.lab.local    
192.168.1.20 master.homelab.lab.local  
192.168.1.20 worker1.homelab.lab.local  
192.168.1.20 worker2.homelab.lab.local  
 
Copie o arquivo para todos o nodes. (Será solicitada a senha de root dos servidores)    
scp /etc/hosts root@192.168.1.19:/etc/       
scp /etc/hosts root@192.168.1.20:/etc/    
scp /etc/hosts root@192.168.1.21:/etc/      
scp /etc/hosts root@192.168.1.22:/etc/      

 
## Ajustes 
- Logue no servidor do Ansible e execute os comandos abaixo:
mkdir /tmp/ajustes  
touch /tmp/ajustes/lista_nodes.txt  
touch /tmp/ajustes/ajustes.sh  

- Crie o arquivo abaixo, esse será a lista dos seus nodes:  
vim /tmp/ajustes/lista_nodes.txt  
192.168.1.19  
192.168.1.20  
192.168.1.21  
192.168.1.22

- Insira dentro do arquivo /tmp/ajustes/ajustes.sh
Esse pequeno script cria em todos os servidores o usuário admincluster e seta uma senha (s&nh@)  e ajusta o sudoers. (Altere o trecho echo "s&nh@" para a sua senha de preferência )

  #!/bin/bash
  for i in $(cat /tmp/ajustes/lista_nodes.txt);do  
  ssh root@$i 'echo "Criando o usuario comun admincluster...";  
  echo "###############################################";
  sleep 3 ;  
  useradd admincluster ;  
  echo "Configura senha do usuario admincluster...";  
  echo "###############################################";  
  sleep 3 ;  
  echo "s&nh@" | passwd --stdin admincluster;  
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

-  Execute o script
   sh /tmp/ajustes/ajustes.sh
   

- Faça um clone do projeto
  cd /tmp/ajustes  
  git clone https://github.com/leandrojpg/ansible-rke-rancher.git  

- Copie o arquivo play-cria-cluster-rancher.yml para dentro da raiz de execução do ansible (Assumo que esteja em /etc/ansible - Altera se necessário).  
cp ansible-cluster-rke/play-cria-cluster-rancher.yml /etc/ansible   


- Logue com o usuário admincluster no servidor do ANSIBLE aqui representado pelo 192.168.1.19  
 su - admincluster

- Gere a chave rsa do usuario admincluster  
 ssh-keygen (Pressione enter em todas as perguntas)  

- Copie a chave para todos os servidores incluindo o bastion    
 ssh-copy-id 192.168.1.19 (Será solicitada a senha de root de todos os nodes)   
 ssh-copy-id 192.168.1.20 (Será solicitada a senha de root de todos os nodes)   
 ssh-copy-id 192.168.1.21 (Será solicitada a senha de root de todos os nodes)   
 ssh-copy-id 192.168.1.22 (Será solicitada a senha de root de todos os nodes)   
 

- A criação do cluster é dependente do binário do rke mediante o carregamento do arquivo cluster.yml  
  que deve ser gerado, a partir desse arquivo é que se dará a instalação. (Vamos deixar pronto)  

   cd /etc/ansible   
   wget https://github.com/rancher/rke/releases/download/v1.2.1/rke_linux-amd64  
   sudo mv rke_linux-amd64 rke  
   sudo mv rke /usr/local/bin/  
   sudo chmod +x /usr/local/bin/rke  

- Gerando o arquivo de criação do cluster  
rke config cluster.yml ( O nome do arquivo fica a seu critério) Será gerada uma saída semelhante a demonstrada abaixo  

[+] Cluster Level SSH Private Key Path [~/.ssh/id_rsa]: -> /home/admincluster/.ssh/id_rsa  

[+] Number of Hosts [1]: - > 3 ( Exceto o servidor do Ansible )  

[+] SSH Address of host (1) [none]: 192.168.1.20 -> IP do primeiro servidor  

[+] SSH Port of host (1) [22]: -> Enter se o servidor estiver ouvindo o ssh na Porta 22  

[+] SSH Private Key Path of host (192.168.1.20) [none]: -> Pressione Enter  

[-] You have entered empty SSH key path, trying fetch from SSH key parameter

[+] SSH Private Key of host (192.168.1.20) [none]: -- > Pressione Enter

[-] You have entered empty SSH key, defaulting to cluster level SSH key: /home/admincluster/.ssh/id_rsa

[+] SSH User of host (192.168.1.20) [ubuntu]: -- > admincluster

[+] Is host (192.168.1.20) a Control Plane host (y/n)? [y]: y ( Iremos eleger 1 servidor Master )

[+] Is host (192.168.1.20) a Worker host (y/n)? [n]: n ( Nao )

[+] Is host (192.168.1.20) an etcd host (y/n)? [n]: y (Esse sera nosso master + banco de dados etcd)

[+] Override Hostname of host (192.168.1.20) [none]: (Enter)

[+] Internal IP of host (192.168.1.20) [none]: (Enter)

[+] Docker socket path on host (192.168.1.20) [/var/run/docker.sock]: (Enter)

[+] SSH Address of host (2) [none]: 192.168.1.21 -- > IP do segundo servidor 

[+] SSH Port of host (2) [22]: -- > Enter se o servidor estiver ouvindo o ssh na Porta 22

[+] SSH User of host (192.168.1.21) [ubuntu]: -- > admincluster

[+] Is host (192.168.1.21) a Control Plane host (y/n)? [y]:n ( Não será um master, e sim um worker )

[+] Is host (192.168.1.21) a Worker host (y/n)? [n]: y ( Sim, será um Worker )

[+] Is host (192.168.1.21) an etcd host (y/n)? [n]: ( Não sera um master, e sim um worker )

[+] Override Hostname of host (192.168.1.21) [none]: (Enter)

[+] Internal IP of host (192.168.1.21) [none]: (Enter)

[+] Docker socket path on host (192.168.1.21) [/var/run/docker.sock]: (Enter)

[+] SSH Address of host (3) [none]: 192.168.1.22 -- >  IP do segundo servidor

[+] SSH Port of host (3) [22]:

[+] SSH Private Key Path of host (192.168.1.22) [none]: /home/admincluster/.ssh/id_rsa

[+] SSH User of host (192.168.1.22) [ubuntu]: -- > admincluster

[+] Is host (192.168.1.22) a Control Plane host (y/n)? [y]:n ( Não será um master, e sim um worker ) 

[+] Is host (192.168.1.22) a Worker host (y/n)? [n]: y -- > ( Sim, será um Worker )

[+] Is host (192.168.1.22) an etcd host (y/n)? [n]:n ( Não sera um master e sim um worker )

[+] Override Hostname of host (192.168.1.22) [none]: (Enter)

[+] Internal IP of host (192.168.1.22) [none]: (Enter)

[+] Docker socket path on host (192.168.1.22) [/var/run/docker.sock]: (Enter)

[+] Network Plugin Type (flannel, calico, weave, canal) [canal]: (Enter)

[+] Authentication Strategy [x509]: (Enter)

[+] Authorization Mode (rbac, none) [rbac]: (Enter)

[+] Kubernetes Docker image [rancher/hyperkube:v1.19.3-rancher1]: (Enter)

[+] Cluster domain [cluster.local]: (Enter)

#####################################################################################
Edite novamente o arquivo acima cluster.yml que acabamos de gerar e modifique  a linha addon_job_timeout: 120  
#####################################################################################

- Crie um arquivo hosts que será usado pelo ANSIBLE especificando os grupos como no exemplo abaixo com o nome que voce quiser: .

  sudo vim /etc/ansible/hosts  
  ansible_user=admincluster  
  become=true  
  [nodes]  
  192.168.1.20  
  192.168.1.21  
  192.168.1.22  
  [admin]  
  192.168.1.19  
- Execute esse playbook como usuario admincluster  
  su - admincluster  
  cd /etc/ansible  
  ansible-playbook play-cria-cluster-rancher.yml --skip-tag remove  

- Uma vez o cluster criado para poder testar edite o arquivo hosts da sua máquina e aponte para o worker1   (Geralmente o pod sobe no worker1)  
 
  192.168.1.21 rancher.homelab.lab.local (Altere para o nome que você escolheu para seu cluster)    
 
  Acesse por ex: https://rancher.homelab.lab.local  
 
########################################################

                          DICAS IMPORTANTES    
########################################################

Para instalação desse ambiente foi usado o cert-manager um gerenciador de certificado auto-assinado
Para evitar erros é de suma importância que a hora/data estejam sincronizadas .   
Sempre que você executa esse playbook é criado um arquivo rke-cluster-state ele guarda o estado do seu    cluster, ele é removido automaticamente quando você remover o cluster  

-  CASO QUEIRA REMOVER O CLUSTER RANCHER FAÇA:  
 cd /etc/ansible    
  rke remove  
  
  PARA REMOVER O DOCKER  
  ansible-playbook play-cria-cluster-rancher.yml --tag remove  
