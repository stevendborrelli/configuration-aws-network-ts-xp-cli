# Crossplane AWS Network Configuration in TypeScript <!-- omit from toc -->

This repository contains a TypeScript implementation of [configuration-aws-network](https://github.com/upbound/configuration-aws-network), using Crossplane's [function-sdk-typescript](https://www.npmjs.com/package/@crossplane-org/function-sdk-typescript) and the [Crossplane CLI](https://docs.crossplane.io/latest/cli/) project commands.

- [Installing and Running the Configuration and Function](#installing-and-running-the-configuration-and-function)
  - [Installation of the Package](#installation-of-the-package)
  - [Configuring AWS Authentication](#configuring-aws-authentication)
    - [AWS static credentials](#aws-static-credentials)
  - [Create the ProviderConfig](#create-the-providerconfig)
  - [Create the Example](#create-the-example)
  - [Deleting the Example](#deleting-the-example)
- [Project Structure](#project-structure)
- [Development](#development)
  - [Prerequisites](#prerequisites)
  - [Building the Project](#building-the-project)
  - [Running Locally with Project Run](#running-locally-with-project-run)
  - [Updating the Function](#updating-the-function)
  - [Building TypeScript Manually](#building-typescript-manually)
  - [Running the Function Locally](#running-the-function-locally)
  - [Crossplane Composition Render](#crossplane-composition-render)
  - [Available CLI Options](#available-cli-options)
- [Packaging and Pushing](#packaging-and-pushing)
  - [Building the Project](#building-the-project-1)
  - [Pushing the Project](#pushing-the-project)
- [License](#license)
- [Author](#author)

## Installing and Running the Configuration and Function

### Installation of the Package

The Configuration Package can be installed using a manifest. The package will install the function and AWS providers as dependencies.

```yaml
apiVersion: pkg.crossplane.io/v1
kind: Configuration
metadata:
  name: configuration-aws-network
spec:
  package: xpkg.upbound.io/stevendborrelli/configuration-aws-network-ts:v0.1.0
```

Verify the package is healthy. If not, run `kubectl describe configuration.pkg configuration-aws-network`.

```sh
$ kubectl get configuration.pkg configuration-aws-network
NAME                        INSTALLED   HEALTHY   PACKAGE                                                                  AGE
configuration-aws-network   True        True      xpkg.upbound.io/stevendborrelli/configuration-aws-network-ts:v0.1.0   18m
```

### Configuring AWS Authentication

Before running the example, we will need to configure authentication to the AWS API.

#### AWS static credentials

AWS Static credentials can be useful in testing, but more secure methods like IRSA or WebIdentity should
be used in production, see [AUTHENTICATION.md](https://github.com/crossplane-contrib/provider-upjet-aws/blob/main/AUTHENTICATION.md) for more information.

Create `[default]` credentials config file from AWS that contains the access key, secret access key and
optionally the session token:

```ini
[default]
aws_access_key_id=ASIA.....
aws_secret_access_key=5XgS...
aws_session_token=IQoJb3H...
```

Next, create a kubernetes secret from this file:

```shell
kubectl create secret generic aws-creds -n crossplane-system --from-file=creds=creds.conf
```

### Create the ProviderConfig

The ProviderConfig sets up authentication for the resource. Since we are using a secret, we will use a `source: Secret` in the configuration. The example will create resources in the `network-team` namespace, so the ProviderConfig will be created in the same namespace:

```shell
kubectl create ns network-team
```

```shell
$ cat <<'EOF' | kubectl apply -f -
apiVersion: aws.m.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
  namespace: network-team
spec:
  credentials:
    source: Secret
    secretRef:
      name: aws-creds
      namespace: crossplane-system
      key: creds
EOF
```

### Create the Example

Now apply the example manifest at [examples/networks/configuration-aws-network.yaml](examples/networks/configuration-aws-network.yaml).

```shell
$ kubectl apply -f examples/networks/configuration-aws-network.yaml
network.aws.platform.upbound.io/configuration-aws-network created
```

Watch the progress of the composition using `crossplane beta trace`:

```shell
$ crossplane resource trace -n network-team network.aws.platform.upbound.io/configuration-aws-network                                                      Stevens-MacBook-Pro.
NAME                                                                                 SYNCED   READY   STATUS
Network/configuration-aws-network (network-team)                                     True     True    Available
├─ InternetGateway/configuration-aws-network-697058725aed (network-team)             True     True    Available
├─ MainRouteTableAssociation/configuration-aws-network-eddcd63cb10d (network-team)   True     True    Available
├─ RouteTableAssociation/configuration-aws-network-2967dec5ff24 (network-team)       True     True    Available
├─ RouteTableAssociation/configuration-aws-network-795ad2be7b91 (network-team)       True     True    Available
├─ RouteTableAssociation/configuration-aws-network-8a555e4906cd (network-team)       True     True    Available
├─ RouteTableAssociation/configuration-aws-network-d1cce7ad8b2f (network-team)       True     True    Available
├─ RouteTable/configuration-aws-network-2a2f7791cfef (network-team)                  True     True    Available
├─ Route/configuration-aws-network-051652a2d765 (network-team)                       True     True    Available
├─ SecurityGroupRule/configuration-aws-network-3367ea7efca4 (network-team)           True     True    Available
├─ SecurityGroupRule/configuration-aws-network-b62204d142ef (network-team)           True     True    Available
├─ SecurityGroup/configuration-aws-network-5643bd4c9d37 (network-team)               True     True    Available
├─ Subnet/configuration-aws-network-1f2f87ef2464 (network-team)                      True     True    Available
├─ Subnet/configuration-aws-network-962b2c149555 (network-team)                      True     True    Available
├─ Subnet/configuration-aws-network-c87c3d58bf81 (network-team)                      True     True    Available
├─ Subnet/configuration-aws-network-e1dfea945c75 (network-team)                      True     True    Available
└─ VPC/configuration-aws-network-e28e2e104054 (network-team)                         True     True    Available
```

### Deleting the Example

```shell
kubectl delete -n network-team network.aws.platform.upbound.io/configuration-aws-network
```

## Project Structure

This project uses the Crossplane CLI project structure:

```sh
.
├── apis/                         # Crossplane XRD and Composition files
│   └── network/                  # Network API definitions
├── crossplane-project.yaml       # Project definition file
├── examples/                     # Example manifests
│   └── networks/                 # Network example manifests
├── functions/                    # Embedded functions directory
│   └── network/                  # Network function
│       ├── dist/                 # Compiled JavaScript artifacts
│       ├── src/                  # TypeScript source files
│       │   ├── function.ts       # Function logic
│       │   └── main.ts           # Function runtime setup
│       ├── package.json          # NPM dependencies and scripts
│       └── tsconfig.json         # TypeScript configuration
├── schemas/                      # Crossplane CLI generated schemas
│   └── typescript/               # TypeScript type definitions for CRDs
├── _output/                      # Build output directory
└── README.md
```

## Development

### Prerequisites

- [Crossplane CLI](https://docs.crossplane.io/latest/cli/) with WIP 
- Node.js 24+
- Kubernetes Kind (for `project run`)

### Building the Project

The Crossplane CLI `project build` command builds the entire project including:

- Compiling TypeScript functions
- Generating TypeScript schemas from CRDs
- Building Docker images for functions
- Creating Crossplane packages (.xpkg files)

```shell
crossplane project build
```

The built packages will be output to `_output/configuration-aws-network-ts.xpkg`.

### Running Locally with Project Run

The `project run` command builds and deploys the project to a local development control plane using kind:

```shell
crossplane project run
```

This will:

1. Build the project
2. Create a local kind cluster
3. Install Crossplane
4. Load the built packages into the cluster

To stop the local control plane:

```shell
crossplane project stop
```

### Updating the Function

All the logic of the function is located in [functions/network/src/function.ts](functions/network/src/function.ts). This project uses generated TypeScript types from the `crossplane-models` package for type-safe resource creation.

To create a resource:

1. Create a new type (like a VPC)
2. Run `validate()` against the resource
3. Add the resource to the `desiredComposed` map

Below is an example for the VPC resource:

```typescript
const vpc = new VPC({
  metadata: {
    ...commonMetadata,
  },
  spec: {
    ...commonSpec,
    forProvider: {
      cidrBlock: observedComposite?.resource?.spec?.parameters?.vpcCidrBlock,
      enableDnsHostnames: true,
      enableDnsSupport: true,
      region: region,
      tags: {
        Name: observedComposite?.resource?.metadata?.name,
      },
    },
  },
});

vpc.validate();

desiredComposed['vpc'] = Resource.fromJSON({
  resource: vpc.toJSON(),
});
```

### Building TypeScript Manually

To compile TypeScript manually without building the full project:

```shell
cd functions/network
npm run build
```

### Running the Function Locally

After compiling the source code, the function can be run locally for
testing with `crossplane composition render`:

```bash
cd functions/network
node dist/main.js --insecure --debug
```

Using `npm run`:

```bash
npm run local
```

The function must be shut down before running locally again.

### Crossplane Composition Render

The `crossplane composition render` command allows developers to generate function outputs. The
function can be run as a local process running on port 9443, and `crossplane composition render` can
connect to this port and invoke the function.

After running the function locally in one terminal, run the following to render a manifest:

```sh
crossplane composition render examples/networks/configuration-aws-network.yaml \
  apis/network/composition.yaml \
  examples/functions.yaml
```

### Available CLI Options

The function supports several CLI options:

- `--address` - Address to listen for gRPC connections (default: `0.0.0.0:9443`)
- `-d, --debug` - Enable debug logging
- `--insecure` - Run without mTLS credentials (for local development)
- `--tls-server-certs-dir` - Directory containing mTLS certificates (default: `/tls/server`)

## Packaging and Pushing

### Building the Project

Build the project into Crossplane packages:

```shell
crossplane project build
```

The built package will be at `_output/configuration-aws-network-ts.xpkg`.

You can override the repository during build:

```shell
crossplane project build --repository=xpkg.upbound.io/my-org/my-project
```

### Pushing the Project

Push the built packages to an OCI registry:

```shell
crossplane project push
```

The repository is specified in `crossplane-project.yaml` under `spec.repository`.

## License

Apache-2.0

## Author

Stefano Borrelli <steve@borrelli.org>
