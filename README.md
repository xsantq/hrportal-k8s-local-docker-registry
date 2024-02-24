This project is about to setting up internal private docker registry intented to use for on-premise k8s clusters:

Internal docker registry itself is a docker image runs on docker engine. Pulling image of pod in worker nodes needs a secure 
SSL and TLS connection wit registry. A basic authentication feature also needed(like in docker-hub). Following steps is iimplemented:


Step-1: Setup Self Signed Certificate 
    - Create a conf file like that: 
    
      [req]
      default_bits = 4096
      default_md = sha256
      distinguished_name = req_distinguished_name
      x509_extensions = v3_req
      prompt = no
      [req_distinguished_name]
      C = TR
      ST = VA
      L = ANKARA
      O = Organization
      OU = Department
      CN = 192.168.23.13
      [v3_req]
      keyUsage = keyEncipherment, dataEncipherment
      extendedKeyUsage = serverAuth
      subjectAltName = @alt_names
      [alt_names]
      IP.1 = 192.168.23.13


CN field and alt_names sections are important and must be the IP adress of the docker registry server. save file as san.cfg 

- Generate key with following command. Give the created file(san.cfg) as config parameter with flag -config:

      $mkdir -p docker_reg_certs

      $openssl req  -newkey rsa:4096 -nodes -sha256 -keyout docker_reg_certs/domain.key -x509 -days 365 -out docker_reg_certs/domain.crt -config <path/to/req/file/from/above>

- verify the certficate: 

      $openssl s_client -connect localhost:5000 -showcerts </dev/null

If you see the  verification:OK, you can install the certificates to Docker registry. 

Step-3: User Authentication for Docker Registry:

This is a container iorder to create username and password for docker registry. 
        
        $mkdir docker_reg_auth
        $docker run -it --entrypoint htpasswd -v $PWD/docker_reg_auth:/auth -w /auth registry:2 -Bbc /auth/htpasswd admin password
        $service docker restart

Step-3 Starting the Docker Registry Container: 

        $docker run -d -p 5000:5000 --restart=always --name registry -v $PWD/docker_reg_certs:/certs -v $PWD/docker_reg_auth:/auth -v /reg:/var/lib/registry -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd -e REGISTRY_AUTH=htpasswd registry:2.7.0

Step-4 Test the repository with pull and push:

-First tag your local image:

        $docker tag local_image:latest ip_address:5000/local_image:tag

-Then login to local registry;
      
        $ docker login ip_address:5000
        $ docker push ip_address:5000/local_image:tag
        $ docker pull ip_address:5000/local_image:tag

- Test your repo with curl 

       $ $curl -u admin:password -v -X GET https://ip_address:5000/v2/_catalog


IMPORTANT: RESTART YOUR WORKER NODE AFTER CERTIFICATE INSTALLATION. Old certificate maybe cached in CRI of the node, and this will lead to certificate errors while pod trying to pull image from local registry.

Step-5 Use your private local registry in K8S

- create a secret for local registry authentication in k8s cluster. Use admin password given previously.
  
       $kubectl create secret docker-registry regcred --docker-server=<your-registry-server> --docker-username=<your-name> --docker-password=<your-pword> --docker-email=<your-email>

- create a deployment manifest that uses this secret in container spec. and image section must be indicate  your private local docker and image. 

                apiVersion: apps/v1
                kind: Deployment
                metadata:
                  name: hrportal
                  labels:
                    app: hrportal
                spec:
                  replicas: 1
                  selector:
                    matchLabels:
                      app: hrportal
                  template:
                    metadata:
                      labels:
                        app: hrportal
                    spec:
                      containers:
                      - name: hrportal
                        image: ip_address:5000/hrportal:local
                        ports:
                        - containerPort: 8080
                      imagePullSecrets:
                      - name: privatecred
                
                
Apply it and create deployment. 
       
       $kubectl create -f deployment.yaml

Check your logs to verify that pod will pull the image from your local private registry
       
       $kubectl  describe po <pod_name>

IPMD
