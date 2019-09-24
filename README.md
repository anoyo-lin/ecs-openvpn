# Overview

This project will create a docker container with openvpn server, specifically to
be run within Amazon ECS.  The credentials files for the openvpn server are
stored within an s3 bucket, and the actual container instance that is run within
AWS is attached to an IAM profile which has read access to this s3 bucket.  This
keeps the secrets out of this repository.

# Variables needed

Set the following shell variables to use the Rakefile:

| Variable     | Example             | Usage                                                 |
|:-------------|:--------------------|:------------------------------------------------------|
| `AWS_ID`     | `123456789012`      | The account number, used by ECS container repository  |
| `AWS_REGION` | `us-east-1`         | The AWS region you're using                           |
| `S3_BUCKET`  | `my-awesome-bucket` | The S3 bucket that the openvpn credentials are stored |
| `S3_DIR`     | `my/awesome/dir`    | The S3 directory that houses the openvpn crecentials  |

# Container size

An attempt was made to keep the container size as small as possible, thus alpine
linux was selected. In order to retrieve the above mentioned openvpn credentials
from s3, an earlier version used amazons python cli to pull the files.  While
this functioned, it did increase the size of the image substantially, so in
order to reduce the size, a go program was written to pull down the credentials,
which is itself a much smaller binary compared to having to download a full
python runtime, the aws cli, plus other dependencies.

# [Serverspec](http://serverspec.org) within container

This shows a way to execute serverspec within a container without having to
install ruby or serverspec.  This will only do a static analysis though, but for
the purposes of this particular container this is fine since, without the
credentials embedded in the image, the container will not fully start up
anyways.

An interesting side effect of how the entrypoint is used along with how
serverspec is being called here, that the entrypoint is still executed, thus a
file is created at zero bytes due to the gets3files binary attempting to
download a file, but failing due to not having sufficient aws credentials.

## AWS command to get temporary credentials for Amazon Container Service

`aws ecr get-login`

and execute the displayed command locally, and it will add the credential to ~/.docker/config.json

## Misc.

if you want to run docker container with tun device, it needs to activate in parameter --cap-add=NET_ADMIN or --priviledged
and the latter one is insecure and not recommended, only for testing purpose

## activating the docker locally

docker run -p 35000:443/tcp -it --entrypoint=sh --cap-add=NET_ADMIN aws/openvpn:tcp
./openvpn-start.sh --cd /etc/openvpn --script-security 2 --config /etc/openvpn/server-tcp.conf
on debug console prompt

## aws ecs fargate configuration

```
{
    "ipcMode": null,
    "executionRoleArn": "arn:aws:iam:: account_id:role/ecsTaskExecutionRole",
    "containerDefinitions": [
        {
            "dnsSearchDomains": null,
            "logConfiguration": {
                "logDriver": "awslogs",
                "secretOptions": null,
                "options": {
                    "awslogs-group": "/ecs/ovpn2",
                    "awslogs-region": "us-east-1",
                    "awslogs-stream-prefix": "ecs"
                }
            },
            "entryPoint": [
                "/etc/openvpn/openvpn-start.sh"
            ],
            "portMappings": [
                {
                    "hostPort": 443,
                    "protocol": "tcp",
                    "containerPort": 443
                },
                {
                    "hostPort": 80,
                    "protocol": "tcp",
                    "containerPort": 80
                }
            ],
            "command": [
                "--cd",
                "/etc/openvpn",
                "--script-security",
                "2",
                "--log",
                "/var/log/openvpn-tcp-server.log",
                "--config",
                "/etc/openvpn/server-tcp.conf"
            ],
            "linuxParameters": {
      "capabilities": {
        "add": ["NET_ADMIN"],
        "drop": []
        }
      },
            "cpu": 0,
            "environment": [],
            "resourceRequirements": null,
            "ulimits": null,
            "dnsServers": null,
            "mountPoints": [],
            "workingDirectory": "/etc/openvpn",
            "secrets": null,
            "dockerSecurityOptions": null,
            "memory": null,
            "memoryReservation": null,
            "volumesFrom": [],
            "stopTimeout": null,
            "image": " account_id.dkr.ecr.us-east-1.amazonaws.com/aws/openvpn:tcp",
            "startTimeout": null,
            "dependsOn": null,
            "disableNetworking": null,
            "interactive": null,
            "healthCheck": null,
            "essential": true,
            "links": null,
            "hostname": null,
            "extraHosts": null,
            "pseudoTerminal": null,
            "user": null,
            "readonlyRootFilesystem": null,
            "dockerLabels": null,
            "systemControls": null,
            "privileged": null,
            "name": "ovpn2"
        }
    ],
    "memory": "512",
    "taskRoleArn": "arn:aws:iam:: account_id:role/ecsTaskExecutionRole",
    "family": "ovpn2",
    "pidMode": null,
    "requiresCompatibilities": [
        "FARGATE"
    ],
    "networkMode": "awsvpc",
    "cpu": "256",
    "inferenceAccelerators": [],
    "proxyConfiguration": null,
    "volumes": [],
    "tags": []
}
```
it need to assign the S3FullAccess policy to ecsTaskExecutionRole.
and aws ecs fargate cannot using --cap-add/--cap-drop, and --priviledge, so we decide using this docker in native build ec2 instance.

