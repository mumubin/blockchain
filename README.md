# Docker下手工启动Fabric网络

标签（空格分隔）： 未分类
---

## 前言
  [Fabric v1.0.5安装笔记][1]中的network_setup.sh up是一站式的运行,e
 2e cli例子.具体做了什么,我们这里一步一步分开来看
  
  参考资料:
  
- [end-to-end][2]
- [script.sh][3]

# 网络拓扑
## 生成组织关系和身份证书
1. 设置环境变量

 ```
for power or z
os_arch=$(echo "$(uname -s)-$(uname -m)" | awk '{print tolower($0)}')
$for linux, osx or windows
os_arch=$(echo "$(uname -s)-amd64" | awk '{print tolower($0)}')
 ```
2. 检查环境变量的生成

 ```
echo $os_arch
 ```
 
 3.确保自己在e2e_cli目录下
 ```
    /opt/gopath/src/github.com/hyperledger/fabric/examples/e2e_cli
 ```
 
 4.生成密钥文件
 ```
 ./../../release/$os_arch/bin/cryptogen generate --config=./crypto-config.yaml
 ```
 
 5.查看生成的密钥目录
 ```
  tree -L 4 crypto-config
 ```
 ```
 crypto-config
├── ordererOrganizations
│   └── example.com
│       ├── ca
│       │   ├── 06ae0192afb55ee494bbec1095b9a73148ab011d19450df5eee540696ffbcd00_sk
│       │   └── ca.example.com-cert.pem
│       ├── msp
│       │   ├── admincerts
│       │   ├── cacerts
│       │   └── tlscacerts
│       ├── orderers
│       │   └── orderer.example.com
│       ├── tlsca
│       │   ├── 1056171c60cfd8e918b86404bf3976726ce69640dc7d9b81cbfad1d9a6b7b282_sk
│       │   └── tlsca.example.com-cert.pem
│       └── users
│           └── Admin@example.com
└── peerOrganizations
    ├── org1.example.com
    │   ├── ca
    │   │   ├── ca.org1.example.com-cert.pem
    │   │   └── e55f885168a48ea087362c472bb285c28531c8df58b8ac92f1d76c9e8c8aba9e_sk
    │   ├── msp
    │   │   ├── admincerts
    │   │   ├── cacerts
    │   │   └── tlscacerts
    │   ├── peers
    │   │   ├── peer0.org1.example.com
    │   │   └── peer1.org1.example.com
    │   ├── tlsca
    │   │   ├── d6eb52db2db3854a1feccfe66494b2b0e89aac1bdd9b68d35ce286873ff85cb0_sk
    │   │   └── tlsca.org1.example.com-cert.pem
    │   └── users
    │       ├── Admin@org1.example.com
    │       └── User1@org1.example.com
    └── org2.example.com
        ├── ca
        │   ├── 5cfdefb1d98601b287ea75805ba679de94108e61d1e595df88bfd2c5ff332542_sk
        │   └── ca.org2.example.com-cert.pem
        ├── msp
        │   ├── admincerts
        │   ├── cacerts
        │   └── tlscacerts
        ├── peers
        │   ├── peer0.org2.example.com
        │   └── peer1.org2.example.com
        ├── tlsca
        │   ├── e423b8beda388a330c78d4bea3248b464133daaa1fb32abdc71c975fc6c3a7a7_sk
        │   └── tlsca.org2.example.com-cert.pem
        └── users
            ├── Admin@org2.example.com
            └── User1@org2.example.com
 ```

## 生成Ordering服务启动genesis区块
1. 设置环境变量,告诉configtxgen那去找配置文件configtx.yaml

 ```
FABRIC_CFG_PATH=$PWD
 ```

2. 创建orderer的创世区块
 ```
 ./../../release/$os_arch/bin/configtxgen -profile TwoOrgsOrdererGenesis -outputBlock ./channel-artifacts/genesis.block
 ```
3. 设置通道名称mumubin
 ```
  CHANNEL_NAME=mumubin
  
 ```
4. 新建应用通道
 ```
./../../release/$os_arch/bin/configtxgen -profile TwoOrgsChannel -outputCreateChannelTx ./channel-artifacts/channel.tx -channelID ${CHANNEL_NAME}

 ```
 
