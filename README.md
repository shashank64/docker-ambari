***This is a dead project. Use [docker-hdp](https://github.com/randerzander/docker-hdp) instead. ***


**Note**: The guide below assumes you've already setup Docker in your environment. See [Docker Toolbox](http://docs.docker.com/mac/started/) install instructions for help.

To build the container image (see the script in Dockerfile):
```
docker build -t randerzander/docker-ambari .
```

To run the container image once built:
```
# You can remove --privileged if you don't intend to use Kerberos
docker run -d -P -h docker.dev --privileged randerzander/docker-ambari
19f4d657579d 
```
The output of the "docker run" command will be a docker container identifier. You can refer to it in other docker commands using just the first 3 characters (avoids having to copy/paste or type the whole string).

To start a bash prompt inside the running container:
```
docker ps
randy$> docker ps
CONTAINER ID        IMAGE               COMMAND                CREATED             STATUS              PORTS        NAMES
19f4d657579d        randerzander/docker-ambari:latest        "/scripts/start-serv   14 minutes ago      Up 14 minutes       0.0.0.0:32872->2181/tcp, 0.0.0.0:32860->4040/tcp, 0.0.0.0:32845->6080/tcp, .....
randy$> docker exec -it 19f bash
bash-4.1# 
```
The list of mapped ports is long. Edit Dockerfile if you want to add to or remove from the list. You'll need to re-build your image each time to change the exposed ports.

To instruct VirtualBox to map all of your container's exposed ports to the same ports on your host machine's localhost:

**Note**:  Expect to see errors for every port the first time you run this. The script is telling VirtualBox to delete port mappings between the docker-machine vm and localhost, then create a new port-mapping for the new container.
```
python scripts/portbind.py vmname 19f
VBoxManage controlvm vmname natpf1 delete tcp9992
VBoxManage: error: Code NS_ERROR_INVALID_ARG (0x80070057) - Invalid argument value (extended info not available)
VBoxManage: error: Context: "RemoveRedirect(Bstr(a->argv[3]).raw())" at line 523 of file VBoxManageControlVM.cpp
VBoxManage controlvm vmname natpf1 tcp9992,tcp,,9992,,32806
VBoxManage controlvm vmname natpf1 delete tcp9991
VBoxManage: error: Code NS_ERROR_INVALID_ARG (0x80070057) - Invalid argument value (extended info not available)
VBoxManage: error: Context: "RemoveRedirect(Bstr(a->argv[3]).raw())" at line 523 of file VBoxManageControlVM.cpp
VBoxManage controlvm vmname natpf1 tcp9991,tcp,,9991,,32781
```

You should now be able to access Ambari via docker.dev:8080 in your host machine's browser. Most HDP packages are pre-installed via yum, so cluster install time should be short. ambari-agent is already running, so use the manual host-registration radio button instead of keyless ssh. You can ignore the few host-check warnings you see during registration (those are artifacts of host OS-container kernel interaction). If you use Hive in this container, Postgres is already configured for use as the metastore. Leave the settings as their defaults and use "dev" for the database password.

You will want to customise the docker-machine VM settings - I set my VM to have 8 vcores and 8GB of RAM.

If you have problems using "docker ps", "docker run" with network issues, run "eval $(docker-machine env vmname)".

If you build containers frequently, you'll find it's easy to fill up the VM's disk. Add the following function to ~/.bash_profile to make cleanup a little easier:
```
function docker-cleanup(){
    docker ps -a -q | xargs -n 1 -I {} docker rm {}
      docker rmi $(docker images | grep "^<none>" | awk "{print $3}")
}
```

Running the cleanup command "docker-cleanup" will remove unused images and free up disk space. The command will display output similar to:
```
Deleted: 02a980ee44d560e2f5dddc273e41c91f3b3d1817b79c3c0c1d1af33f761b6157
Deleted: 40e86cea7d23664e80d5e8b34264dc42b06a6f4bbb37810bae288f6dabf09266
Deleted: b8e492a21f863c649865181313b53f816a9736179a63eea78edbac9c197f588c
Deleted: c779b6325258ed60beed53ef5422de92b097ad4389df3f19cfcc9818f0ee2916
Deleted: f2cc857903ec084a8d7cc4afcd124cabe7780438df34da338698dc3dce4c2cdc
Deleted: b0bb1feeb73a668e5f36e3d980b360acf167270a4c1694d1acc4026e698c0474
Deleted: 032ebea010cd87468a40ae610a664b4de0f751bb3afe5e80c17b0711db906c12
Error response from daemon: No such image: 15
Error response from daemon: No such image: hours
Error response from daemon: No such image: ago
Error response from daemon: No such image: 7.393
Error response from daemon: No such image: GB
FATA[0009] Error: failed to remove one or more images 
```
Excuse my awful bash scripting. The errors can be ignored.

To list running containers: "docker ps"

When you are finished with your container, you can kill it: "docker stop 19f"

Or, if you'd like to save the state of your container as a new docker image:
```
docker commit container_name new_image_name
```

Then you can start a copy of the new image without having to go through the normal HDP cluster install process. You'll need to 'service ambari-agent restart' and tell Ambari to restart your services.

**Networking Issues:**

If you change networks on your host machine, the docker-machine VM wont pick up the changes. One fix is "docker-machine restart vmname". This has the annoying side effect of stopping any running containers. To fix the networks within currently running containers *and* any new containers started before docker-machine vm itself is restarted, edit /etc/resolv.conf and set nameserver's IP to your DNS server of choice (8.8.8.8 to use Google's, which usually works).

Big thanks to the Hortonworks/SequenceIQ team. Their own [docker-ambari repo](https://github.com/sequenceiq/docker-ambari) was the basis for this effort.

Useful to include in host machine's ~/.bash_profile:
```
function dcp(){
  local_file=$1
  container_file=$2
  container_id=$3
  echo Copying $1 to $2 on docker container: $3
  docker exec -i $container_id /bin/bash -c "cat > $container_file" < $local_file
  echo Done!
}
```
To copy files from host machine to your docker container: dcp local_file /docker/container/file container_id

A sample terminal and dev env setup script is provided in /scripts/dev-setup.sh:
```
docker exec -it 19f bash
./scripts/dev-setup.sh
```
