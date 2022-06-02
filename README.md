How to create a Kubernetes cluster with Kubeadm and Ansible on Ubuntu 22.04
Introduction
Kubernetes is an open source platform, actively developed by a community around the world, which enables the management and orchestration of large-scale application containers.
In the following tutorial its latest version is used, by configuring a Kubernetes cluster from scratch and using the Ansible software for the automation of the procedures and the Kubeadm tool for cluster creation on an Ubuntu 20.04 server.
Provided the servers in the cluster have enough CPU and RAM resources for the applications to run, by the end of this guide you will have a cluster ready to run containerized applications. Before proceeding with this tutorial, make sure you have at least two worker machines, and a master machine with at least 2 CPUs that can work properly with the worker machines.
First, connect to your server via SSH. If you haven't done so yet, following our guide is recommended to  connect securely with the SSH protocol . In case of a local server, go to the next step and open the terminal of your server.
Installing Ansible
First, update the system packages via the commands to proceed with the installation of Ansible:
sudo apt update
 
Then, proceed by installing the necessary software:
sudo apt install software-properties-common -y
sudo apt install ansible -y
 
After a few seconds, Ansible should be ready for use.
Configuring SSH login
To allow Ansibile to correctly work on remote nodes, add SSH keys on the target servers, in order to guarantee access without requiring a password.
Generate the two keys (private and public) first through the command:
# ssh-keygen -t rsa
 
 Generating public/private rsa key pair.
 Enter file in which to save the key (/root/.ssh/id_rsa):
 Enter passphrase (empty for no passphrase):
 Enter same passphrase again:
 
As shown above, you will be prompted for passwords, which you can leave blank.
Once completed, send the public key to the remote servers using the "ssh-copy-id" command:
# ssh-copy-id root@IP_WORKER_1
 
# ssh-copy-id root@IP_WORKER_2
 
# ssh-copy-id root@IP_MASTER
 
For each configured machine, enter the login credentials when required.
Configuring Master node
Create a directory called "kube-cluster" in the home directory of your master machine:
mkdir ~/kube-cluster
cd ~/kube-cluster
 
This will be the directory with all your Ansible playbooks for your workspace throughout the tutorial. In addition, under this directory all the local commands will be run.
Create an "inventory" file, called hosts, using nano or your favorite text editor:
nano ~/kube-cluster/hosts
 
Add the following text to the file specifying the remote machines information in your cluster:
[masters]
 master ansible_host=[IP_MASTER] ansible_user=root
 
[workers]
 worker1 ansible_host=[IP_WORKER_1] ansible_user=root
 worker2 ansible_host=[IP_WORKER_2] ansible_user=root
 
[all:vars]
 ansible_python_interpreter=/usr/bin/python3
 
The inventory files in Ansible are used to specify server information: IP addresses, remote users, and server groupings to be used as a single unit for executing commands. In this tutorial two Ansible groups (master and worker) have been added by specifying the logical structure of your cluster.
In the master group, there is a server entry named "master" that lists the IP of the master node (IP_MASTER) and specifies that Ansible should run remote commands as the system administrator (root user). In the workgroup, there are two entries for the worker servers (worker1 and worker2) which also specify ansible_user as root.
The last line of the file tells Ansible to use the remote servers' Python 3 interpreters for its management operations.
Save and close the file after adding the text.
Master Main user configuration
Now, create a non-root user with sudo privileges on all servers to manually log in via SSH as an unprivileged user.
Using an unprivileged user for typical tasks that occur while maintaining a cluster minimizes the risk of inadvertently performing dangerous operations or modifying or deleting important files.
Create a file called initial.yml in the workspace:
nano initial.yml
 
Then, add the following content:
- hosts: all
 become: yes
 tasks:
 - name: create the 'ubuntu' user
 user: name=ubuntu append=yes state=present createhome=yes shell=/bin/bash
 
 - name: allow 'ubuntu' to have passwordless sudo
 lineinfile:
  dest: /etc/sudoers
  line: 'ubuntu ALL=(ALL) NOPASSWD: ALL'
  validate: 'visudo -cf %s'
 
 - name: set up authorized keys for the ubuntu user
 authorized_key: user=ubuntu key="{{item}}"
 with_file:
  - ~/.ssh/id_rsa.pub
 
This playbook has the task of creating the non-root user "ubuntu", by configuring the "sudoers" file to allow the user "ubuntu" to execute sudo commands without a password prompt.
It is also responsible for adding the public key on your local machine (usually ~ / .ssh / id_rsa.pub) to the list of authorized keys of the remote "ubuntu" user. This will allow you to access each server via SSH as the "ubuntu" user.
Save and close the file after adding the text.
Next, run the playbook by running locally:
ansible-playbook -i hosts initial.yml
 