## 生成锚节点配置更新文件
1. 生成Org1锚节点配置更新文件
 ```
  ./../../release/$os_arch/bin/configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/Org1MSPanchors.tx -channelID ${CHANNEL_NAME} -asOrg Org1MSP
  
 ```
2. 生成Org2锚节点配置更新文件
 ```
 ./../../release/$os_arch/bin/configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/Org2MSPanchors.tx -channelID ${CHANNEL_NAME} -asOrg Org2MSP
 
 ```
至此,网络拓扑就生成完毕了.
 ```
 ll channel-artifacts/
 ```
 ```
 -rw-r--r-- 1 root root  390 1月  18 09:02 channel.tx
 -rw-r--r-- 1 root root 9085 1月  18 08:59 genesis.block
 -rw-r--r-- 1 root root  280 1月  18 09:04 Org1MSPanchors.tx
 -rw-r--r-- 1 root root  280 1月  18 09:05 Org2MSPanchors.tx
 ```

# 启动网络

##准备工作
 1. 清理启动的线程
 ```
 docker rm -f $(docker ps -aq)
 
 ```
 2. 查验(结果应该为空)
 ```
    docker ps
    
 ```
 3. 查看images
 
 ```
    docker images
    
  dev-peer0.org1.example.com-mycc-1.0-384f11f484b9302df90b453200cfb25174305fce8f53f4e94d45ee3b6cab0ce9   latest                          647439adf7f6        14 hours ago        145 MB
hyperledger/fabric-tools                                                                               latest                          3275ebd1bb71        2 days ago          1.328 GB
hyperledger/fabric-tools                                                                               x86_64-1.0.6-snapshot-78e18d1   3275ebd1bb71        2 days ago          1.328 GB
hyperledger/fabric-orderer                                                                             latest                          6b311f088ccb        2 days ago          151.3 MB
hyperledger/fabric-orderer                                                                             x86_64-1.0.6-snapshot-78e18d1   6b311f088ccb        2 days ago          151.3 MB
hyperledger/fabric-peer                                                                                latest                          725c3f9ca713        2 days ago          154.3 MB
hyperledger/fabric-peer                                                                                x86_64-1.0.6-snapshot-78e18d1   725c3f9ca713        2 days ago          154.3 MB
hyperledger/fabric-ccenv                                                                               latest                          b2b067a6c6d9        2 days ago          1.282 GB
hyperledger/fabric-ccenv                                                                               x86_64-1.0.6-snapshot-78e18d1   b2b067a6c6d9        2 days ago          1.282 GB
hyperledger/fabric-kafka                                                                               latest                          b8c5172bb83c        6 weeks ago         1.286 GB
docker.io/hyperledger/fabric-kafka                                                                     x86_64-1.0.5                    b8c5172bb83c        6 weeks ago         1.286 GB
docker.io/hyperledger/fabric-zookeeper                                                                 x86_64-1.0.5                    68945f4613fc        6 weeks ago         1.316 GB
hyperledger/fabric-zookeeper                                                                           latest                          68945f4613fc        6 weeks ago         1.316 GB
docker.io/hyperledger/fabric-baseimage                                                                 x86_64-0.3.2                    c92d9fdee998        4 months ago        1.257 GB
hyperledger/fabric-baseimage                                                                           latest                          c92d9fdee998        4 months ago        1.257 GB
docker.io/hyperledger/fabric-baseos                                                                    x86_64-0.3.2                    bbcbb9da2d83        4 months ago        128.8 MB
hyperledger/fabric-baseos                                                                              latest                          bbcbb9da2d83        4 months ago        128.8 MB
 ```
 4. 删除直接生成的无用的images(带mycc字段)
 
 ```
    docker rmi -f 647439adf7f6
 ```
 5. 修改掉docker-compose-cli.yaml,防止其自动跑所有流程


 ```
   git diff docker-compose-cli.yaml
   
   diff --git a/examples/e2e_cli/docker-compose-cli.yaml b/examples/e2e_cli/docker-compose-cli.yaml
index e6290cf..27b92f9 100644
--- a/examples/e2e_cli/docker-compose-cli.yaml
+++ b/examples/e2e_cli/docker-compose-cli.yaml
@@ -54,7 +54,7 @@ services:
       - CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
       - CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
     working_dir: /opt/gopath/src/github.com/hyperledger/fabric/peer
-    command: /bin/bash -c './scripts/script.sh ${CHANNEL_NAME}; sleep $TIMEOUT'
+    #command: /bin/bash -c './scripts/script.sh ${CHANNEL_NAME}; sleep $TIMEOUT'
     volumes:
         - /var/run/:/host/var/run/
         - ../chaincode/go/:/opt/gopath/src/github.com/hyperledger/fabric/examples/chaincode/go

 ```
 
