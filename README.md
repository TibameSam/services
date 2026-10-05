# services

本 repo 收錄 FinMind 教學中，所有後端服務的部屬設定檔（docker-compose / swarm yml），
方便讀者一鍵啟動完整的爬蟲、資料庫、訊息佇列、API、視覺化等基礎設施。

## 服務簡介

- **Portainer** (`portainer.yml`)：Docker Swarm 視覺化管理介面，可在瀏覽器查看與操作各 service。
- **MySQL** (`mysql.yml`, `mysql-gce.yml`)：主要資料庫，儲存爬取下來的金融資料。
  - `mysql.yml`：本機或自架機器使用
  - `mysql-gce.yml`：GCP GCE 環境使用
- **RabbitMQ** (`rabbitmq.yml`, `rabbitmq-gce.yml`)：分散式任務佇列，串接 producer 與 worker。
- **Crawler Worker** (`docker-compose-worker-network-version-swarm.yml`)：分散式爬蟲 worker，從 RabbitMQ 領取任務並執行爬蟲。
- **Crawler Producer** (`docker-compose-producer-duplicate-network-version.yml`)：發送爬蟲任務到 RabbitMQ。
- **API** (`docker-compose-api-network-version-swarm.yml`)：對外提供查詢服務的 FastAPI server。
- **Redash**
  - `docker-compose-redash.yml`：swarm 版本，用於 production／多機部屬
  - `docker-compose-redash-local.yml`：docker-compose 版本，用於本機教學或單機開發
- **Superset** (`superset.yml`)：BI 視覺化平台（教學精簡版，metadata 使用官方映像內建的 SQLite，免額外資料庫）。
- **Upload 任務** (`docker-compose-upload_taiwan_stock_price_to_mysql.yml`,
  `docker-compose-upload_taiwan_stock_margin_purchase_short_sale.yml`)：
  將爬取下來的資料寫入 MySQL 的批次任務。
  - `docker-compose-upload_taiwan_stock_margin_purchase_short_sale.yml`：swarm 版本
  - `docker-compose-upload_taiwan_stock_margin_purchase_short_sale-local.yml`：docker-compose 版本，用於本機教學或單機開發

## 部屬流程概覽

1. 安裝 docker（見下方「ubuntu 安裝 docker」）。
2. 初始化 swarm 並建立 overlay network。
3. 依序部屬 MySQL、RabbitMQ 等基礎服務。
4. 部屬 crawler worker / producer 開始爬取資料。
5. 部屬 API 對外提供查詢，或啟動 Redash 進行資料視覺化。

以下為各步驟對應的指令。

## 刪除所有 container

    docker rm -f $(docker ps -a -q)

# Swarm

## init

    docker swarm init

## 初始化後，會出現一段訊息，讓 Node 加入 Swarm 集群

    docker swarm join --token SWMTKN-xxxx xxx.xxx.xxx.xx:2377

## 忘記 docker swarm join 以上指令怎麼辦?

	docker swarm join-token worker
	docker swarm join-token manager

## 部屬 Portainer 服務

	docker pull portainer/portainer-ce:2.0.1
	docker pull portainer/agent
	docker stack deploy -c portainer.yml por

## 打開 Portainer 服務

http://127.0.0.1:9000

## create-mysql-volume:
	docker volume create mysql

## remove-network
	docker network rm my_swarm_network

## create-network:
	docker network create --scope=swarm --driver=overlay my_swarm_network
	docker network create --scope=swarm --driver=overlay --attachable my_attachable_network

## deploy-mysql:
	docker stack deploy --with-registry-auth -c mysql.yml mysql
	docker stack deploy --with-registry-auth -c mysql-gce.yml mysql

## deploy-rabbitmq:
	docker stack deploy --with-registry-auth -c rabbitmq.yml rabbitmq
	docker stack deploy --with-registry-auth -c rabbitmq-gce.yml rabbitmq

## 離開 docker swarm
	docker swarm leave --force

## deploy-crawler:
	DOCKER_IMAGE_VERSION=0.0.6 docker stack deploy --with-registry-auth -c docker-compose-worker-network-version-swarm.yml crawler
	DOCKER_IMAGE_VERSION=0.0.6.arm64 docker stack deploy --with-registry-auth -c docker-compose-worker-network-version-swarm.yml crawler

