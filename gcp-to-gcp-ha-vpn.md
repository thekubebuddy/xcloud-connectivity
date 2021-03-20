# Setting-up HA VPN between GCP networks

* This doc. describe setting up the HA VPN between 2 GCP networks.



#### VPCs & Subnets creation 
```
# vpc creation
gcloud compute networks create ha-vpn-vpc-1 --subnet-mode custom
gcloud compute networks create ha-vpn-vpc-2 --subnet-mode custom

# Setting up the GCP region for further resource deployment
export GCP_REGION=us-central1

# subnets creation
gcloud beta compute networks subnets create vpc-01-subnet-1 \
--network ha-vpn-vpc-1 --range 10.1.1.0/24 --region $GCP_REGION

gcloud beta compute networks subnets create vpc-02-subnet-1 \
--network ha-vpn-vpc-2 --range 10.2.1.0/24 --region $GCP_REGION
```

#### Cloud routers creation
```
gcloud compute routers create ha-vpn-vpc-crt-01    --region=$GCP_REGION    --network=ha-vpn-vpc-1    --asn=65001
gcloud compute routers create ha-vpn-vpc-crt-02    --region=$GCP_REGION    --network=ha-vpn-vpc-2    --asn=65002
```

#### HA VPN Gateway creation
```
gcloud compute vpn-gateways create ha-vpn-gw-01    --network=ha-vpn-vpc-1    --region=$GCP_REGION
gcloud compute vpn-gateways create ha-vpn-gw-02    --network=ha-vpn-vpc-2    --region=$GCP_REGION
```

#### VPN Tunnel creations
```

# Generating the random 32byte string for the tunnels shared-secret-key 
export SHARED_SECRET_KEY=`tr -dc A-Za-z0-9 </dev/urandom | head -c 32 ; echo '' `
echo $SHARED_SECRET_KEY

# Tunnels for the vpc-1 for interface 0,1
gcloud beta compute vpn-tunnels create ha-vpn-vpc1-tunnel0 \
    --vpn-gateway ha-vpn-gw-01 \
    --peer-gcp-gateway ha-vpn-gw-02 \
    --shared-secret $SHARED_SECRET_KEY \
    --interface 0
    --region $GCP_REGION \
    --ike-version 2 \
    --router ha-vpn-vpc-crt-01 \

gcloud beta compute vpn-tunnels create ha-vpn-vpc1-tunnel1 \
    --vpn-gateway ha-vpn-gw-01 \
    --peer-gcp-gateway ha-vpn-gw-02 \
    --shared-secret $SHARED_SECRET_KEY \
    --interface 1
    --region $GCP_REGION \
    --ike-version 2 \
    --router ha-vpn-vpc-crt-01 \

# Tunnels for the vpc-2 for interface 0,1
gcloud beta compute vpn-tunnels create ha-vpn-vpc2-tunnel0 \
    --vpn-gateway ha-vpn-gw-02 \
    --peer-gcp-gateway ha-vpn-gw-01 \
    --shared-secret $SHARED_SECRET_KEY \
    --interface 0
    --region $GCP_REGION \
    --ike-version 2 \
    --router ha-vpn-vpc-crt-02 \

gcloud beta compute vpn-tunnels create ha-vpn-vpc2-tunnel1 \
    --vpn-gateway ha-vpn-gw-02 \
    --peer-gcp-gateway ha-vpn-gw-01 \
    --shared-secret $SHARED_SECRET_KEY \
    --interface 1
    --region $GCP_REGION \
    --ike-version 2 \
    --router ha-vpn-vpc-crt-02 \
```


#### BJP sessions configurations

* Configure the bgp session from the VPN tunnels console page, for each respective tunnels
```
"bgp-01" session for the ha-vpn-vpc1-tunnel0 
Cloud Router BGP IP: 169.254.0.1
BGP Peer IP: 169.254.0.2
Peer router ASN: 65002

"bgp-02" session for the ha-vpn-vpc1-tunnel1 
Cloud Router BGP IP: 169.254.1.1
BGP Peer IP: 169.254.1.2
Peer router ASN: 65002

"bgp-03" session for the ha-vpn-vpc2-tunnel0 
Cloud Router BGP IP: 169.254.0.2
BGP Peer IP: 169.254.0.1
Peer router ASN: 65001

"bgp-04" session for the ha-vpn-vpc2-tunnel0 
Cloud Router BGP IP: 169.254.1.2
BGP Peer IP: 169.254.1.1
Peer router ASN: 65001
```

