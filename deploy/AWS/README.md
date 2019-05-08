# AWS BOSH Lite and Cloud Foundry

## 1. AWS Setup

- Select VPC menu in AWS Console

- Create VPC through VPC Wizard

  Created in the following format :

  | Field                | Value                              |
  | -------------------- | ---------------------------------- |
  | IP CIDR block        | 10.0.0.0/16                        |
  | VPC name             | bosh                               |
  | Public subnet        | 10.0.0.0/24                        |
  | Availability Zone    | *choose a zone for your region*    |
  | Subnet name          | public<br/> *Set to public subnet* |
  | Enable DNS hostnames | Yes                                |
  | Hardware tenancy     | Default                            |

- Get Elastic IP Assignment

- Creating a key pair in the EC2 Dashboard

- Generate Downloaded Key Pair Permissions

  ```bash
  $ chmod 400 <path to parivate key>
  ```

- Create a security group in EC2 Dashboard

  Created in the following format :

  | Field               | Value                                     |
  | ------------------- | ----------------------------------------- |
  | Security group name | bosh                                      |
  | Description         | BOSH deployed VMs                         |
  | VPC                 | *select the bosh VPC you created earlier* |

- Add **inbound** rules to the security group you created

  Add in the following format : 

  | Type            | Port Range | Source                      |
  | --------------- | ---------- | --------------------------- |
  | SSH             | 22         | *my IP*                     |
  | Custom TCP Rule | 6868       | *my IP*                     |
  | Custom TCP Rule | 25555      | *my IP*                     |
  | All ICMP-IPv4   | All        | *my IP*                     |
  | All Tracffic    | All        | *id of your security group* |

- Add one instance for deployment work



## 2. Deploying the Director

- Download the bosh cli release from the link below

  Release Download link : <https://github.com/cloudfoundry/bosh-cli/releases>

- bosh cli settings

  ```bash
  $ chmod +x ./bosh-*
  $ sudo mv ./bosh-* /usr/local/bin/bosh
  $ bosh -v
  # version 5.4.0-891ff634-2018-11-14T00:22:02Z
  # Succeeded
  ```

  Ubuntu Xenial (16.04) or Ubuntu Trusty (14.04)

  ```bash
  $ sudo apt-get install -y build-essential zlibc zlib1g-dev ruby ruby-dev openssl libxslt-dev libxml2-dev libssl-dev libreadline6 libreadline6-dev libyaml-dev libsqlite3-dev sqlite3
  ```

  Ubuntu Bionic (18.04)

  ```bash
  sudo apt-get install -y build-essential zlibc zlib1g-dev ruby ruby-dev openssl libxslt1-dev libxml2-dev libssl-dev libreadline7 libreadline-dev libyaml-dev libsqlite3-dev sqlite3
  ```

- Get bosh deployment pull from the link below

  Git Pull link :  https://github.com/cloudfoundry/bosh-deployment.git

  ```bash
  $ git clone https://github.com/cloudfoundry/bosh-deployment ~/workspace/bosh-deployment
  ```

- Deploying BOSH Director with the following command

  ```bash
  bosh create-env $path/bosh-deployment/bosh.yml \
          --state=$path"/deployment/state.json" \
          -o $path/bosh-deployment/aws/cpi.yml \
          -o $path/bosh-deployment/bosh-lite.yml \
          -o $path/bosh-deployment/bosh-lite-runc.yml \
          -o $path/bosh-deployment/jumpbox-user.yml \
          -o $path/bosh-deployment/uaa.yml \
          -o $path/bosh-deployment/credhub.yml \
          -o $path/bosh-deployment/external-ip-with-registry-not-recommended.yml \
          --vars-store=$path"/deployment/creds.yml" \
          -v director_name="$DIRECTOR_NAME" \
          -v internal_cidr="$INTERNAL_CIDR" \
          -v internal_gw="$INTERNAL_GW" \
          -v internal_ip="$INTERNAL_IP" \
          -v access_key_id="$AWS_ACCESS_KEY_ID" \
          -v secret_access_key="$AWS_SECRET_ACCESS_KEY" \
          -v region="ap-northeast-2" \
          -v az="ap-northeast-2a" \
          -v default_key_name=$DEFAULT_KEY_NAME \
          -v default_security_groups=[bosh] \
          --var-file private_key=<path/to/private/key> \
          -v subnet_id="$SUBNET_ID" \
          -v external_ip="$EXTERNAL_IP"
  ```

  > **Note:** $EXTERNAL_IP is the Elastic IP assigned in AWS Setup.

  

