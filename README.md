# FAST-EF

This repository contains the code and setup files for FAST-EF. FAST-EF has been published as a journal paper in [IEEE Access](https://ieeeaccess.ieee.org/) with title ["Federated 3GPP Mobile Edge Computing Systems: A Transparent Proxy for 3rd-Party Authentication with Application Mobility Support (2022)"](https://ieeexplore.ieee.org/document/9743909). 


## Repository structure
	-- openair-epc-fed
	-- openairinterface5g
	-- MEC
		
	-- utils
		
	-- scripts (need to clean and generalise)
		-- installations, network setup, docker setups and run scripts
	-- assets
		-- image-of-implementation-setup.png

## Setup steps

### LTE Network
#### Core network
Clone the core network repository from [saadsakib/openair-epc-fed](https://github.com/saadsakib/openair-epc-fed.git). This repository is forked from [OPENAIRINTERFACE/openair-epc-fed.git](https://github.com/OPENAIRINTERFACE/openair-epc-fed.git) by openairinterface. 

```
git clone https://github.com/saadsakib/openair-epc-fed.git
```

Inside the `OPENAIRINTERFACE/openair-epc-fed` repository, [openair-mme](https://github.com/OPENAIRINTERFACE/openair-mme) and [openair-hss](https://github.com/OPENAIRINTERFACE/openair-hss) repositories are used as submodules. Some changes were also made in these two repositories which can be found in [saadsakib/openair-mme](https://github.com/saadsakib/openair-mme) and [saadsakib/openair-hss](https://github.com/saadsakib/openair-hss.git). The author @Sadman has added roaming protocol in the core network which was not present in the original repositories. The changes can be found in `roaming` branch in all 3 repositories.
	
Try following this documentation provided by openairinterface in order to deploy the core network [https://github.com/saadsakib/openair-epc-fed/blob/roaming/docs/DEPLOY_HOME.md](https://github.com/saadsakib/openair-epc-fed/blob/roaming/docs/DEPLOY_HOME.md)


#### RAN
Clone the RAN repository from [https://gitlab.eurecom.fr/oai/openairinterface5g.git](https://gitlab.eurecom.fr/oai/openairinterface5g.git)
	```git clone https://gitlab.eurecom.fr/oai/openairinterface5g.git```
Try following this documentation provided by openairinterface in order to deploy the RAN: Building the RAN [https://gitlab.eurecom.fr/oai/openairinterface5g/-/blob/master/doc/BUILD.md](https://gitlab.eurecom.fr/oai/openairinterface5g/-/blob/master/doc/BUILD.md) and running the RAN [https://gitlab.eurecom.fr/oai/openairinterface5g/-/blob/master/doc/RUNMODEM.md](https://gitlab.eurecom.fr/oai/openairinterface5g/-/blob/master/doc/RUNMODEM.md)

We have used a script to setup the enb and ue. You can try and use that as well.
	```./open-air-fed/ran_files/```
The conf files we have use for setup can also be found in this folder.
	
### MEC Setup

1. Installing dependencies for MEC dataplane. This can be found at [MEC/mec-dataplane/MEC-setup.md](MEC/mec-dataplane/MEC-setup.md)

2. Installing depepndencies for MEC controlplane. Installing pycrate. You can follow instructions provided in the repository of pycrate [https://github.com/P1sec/pycrate](https://github.com/P1sec/pycrate)

3. Setup network for the experiment. The network needs to be setup according to the number of machines used. We have setup the networks using the following scripts.

## Run

We used 2 PCs. eNB and MEC proxy was run in 1 PC. EPC and freediameter relay nodes were run in another.

The components were run in the following sequence. Where not stated explicitly, the commands were run in FAST-EF repository.

### In eNB PC,

Create iptables rules to send traffic from eNB and MEC proxy to EPC PC

```
sh scripts/iptables91.sh
```

Run MEC proxy

```
npm run debug
```

### In EPC PC,

Create iptables rules to forward traffic to EPC and MEC components

```
sh scripts/iptables92.sh
```

We created two network namespaces that represent MEC. In each, a GTP interface is created and corresponding MEC components are run. (home = ne2 / ns2 / veth2 -> veth3, foreign = ne1 / ns1 / veth0 -> veth1)

```
cd scripts/
sh create_netns_home.sh
sh create_netns_foreign.sh
ne1='ip netns exec ns1'
ne2='ip netns exec ns2'
```

Run EPC components and freediameter relays. To run freediameter relays, you need to have freediameter installed. In openair-epc-fed repository,

```
sh two_epc_run.sh
```

Run tshark in 2 terminals to capture sctp, gtp or icmp packets.

```
tshark -ta -ni any -f sctp
tshark -ta -ni any -Y "gtp or icmp"
```

Run userplane dispatcher home, userplane dispatcher foreign (restart each time with epc)

```
cd MEC/mec-dataplane/mec-dataplane/Dispatcher/
python3 dispatcher.py -n [ home | foreign ]
```

Run vif home + foreign. This creates GTP interfaces with libgtpnl library. Make sure you install necessary requirements first

```
cd MEC/mec-dataplane/mec-dataplane/VirtualInterface/
make tc.out && [$ne2 | $ne1] ./tc.out
```

Create dummy interface inside network namespaces.

```
cd scripts
[ $ne2 | $ne1] ./create_device.sh
http server home, http server foreign
```

Run mec controller in home & foreign namespaces. This requires CryptoMobile and pycrate to be installed.

```
[ $ne2 |  $ne1 ] python3 controller.py [-u] [-s]
```

Run sctp proxy

```
cd MEC/sctp-proxy
make && ./sctp_dispatcher [ home | foreign ]
```

Run cellular oidc idp provider (restart each time with epc)

```
cd MEC/cellular-oidc/with-node
[ $ne2 | $ne1 ] npm run debug:inmec or , debug:[home | foreign]
```

Run MEC manager and AMS in home & foreign namespaces (restart with epc)

```
cd MEC/mec-controller/
[ $ne2 | $ne1 ] npm run debug:[home | foreign]
```

Run app backend home + foreign (restart with epc)

```
cd MEC/mec-app/backend
[ $ne2 | $ne1] npm run debug:[home | foreign]
```

Run app frontend home + foreign

```
cd MEC/mec-app/frontend-vanilla
npm run build
[ $ne2 | $ne1] http-server
```

Run tshark inside network namespaces

```
[$ne2 | $ne1] tshark -ni any
```

### Again in eNB PC,  
Run enb, ue from openairinterface5G repository

```
./enb_run.sh -n (home | foreign)
./ue_run.sh
```

After ue connects,

```
cd scripts
sh create_ue_routes.sh
```
