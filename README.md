# services

## 刪除所有 container

    docker rm -f $(docker ps -a -q)

# Swarm Init

    docker swarm init

## 初始化後，會出現一段訊息，讓 Node 加入 Swarm 集群

    docker swarm join --token SWMTKN-xxxx xxx.xxx.xxx.xx:2377

## 忘記 swarm join 以上指令怎麼辦?




## create-mysql-volume:
	docker volume create mysql

## create-network:
	docker network create --scope=swarm --driver=overlay my_network

## deploy-mysql:
	docker stack deploy --with-registry-auth -c mysql.yml mysql

## deploy-rabbitmq:
	docker stack deploy --with-registry-auth -c rabbitmq.yml rabbitmq

