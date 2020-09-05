# Kubernetes Egress SFTP experiment

This experiment demonstrates the egress behavior of Kubernetes.

First we bring up a SFTP server and SFTP client as docker containers.
Do docker network inspect to get the IPs allocated to the containers.
Then try to connect to SFTP server from the SFTP client.

Second we bring up K3S(or can be done via K3D) i.e. Kubernetes cluster inside docker.
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

1. Start K3S port forwarding

2. Upload sftp-client to K3S Docker repo

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
root@sftp-client:/# sftp -P 22 sftpuser@172.17.0.3
The authenticity of host '172.17.0.3 (172.17.0.3)' can't be established.
ED25519 key fingerprint is SHA256:JeCyNZS7AakU3VvyXKSF5y5vCoARa+2MsUlRqd1L/CQ.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '172.17.0.3' (ED25519) to the list of known hosts.
sftpuser@172.17.0.3's password: 
Connected to 172.17.0.3.
sftp> bye
root@sftp-client:/# exit
exit
```

# References
- https://hub.docker.com/r/atmoz/sftp/