## openvpn profile template for client

```
client
proto tcp
remote hostname
port 35000
dev tun
nobind
#it will show the HMAC invalid error message from openvpn server
cipher AES-128-CBC
auth SHA256
key-direction 1
#it needs compression lzo header keep same both in server and client
comp-lzo
<ca>
-----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----
</ca>
<cert>
-----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----
</cert>
<key>
-----BEGIN PRIVATE KEY-----
...
-----END PRIVATE KEY-----
</key>
<tls-auth>
-----BEGIN OpenVPN Static key V1-----
...
-----END OpenVPN Static key V1-----
</tls-auth>
```

## RSpec testing

due to the rspec can not run the container with --cap-add=NET_ADMIN or --priviledged, and it echo the container is not running, due to docker finished openvpn-start.sh and closed, if you want to the static analysis for openvpn docker you need to change `exec openvpn $*` to other command e.g. `tail -f /var/log`

# OpenVPN auth related
* certificates authority ca.crt 
* openvpn server certificates server.crt 
* openvpn server private key server.key 
* credential revocation list crl.pem 
* diffie-hellman 4096bits  dh4096.pem 
	* Generate Diffie-Hellman parameter. This is a set of randomly generated data used when establishing Perfect Forward Secrecy during creation of a client’s session key. The default size is 2048 bits, but OpenVPN’s documentation recommends to use a prime size equivalent to your RSA key size. Since you will be using 4096 bit RSA keys, create a 4096 bit DH prime. Depending on the size of your instance, this could take approximately 10 minutes to complete.
* tls-auth ta.key 
	* Require a matching HMAC signature for all packets involved in the TLS handshake between the server and connecting clients. Packets without this signature are dropped. To generate the HMAC signature file.

```
openvpn --genkey --secret /etc/openvpn/server/ta.key
openssl genpkey -genparam -algorithm DH -out /etc/openvpn/server/dhp4096.pem -pkeyopt dh_paramgen_prime_len:4096
make-cadir ~/ca && cd ~/ca
ln -s openssl-1.0.0.cnf openssl.cnf
# ~/ca/vars
# These are the default values for fields
# which will be placed in the certificate.
# Don't leave any of these fields blank.
export KEY_COUNTRY="CN"
export KEY_PROVINCE="TJ"
export KEY_CITY="Tianjin"
export KEY_ORG="gene's secret garden"
export KEY_EMAIL="mail@domain.com"
export KEY_OU="God Hand"

source ./vars
./clean-all
./build-ca
./build-key-server server
scp ./keys/{ca.crt,server.crt,server.key} root@<your_instance's_IP>:/etc/openvpn/server
scp root@<your_instance's_IP>:/etc/openvpn/server/ta.key ./keys

cd ~/ca && source ./vars && ./build-key client1
```
## Summary
server: ca.crt server.crt server.key from(easy-rsa's CA generated server) 
dh4096.pem (generate by openssl)
ta.key (generated by openvpn-server-side)

client: ca.crt(get from upper one) 
ta.key(get from upper one)
client1.crt client1.key(easy-rsa's CA generated client1)