The command will be completed within a few minutes. When finished, outputs similar to the following will be shown:
Output
 PLAY [all] ****
 
 TASK [Gathering Facts] ****
 ok: [master]
 ok: [worker1]
 ok: [worker2]
 
 TASK [create the 'ubuntu' user] ****
 changed: [master]
 changed: [worker1]
 changed: [worker2]
 
 TASK [allow 'ubuntu' user to have passwordless sudo] ****
 changed: [master]
 changed: [worker1]
 changed: [worker2]
 
 TASK [set up authorized keys for the ubuntu user] ****
 changed: [worker1] => (item=ssh-rsa AAAAB3...)
 changed: [worker2] => (item=ssh-rsa AAAAB3...)
 changed: [master] => (item=ssh-rsa AAAAB3...)
 
 PLAY RECAP ****
 master  : ok=5 changed=4 unreachable=0 failed=0 
 worker1  : ok=5 changed=4 unreachable=0 failed=0 
 worker2  : ok=5 changed=4 unreachable=0 failed=0 
 Preliminary setup is complete.
Creation of the Kubernetes cluster
Now, install the required packages by Kubernetes at the operating system level with Ubuntu's package manager.
Create a file called kube-dependencies.yml in the workspace:
nano ~/kube-cluster/kube-dependencies.yml
 
Add the following instructions to the file to install these packages on your servers:
- hosts: all
 become: yes
 tasks:
 - name: install Docker
 apt:
  name: docker.io
  state: present
  update_cache: true
 
 - name: install APT Transport HTTPS
 apt:
  name: apt-transport-https
  state: present
 
 - name: add Kubernetes apt-key
 apt_key:
  url: https://packages.cloud.google.com/apt/doc/apt-key.gpg
  state: present
 
 - name: add Kubernetes' APT repository
 apt_repository:
 repo: deb http://apt.kubernetes.io/ kubernetes-xenial main
 state: present
 filename: 'kubernetes'
 
 - name: install kubelet
 apt:
  name: kubelet
  state: present
  update_cache: true
 
 - name: install kubeadm
 apt:
  name: kubeadm
  state: present
 
 - hosts: master
 become: yes
 tasks:
 - name: install kubectl
 apt:
  name: kubectl
  state: present
 
The newly pasted playbook has the task of:
•	Installing Docker, the container runtime;
•	Installing apt-transport-https, allowing you to add external HTTPS sources to the APT source list;
•	Adding the apt key of the Kubernetes APT repository for key verification;
•	Adding the Kubernetes APT repository to the remote server APT sources list;
•	Installing Kubelet and Kubeadm.
Save and close the file when done.
Next, run the playbook by running locally:
ansible-playbook -i hosts kube-dependencies.yml
 
