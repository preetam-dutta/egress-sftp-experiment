# Kubernetes Egress SFTP experiment

This experiment demonstrates the egress behavior of Kubernetes.

First we bring up a SFTP server and SFTP client as docker containers.
Do docker network inspect to get the IPs allocated to the containers.
Then try to connect to SFTP server from the SFTP client.

Second we bring up K3D i.e. Kubernetes cluster inside docker.
And then bring up the SFTP client as a pod running inside K8s cluster.
And we connect to the SFTP server running outside of the K8s cluster from the pod running inside the K8s cluster.

## Components setup
1. SFTP server container: Running as docker container outside K8s cluster.
2. SFTP client container: To test from outside K8s cluster.
3. SFTP Pod: To test from inside K8s cluster.

**NOTE**: Egress Network Policies not covered.

# SFTP Server Docker Container Test

## Start a sftp-server container in Docker

```
docker run \
    --name sftp-server \
    -p 22:22 \
    -d atmoz/sftp sftpuser:sftppass:::upload
```

## Start sftp-client container

```
docker build -t sftp-client sftp-client/

docker run \
    --name sftp-client \
    -d sftp-client:latest
```

## Docker networking

List docker networks

```
>> docker network ls

NETWORK ID          NAME                DRIVER              SCOPE
1305b89cbcd0        bridge              bridge              local
0799c236080b        host                host                local
1864c380536a        none                null                local
```

Inspect docker network to ensure both containers on same network. 
The containers section should list both the sftp-server and sftp-client containers.

```
>> docker network inspect bridge
[
    {
        "Name": "bridge",
        "Id": "75ae611cf3019ef1bf8c930e2128d1dd7788d2f9b949b37f0a62df337d509e44",
        "Created": "2020-09-05T02:53:44.219239267Z",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.17.0.0/16",
                    "Gateway": "172.17.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "0c82953cbd8e465f15a160444039916e0607559651ed5aa590d7eeb6b00f9908": {
                "Name": "sftp-server",
                "EndpointID": "8d526ce99da8f6c40c8fe77b6fd3bc4981dca47aefc9ca6591140dd8f54b7a3a",
                "MacAddress": "02:42:ac:11:00:03",
                "IPv4Address": "172.17.0.3/16",
                "IPv6Address": ""
            },
            "50ea33f071c33d0fc612fc8f906484317ad2f9b8acaf5177886ef643eb0b5beb": {
                "Name": "sftp-client",
                "EndpointID": "3aa86cb986c9897bb36622a61acf7d9ff1b00cc965b7ec640f864a0bba820221",
                "MacAddress": "02:42:ac:11:00:04",
                "IPv4Address": "172.17.0.4/16",
                "IPv6Address": ""
            },
            "93ef896681c95c5a6937cf0a05c700ab185b9a67bfb7b7b187b5e5e85fc87028": {
                "Name": "k3s-docker",
                "EndpointID": "01ace3eac470df3ee06742630e5a24cafe3b0dc94695becf9f0a774b401f4e8e",
                "MacAddress": "02:42:ac:11:00:02",
                "IPv4Address": "172.17.0.2/16",
                "IPv6Address": ""
            }
        },
        "Options": {
            "com.docker.network.bridge.default_bridge": "true",
            "com.docker.network.bridge.enable_icc": "true",
            "com.docker.network.bridge.enable_ip_masquerade": "true",
            "com.docker.network.bridge.host_binding_ipv4": "0.0.0.0",
            "com.docker.network.bridge.name": "docker0",
            "com.docker.network.driver.mtu": "1500"
        },
        "Labels": {}
    }
]

```


# sftp test

1. Get the sftp-server IP from the above (i.e. 172.17.0.4)
2. docker exec to the sftp-client container and execute the below command

```
root@50ea33f071c3:/# sftp -P 22 sftpuser@172.17.0.4
sftpuser@172.17.0.4's password: 
Connected to 172.17.0.4.
sftp> ls -l
drwxr-xr-x    2 1000     100          4096 Sep  4 16:21 upload
sftp>
``` 
3. Logs of sftp-server ensuring the hit has received

```
docker logs  sftp-server | tail
[/entrypoint] Executing sshd
Server listening on 0.0.0.0 port 22.
Server listening on :: port 22.
Connection closed by 172.17.0.3 port 59294 [preauth]
Connection closed by 172.17.0.3 port 59570 [preauth]
Connection closed by 172.17.0.3 port 59626 [preauth]
Connection closed by 172.17.0.3 port 59844 [preauth]
Connection closed by 172.17.0.3 port 59972 [preauth]
Accepted password for sftpuser from 172.17.0.3 port 32874 ssh2
Received disconnect from 172.17.0.3 port 32874:11: disconnected by user
Disconnected from user sftpuser 172.17.0.3 port 32874
Accepted password for sftpuser from 172.17.0.3 port 33950 ssh2
```

# SFTP Server Kubernetes Test

1. Start K3D cluster

**Note**: Install K3D if required - https://github.com/rancher/k3d

```
>> k3d cluster create demo --servers 3 --agents 3

INFO[0000] Created network 'k3d-demo'                   
INFO[0000] Created volume 'k3d-demo-images'             
INFO[0000] Creating initializing server node            
INFO[0000] Creating node 'k3d-demo-server-0'            
INFO[0009] Creating node 'k3d-demo-server-1'            
INFO[0010] Creating node 'k3d-demo-server-2'            
INFO[0011] Creating node 'k3d-demo-agent-0'             
INFO[0011] Creating node 'k3d-demo-agent-1'             
INFO[0012] Creating node 'k3d-demo-agent-2'             
INFO[0013] Creating LoadBalancer 'k3d-demo-serverlb'    
INFO[0016] Pulling image 'docker.io/rancher/k3d-proxy:v3.0.1' 
INFO[0034] Cluster 'demo' created successfully!         
INFO[0035] You can now use it like this:                
kubectl cluster-info
```

