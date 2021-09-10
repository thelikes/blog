# Terraform a VULTR box

tags: devops, terraform, vultr, vps

date: mon sep 28, 2020

## resources
- https://github.com/vultr/terraform-provider-vultr
- https://registry.terraform.io/providers/vultr/vultr/latest/docs/resources/server
- https://www.vultr.com/api/

## How to

Assuming you have terraform installed, first you'll need to install the provider:

```
mkdir -p $GOPATH/src/github.com/vultr; cd $GOPATH/src/github.com/vultr
git clone git@github.com:vultr/terraform-provider-vultr.git
cd $GOPATH/src/github.com/vultr/terraform-provider-vultr
make build
make install
```

To fill out the terraform resource, these may come in handy:

- List OSs
    ```
    curl -s "https://api.vultr.com/v1/os/list" | jq
    ```
- List configured SSH key IDs
    ```
    curl -s -H 'API-Key: xxx' https://api.vultr.com/v1/sshkey/list | jq
    ```
- List region IDs
    ```
    curl -s  https://api.vultr.com/v1/regions/list | jq
    ```
- List plans
    ```
    curl -s  https://api.vultr.com/v1/plans/list | jq
    ```

Finally, create your `main.tf`

```
resource "vultr_server" "box" {
    plan_id = "203"
    region_id = "1"
    os_id = "270"
    label = "test"
    tag = "test-tag"
    hostname = "test"
    enable_ipv6 = true
    auto_backup = false
    ssh_key_ids = var.vultr_ssh_key_id # provided in secrets.tf
}

# print out the IP
output "Box_PublicIP" {
    value = vultr_server.box[*].main_ip
}
```

`secrets.tf`:

```
provider "vultr" {
  api_key = "xxx"
  rate_limit = 700
  retry_limit = 3
}

variable "vultr_ssh_key_id" {
    description = "vultr ssh key id"
    default = [ "xxx" ]
}
```

Hit it:

```
tf apply -auto-approve
```