## 發送任務:
	DOCKER_IMAGE_VERSION=0.0.6 docker stack deploy --with-registry-auth -c docker-compose-producer-duplicate-network-version.yml crawler
	DOCKER_IMAGE_VERSION=0.0.6.arm64 docker stack deploy --with-registry-auth -c docker-compose-producer-duplicate-network-version.yml crawler

## 部屬 API:
	DOCKER_IMAGE_VERSION=0.0.1 docker stack deploy --with-registry-auth -c docker-compose-api-network-version-swarm.yml api
	DOCKER_IMAGE_VERSION=0.0.1.arm64 docker stack deploy --with-registry-auth -c docker-compose-api-network-version-swarm.yml api

## rm stack
	docker stack rm airflow api crawler mysql rabbitmq

## ubuntu 安裝 docker

	sudo apt-get update
	sudo apt-get install docker.io -y

## 將你的帳號加入 docker group
	sudo usermod -aG docker $USER

## 登入 docker
	docker login -u linsamtw

## upload_taiwan_stock_price_to_mysql
	DOCKER_IMAGE_VERSION=0.0.6 docker stack deploy --with-registry-auth -c docker-compose-upload_taiwan_stock_price_to_mysql.yml upload

## 設定 linode hostname
	sudo hostname manager
	sudo hostname mysql
	sudo hostname rabbitmq
	sudo hostname airflow

## 啟動 redash
	docker stack deploy -c docker-compose-redash.yml redash

## 啟動 redash (docker-compose 版本，非 swarm)
	docker compose -f docker-compose-redash-local.yml up -d

## 關閉 redash (docker-compose 版本)
	docker compose -f docker-compose-redash-local.yml down

## 移除已建立完 table 的 create_table container (docker-compose 版本)
	docker compose -f docker-compose-redash-local.yml rm -f create_table

## 啟動 superset
	# 官方映像 + SQLite，免預建資料庫；首次啟動會自動初始化（約 1~2 分鐘）
	# 預設管理者帳密 admin / admin，介面 http://localhost:8088
	docker stack deploy --with-registry-auth -c superset.yml superset

## 關閉 superset
	docker stack rm superset

## superset 連線 MySQL

`superset.yml` 使用 `apache/superset:4.1.4-dev` 映像（dev 版），已內建 `mysqlclient`
等多種 DB driver，故 "Connect a database" 可直接選 MySQL，免額外安裝。
（lean 版 `apache/superset:4.1.4` 不含 MySQL driver，清單才不會出現 MySQL。）

連線時於 UI 填入 SQLAlchemy URI：

	mysql://<帳號>:<密碼>@<host>:3306/<資料庫名>

對照 `mysql.yml` 的設定，直接填以下任一即可：

	# root 帳號
	mysql://root:ppWgnb_mfGe2m_@mysql_mysql:3306/mydb

	# 一般帳號（權限限於 mydb）
	mysql://user:ppWgnb_mfGe2m_@mysql_mysql:3306/mydb

各欄位來源（皆來自 `mysql.yml`）：

- 帳號／密碼：`MYSQL_USER` / `MYSQL_PASSWORD`，或 root / `MYSQL_ROOT_PASSWORD`
- 資料庫：`MYSQL_DATABASE`（預設 `mydb`）
- host：用 `mysql.yml` 部署時，stack 名與 service 名皆為 `mysql`，
  Superset 在不同 stack，須用完整 DNS 名 `<stack>_<service>` 即 `mysql_mysql`
  （兩者都在 `my_swarm_network` overlay 網路上，故用服務名即可互通，不需 IP）

## upload_taiwan_stock_margin_purchase_short_sale_to_mysql
	DOCKER_IMAGE_VERSION=0.0.9 docker stack deploy --with-registry-auth -c docker-compose-upload_taiwan_stock_margin_purchase_short_sale.yml upload

## 啟動 upload_taiwan_stock_margin_purchase_short_sale (docker-compose 版本，非 swarm)
	DOCKER_IMAGE_VERSION=0.0.9 docker compose -f docker-compose-upload_taiwan_stock_margin_purchase_short_sale-local.yml up -d

## 關閉 upload_taiwan_stock_margin_purchase_short_sale (docker-compose 版本)
	docker compose -f docker-compose-upload_taiwan_stock_margin_purchase_short_sale-local.yml down
