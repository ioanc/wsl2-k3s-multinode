# wsl2 k3s multinode

## Info
- [K3S](https://k3s.io/) is a certified kubernetes distro for edge and other scenarios
- [WSL2](https://docs.microsoft.com/en-us/windows/wsl/wsl2-install) runs Linux on windows, nicer and more integrated than a traditional vm on hyper-v


## Tools and resources
- [Install Rio on K3S running in WSL2 on Windows](https://gist.github.com/juliostanley/622486d82cb8ed334d270ead5f684e89)
- [k3d](https://github.com/rancher/k3d) - K3S in docker
- [Windows Terminal](https://github.com/microsoft/terminal) - great experience
- [docker export](https://docs.docker.com/engine/reference/commandline/export/) - export running container to rootfs


## Requirements
- Windows 10 2004 OS Build 19041.xxx or newer with WSL installed
- A Linux system where [docker](https://docs.docker.com/) is installed and running
- [Ubuntu](https://aka.ms/wsl-ubuntu-1804) installed from the Microsoft Store
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) installed on your system 


## Steps


 #### Prepare and created the K3S image to be imported in WSL2

- open Windows Terminal and connect to your WLS Ubuntu instance

  - connect to your Linux system that runs docker using ssh ; I am using a VM running Ubuntu 16.04.6 LTS
  
  - create a working folder, to be used for creating the K3S image for to be used in WSL2

    ``` 
      mkdir k3s-wsl2
    ```
    
  - prepare the files to created the K3S image and container ```profile``` and ```Dockerfile```
  
    ```
      cat <<EOF > ./profile
      export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/bin/aux
      mkdir /sys/fs/cgroup/systemd
      mount -t cgroup -o none,name=systemd cgroup /sys/fs/cgroup/systemd
      EOF
    ```
     
    ```
      cat <<EOF > ./Dockerfile
      FROM rancher/k3s:v1.17.4-k3s1-amd64
      COPY ./profile /etc/profile
      EOF 
    ```
  - create the K3S image, start a container and export container’s filesystem as a tar archive
  
    ```
      docker build . --tag wsl2-k3s-v1.17.4-k3s1
      docker run --name wsl2-k3s wsl2-k3s-v1.17.4-k3s1
      docker export --output wsl2-k3s-v1.17.4-k3s1.tar wsl2-k3s
    ```
  
  - copy the tar file, ```wsl2-k3s-v1.17.4-k3s1.tar```, to your local Windows system, e.g ```C:\wsl2\```
  

 #### Create a new WSL2 instance using the K3S image and K3S as a server, in the current user context

- in Windows Terminal open a PowerShell instance

  - change the working directory to ```c:\wsl2```
  
  - create a folder, where the files of the k3s-node-0 WSL2 instace will be saved
  
    ```powershell 
      PS C:\>C:\wsl2\ mkdir C:\wsl2\k3s-node-0 
    ```
  
  - import the K3S image, that was copied eralier from your Linux box, as a WSL instance version 2
  
    ```powershell 
      PS C:\>C:\wsl2\ wsl --import k3s-node-0 C:\wsl2\k3s-node-0\ C:\wsl2\wsl2-k3s-v1.17.4-k3s1.tar --version 2 
    ```
  
  - check created instance - it will use version 2
  
    ```powershell 
      PS C:\>C:\wsl2\ wsl -l -v
      NAME          STATE           VERSION
      * Ubuntu      Running         1
      k3s-node-0    Stopped         2
      PS C:\> 
    ```
  
  - start the K3S instance

    ```powershell 
      PS C:\>C:\wsl2\ wsl -d k3s-node-0 
    ```
  
  - at the new promp, check k3s version, add and ID to ```/etc/machine-id``` and start the K3S server
    
    ```
      / #
      / # k3s --version
      k3s version v1.17.4+k3s1 (3eee8ac3)
      / #
      / # od -x /dev/urandom | head -1 | awk '{OFS="-"; print $2$3,$4,$5,$6,$7$8$9}' > /etc/machine-id
      / #
      / # nohup k3s server &
      / #      
    ```
  
  - several seconds later, K3s is running
  
    ```
      / # kubectl get nodes
      NAME       STATUS   ROLES    AGE   VERSION
      pc0qap6g   Ready    master   10m   v1.17.4+k3s1
      / #
      
      / # kubectl get pods --all-namespaces
      NAMESPACE     NAME                                      READY   STATUS      RESTARTS   AGE
      kube-system   local-path-provisioner-58fb86bdfd-wqcll   1/1     Running     0          17m
      kube-system   coredns-6c6bb68b64-xz8pd                  1/1     Running     0          17m
      kube-system   metrics-server-6d684c7b5-hkxfp            1/1     Running     0          17m
      kube-system   helm-install-traefik-sr67m                0/1     Completed   0          17m
      kube-system   svclb-traefik-nppmc                       2/2     Running     0          14m
      kube-system   traefik-7b8b884c8-glxdp                   1/1     Running     0          14m
      / #      
    ```
  - get the IP address of the K3S server and the token; we need this to add the second node to the cluster

    ```
      / # ip addr sho eth0 scope global|grep inet|awk '{print $2}'|awk -F'/' '{print $1}'
      172.23.120.165
      / #
      / # cat /var/lib/rancher/k3s/server/node-token
      / #
      K1034daef0594ec255977adc416af8c57bf2eb914167fc3ee241ea134a246621c6c::server:175d0215b69ffacc9bd03982ae8a3bf9
      / #
      
    ```

 #### Create a new WSL2 instance, in the context of another local user, using the K3S image and add the new K3S node to the cluster

- open PowerShell with Admin rights

  - create a new local user and assign it a password
    ```powershell
      PS C:\> $Password = Read-Host -AsSecureString
      *********
      PS C:\> New-LocalUser -Name "k3s-node-1" -AccountNeverExpires -Password $Password

      Name       Enabled Description
      ----       ------- -----------
      k3s-node-1 True
 
      PS C:\>
    ```
  - next, we will run PowerShell in the new user context to create the folder to install the new WSL2 instance, 
    and import the K3S image

    ```powershell
      PS C:\>

      PS C:\> $Credential = Get-Credential k3s-node-1
      PS C:\> start-process powershell -credential $Credential -LoadUserProfile -ArgumentList "mkdir c:\wsl2\k3s-node-1;exit"
      PS C:\> start-process powershell -credential $Credential -LoadUserProfile -ArgumentList "wsl --import k3s-node-1 C:\wsl2\k3s-node-1\ C:\wsl2\wsl2-k3s-v1.17.4-k3s1.tar --version 2"
      PS C:\> start-process powershell -credential $Credential -LoadUserProfile -ArgumentList "wsl -l -v;start-sleep 10;exe"
      PS C:\>
    ```
    
  - we can now start the new WSL2 instance and check the verison of the K3S; a new window will pop up and close in 10sec
    ```powershell
      PS C:\> start-process powershell -credential $Credential -LoadUserProfile -WorkingDirectory c:\wsl2 -ArgumentList "wsl -d k3s-node-1 k3s --version;start-sleep 10;exit"
    ```
  - before starting ```k3s agent``` we need to create a soft link for ```mount``` in ```/bin``` folder
    in the interactive mode ```/bin/aux/``` is part of the ```$PATH``` but not when running ```wsl ---exec $PATH```
    
    ```powershell
      PS C:\> Start-Job -Credential $Credential -Name k3s-node-1 -ScriptBlock {wsl -d k3s-node-1 ln -s /bin/aux/mount /bin/mount}    
      
      Id     Name            PSJobTypeName   State         HasMoreData     Location             Command
      --     ----            -------------   -----         -----------     --------             -------
      19     k3s-node-1      BackgroundJob   Running       True            localhost            wsl -d k3s-node-1 ln -...
    ```
  
  - we prepare the arguments and start the ```k3s agent``` using ```Start-Job```
    this is needed becase ```wsl --exe``` does not accept special characters like ```&```

    ```powershell
      PS C:\> $script = {wsl -d k3s-node-1 -e k3s agent --server https://172.23.120.165:6443 --token K1034daef0594ec255977adc416af8c57bf2eb914167fc3ee241ea134a246621c6c::server:175d0215b69ffacc9bd03982ae8a3bf9 --with-node-id}
      PS C:\>   
      PS C:\> Start-Job -Credential $Credential -Name k3s-node-1 -ScriptBlock $script

      Id     Name            PSJobTypeName   State         HasMoreData     Location             Command
      --     ----            -------------   -----         -----------     --------             -------
      21     k3s-node-1      BackgroundJob   Running       True            localhost            wsl -d k3s-node-1 -e k...
    ``` 
  
 #### Checking results and exporting kubeconfig

- in Windows Terminal open a PowerShell instance

  - check using ```wsl``` the status of the cluster and nodes
  
    ```powershell
      PS C:\> wsl -l -v
      NAME          STATE           VERSION
      * Ubuntu      Running         1
      k3s-node-0    Running         2
      PS C:\>
      PS C:\> wsl -d k3s-node-0 kubectl get nodes
      NAME                STATUS   ROLES    AGE     VERSION
      pc0qap6g-2ea44fa8   Ready    <none>   63m     v1.17.4+k3s1   172.23.114.210   <none>        Unknown    4.19.84-microsoft-standard   containerd://1.3.3-k3s2
      pc0qap6g            Ready    master   5h20m   v1.17.4+k3s1   172.23.120.165   <none>        Unknown    4.19.84-microsoft-standard   containerd://1.3.3-k3s2
      PS C:\>
      PS C:\> wsl -d k3s-node-0 kubectl top nodes
      NAME                CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
      pc0qap6g            133m         3%     684Mi           5%
      pc0qap6g-2ea44fa8   19m          0%     109Mi           0%
      PS C:\>
      PS C:\> wsl -d k3s-node-0 kubectl get pods --all-namespaces
      NAMESPACE     NAME                                      READY   STATUS      RESTARTS   AGE
      kube-system   local-path-provisioner-58fb86bdfd-wqcll   1/1     Running     0          5h18m
      kube-system   coredns-6c6bb68b64-xz8pd                  1/1     Running     0          5h18m
      kube-system   metrics-server-6d684c7b5-hkxfp            1/1     Running     0          5h18m
      kube-system   helm-install-traefik-sr67m                0/1     Completed   0          5h18m
      kube-system   svclb-traefik-nppmc                       2/2     Running     0          5h15m
      kube-system   traefik-7b8b884c8-glxdp                   1/1     Running     0          5h15m
      kube-system   svclb-traefik-mntpx                       2/2     Running     0          24m
      PS C:\>
    ```
    
  - extract the kubeconfig file from the cluster, replace 127.0.0.1 with the eth0 ip address of the k3s server
    config file saved to c:\wsl2\k3s.yaml
    
    ```powershell
      PS C:\wsl2> (wsl -d k3s-node-0 cat /etc/rancher/k3s/k3s.yaml) -replace '127.0.0.1',(wsl -d k3s-node-0 ip addr show eth0|?{$_ -match "inet "}).split()[5].split("/")[0] > c:\wsl2\k3s.yaml
    ```
   
  - use local ```kubectl``` and check cluster resource
  
    ```powershell
      PS C:\wsl2>
      PS C:\wsl2> kubectl --kubeconfig=c:\wsl2\k3s.yaml get nodes
      NAME                STATUS    ROLES     AGE       VERSION
      pc0qap6g-2ea44fa8   Ready     <none>    83m       v1.17.4+k3s1
      pc0qap6g            Ready     master    5h41m     v1.17.4+k3s1
      PS C:\wsl2>
      PS C:\wsl2> kubectl --kubeconfig=c:\wsl2\k3s.yaml top nodes
      NAME                CPU(cores)   CPU%      MEMORY(bytes)   MEMORY%
      pc0qap6g            125m         3%        686Mi           5%
      pc0qap6g-2ea44fa8   20m          0%        109Mi           0%
      PS C:\wsl2>
      PS C:\wsl2> kubectl --kubeconfig=c:\wsl2\k3s.yaml get pods --all-namespaces
      NAMESPACE     NAME                                      READY     STATUS      RESTARTS   AGE
      kube-system   local-path-provisioner-58fb86bdfd-wqcll   1/1       Running     0          5h41m
      kube-system   coredns-6c6bb68b64-xz8pd                  1/1       Running     0          5h41m
      kube-system   metrics-server-6d684c7b5-hkxfp            1/1       Running     0          5h41m
      kube-system   helm-install-traefik-sr67m                0/1       Completed   0          5h41m
      kube-system   svclb-traefik-nppmc                       2/2       Running     0          5h38m
      kube-system   traefik-7b8b884c8-glxdp                   1/1       Running     0          5h38m
      kube-system   svclb-traefik-mntpx                       2/2       Running     0          47m
      PS C:\wsl2>
    ```

## DO TO
  still looking for a way to get the rootfs of an image , without the need to have a Linux  /Windows computer that is running docker
