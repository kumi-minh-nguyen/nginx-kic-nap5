# nginx-kic-nap5

### Summary
This repository provides guidance to set up Nginx Plus Ingress Controller with NGINX App Protect WAF v5. 

References:

https://docs.nginx.com/nginx-app-protect-waf/v5/admin-guide/compiler/

https://docs.nginx.com/nginx-ingress-controller/installation/integrations/app-protect-waf-v5/configuration/

https://docs.nginx.com/nginx-ingress-controller/installation/integrations/app-protect-waf-v5/installation/

### Prerequisites
- NGINX Plus trial keys have been downloaded from F5 portal
- NGINX Plus Ingress Controller with Custom Resource Definition has been installed

### Implementation

#### Step 1: Clone the repo to use

`cd nginx-kic-nap5`

#### Step 2: Build Waf Complier
*Make sure you have nginx-repo.crt and nginx-repo.key in the respective folder (/etc/ssl/nginx)*

`sudo docker build --no-cache --secret id=nginx-crt,src=/etc/ssl/nginx/nginx-repo.crt --secret id=nginx-key,src=/etc/ssl/nginx/nginx-repo.key -t waf-compiler:1.0.0 .`

#### Step 3: Compile waf policy and log bundles

`docker run --rm -v $(pwd):$(pwd) waf-compiler:1.0.0 -g $(pwd)/nap5-global-settings.json -p $(pwd)/nap5-custom-policy.json -o $(pwd)/nap5-policy.tgz`

`docker run --rm -v $(pwd):$(pwd) waf-compiler:1.0.0 -l $(pwd)/nap5-custom-log-profile.json -o $(pwd)/nap5-log-profile.tgz`

#### Step 4: Copy the policy and log bundle to your storage folder. 
*For example: /mnt/kic_nap5_bundles_pv_data/*

`cp nap5-custom-policy.tgz /mnt/kic_nap5_bundles_pv_data/`

*Note: the storage folder need to be owned by 101:101 user*

`sudo chown -R 101:101 /mnt/kic_nap5_bundles_pv_data/`

#### Step 5: Create the policy with bundles information

`kubectl apply -f waf-policy.yaml`

`kubectl apply -f rate-limit-policy.yaml`

*Check if policies are created properly*

`kubectl get policies`

#### Step 6: Create persistent volume and claim

`kubectl apply -f storage.yaml`

#### Step 7: Create nginx plus ingress with waf config manager and waf enforcer

`kubectl apply -f nginx-plus-ingress.yaml`

*Check if ingress is created properly*

`kubectl get pods -n nginx-ingress`

#### Step 8: Create load balancer service

`kubectl apply -f service.yaml`

#### Step 9: Create coffee and tea services

`kubectl apply -f cafe-secret.yaml`

`kubectl apply -f cafe-deployment.yaml`

`kubectl apply -f cafe-virtualserver.yaml`

#### Step 10: Test the policies
*SQL Injection*
- Testing with single quote (this should pass)

`curl -I "http://webapp.example.com/?id=1'"`

- Testing with union-based injection (this should fail)

`curl -I "http://webapp.example.com/?id=1 UNION SELECT null, username, password FROM users--"`

- Testing with comment-based injection (this should fail)
  
`curl -I "http://webapp.example.com/?id=1; --"`

*Rate Limit*
- Rate limit is not set to `coffee` service. All will pass.
  
`for i in {1..11}; do curl -k https://cafe.example.com/coffee; done; echo`

- Rate limit is set to `tea` service with 10 requests per seconds. Some requests will be blocked.
  
`for i in {1..11}; do curl -k https://cafe.example.com/tea; done; echo`

