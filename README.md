# EC2 Proxies

Read short introduction here: [Create free HTTPS/SOCKS5 proxy servers using AWS Free Tier EC2 instances automatically on demand within Terraform and simple HTTP API](https://medium.com/@vfreefly/create-free-https-socks5-proxy-servers-using-aws-free-tier-ec2-instances-automatically-on-demand-517ad5c9c0bb).

**Tools:**
* [Terraform](https://www.terraform.io/) to automatically create/install software/destroy EC2 instances
* Proxy server - [Goproxy](https://github.com/snail007/goproxy)
* Ruby [Sinatra gem](http://sinatrarb.com/) for HTTP API to manage proxy instances (optional)
* Ubuntu 16.04 server
* Systemd to convert goproxy process to the system daemon service


## Installation

* Clone the repo
* Install [CLI Terraform](https://www.terraform.io/intro/getting-started/install.html) and run `$ terraform init` inside of project directory.
* For HTTP API (optionally): [install ruby](https://www.ruby-lang.org/en/documentation/installation/) (if you don't have it yet. Minimal supported version is 2.3), install bundler gem `$ gem install bundler` and then run `$ bundle install` inside of project directory.


## Configuration

**1)** Provide your `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` credentials to manage EC2 instances. It is good practice to have separate user roles with restricted permissions for different projects.

[Check here](https://medium.com/@vfreefly/how-to-create-aws-restricted-credentials-example-for-s3-eb360189f383) how to create a new AWS user role and copy credentials. You'll need a user role with `AmazonEC2FullAccess` permission. Then create file `terraform.tfvars` (inside of project directory) and put there `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`, example:

```
AWS_ACCESS_KEY_ID="78J347ZVBPY5R4EPXYGQ"
AWS_SECRET_ACCESS_KEY="WvrNVw38ZJT8pbMV6Vy75RQuLoBdgW6ijtRLMgdt"
```

**2)** Generate SSH key pair for EC2 instances and save it to the .ssh subfolder: `$ ssh-keygen -f .ssh/ec2_key -N ''`


## Settings

All default settings located in the `config.tf` file. If you want to change the value of variable, don't edit `config.tf` file, instead put your configuration to the `terreform.tfvars` file (create this file if it doesn't exists). Use format `VARIABLE_NAME="value"` inside of `terreform.tfvars` file.

You'll probably want to tweak following settings:

* `AWS_INSTANCES_COUNT` - the number of proxy servers to create. Default is 5. You can set it [up to 20](https://aws.amazon.com/ec2/faqs/#How_many_instances_can_I_run_in_Amazon_EC2).
* `AWS_DEFAULT_REGION` - region of instances (proxy servers) where they will be created. Default is `us-east-1`. Check [available regions here](https://docs.aws.amazon.com/general/latest/gr/rande.html#ec2_region). Keep in mind that `AWS_INSTANCE_AMI` should match AWS_DEFAULT_REGION. You can check all Ubuntu [16.04 server AMI's here](https://cloud-images.ubuntu.com/locator/ec2/).
* `PROXY_TYPE` - type of proxy server. Default is `socks` (socks5). If you need HTTP/HTTPS anonymous proxy instead, set variable to `http`.
* `PROXY_PORT` - port of proxy server. Default is `46642`.
* `PROXY_USER` and `PROXY_PASSWORD` - set these variables if you want proxy server use authorization. Defaut is empty (proxy without authorization).

## Usage
### Command line
#### apply

Command `$ terraform apply` will create EC2 instances and make instances setup (install and run goproxy server). From output you'll get IP addresses of created instances. Example:

```
$ terraform apply
...

Apply complete! Resources: 5 added, 0 changed, 0 destroyed.

Outputs:

instances = [
    54.225.911.634,
    31.207.37.49,
    53.235.228.205,
    52.31.233.217,
    35.213.244.142
]
```

Use these IP addresses to connect to proxy servers (proxy type, port and user/password settings were applied from config.tf, see Settings above).

#### output

Command `$ terraform output` will print IP addresses of created instances. Example:

```
$ terraform output

instances = [
    54.225.911.634,
    31.207.37.49,
    53.235.228.205,
    52.31.233.217,
    35.213.244.142
]
```

#### destroy

Command `$ terraform destroy` will destroy all created instances. Example:

```
$ terraform destroy
...

aws_instance.ProxyNode[4]: Destruction complete after 57s
aws_instance.ProxyNode[0]: Destruction complete after 57s
aws_instance.ProxyNode[3]: Destruction complete after 57s
aws_instance.ProxyNode[2]: Destruction complete after 57s
aws_instance.ProxyNode[1]: Destruction complete after 57s
aws_security_group.ec2_proxies_sg: Destroying... (ID: sg-2543a86e)
aws_key_pair.ec2_key: Destroying... (ID: ec2_key)
aws_key_pair.ec2_key: Destruction complete after 2s
aws_security_group.ec2_proxies_sg: Destruction complete after 2s

Destroy complete! Resources: 7 destroyed.
```


### HTTP API

First, start the API server: `$ RACK_ENV=production bundle exec ruby app.rb`. API will be available on `http://localhost:4567`.

#### POST /api/v1/apply

Make post request `/api/v1/apply` to create proxy servers. Configuration in `config.tf` will be used to create instances.

You can pass additional parameters in request body to apply custom settings: `aws_instances_count`, `proxy_type`, `proxy_port`, `proxy_user` and `proxy_password`.

Example:

```
$ curl -X POST http://localhost:4567/api/v1/apply -d 'proxy_type=http&aws_instances_count=3&proxy_user=admin&proxy_password=123456'

{
  "status": "Ok",
  "message": "Successfully performed apply action",
  "data": {
    "instances": [
      "57.94.21.138",
      "28.206.169.144",
      "44.240.61.89"
    ]
  }
}
```

#### GET /api/v1/instances_list

Make get request `/api/v1/instances_list` to get IP addresses of created proxy servers. Example:

```
$ curl -X GET http://localhost:4567/api/v1/instances_list

{
  "status": "Ok",
  "message": "Running 3 instances",
  "data": {
    "instances": [
      "57.94.21.138",
      "28.206.169.144",
      "44.240.61.89"
    ]
  }
}
```

#### POST /api/v1/destroy

Make post request `/api/v1/destroy` to destroy all proxy servers. Example:

```
$ curl -X POST http://localhost:4567/api/v1/destroy

{
  "status": "Ok",
  "message": "Successfully performed destroy action"
}
```