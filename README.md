# sonarqube_and_jenkins_demo

## docker-composer
sonarqube 中的 elasticsearch 需要 vm.max_map_count=262144

### linux 
#### 一次性

```shell
sysctl -w vm.max_map_count=262144
```

#### 永久修改
修改 /etc/sysctl.conf

```conf
vm.max_map_count=262144
```

### Windows 
Windows 使用 WSL2 來使用 Docker Desktop
所以只能次開機後修改，使用powershell
#### 一次性

```shell
wsl -d docker-desktop 
sysctl -w vm.max_map_count=262144
```
