# Running Docker Compose with ECS 

## 1. Setup:

### Mac: 
- Install Docker Desktop
- If you are using MacBook with Apple Silicon (M1), the image building process can be more complex as it requires using `docker buildx build --platform linux/amd64,linux/arm64` 

### Linux
0. Create a Cloud 9 IDE instance with `m5.large` instance type
1. Install Docker-CE and jq 

```
$ sudo yum install docker jq -y
## Post Docker Installation
$ sudo groupadd docker
$ sudo usermod -aG docker $USER
$ ln -s /path/to/existing/docker /directory/in/PATH/com.docker.cli
```
2. Install Compose Cloud Intergration 

Installation:
```
pushd /tmp
curl -L https://raw.githubusercontent.com/docker/compose-cli/main/scripts/install/install_linux.sh | sh
curl -LO https://github.com/docker/compose-cli/releases/download/v1.0.17/docker-linux-amd64
chmod +x docker-linux-amd64
sudo mv docker-linux-amd64 /usr/local/bin/docker
popd
```

Verify it works:
```
admin:/tmp $ which docker
/usr/local/bin/docker

admin:/tmp $ docker version
Client:
 Cloud integration: 1.0.17
 Version:           20.10.4
 API version:       1.41
 Go version:        go1.15.8
 Git commit:        d3cb89e
 Built:             Mon Mar 29 18:54:36 2021
 OS/Arch:           linux/amd64
 Context:           default
 Experimental:      true
...
```
3. Install docker-compose

```
pushd /tmp
curl -LO https://github.com/docker/compose-cli/releases/download/v2.0.0-beta.6/docker-compose-linux-amd64
chmod +x docker-compose-linux-amd64
mkdir -p ~/.docker/cli-plugins/
mv ./docker-compose-linux-amd64 ~/.docker/cli-plugins/docker-compose
popd
```

4. Run `aws configure`

```
admin:~/environment/ $ aws configure
AWS Access Key ID [None]: 
AWS Secret Access Key [None]: 
Default region name [None]: us-east-1
```

5. Create ECS context 

```
admin:~/environment/wordpress $ docker context create ecs myecs
? Create a Docker context using: An existing AWS profile
? Select AWS Profile default
Successfully created ecs context "myecs"

admin:~/environment/wordpress $ docker context ls 
NAME                TYPE                DESCRIPTION                               DOCKER ENDPOINT               KUBERNETES ENDPOINT   ORCHESTRATOR
default *           moby                Current DOCKER_HOST based configuration   unix:///var/run/docker.sock                         swarm
myecs               ecs                                                                                                   
```

## 2. Let's Start with Running WordPress 

- Create a `wordpress` directory by running `mkdir wordpress; cd wordpress`
- Get the Docker-Compose file for WordPress

```
echo << EOF > docker-compose.yaml
version: "3.9"
    
services:
  db:
    image: mysql:5.7
    volumes:
      - db_data:/var/lib/mysql
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: somewordpress
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress
      MYSQL_PASSWORD: wordpress
    
  wordpress:
    depends_on:
      - db
    image: wordpress:latest
    volumes:
      - wordpress_data:/var/www/html
    ports:
      - "8000:80"
    restart: always
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: wordpress
      WORDPRESS_DB_NAME: wordpress
volumes:
  db_data: {}
  wordpress_data: {}

EOF
```

- Modify the docker-compose.yaml to use port 8080. This step is because Cloud9 support port 8080, 8081, 8082 only. 

```
...
  wordpress:
    depends_on:
      - db
    image: wordpress:latest
    volumes:
      - wordpress_data:/var/www/html
    ports:
      - "8080:80"
...
```

- Run `docker compose up` 

```
➜  wordpress git:(master) ✗ docker compose up -d    
[+] Running 2/2
 ⠿ Container wordpress_db_1         Running                                                                                                                                                                                                                     0.0s
 ⠿ Container wordpress_wordpress_1  Started  
```

- The WordPress and MySQL Containersare running. View the logs: 
```
➜  wordpress git:(master) ✗ docker compose logs -f
...
wordpress_1            | 172.27.0.1 - - [21/Jul/2021:09:00:52 +0000] "GET /favicon.ico HTTP/1.1" 302 404 "http://localhost:8080/wp-admin/install.php" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.164 Safari/537.36"
```

- Let's change back the port mapping to the same port number so that it is supported for ECS Docker Compose Integration. 
```
  wordpress:
    depends_on:
      - db
    image: wordpress:latest
    volumes:
      - wordpress_data:/var/www/html
    ports:
      - "80:80"
```