#### Creating ingress firewall rule in VPCs to allow internal communication.
```
gcloud compute firewall-rules create ha-vpn-vpc-1-allow-access-from-ha-vpn-vpc-2-fw \
    --network ha-vpn-vpc-1 \
    --allow tcp,udp,icmp \
    --source-ranges 10.2.1.0/24


gcloud compute firewall-rules create ha-vpn-vpc-2-allow-access-from-ha-vpn-vpc-1-fw \
    --network ha-vpn-vpc-2 \
    --allow tcp,udp,icmp \
    --source-ranges 10.1.1.0/24
```

#### Creating the IAP rule for the 2 VPCs to take the SSH access with IAP
```
gcloud compute firewall-rules create ha-vpn-vpc-1-allow-iap-access-fw \
    --network ha-vpn-vpc-1 \
    --allow tcp:22 \
    --source-ranges 35.235.240.0/20 \
    --target-tags allow-iap-access

gcloud compute firewall-rules create ha-vpn-vpc-2-allow-iap-access-fw \
    --network ha-vpn-vpc-2 \
    --allow tcp:22 \
    --source-ranges 35.235.240.0/20 \
    --target-tags allow-iap-access
```

#### Creating the GCE VMs in the respective VPCs for testing out the connectivity through IAP
```
# GCE VM in VPC1
gcloud compute instances create ha-vpn-vpc-1-instance1 --zone $GCP_REGION-a --subnet vpc-01-subnet-1 --no-address  --labels=owner=user1 --scopes=https://www.googleapis.com/auth/cloud-platform --tags=allow-iap-access

# GCE VM in VPC2
gcloud compute instances create ha-vpn-vpc-2-instance1 --zone $GCP_REGION-a --subnet vpc-02-subnet-1 --no-address --labels=owner=user1 --scopes=https://www.googleapis.com/auth/cloud-platform --tags=allow-iap-access
```

#### Cleanup
```
export CLOUDSDK_CORE_DISABLE_PROMPTS=1
export GCP_REGION=us-central1

gcloud compute instances delete ha-vpn-vpc-1-instance1 --zone $GCP_REGION-a 
gcloud compute instances delete ha-vpn-vpc-2-instance1 --zone $GCP_REGION-a 

gcloud compute firewall-rules delete ha-vpn-vpc-1-allow-iap-access-fw
gcloud compute firewall-rules delete ha-vpn-vpc-2-allow-iap-access-fw
gcloud compute firewall-rules delete ha-vpn-vpc-1-allow-access-from-ha-vpn-vpc-2-fw
gcloud compute firewall-rules delete ha-vpn-vpc-2-allow-access-from-ha-vpn-vpc-1-fw

gcloud beta compute vpn-tunnels delete ha-vpn-vpc1-tunnel0 --region $GCP_REGION
gcloud beta compute vpn-tunnels delete ha-vpn-vpc1-tunnel1 --region $GCP_REGION
gcloud beta compute vpn-tunnels delete ha-vpn-vpc2-tunnel0 --region $GCP_REGION
gcloud beta compute vpn-tunnels delete ha-vpn-vpc2-tunnel1 --region $GCP_REGION

gcloud compute vpn-gateways delete ha-vpn-gw-01 --region=$GCP_REGION
gcloud compute vpn-gateways delete ha-vpn-gw-02 --region=$GCP_REGION

gcloud compute routers delete ha-vpn-vpc-crt-01    --region=$GCP_REGION
gcloud compute routers delete ha-vpn-vpc-crt-02    --region=$GCP_REGION

gcloud beta compute networks subnets delete vpc-01-subnet-1 --region=$GCP_REGION
gcloud beta compute networks subnets delete vpc-02-subnet-1 --region=$GCP_REGION

gcloud beta compute networks delete ha-vpn-vpc-1 
gcloud beta compute networks delete ha-vpn-vpc-1 
```


References:

https://cloud.google.com/sdk/gcloud/reference/config/set

