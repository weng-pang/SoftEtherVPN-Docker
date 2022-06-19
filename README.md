# A [SoftEther VPN][1] Bridge Docker image (WIP)

![](https://github.com/weng-pang/SoftEtherVPN-Docker/workflows/Docker%20Image%20CI/badge.svg)


## Credit 

This image repository is derived directly from siomiz's [Softether VPN Docker ](https://github.com/siomiz/SoftEtherVPN/)

It is a heavily scaled down version - the image covers the bridge function only.

### Quick Background

Siomiz is very kind to provide a solution for implementing the Softether server as a docker image.

That makes me thinking: we now have a vpn server, how about the other components (like vpn bridge)? My use case is to containerise the vpn bridge, so I can deploy my vpn branches a lot more rapidly. It turns out the only thing I have to do is to switch the vpn server to vpn bridge on the dockerfile, without the key creation, and it is ready to go. 

As of the current status, the alpine docker is created. Given time and resources, I will cover all other dockerfiles as well.

## Image Tags
Base OS Image | Available | Latest Stable ([v4.38-9760-rtm](https://github.com/SoftEtherVPN/SoftEtherVPN_Stable/tree/v4.38-9760-rtm)) | Previous Base | [v4.36-9754-beta](https://github.com/SoftEtherVPN/SoftEtherVPN_Stable/tree/v4.36-9754-beta)
------------- | -- | -- | -- | --
`alpine:3.14` | YES | `:alpine`, `:9760-alpine`, `:4.38-alpine` | `alpine:3.9` | `:9754-alpine`, `:4.36-alpine`
`centos:8` | NO |**`:latest`**, `:centos`, `:9760`, `:4.38`, `:9760-centos`, `:4.38-centos` | `centos:7` | `:9754`, `:4.36`, `:9754-centos`, `4.36-centos`
`debian:10-slim` | NO|  `:debian`, `:9760-debian`, `:4.38-debian` | `debian:10-slim` | `:9754-debian`, `:4.36-debian`
`ubuntu:20.04` | NO | `:ubuntu`, `:9760-ubuntu`, `:4.38-ubuntu` | `ubuntu:18.04` | `:9754-ubuntu`, `:4.36-ubuntu`

## Setup
 - SecureNAT enabled
 - make'd from [the official SoftEther VPN GitHub Stable Edition Repository][2].
 - vpn bridge selected

`docker run -d --cap-add NET_ADMIN -p 5555:5555/tcp wengpang/softethervpn`

Supported published ports: 
- `-p 5555:5555/tcp` for SoftEther VPN (recommended by vendor).

#### Notice ####

If you specify credentials using environment variables (`-e`), they may be revealed via the process list on host (ex. `ps(1)` command) or `docker inspect` command. It is recommended to mount an already-configured SoftEther VPN config file at `/opt/vpn_server.config`, which contains hashed passwords rather than raw ones. The initial setup will be skipped if this file exists at runtime (in entrypoint script). You can obtain this file from a running container using [`docker cp` command](https://docs.docker.com/engine/reference/commandline/cp/).

## Use of Privilege Mode ##

As mentioned by the [Softether documentation](https://www.softether.org/4-docs/1-manual/3._SoftEther_VPN_Server_Manual/3.6_Local_Bridges#3.6.5_Supported_Network_Adapter_Types), the network bridge requires promiscuous mode or the bridge adapter will return an error status. 
Noramlly, on a physical machine, we can turn on the network adaptor the promiscuous mode on. Under docker environment, some preparation must be done:
- The docker privilege mode must be turned on
- The mode is available by giving the `--privilege`flag.
Of course, this is a potential security concern. I am happy to see if there are any possible alternatives, provided that the Softether's operation is very severally limited. (For example, the [official documentation](https://www.softether.org/4-docs/1-manual/3._SoftEther_VPN_Server_Manual/3.6_Local_Bridges#3.6.6_Use_of_network_adapters_not_supporting_Promiscuous_Mode) does say so, but it does not say clearly what sort of limitation it encounters)

## Configurations ##

To make the bridge configurations persistent beyond the container lifecycle (i.e. to make the config survive a restart), mount a complete config file at `/usr/vpnbridge/vpn_bridge.config`. If this file is mounted the initial setup will be skipped.
To obtain a config file template, `docker run` the initial setup with Server & Hub passwords, then `docker cp` out the config file:

    $ docker run --name vpnconf -e SPW=<serverpw> -e HPW=<hubpw> siomiz/softethervpn echo
    $ docker cp vpnconf:/usr/vpnbridge/vpn_bridge.config /path/to/vpn_bridge.config
    $ docker rm vpnconf
    $ docker run ... -v /path/to/vpn_bridge.config:/usr/vpnbridge/vpn_bridge.config siomiz/softethervpn

Refer to [SoftEther VPN Server Administration manual](https://www.softether.org/4-docs/1-manual/3._SoftEther_VPN_Server_Manual/3.3_VPN_Server_Administration) for more information.

## Logging ##

By default SoftEther has a very verbose logging system. For privacy or space constraints, this may not be desirable. The easiest way to solve this create a dummy volume to log to /dev/null. In your docker run you can use the following volume variables to remove logs entirely.
```
-v /dev/null:/usr/vpnbridge/server_log \
-v /dev/null:/usr/vpnbridge/packet_log \
-v /dev/null:/usr/vpnbridge/security_log
```
## Server & Hub Management Commands ##

Management commands can be executed just before the server & hub admin passwords are set via:
- `-e VPNCMD_SERVER`: `;`-separated [Server management commands](https://www.softether.org/4-docs/1-manual/6._Command_Line_Management_Utility_Manual/6.3_VPN_Server_%2F%2F_VPN_Bridge_Management_Command_Reference_(For_Entire_Server)).
- `-e VPNCMD_HUB`: `;`-separated [Hub management commands](https://www.softether.org/4-docs/1-manual/6._Command_Line_Management_Utility_Manual/6.4_VPN_Server_%2F%2F_VPN_Bridge_Management_Command_Reference_(For_Virtual_Hub)) (currently only for `DEFAULT` hub).

Example: Set MTU via [`NatSet`](https://www.softether.org/4-docs/1-manual/6._Command_Line_Management_Utility_Manual/6.4_VPN_Server_%2F%2F_VPN_Bridge_Management_Command_Reference_(For_Virtual_Hub)#6.4.97_.22NatSet.22:_Change_Virtual_NAT_Function_Setting_of_SecureNAT_Function) Hub management command:
`-e VPNCMD_HUB='NatSet /MTU:1500'`

Note that commands run only if the config file is not mounted. Some commands (like `ServerPasswordSet`) will cause problems.

## License ##

[MIT License][4].

  [1]: https://www.softether.org/
  [2]: https://github.com/SoftEtherVPN/SoftEtherVPN_Stable
  [3]: https://docs.docker.com/engine/reference/commandline/run/#set-environment-variables-e-env-env-file
  [4]: https://github.com/siomiz/SoftEtherVPN/raw/master/LICENSE