- Let's run the WordPress on ECS now! 
```
➜  docker -c myecscontext compose up
WARNING services.platform: unsupported attribute     
WARNING services.restart: unsupported attribute      
WARNING services.restart: unsupported attribute      
[+] Running 8/19
[+] Running 8/19                                        CreateInProgress User Initiated                                        13.9s
 ⠏ wordpress-blog                                        CreateInProgress User Initiated                                        14.0s
 ⠿ WordpressdataFilesystem                               CreateComplete                                                          6.0s
 ⠏ WordpressTaskExecutionRole                            CreateInProgress Resource creation Init...                             10.0s
 ⠿ LogGroup                                              CreateComplete                                                          2.0s
 ⠏ CloudMap                                              CreateInProgress Resource creation Initiated                           10.0s
 ⠿ DbdataFilesystem                                      CreateComplete                                                          7.0s
 ⠿ DefaultNetwork                                        CreateComplete                                                          5.0s
 ⠿ Cluster                                               CreateComplete                                                          5.0s
 ⠏ DbTaskExecutionRole                                   CreateInProgress Resource creation Initiated                           10.0s
 ⠿ WordpressTCP80TargetGroup                             CreateComplete                                                          1.0s
 ⠿ DefaultNetworkIngress                                 CreateComplete                                                          1.0s
 ⠋ LoadBalancer                                          CreateInProgress Resource creation Initiated                            3.0s
 ⠿ Default80Ingress                                      CreateComplete                                                          1.0s
 ⠋ WordpressdataNFSMountTargetOnSubnet0ec626afda779eabc  CreateInProgr...                                                        3.0s
 ⠋ WordpressdataAccessPoint                              CreateInProgress Resource creation Initia...                            3.0s
 ⠏ WordpressdataNFSMountTargetOnSubnet08b349ec5f3a7870a  CreateInProgr...                                                        2.0s
 ⠏ DbdataNFSMountTargetOnSubnet08b349ec5f3a7870a         CreateInProgress                                                        2.0s
 ⠏ DbdataNFSMountTargetOnSubnet0ec626afda779eabc         CreateInProgress                                                        2.0s
 ⠋ DbdataAccessPoint                                     CreateInProgress     
 ```
You will see that docker compose cloud intergration is creating AWS resources including ECS Cluster, ECS Task Definitions, Load Balancer, CloudMap Resources, etc. 

Now you can visit the WordPress on AWS by visiting the URL `wordp-LoadB-ZAVSOUPG6KPN-2065488463.us-west-2.elb.amazonaws.com:` shown below.  

```
➜  wordpress git:(master) ✗ docker -c myecscontext compose ps                    
NAME                                              SERVICE             STATUS              PORTS
task/wordpress/477f7a92104b49c799fad6494cc1105d   db                  Running             
task/wordpress/83e0c0e6cf654f918ab8000a202d4369   wordpress           Running             wordp-LoadB-ZAVSOUPG6KPN-2065488463.us-west-2.elb.amazonaws.com:80->80/http
```

## 3. How does the stack look like? 

ECS integration relies on CloudFormation to manage AWS resrouces as an atomic operation. 

- Each compose application service is mapped to an ECS Service
- Service’s ports get mapped into security group’s IngressRules and load balancer Listeners. 
- Compose application with HTTP services only (using ports 80/443 or x-aws-protocol set to http) get an Application Load Balancer created, otherwise a Network Load Balancer is used.
- Fargate first, EC2 when you request GPU support for a container.
- Docker volumes are mapped to EFS file systems. Data won’t be deleted on application shut-down.
- Secrets defined will be stored in AWS Secret Manager and can be used by containers. We will demo this later. 

[1] https://docs.docker.com/cloud/ecs-architecture/
[2] https://aws.amazon.com/blogs/containers/deploy-applications-on-amazon-ecs-using-docker-compose/

If you would like to further modify the stack, you can take a preview by running `docker -c myecscontext compose convert | tee cloudformation.yaml` to see and save the CloudFormation template for creating the stack. 

Once completed, you can turn the stack down.
```
docker -c myecscontext compose down --volume
```


```
➜  wordpress git:(master) ✗ docker -c myecscontext compose convert | tee cloudformation.yaml
....
AWSTemplateFormatVersion: 2010-09-09
Resources:
  CloudMap:
    Properties:
      Description: Service Map for Docker Compose project wordpress
      Name: wordpress.local
      Vpc: vpc-f919379f
    Type: AWS::ServiceDiscovery::PrivateDnsNamespace
...
```



