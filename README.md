# Kubernetes Egress SFTP experiment

This experiment demonstrates the egress behavior of Kubernetes.

First we bring up a SFTP server and SFTP client as docker containers.
Do docker network inspect to get the IPs allocated to the containers.
Then try to connect to SFTP server from the SFTP client.

Second we bring up K3D i.e. Kubernetes cluster inside docker.
And then bring up the SFTP client as a pod running inside K8s cluster, along with Network Policy.
And we connect to the SFTP server running outside of the K8s cluster from the pod running inside the K8s cluster.

## Components setup
1. SFTP server container: Running as docker container outside K8s cluster.
2. SFTP client container: To test from outside K8s cluster.
3. SFTP Pod & respectice Network Policy: To test from inside K8s cluster.

# SFTP Server Docker Container Test

## Start a sftp-server container in Docker

```
>> docker run \
    --name sftp-server \
    -p 22:22 \
    -d atmoz/sftp sftpuser:sftppass:::upload
```

## Start sftp-client container

```
>> docker build -t sftp-client sftp-client/
```

```
>> docker run \
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
>> k3d cluster create cluster2 --k3s-server-arg --flannel-backend="vxlan" --k3s-server-arg --disable=traefik --verbose --servers 1 --agents 2
```

Or

```
>> k3d cluster create demo --servers 3 --agents 3
```

```
>> docker network ls

NETWORK ID          NAME                DRIVER              SCOPE
75ae611cf301        bridge              bridge              local
0799c236080b        host                host                local
733027fe0855        k3d-cluster2        bridge              local
4feacf8c60b1        k3d-demo            bridge              local

```

2. Connect sftp-server container to the k3d-demo network
```
>> docker network connect k3d-cluster2 sftp-server
```

```
>> docker network inspect k3d-cluster2 

[
    {
        "Name": "k3d-cluster2",
        "Id": "733027fe08558f7d32b81f1e3a65d5678c3129ccf726b5d7eaa748e09d7edfc3",
        "Created": "2020-09-05T14:11:31.593053913Z",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.19.0.0/16",
                    "Gateway": "172.19.0.1"
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
                "EndpointID": "28870dde17227ae58616b1590fbd015f665ff0f4cd48145603b98aad65531cee",
                "MacAddress": "02:42:ac:13:00:06",
                "IPv4Address": "172.19.0.6/16",
                "IPv6Address": ""
            },
            "5292e1ada10ddfa878faf51900a4c0cdc08a5462ff98303f128a1734c505df43": {
                "Name": "k3d-cluster2-server-0",
                "EndpointID": "1f26feb4fac2e8e72c1c385b4b6b668e22fac5a8da4032857e1c98a51fbc3052",
                "MacAddress": "02:42:ac:13:00:02",
                "IPv4Address": "172.19.0.2/16",
                "IPv6Address": ""
            },
            "9cdf6dd190f986ca3d2812b33b3e9c5c06288ca6547e14bd29e2248afcf187dc": {
                "Name": "k3d-cluster2-serverlb",
                "EndpointID": "9495cec7b22542372a8936275b170cb6a44accf7c7b94a8b79a80d103bc2b8c0",
                "MacAddress": "02:42:ac:13:00:05",
                "IPv4Address": "172.19.0.5/16",
                "IPv6Address": ""
            },
            "afe65bb5ac3ace7d2c36bfb688751c68da30225b0a34cb9c3d23fe6e87a32152": {
                "Name": "k3d-cluster2-agent-0",
                "EndpointID": "168d451a0c17771747ba8c95a16788bc50c0027550fcc0fd7330de75bd049e4b",
                "MacAddress": "02:42:ac:13:00:03",
                "IPv4Address": "172.19.0.3/16",
                "IPv6Address": ""
            },
            "febb58e31c4b38610e2714b0828729e9daacd2d66532df08f76c873d5c9f3dc3": {
                "Name": "k3d-cluster2-agent-1",
                "EndpointID": "0c5c07a5ec1b411040bd6f3348aa7715e5b5e9ea9f37ed05bd45dc1a3a144b59",
                "MacAddress": "02:42:ac:13:00:04",
                "IPv4Address": "172.19.0.4/16",
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

3. Build, Tag and Push sftp-client image

```
>> docker build -t sftp-client:latest ./sftp-client/
```

```
>> docker tag sftp-client:latest preetamdutta/sftp-client:latest
```

```
>> docker push preetamdutta/sftp-client:latest
```

4. Create pod

```
>> kubectl apply -f ./sftp-client/pod.yaml
```

5. Create network policy

```
>> kubectl apply -f ./sftp-client/network-policy.yaml
```

6. Test SFTP from the pod (sftp-client)
```
kubectl exec -it sftp-client bash

root@sftp-client:/# sftp -P 22 sftpuser@172.19.0.6
sftpuser@172.19.0.6's password: 
Connected to 172.19.0.6.
sftp> ls -l
drwxr-xr-x    2 1000     100          4096 Sep  4 16:21 upload
sftp> bye 
```

# References
- https://hub.docker.com/r/atmoz/sftp/
- https://github.com/rancher/k3d