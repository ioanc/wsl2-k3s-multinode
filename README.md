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


## Steps


 #### Prepare and created the K3S image to be imported in WSL2

- open Windows Terminal and connect to your WLS Ubuntu instance

  - connect to your Linux system using ssh ; I am using a VM running Ubuntu 16.04.6 LTS
  
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
  

 #### Create a new WSL2 instance using the K3S image and that K3S as a server

- in Windows Terminal open a PowerShell instance

  - change the working directory to ```c:\wsl2```
  
  - create a folder, where the files of the k3s-node-0 WSL2 instace will be saved
  
    ``` 
      PS C:\>C:\wsl2\ mkdir C:\wsl2\k3s-node-0 
    ```
  
  - import the K3S image, that was copied eralier from your Linux box, as a WSL instance version 2
  
    ``` 
      PS C:\>C:\wsl2\ wsl --import k3s-node-0 C:\wsl2\k3s-node-0\ C:\wsl2\wsl2-k3s-v1.17.4-k3s1.tar --version 2 
    ```
  
  - check created instance - it will use version 2
  
    ``` 
      PS C:\>C:\wsl2\ wsl -l -v
      NAME          STATE           VERSION
      * Ubuntu      Running         1
      k3s-node-0    Stopped         2
      PS C:\> 
    ```
  
  - start the K3S instance

    ``` 
      PS C:\>C:\wsl2\ wsl -d k3s-node-0 
    ```
  
  - at the new promp, chec kk3s version and start the K3S server
    
    ```
      / #
      / # k3s --version
      k3s version v1.17.4+k3s1 (3eee8ac3)
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
