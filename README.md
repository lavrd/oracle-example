# oracle-example
DIMO Connection Oracle example repository

## Key Concepts

Recommended reading: [DIMO Developer docs](https://docs.dimo.org/developer-platform).

### DIMO Oracle

An application that exposes an API and performs data streaming from your data source to a DIMO Node.
The API is used for a frontend to handle onboarding and removal. The data streaming pulls data however you need it to 
from your systems and then forwards it via an http POST to a DIMO Node. This app also does minting of necessary on-chain records. 
This repository is an example in Golang of a DIMO Oracle, which you can base your solution on and just replace necessary parts.

### Connection License

This is an on-chain object that basically represents your Oracle on-chain. It is required to be able to send data to a DIMO
Node and run an Oracle.

### Vehicle NFT

This represents the vehicle on-chain. It stores basic information such as the owner wallet address as well as the Make Model Year.
You can query for them using our Identity-API - [docs](https://docs.dimo.org/developer-platform/api-references/identity-api)

When you onboard a vehicle, the oracle will mint a vehicle NFT as part of the process. The two key inputs are the definition_id 
and the owner 0x address.

### Synthetic Device NFT

This represents the software connection between the Vehicle NFT and the Connection, eg. your oracle.
When the connection is removed, this should be burned. [docs](https://docs.dimo.org/developer-platform/api-references/identity-api/nodes-and-objects/syntheticdevice#definition)
Every payload is signed by the Synthetic Device. We have examples of this in the repository.

### Login with DIMO (LIWD)

An easy way to handle for DIMO users to login with DIMO via an auth redirect flow. You will likely need to implement this
to handle the login for the vehicle onboarding. If you're doing Fleet onboarding, where a single user or few users manage
many vehicles, your UI should reflect that. If you're building for retail/consumer users, you'll want onboarding to be centered
around a single or few vehicles with explicit ownership verification mechanisms, eg. maybe you do an OAuth flow with your system
or require the user to input something specific to that vehicle or PIN code etc. 

# Implementing your Oracle

1. Register an account on https://console.dimo.org/ 
2. Generate an API key and add your preferred redirect URI
3. Create your Connection License. In the future this will be in the dev console but for now provide your ClientID to your developer relations person at DIMO.
4. Obtain the required Synthetic Device Minting roles - engineers at DIMO will do this for you.
5. Create an Account Abstracted wallet with zerodev, we'll call this the Developer AA Wallet - engineers at DIMO can do this for you. Future state will be in Console.
6. Fund your Developer AA Wallet with some DCX. You can do this from the DIMO Dev Console. Required for the minting operations.
7. After you install the Oracle and have it running on a public URL, eg. `https://dimo.mycompany.com`, copy it and add it to the Authorized URL's in the [DIMO Console](https://console.dimo.org)
   Authorized Redirect URIs are under the "License Details" section.

## Onboarding

If you look at the go package `internal/onboarding` you'll see an example we've built for this. 
In general, onboarding in this example codebase happens through a couple API endpoints, and then all the operations are handled
by a backend job (we use river for the jobs). Jobs are stored in the database. 

THe REST endpoints that handle onboarding are in `internal/app/app.go`. The process of onboarding boils down to:
- Decoding the vehicle's VIN or whatever identifier is used. The example code uses DIMO Protocol decoder. If your VINs are uncommon to decode, please reach out.
- Enrolling the vehicle with your backend, whatever way makes sense to you. Basically it tells your external system it's ok for this Oracle app to connect to it. 
- Minting the Vehicle NFT. This creates the on-chain representation of the vehicle, with the owner set to your logged in wallet 0x. 
- Minting the Synthetic Device NFT. This represents the connection of the Vehicle NFT to your software connection. Contains your oracle's connection license 0x. 
- Optionally Sharing & setting Permissions via DIMO SACD. If you know you'll be sharing vehicles you onboard immediately with a customer for example, you can include SACD permissions.

Minting operations above require your Developer AA Wallet address to have DCX balance to pay for the operations. 

### vendor.go file

This implements the onboarding process with your external system. It has a common interface with 2 functions:
- Validate: used to check if the VIN (or identifier) is compatible with your system. You could imagine this calling out to some endpoint in your system that checks.
- Connect: actually onboards the VIN (or identifier) into your system so that it knows to allow connections however you with to implement it - Streaming, grpc, kafka, REST etc.

There is an example struct implementation `ExternalOnboardingService` in this file, but feel free to change it up as needed. 

## Sending data

Data is sent to DIS (DIMO Ingest Server). DIS runs on a DIMO Node, there can be multiple and you can even run your own, but for now we'll assume a 
single fixed node run by DIMO itself.

In this example we read from a kafka stream and then POST to the DIS ingest endpoint. 
The expectation is that payloads come through in the DIMO CloudEvent format. Cloud Event repo
https://github.com/DIMO-Network/dis

Their is a configuration option to disable any data mappings. If you want to just send messages via Kafka and convert them on your end, 
you can do so and just disable `CONVERT_TO_CLOUD_EVENT` by setting it to false.

DIMO Ingest Service (DIS) uses mTLS auth via public private certificates. These are configured via two settings (get them from DIMO):
- `CERT`
- `CERT_KEY`

There is also a `CA_CERT` setting that is the dimo root ca which gets created from `certificateconfigmap.yaml`. It is the same for all oracles and not a secret.

# Installation via HELM charts to Kubernetes cluster

First step is updating your values.yaml in the `/charts/oracle-example/values.yaml`, primarily the `env` and the `ingress` sections.
Do a search for REPLACE_ME to help find values that should be replaced.

Secrets should be created in your preffered setup way, or manually with kubectl eg. 
```shell
kubectl create secret generic <secret-name> \
  --from-literal=<key>=<value> \
  --from-literal=<another-key>=<another-value> \
  -n <namespace>
```
In this example, we have secrets being fed in from AWS Secretsmanager under `templates/secret.yaml` - change or remove this if you do it different.

Consider renaming anything in the chart to match your desired k8s service naming. 
For example in Chart.yaml we have `name: dimo-oracle`, similarly if you do a search for anything `dimo-oracle` you should find what you may want to change. 

In the templates folder, there are some thing you may or may not want - things would consider optional:
- alerts.yaml
- hpa.yaml
- prometheusrule.yaml
- servicemonitor.yaml

### More on settings and secrets

**Secrets you'll get from DIMO through your contact:**
- DEVELOPER_AA_WALLET_ADDRESS
- DEVELOPER_PK
- SD_WALLETS_SEED
- CERT
- CERT_KEY
- DIMO_AUTH_PRIVATE_KEY: this you'll need to get from DIMO [dev console](https://console.dimo.org) -> Developer License -> API Keys -> Generate -> grab the key

**Non secret settings you'll get from DIMO:**
- DIMO_AUTH_CLIENT_ID: from the [dev console](https://console.dimo.org) -> Developer License -> Client ID -> copy the 0x address
- DIMO_AUTH_DOMAIN: whatever your deployment url is eg. dimo-oracle.yourcompany.io
- CONNECTION_TOKEN_ID: token ID of your connection on chain
- INTEGRATION_TOKEN_ID: (legacy, deprecated soon) token ID of your integration on chain

**Secrets you'll need to figure out:**
- DB_HOST: (postgres db)
- DB_USER
- DB_PASSWORD
- DB_NAME: not really a secret, but you'll need to create this in your system
- DB_SSL_MODE: set to `require` by default in values.yaml

- RPC_URL: ethereum compatible RPC. Create an account if you don't have one already, we've had good luck with https://www.alchemy.com & their free tier is likely more than enough for this. Be sure to have Polygon Enabled.
- PAYMASTER_URL: create account in https://dashboard.zerodev.app -> Just login with eg. google if you don't have an account, then Create Project. Enable Polygon mainnet. For some Oracles we may sponsor, discuss with your DIMO contact.
- BUNDLER_URL: also from https://zerodev.app

### Installing once you're ready

Once you have everything ready ready, from the root you can just install with typical helm command:
```shell
helm install <release-name eg. my-dimo-oracle> ./charts/oracle-example -f values.yaml
```

Don't forget to do step number 7 [above](# Implementing your Oracle) to add your Oracle's URL to the Authorized URI's 
section in the [DIMO Console](https://console.dimo.org).