- Once you have successfully deployed, check with the following command

  ```bash
  export BOSH_ENVIRONMENT=$EXTERNAL_IP
  export BOSH_CA_CERT="$(bosh int $path/deployment/creds.yml --path /director_ssl/ca)"
  export BOSH_CLIENT=admin
  export BOSH_CLIENT_SECRET="$(bosh int $path/deployment/creds.yml --path /admin_password)"
  export BOSH_GW_HOST=$BOSH_ENVIRONMENT
  export BOSH_GW_USER=vcap
  export BOSH_GW_PRIVATE_KEY=<path/to/private/key>
  ```

  If the following statement is displayed, it is successfully distributed

  ```bash
    Name      BOSH Lite Director
    UUID      b1c3a0d6-cd0b-4ff9-9b6a-c80f9c34cf79
    Version   264.7.0 (00000000)
    CPI       warden_cpi
    Features  compiled_package_cache: disabled
              config_server: disabled
              dns: disabled
              snapshots: disabled
    User      admin
  ```

- Log in to BOSH Director after distributing the alias

  ```bash
  $ bosh alias-env $DIRECTOR_NAME -e $EXTERNAL_IP --ca-cert <(bosh int $path/deployment/creds.yml --path /director_ssl/ca)
  ```

- BOSH Alias check.

  ```bash
  $ bosh envs
  # URL             Alias  
  # $EXTERNAL_IP    $DIRECTOR_NAME  
  # 1 environments
  # Succeeded
  ```



## 3. Deploying the CF (Cloud Foundry)

- Get the cf deployment pull from the link below

  Git Pull link : https://github.com/cloudfoundry/cf-deployment

  ```bash
  $ git clone https://github.com/cloudfoundry/cf-deployment ~/workspace/cf-deployment
  ```

- Update Cloud Config

  ```bash
  $ cd ~/workspace/cf-deployment
  $ bosh -e $DIRECTOR_NAME update-cloud-config iaas-support/bosh-lite/cloud-config.yml
  ```

- Upload Stemcell

  ```bash
  $ export IAAS_INFO=warden-boshlite
  $ export STEMCELL_VERSION=$(bosh interpolate cf-deployment.yml --path=/stemcells/alias=default/version)
  $ bosh -e vbox upload-stemcell https://bosh.io/d/stemcells/bosh-${IAAS_INFO}-ubuntu-xenial-go_agent?v=${STEMCELL_VERSION}
  ```

- Upload a Runtime-Config

  ```bash
  $ bosh -e $DIRECTOR_NAME update-runtime-config ~/worspace/bosh-deployment/runtime-configs/dns.yml --name dns
  ```

- Deploy CF

  ```bash
  $ bosh -e $DIRECTOR_NAME -d cf deploy cf-deployment.yml \
     -o operations/bosh-lite.yml \
     -v system_domain=$PUBLIC_DOMAIN
  ```

- Download the credhub cli release from the link below

  Release Download link : <https://github.com/cloudfoundry-incubator/credhub-cli/releases>

- CredHub cli settings

  ```bash
  $ tar -xvf credhub-*.tgz
  $ chmod +x credhub
  $ sudo mv credhub /usr/local/bin
  $ export BOSH_CA_CERT="$(bosh interpolate $path/deployment/creds.yml --path /director_ssl/ca)"
  $ export CREDHUB_SERVER=https://$PUBLIC_DOMAIN:8844
  $ export CREDHUB_CLIENT=credhub-admin
  $ export CREDHUB_SECRET=$(bosh interpolate $path/deployment/creds.yml --path=/credhub_admin_client_secret)
  $ export CREDHUB_CA_CERT="$(bosh interpolate $path/deployment/creds.yml --path=/credhub_tls/ca )"$'\n'"$( bosh interpolate $path/deployment/creds.yml --path=/uaa_ssl/ca)"
  
  $ credhub login -s https://$PUBLIC_DOMAIN:8844 --skip-tls-validation
  ```

- Insatll CF cli

  ```bash
  $ wget -q -O - https://packages.cloudfoundry.org/debian/cli.cloudfoundry.org.key | sudo apt-key add -
  $ echo "deb https://packages.cloudfoundry.org/debian stable main" | sudo tee /etc/apt/sources.list.d/cloudfoundry-cli.list
  $ sudo apt-get update
  $ sudo apt-get install cf-cli
  $ cf -v
  # cf version 6.43.0+815ea2f3d.2019-02-20
  ```

- CF login after normal installation

  ```bash
  $ cf api https://api.$PUBLIC_DOMAIN --skip-ssl-validation
  $ export CF_ADMIN_PASSWORD=$(bosh int <(credhub get -n /bosh-lite/cf/cf_admin_password --output-json) --path /value)
  $ cf auth admin $CF_ADMIN_PASSWORD
  ```

