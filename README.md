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

##openvpn profile template for client

```
client
proto tcp
remote hostname
port 35000
dev tun
nobind
cipher AES-128-CBC
auth SHA256
key-direction 1
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

##RSpec testing

due to the rspec can not run the container with --cap-add=NET_ADMIN or --priviledged, and it echo the container is not running, due to docker finished openvpn-start.sh and closed, if you want to the static analysis for openvpn docker you need to change `exec openvpn $*` to other command e.g. `tail -f /var/log`
