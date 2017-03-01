# OpenVPN AMI

AMI that should be used to create virtual machines with OpenVPN installed.

## Synopsis

This script will create an AMI with OpenVPN installed and with all of the
required initialization scripts.

The AMI resulting from this script should be the one used to instantiate a
OpenVPN server.

This AMI can also be used as a NAT instance but extra configurations are
required.
To use this AMI as a NAT instance the "Change Source / Dest. Check" option
of the resulting EC2 instance needs to be disabled.

## Getting Started

There are a couple of things needed for the script to work.

### Prerequisites

Packer and AWS Command Line Interface tools need to be installed on your local
computer.
To build a base image you have to know the id of the latest Debian AMI files
for the region where you wish to build the AMI.

#### Packer

Packer installation instructions can be found
[here](https://www.packer.io/docs/installation.html).

#### AWS Command Line Interface

AWS Command Line Interface installation instructions can be found [here](http://docs.aws.amazon.com/cli/latest/userguide/installing.html)

#### Debian AMI's

A list of all the Debian AMI id's can be found at the Debian official page:
[Debian oficial Amazon EC2 Images](https://wiki.debian.org/Cloud/AmazonEC2Image/)

### Usage

In order to create the AMI using this packer template you need to provide a
few options.

```
Usage:
  packer build \
    -var 'aws_access_key=AWS_ACCESS_KEY' \
    -var 'aws_secret_key=<AWS_SECRET_KEY>' \
    -var 'aws_region=<AWS_REGION>' \
    -var 'aws_base_ami=<BASE_IMAGE>' \
    [-var 'option=value'] \
    openvpn.json
```

#### Script Options

- `aws_access_key` - *[required]* The AWS access key.
- `aws_ami_name` - The AMI name (default value: "openvpn").
- `aws_ami_name_prefix` - Prefix for the AMI name (default value: "").
- `aws_base_ami` - *[required]* The AWS base AMI id to use. See [here](https://wiki.debian.org/Cloud/AmazonEC2Image/) for a list of available options.
- `aws_instance_type` - The instance type to use for the build (default value: "t2.micro").
- `aws_region` - *[required]* The regions were the build will be performed.
- `aws_secret_key` - *[required]* The AWS secret key.
- `easyrsa_req_city` - The City value to be used on the CA certificate (default value: "San Francisco").
- `easyrsa_req_country` - The Country value to be used on the CA certificate (default value: "US".)
- `easyrsa_req_email` - The Email value to be used on the CA certificate (default value: "private").
- `easyrsa_req_org` - The Organization Name value to be used on the CA certificate (default value: "Private Company").
- `easyrsa_req_ou` - The Organizational Unit value to be used on the CA certificate (default value: "IT").
- `easyrsa_req_state` - The State value to be used on the CA certificate (default value: "California").
- `easyrsa_version` - The EasyRSA version to install (default value:  "3.0.1").
- `system_locale` - Locale for the system (default value: "en_US").

### Instantiate a Server

In order to end up with a functional OpenVPN service some configurations have
to be performed after instantiating the servers.

To help perform those configurations a small set of scripts is included on the
AWS image.

- `ovpn_initpki` - This script will initialize the CA using the EasyRSA toolset.
- `ovpn_config` - This script will configure the OpenVPN service.
- `ovpn_addclient` - This script will create the required credentials for a client using the EasyRSA toolset.
- `ovpn_delclient` - This script will revoke the credentials for a given client using the EasyRSA toolset. This script will restart the OpenVPN service.
- `ovpn_getclient` - This script will create the configuration file for a given client.
- `ovpn_status` - This script will show the OpenVPN service status. Will include active connections.

#### CA Configuration Script

The OpenVPN service will require clients to authenticate using certificates.
The EasyRSA tools installed on the AMI will allow for the creation of the
client-side certificates.

To be able to manage the certificates a CA needs to be created and configured.
That can be done using the **ovpn_initpki** script.

```
Usage: ovpn_initpki [options]
```

##### Options

* `-c <CN>` - The Common Name to use for the CA certificate.

#### Configuring a CA

To create/initialize a CA that can be used to manage the client authentication
credentials the following steps need to be performed.

Run the configuration tool (*ovpn_initpki*) to create/initialize the CA.

```
ovpn_initpki -c vpn.mydomain.tld
```

After this steps a new CA should be configured and ready to be used to create client credentials.
This also created the required certificates for the OpenVPN service.

#### OpenVPN Configuration Script

The OpenVPN service will require clients to authenticate using certificates.

After taking care of that part (see the [Configuring a CA](#configuring-a-ca)
section for more details), configuring the OpenVPN service can be done using
the **ovpn_config** script.

```
Usage: ovpn_config [options]
```

##### Options

* `-c` - Enables the client-to-client option.
* `-d` - Disables the built in external DNS.
* `-D` - Disables the OpenVPN service from start at boot time.
* `-E` - Enables the OpenVPN service to start at boot time.
* `-g` - Disables the NAT routing and Default Gateway.
* `-n <ADDRESS>` - Sets a Name Server to be pushed to the clients. Several Name Server endpoints can be set by using extra `-n` options.
* `-N` - Configures NAT to access external server network.
* `-p <RULE>` - Sets a rule to be pushed to the clients. Several rules can be set by using extra `-p` options.
* `-r <ROUTE>` - Sets a route to be added on the client side (e.g.: '10.0.0.0/16'). Several routes can be set by using extra `-r` options.
* `-s <CIDR>` - The OpenVPN service subnet (e.g.: '172.16.0.0/12').
* `-S` - Starts the OpenVPN service after performing the required configurations.
* `-W <SECONDS>` - Waits the specified amount of seconds before starting the OpenVPN service (default value is '0').
* `-u <ADDRESS>` - The OpenVPN server public DNS name. Should be in the form of (udp|tcp)://<server_dns_name>:<server_port> .

Most likely the `-u` option will have the value used for the `-c` option on the
*ovpn_initpki* script (see the [Configuring a CA](#configuring-a-ca) section
for more details).

#### Configuring the OpenVPN service

To configure the OpenVPN service the following steps need to be performed.

Run the configuration tool (*ovpn_config*) to configure the OpenVPN service
and have it starting at boot time.

```
ovpn_config -d -E -n 8.8.8.8 -n 8.8.4.4 -p "route 10.0.0.0 255.255.0.0" -s 172.16.0.0/12 -S -u udp://vpn.mydomain.tld:1194
```

After this steps a OpenVPN service should be running and configured to start on
server boot.

More options can be used on the service configuration, see the
[OpenVPN Configuration Script](#openvpn-configuration-script) section for more
details.

#### Adding OpenVPN Users

Creating credentials for the OpenVPN service can be done using the
**ovpn_addclient** script.

```
Usage: ovpn_addclient [options]
```

##### Options

* `-u` - The username for the OpenVPN client.

#### Deleting OpenVPN Users

Removing credentials from the OpenVPN service can be done using the
**ovpn_delclient** script.

```
Usage: ovpn_delclient [options]
```

##### Options

* `-u` - The username for the OpenVPN client.

#### Obtain the OpenVPN Client Configurations

Getting the OpenVPN client configurations for one user can be done using the
**ovpn_getclient** script.

```
Usage: ovpn_getclient [options] > myuser-vpn_mydomain_tld.ovpn
```

The resulting *.ovpn* file should be the one used by the user to configure the
OpenVPN client.

##### Options

* `-u` - The username for the OpenVPN client.

### Maintaining the OpenVPN service

It is possible to add, delete, and/or get a user's credentials to the OpenVPN
service from outside the server (assuming that SSH access is possible).

To perform the maintenance tasks there are a set of scripts on the *scripts*
folder of this recipe.

- `ovpn-add-user.sh` - This script will create a user on the OpenVPN server.
- `ovpn-del-user.sh` - This script will revoke a user's access to the OpenVPN server.
- `ovpn-get-user.sh` - This script will obtain the OpenVPN user configurations.

This scripts will execute the respective *ovpn* script through an SSH
connection.

#### Adding OpenVPN Users

Creating credentials for the OpenVPN service can be done using the
**ovpn-add-user.sh** script.

```
Usage: ovpn-add-user.sh [options]
```

##### Options

* `-i <FILE>` - The SSH authentication key.
* `-p <PORT>` - The SSH server port (default value: 222).
* `-s <ADDRESS>` - The SSH server address.
* `-u <USERNAME>` - The SSH username.
* `-v <USERNAME>` - The OpenVPN username.

#### Deleting OpenVPN Users

Removing credentials from the OpenVPN service can be done using the
**ovpn-del-user.sh** script.

```
Usage: ovpn-del-user.sh [options]
```

##### Options

* `-i <FILE>` - The SSH authentication key.
* `-p <PORT>` - The SSH server port (default value: 222).
* `-s <ADDRESS>` - The SSH server address.
* `-u <USERNAME>` - The SSH username.
* `-v <USERNAME>` - The OpenVPN username.

#### Obtain the OpenVPN Client Configurations

Getting the OpenVPN client configurations for one user can be done using the
**ovpn-get-user.sh** script.

```
Usage: ovpn-get-user.sh [options]
```

The resulting *.ovpn* file should be the one used by the user to configure the
OpenVPN client.

##### Options

* `-i <FILE>` - The SSH authentication key.
* `-p <PORT>` - The SSH server port (default value: 222).
* `-s <ADDRESS>` - The SSH server address.
* `-u <USERNAME>` - The SSH username.
* `-v <USERNAME>` - The OpenVPN username.

## Services

This AMI will have the SSH service running as well as the OpenVPN services.
The following ports will have to be configured on Security Groups.

| Service    | Port      | Protocol |
|------------|:---------:|:--------:|
| SSH        | 22        |    TCP   |
| OpenVPN    | 1194      |    UDP   |

If this image is tu be used as a NAT instance other ports may have to be
added to the Security groups.

## Contributing

1. Fork it!
2. Create your feature branch: `git checkout -b my-new-feature`
3. Commit your changes: `git commit -am 'Add some feature'`
4. Push to the branch: `git push origin my-new-feature`
5. Submit a pull request

## Versioning

This project uses [SemVer](http://semver.org/) for versioning. For the versions
available, see the [tags on this repository](https://github.com/fscm/packer-aws-openvpn/tags).

## Authors

* **Frederico Martins** - [fscm](https://github.com/fscm)

See also the list of [contributors](https://github.com/fscm/packer-aws-openvpn/contributors)
who participated in this project.

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE)
file for details