## 4. Using AWS Secret Manager for Database Password

We are going to take advanage of AWS Secert Manager to store the password of MySQL database. You can also use Amazon RDS and store the password using AWS Secert Manager as well.

We are going to store the password in ./secrets/mysecret.txt 
```
➜  mkdir ~/environment/wordpress-aws/; cd ~/environment/wordpress-aws/
➜  vi ./secrets/mysecret.txt 
[Entering some password like abcd1234]
```
Now create the new docker compose file:
```
➜  cat << EOF > docker-compose.yaml 
version: "3.9"
    
services:
  db:
    image: mysql:5.7
    volumes:
      - db_data:/var/lib/mysql
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: somewordpress
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress
      MYSQL_PASSWORD_FILE: /run/secrets/mysql_password
    secrets:
      - mysql_password

  wordpress:
    depends_on:
      - db
    image: wordpress:latest
    volumes:
      - wordpress_data:/var/www/html
    ports:
      - "80:80"
    #   - "8080:80"
    restart: always
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_NAME: wordpress
      WORDPRESS_DB_PASSWORD_FILE: /run/secrets/wordpress_db_password

    secrets:
      - source: mysql_password
        target : wordpress_db_password
        mode: 0400

volumes:
  db_data: {}
  wordpress_data: {}

secrets:
  mysql_password:
    file: ./secrets/mysecret.txt 
EOF
```

In the docker compose file:
- The password will be stored in AWS Secret Manager.
- The password will be obtained by the `mysql` and `wordpress` container. 
- The related IAM roles to access AWS Secret Manager will be created. 

Now let's run it on ECS! 

```
➜  docker -c myecscontext compose up
➜  wordpress-aws docker -c myecscontext compose up 
WARNING services.restart: unsupported attribute      
WARNING services.restart: unsupported attribute      
WARNING services.secrets.mode: unsupported attribute 
[+] Running 14/27
 ⠦ wordpress-aws                                         CreateInProgress User Initiated                          45.7s
 ⠿ WordpressTCP80TargetGroup                             CreateComplete                                            1.0s
 ⠿ LogGroup                                              CreateComplete                                            2.0s
 ⠿ DbdataFilesystem                                      CreateComplete                                            6.0s
 ⠿ Cluster                                               CreateComplete                                            6.0s
 ⠿ WordpressdataFilesystem                               CreateComplete                                            6.0s
 ⠦ CloudMap                                              CreateInProgress Resource creation Initiate...           41.7s
 ⠿ WordpressawsmysqlpasswordSecret                       CreateComplete                                            2.0s
 ⠿ DefaultNetwork                                        CreateComplete                                            6.0s
 ⠿ WordpressTaskExecutionRole                            CreateComplete                                           18.5s
 ⠿ DbTaskExecutionRole                                   CreateComplete                                           18.5s
 ⠦ LoadBalancer                                          CreateInProgress Resource creation Init...               33.6s
 ⠦ WordpressdataNFSMountTargetOnSubneta20696ea           CreateIn...                                              33.6s
 ⠿ Default80Ingress                                      CreateComplete                                            1.0s
 ⠿ DefaultNetworkIngress                                 CreateComplete                                            1.0s
 ⠿ WordpressdataAccessPoint                              CreateComplete                                            6.0s
 ⠦ WordpressdataNFSMountTargetOnSubnetc1b2809a           CreateIn...                                              33.6s
 ⠦ WordpressdataNFSMountTargetOnSubnet05a5660b192765703  CreateInProgress Resource creation Initiated             33.6s
 ⠦ DbdataNFSMountTargetOnSubnet09714186df2be7df9         Create...                                                33.6s
 ⠿ DbdataAccessPoint                                     CreateComplete                                            6.0s
 ⠦ WordpressdataNFSMountTargetOnSubnet09714186df2be7df9  CreateInProgress Resource creation Initiated             33.6s
 ⠦ DbdataNFSMountTargetOnSubneta20696ea                  CreateInProgres...                                       33.6s
 ⠦ DbdataNFSMountTargetOnSubnet05a5660b192765703         Create...                                                33.6s
 ⠦ DbdataNFSMountTargetOnSubnetc1b2809a                  CreateInProgres...                                       33.6s
 ⠿ WordpressTaskRole                                     CreateComplete                                           19.0s
 ⠦ DbTaskRole                                            CreateInProgress Resource creation Initia...              9.6s
 ⠦ WordpressTaskDefinition                               CreateInProgress                                          2.6s

```

- Please turn it down after complated. 

`docker -c myecs compose down`