When finished, outputs similar to the following will be shown:
Output
 PLAY [all] ****
 
 TASK [Gathering Facts] ****
 ok: [worker1]
 ok: [worker2]
 ok: [master]
 
 TASK [install Docker] ****
 changed: [master]
 changed: [worker1]
 changed: [worker2]
 
 TASK [install APT Transport HTTPS] *****
 ok: [master]
 ok: [worker1]
 changed: [worker2]
 
 TASK [add Kubernetes apt-key] *****
 changed: [master]
 changed: [worker1]
 changed: [worker2]
 
 TASK [add Kubernetes' APT repository] *****
 changed: [master]
 changed: [worker1]
 changed: [worker2]
 
 TASK [install kubelet] *****
 changed: [master]
 changed: [worker1]
 changed: [worker2]
 
 TASK [install kubeadm] *****
 changed: [master]
 changed: [worker1]
 changed: [worker2]
 
 PLAY [master] *****
 
 TASK [Gathering Facts] *****
 ok: [master]
 
 TASK [install kubectl] ******
 ok: [master]
 
 PLAY RECAP ****
 master  : ok=9 changed=5 unreachable=0 failed=0 
 worker1  : ok=7 changed=5 unreachable=0 failed=0 
 worker2  : ok=7 changed=5 unreachable=0 failed=0 
 
After execution, Docker, Kubeadm and Kubelet will be installed on all the remote servers.
Kubectl is not a required component and is only needed to run cluster commands. Installing it on the master node is useful only in this context, as only Kubectl commands will be run from the master.
However, Kubectl commands can be run from any worker node or any machine on which it can be installed and configured to point to a cluster.
All system dependencies are now installed.
At this point, configure the master node and initialize the cluster, before creating any playbooks.
Create an Ansible playbook named master.yml on your local computer:
nano ~/kube-cluster/master.yml
 
Add the following playback to the file to initialize the cluster and install Flannel:
- hosts: master
 become: yes
 tasks:
 - name: remove swap
 shell: "swapoff -a"
 
 - name: initialize the cluster
 shell: kubeadm init --pod-network-cidr=10.244.0.0/16 >> cluster_initialized.txt
 args:
  chdir: $HOME
  creates: cluster_initialized.txt
 
 - name: create .kube directory
 become: yes
 become_user: ubuntu
 file:
  path: $HOME/.kube
  state: directory
  mode: 0755
 
 - name: copy admin.conf to user's kube config
 copy:
  src: /etc/kubernetes/admin.conf
  dest: /home/ubuntu/.kube/config
  remote_src: yes
  owner: ubuntu
 
 - name: install Pod network
 become: yes
 become_user: ubuntu
 shell: kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml >> pod_network_setup.txt
 args:
  chdir: $HOME
  creates: pod_network_setup.txt
 
In detail:
•	By running the first Kubeadm task "init" the cluster is initialized, passing the argument --pod-network-cidr = 10.244.0.0 / 16 and therefore specifying the private subnet from which the pod IPs will be assigned. Flannel uses the old subnet by default - Kubeadm is told to use the same subnet.
•	With the second task the creation of a .kube directory in / home / ubuntu is initialized, and it will contain all the configuration information, such as the admin key files, needed to connect to the cluster, and the cluster API address.
•	With the third task the /etc/kubernetes/admin.conf file generated by kubeadm init to the non-root user's home directory will be copied. This will allow you to use kubectl to access the newly created cluster.
•	With the fourth task, you are running Kubectl apply to install Flannel. kubectl apply -f descriptor. [yml | json] is the syntax for telling Kubectl to create the objects described in the descriptor file. [yml | json]. The kube-flannel.yml file contains descriptions of the objects needed to configure Flannel in the cluster.
Save and close the file at the end.
Run the playbook locally, by running the following command:
ansible-playbook -i hosts master.yml
 
When done, outputs similar to the followingwill be shown:
PLAY [master] ****
 
 TASK [Gathering Facts] ****
 ok: [master]
 
 TASK [initialize the cluster] ****
 changed: [master]
 
 TASK [create .kube directory] ****
 changed: [master]
 
 TASK [copy admin.conf to user's kube config] *****
 changed: [master]
 
 TASK [install Pod network] *****
 changed: [master]
 
 PLAY RECAP ****
 master  : ok=5 changed=4 unreachable=0 failed=0 
 
To check the status of the master node, use this command:
ssh ubuntu@master_ip 
Once inside the master node, run:
kubectl get nodes
 
Now, the following output can be seen:
NAME STATUS ROLES AGE VERSION master Ready master 1d v1.14.0 
The output indicates that all the initialization tasks have been completed by the master node, which is now in a ready state and therefore ready to begin accepting worker nodes and executing tasks sent to the API server.
Configure worker nodes
Adding workers to the cluster involves running a single command on each of them.
This command includes the necessary cluster information, such as IP address, master API Server port, and a secure token. Nodes that pass into the secure token will be able to join the cluster later.
Go back to your workspace and create a playbook called workers.yml:
nano ~/kube-cluster/workers.yml
 
To add workers to the cluster, add the following configuration to the file:
- hosts: master
 become: yes
 gather_facts: false
 tasks:
 - name: get join command
 shell: kubeadm token create --print-join-command
 register: join_command_raw
 
 - name: set join command
 set_fact:
  join_command: "{{ join_command_raw.stdout_lines[0] }}"
 
 - hosts: workers
 become: yes
 tasks:
 - name: remove swap
 shell: "swapoff -a"
 
 - name: join cluster	
 shell: "{{ hostvars['master'].join_command }} >> node_joined.txt"
 args:
  chdir: $HOME
  creates: node_joined.txt
 
This playbook contains the instructions that have to be executed on worker nodes.
With the second replay, a single task executes the join command on all worker nodes.
At the end of this activity, the two worker nodes will be part of the cluster.
Save and close the file when done and run the playbook by running locally:
ansible-playbook -i hosts workers.yml
 
When done, outputs similar to the following will be shown:
Output
 PLAY [master] ****
 
 TASK [get join command] ****
 changed: [master]
 
 TASK [set join command] *****
 ok: [master]
 
 PLAY [workers] *****
 
 TASK [Gathering Facts] *****
 ok: [worker1]
 ok: [worker2]
 
 TASK [join cluster] *****
 changed: [worker1]
 changed: [worker2]
 
 PLAY RECAP *****
 master  : ok=2 changed=1 unreachable=0 failed=0 
 worker1  : ok=2 changed=1 unreachable=0 failed=0 
 worker2  : ok=2 changed=1 unreachable=0 failed=0 
 
With the addition of worker nodes, your cluster is now fully configured and functional, with workers ready to run workloads.
Check the correct functioning of the newly added clusters, using the Kubectl tool, as previously done:
ssh ubuntu master_ip
Run the following command to get the cluster status:
kubectl get nodes
Outputs similar to the following will be shown: 
Output NAME STATUS ROLES AGE VERSION master Ready master 1d v1.14.0 worker1 Ready  1d v1.14.0 worker2 Ready  1d v1.14.0
 
However, if some nodes have the status "NotReady", this may imply that the worker nodes have not completed the configuration yet. Within ten minutes, re-run the "kubectl get" nodes command and inspect the new output.
If the status change from NotReady to Ready still does not occur, check if you have correctly executed the commands in the previous steps and repeat them if necessary.
Now that your cluster has been successfully verified, you are ready to schedule remote operations within the Kubernetes cluster on your Ubuntu 22.04 server.