```
>> kubectl cluster-info

Kubernetes master is running at https://0.0.0.0:61192
CoreDNS is running at https://0.0.0.0:61192/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://0.0.0.0:61192/api/v1/namespaces/kube-system/services/https:metrics-server:/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

```
>> docker network ls

NETWORK ID          NAME                DRIVER              SCOPE
75ae611cf301        bridge              bridge              local
0799c236080b        host                host                local
4feacf8c60b1        k3d-demo            bridge              local
1864c380536a        none                null                local
```

2. Connect sftp-server container to the k3d-demo network
```
>> docker network connect k3d-demo sftp-server
```

```
>> [
    {
        "Name": "k3d-demo",
        "Id": "4feacf8c60b1a5f23232dc4bcf42212ac11902fd6879fe504d6df475c770227a",
        "Created": "2020-09-05T03:30:14.521736387Z",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.18.0.0/16",
                    "Gateway": "172.18.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "00b59b50ebccc5a7a596cb745259cd4c97d670a44675e8b13e15d5e65a9bed32": {
                "Name": "k3d-demo-agent-1",
                "EndpointID": "c9d03dc906c9ecd07c8f04037270ad55cdb8d16af7a183437eabae61b49538d3",
                "MacAddress": "02:42:ac:12:00:06",
                "IPv4Address": "172.18.0.6/16",
                "IPv6Address": ""
            },
            "0c82953cbd8e465f15a160444039916e0607559651ed5aa590d7eeb6b00f9908": {
                "Name": "sftp-server",
                "EndpointID": "79226855896bab06edea7b95763930cd183d591edc91dcc4817d828becc505e6",
                "MacAddress": "02:42:ac:12:00:09",
                "IPv4Address": "172.18.0.9/16",
                "IPv6Address": ""
            },
            "1d8f5caefd0fba7ba8bf3ae7f0c7da50d32a291bd5873bdd9a150810a3017a1f": {
                "Name": "k3d-demo-serverlb",
                "EndpointID": "bb1d9567868f45e7ac7ac89ce221576e00f2c6064a064424b01c243c01ec92a4",
                "MacAddress": "02:42:ac:12:00:08",
                "IPv4Address": "172.18.0.8/16",
                "IPv6Address": ""
            },
            "32896f9d592545e96a3004fe0d8a740d3652154f89f53cbce2d8a44045adbde5": {
                "Name": "k3d-demo-server-1",
                "EndpointID": "aa15869882ec5a7059291ee3058e44d1d49ad6e3a7ac2a4b4f906c8b3b8ba60d",
                "MacAddress": "02:42:ac:12:00:03",
                "IPv4Address": "172.18.0.3/16",
                "IPv6Address": ""
            },
            "6616a8657b2d4c0052ba2b142e5c7294526027f755f7dac1cca8d6bae4623733": {
                "Name": "k3d-demo-server-2",
                "EndpointID": "6010d59afbf4480ef2e23189bcf01493131758e2bb4b6c0a1926e28795d4e53f",
                "MacAddress": "02:42:ac:12:00:04",
                "IPv4Address": "172.18.0.4/16",
                "IPv6Address": ""
            },
            "b31559e6bd95b887eca0cb2eed5a4445880b8a8958b79370923910ab9855af53": {
                "Name": "k3d-demo-agent-2",
                "EndpointID": "f68b73330f364bf02f3f702b233df9ed6020b0967d99fa6aeb498f5724c4701a",
                "MacAddress": "02:42:ac:12:00:07",
                "IPv4Address": "172.18.0.7/16",
                "IPv6Address": ""
            },
            "bb66dd5d663af04956f3c52062a6ea715975dc1be4e1f995398fd41ab18d9750": {
                "Name": "k3d-demo-agent-0",
                "EndpointID": "b616c88f7f63a00b6be61c7a929efe4efb690c733cfdff0a598a05683714a200",
                "MacAddress": "02:42:ac:12:00:05",
                "IPv4Address": "172.18.0.5/16",
                "IPv6Address": ""
            },
            "cf24bf63320fa8c3058b455e24791dc74a1d2ee7a62ff456caf49ddd2962d68f": {
                "Name": "k3d-demo-server-0",
                "EndpointID": "feadd823a54d9815cb4329cddb40093f0d7319b17d9e5e4f0c6bd493249ef81c",
                "MacAddress": "02:42:ac:12:00:02",
                "IPv4Address": "172.18.0.2/16",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {
            "app": "k3d"
        }
    }
]
```

2. Create sftp-client pod

```
docker tag sftp-client:latest preetamdutta/sftp-client:latest
docker push preetamdutta/sftp-client:latest
```

3. Create pod
```
kubectl apply -f ./sftp-client/pod.yaml
```

4. Test SFTP from the pod (sftp-client)
```
kubectl exec -it sftp-client bash
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl kubectl exec [POD] -- [COMMAND] instead.

root@sftp-client:/# sftp -P 22 sftpuser@172.18.0.9
The authenticity of host '172.18.0.9 (172.18.0.9)' can't be established.
ED25519 key fingerprint is SHA256:JeCyNZS7AakU3VvyXKSF5y5vCoARa+2MsUlRqd1L/CQ.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '172.18.0.9' (ED25519) to the list of known hosts.
sftpuser@172.18.0.9's password: 
Connected to 172.18.0.9.
sftp> ls 
upload  
sftp> 
```

# References
- https://hub.docker.com/r/atmoz/sftp/
- https://github.com/rancher/k3d