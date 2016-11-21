docker 1.12 可以使用docker swarn mode 搭建docker 集群   
自动负载均衡 创建一个v(virtual)ip 
1. 创建管理节点
```
docker swarm init --advertise-addr 192.168.0.13
```
> Swarm initialized: current node (bd9efxu1gkqdsh08huby4p7vm) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-4asav6wyxm1rfz90hsb2l6fpjjd3pmu1qn2rr9cjl2q7tnbhvs-1bj23uvuez3dxin3y3r02frcx \
    192.168.0.13:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
2. 创建worker 节点

```
docker swarm join \
    --token SWMTKN-1-4asav6wyxm1rfz90hsb2l6fpjjd3pmu1qn2rr9cjl2q7tnbhvs-1bj23uvuez3dxin3y3r02frcx \
    192.168.0.13:2377
```
*查看节点*
```
docker node ls
```

```
ID                           HOSTNAME               STATUS  AVAILABILITY  MANAGER STATUS
325ojklfqsx1v7equy92893vc    localhost.localdomain  Ready   Active        
bd9efxu1gkqdsh08huby4p7vm *  ubuntu                 Ready   Active        Leader

```
3. 部署测试服务
replicas 是
```
docker service create --replicas 2  --publish 8090:80 --name helloworld nginx
```
4 增加服务

```
 docker service scale helloworld=5
```
5. 更新服务

```
docker service create \
  --replicas 3 \
  --name redis \
  --update-delay 10s \
  redis:3.0.6
```

```
docker service update --image redis:3.0.7 redis
redis
```

6 节点停止服务
*prevents a node from receiving new tasks from the swarm manager*

```
docker node update --availability drain worker1
```

```
docker node update --availability active worker1
```