## 一步步启动网络
1. 设置环境变量peer0
 ```
 CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
CORE_PEER_ADDRESS=peer0.org1.example.com:7051
CORE_PEER_LOCALMSPID="Org1MSP"
CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt

 ```
2. 启动容器
 ```
 CHANNEL_NAME=mumubin TIMEOUT=1000 docker-compose -f docker-compose-cli.yaml up 
 
 ```
 P.S. 不要加-d参数,这样可以打日志
3. 另起窗口,进入cli容器
 ```
docker exec -it cli bash

 ```

4. 设置环境变量
 ```
 CHANNEL_NAME=mumubin
 ORDERER_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem

 ```
5. 创建channel(这里end-to-end文档有错,参见script脚本)
 ```
 peer channel create -o orderer.example.com:7050 -c $CHANNEL_NAME -f ./channel-artifacts/channel.tx --tls $CORE_PEER_TLS_ENABLED --cafile $ORDERER_CA
 
 ```
6. 加入Channel(目前是Org1的peer0加入的网络)
 ```
peer channel join -b mumubin.block
 
 ```
7. Org1的peer1加入的网络
 ```
CORE_PEER_LOCALMSPID="Org1MSP"
CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
CORE_PEER_ADDRESS=peer1.org1.example.com:7051

 ```
 ```
peer channel join -b mumubin.block
 ```

8. Org2的peer0加入的网络
 ```
CORE_PEER_LOCALMSPID="Org2MSP"
CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt
CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp
CORE_PEER_ADDRESS=peer0.org2.example.com:7051

 ```
 ```
peer channel join -b mumubin.block
 ```

9. Org2的peer0加入的网络
 ```
CORE_PEER_LOCALMSPID="Org2MSP"
CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt
CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp
CORE_PEER_ADDRESS=peer0.org2.example.com:7051
 ```
 ```
peer channel join -b mumubin.block
 ```

10. Org2的peer1加入的网络
 ```
CORE_PEER_LOCALMSPID="Org2MSP"
CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt
CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp
CORE_PEER_ADDRESS=peer1.org2.example.com:7051

 ```
  ```
peer channel join -b mumubin.block
 ```
 
## 交易运行
1. 回归peer0
 ```
  CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
CORE_PEER_ADDRESS=peer0.org1.example.com:7051
CORE_PEER_LOCALMSPID="Org1MSP"
CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt

 ```
2. 安装 install chaincode 
 ```
peer chaincode install -n mycc -v 1.0 -p github.com/hyperledger/fabric/examples/chaincode/go/chaincode_example02

 ```
3. 实例化 instantiate chaincode
 ```
    peer chaincode instantiate -o orderer.example.com:7050 --tls $CORE_PEER_TLS_ENABLED --cafile $ORDERER_CA -C $CHANNEL_NAME -n mycc -v 1.0 -c '{"Args":["init","a","100","b","200"]}' -P "OR	('Org1MSP.member','Org2MSP.member')"
    
 ```
 验证实例化是否成功
 ```
 peer chaincode query -C $CHANNEL_NAME -n mycc -c '{"Args":["query","a"]}'
 ```
 ```
Query Result: 100
 ```

4. 触发交易 Invoke chaincode
 ```
 peer chaincode invoke -o orderer.example.com:7050  --tls $CORE_PEER_TLS_ENABLED --cafile $ORDERER_CA -C $CHANNEL_NAME -n mycc -c '{"Args":["invoke","a","b","10"]}'
 ```
5. 查询交易

 ```
 peer chaincode query -C $CHANNEL_NAME -n mycc -c '{"Args":["query","a"]}'
 ```
 ```
Query Result: 90
 ```

交易顺利执行成功

   
 
  


  [1]: https://segmentfault.com/a/1190000012855320
  [2]: https://github.com/hyperledger/fabric/blob/v1.0.5/examples/e2e_cli/end-to-end.rst
  [3]: https://github.com/hyperledger/fabric/blob/v1.0.5/examples/e2e_cli/scripts/script.sh
