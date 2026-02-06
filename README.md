# AKS_RP_E2E_TEST_DOC

# AKS-RP E2E Local Testing: Complete Setup Guide

This guide provides step-by-step instructions for setting up and running E2E tests locally, including solutions for common issues like authentication and proxy problems.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Clone the Repository](#clone-the-repository)
3. [Setup go proxy](#setup-go-proxy)
4. [Azure Authentication](#azure-authentication)
5. [Building aksdev CLI](#building-aksdev-cli)
6. [Generating Azure Configuration](#generating-azure-configuration)
7. [Running E2E Tests](#running-e2e-tests)
8. [Troubleshooting](#troubleshooting)
9. [Quick Reference](#quick-reference)

---

## Prerequisites
if you are not part of AKS Team then ensure you have the the have the [TM-AKS-ExtContributor](https://coreidentity.microsoft.com/manage/entitlement/entitlement/tmaksextcont-jawg) entitlement
### Required Tools

| Tool | Purpose | Installation |
|------|---------|--------------|
| **Go 1.21+** | Build aksdev | `brew install go` or [golang.org](https://golang.org) |
| **Azure CLI** | Azure authentication | `brew install azure-cli` or `curl -sL https://aka.ms/InstallAzureCLIDeb \| sudo bash` |
| **kubectl** | Kubernetes operations | `brew install kubectl` |
| **Python 3** | Bazel dependencies | `brew install python` |
| **Bazel** | Build system (optional) | `brew install bazel` |
| **Git** | Clone repository | `sudo apt install git` |

### Verify Prerequisites

```bash
# Check Go version (should be 1.21+)
go version

# Check Azure CLI
az version

# Check kubectl
kubectl version --client

# Check Python
python3 --version
```

---

## Clone the Repository

### Step 1: Set Up Directory Structure

```bash
# Create the Go module path structure
mkdir -p ~/acstor/aks-rp/go.goms.io/aks
cd ~/acstor/aks-rp/go.goms.io/aks
```

### Step 2: Clone the Repository

```bash
# Clone via HTTPS
git clone https://msazure.visualstudio.com/CloudNativeCompute/_git/aks-rp rp

# Or clone via SSH
git clone git@ssh.dev.azure.com:v3/msazure/CloudNativeCompute/aks-rp rp

# Navigate to the repo
cd rp
```

## SetUp Go Proxy 

### Generate PAT Token
####How to get the PAT [Your Personal Access Token of Azure Devops of msazure org]:

Go to https://dev.azure.com/msazure/_usersSettings/tokens
Generate a token for "msazure" account under "Code" scope with "Read" access. You can specify an expiration date. Currently the longest expiration allowed is 7 days

<img width="501" height="97" alt="image" src="https://github.com/user-attachments/assets/16bfc311-ca45-4fcb-9a30-ab007dac3e7b" />

For more info go to https://dev.azure.com/msazure/CloudNativeCompute/_wiki/wikis/CloudNativeCompute.wiki/190872/aks-go-proxy

Test your token with:
```bash
curl -v "https://whatever:$(az account get-access-token -o tsv --resource https://microsoft.onmicrosoft.com/AKSGoProxyMSFT --query accessToken)@goproxyprod.goms.io/aks/rp/protos/hcp/@v/v1.3.12.zip"
```
Successful queries will get a 404. Unsuccessful queries will get a 401. If you are getting a 401 when connecting to goproxyprod.goms.io , check that your JWT has aksgoproxyreader in the roles attribute by decoding it:
```bash
az account get-access-token -o tsv --resource https://microsoft.onmicrosoft.com/AKSGoProxyMSFT --query accessToken | jq -R -r 'split(".") | .[1] ' | base64 --decode | jq .roles
```
You should see output like the following, if you do not, make sure you have the entitlement linked at the top of this page:
```bash
[
 "aksgoproxyreader"
]
```
for your own raw environment
```bash
goproxy_bearer_token=$(az account get-access-token -o tsv --resource https://microsoft.onmicrosoft.com/AKSGoProxyMSFT --query accessToken)
export AKS_GOPROXY_TOKEN="${goproxy_bearer_token}"
export GOPROXY="https://notinuse:${AKS_GOPROXY_TOKEN}@goproxyprod.goms.io"
export GOPRIVATE="goms.io/aks/*,go.goms.io/aks/*,go.goms.io/caravel,go.goms.io/fleet*"
export GONOPROXY=none
```
and a simpler way to set the env in your console is to run command below:
```bash
andliu@devbox:~/go/src/go.goms.io/aks/rp$ source ./hack/aksbuilder_goproxy.sh
```
and for the goproxy used in aksbuilder.
you can do
```bash
export ENABLE_AKS_GOPROXY_AAD_AUTH=true
```
to enable it

---

## Azure Authentication

### Option A: Interactive Login (Standard)

```bash
# Standard login
az login

# Set your subscription
az account set --subscription "<your-subscription-id>"

# Verify
az account show
```

## Building aksdev CLI

### Option A: Quick Build (Recommended)

```bash
cd ~/acstor/aks-rp/go.goms.io/aks/rp

# Build aksdev
make build-aksdev

# Make executable
chmod +x ./bin/aksdev

# Verify
./bin/aksdev --help
```

### Option B: Build with Bazel

```bash
cd ~/acstor/aks-rp/go.goms.io/aks/rp

# Step 1: Run bootstrap (first time only)
bash ./test/e2e/pkg/aksdev/bootstrap.sh

# Step 2: Set up Go proxy
source hack/aksbuilder_goproxy.sh

# Step 3: Update BUILD.bazel files
./hack/aksbuilder.sh tidy -w ./test/e2e

# Step 4: Build aksdev
export AKSDEV_BIN_DIR="./bin"
./hack/aksbuilder.sh build -w ./test/e2e \
  --tags "" \
  --target //cmd/aksdev:aksdev \
  --binaryoutput ${AKSDEV_BIN_DIR}

# Step 5: Make executable
chmod +x ${AKSDEV_BIN_DIR}/aksdev

# Verify
./bin/aksdev --help
```

---

## Generating Azure Configuration

### Create Work Directory

```bash
mkdir -p ~/work
```

### Option A: Production Environment (Recommended for Local Testing)

Use this for testing against the production AKS RP:

```bash
./bin/aksdev e2e generate-azureconfig \
  --e2eEnvironment prod \
  --e2eManagedClusterSubscription <your-subscription-id> \
  --location eastus > ~/work/azureconfig.yaml
```

### Option B: Using E2E Test Subscriptions

Use pre-configured E2E test subscriptions where the standalone SP has permissions:

```bash
# Using ACS Test subscription
./bin/aksdev e2e generate-azureconfig \
  --e2eEnvironment prod \
  --e2eManagedClusterSubscription 8ecadfc9-d1a3-4ea4-b844-0d9f87e4d7c8 \
  --location eastus > ~/work/azureconfig.yaml
```

### Option C: With Service Principal

```bash
./bin/aksdev e2e generate-azureconfig \
  --e2eEnvironment prod \
  --e2eManagedClusterSubscription <subscription-id> \
  --location eastus \
  --clientId <your-sp-app-id> \
  --clientSecret <your-sp-secret> > ~/work/azureconfig.yaml
```

### Option D: Standalone Mode (For Testing Unmerged Changes)

**Note:** Standalone mode requires a deployed underlay cluster.

```bash
export BUILD_VERSION_STRING="standalone-$(whoami)-$(date +%Y%m%d%H%M)"

./bin/aksdev e2e generate-azureconfig \
  --e2eBuildVersion $BUILD_VERSION_STRING \
  --buildMode standalone \
  --e2eManagedClusterSubscription <subscription-id> \
  --e2eUnderlayClusterSubscription <subscription-id> \
  --location westus2 > ~/work/azureconfig.yaml
```

### Verify Configuration

```bash
cat ~/work/azureconfig.yaml
```

### Available Subscriptions for E2E Testing

| Subscription | ID | Purpose |
|--------------|-----|---------|
| ACS Test | `8ecadfc9-d1a3-4ea4-b844-0d9f87e4d7c8` | Primary E2E testing |
| ACS Test 2 | `26ad903f-2330-429d-8389-864ac35c4350` | Secondary E2E testing |
| ACS Development | `c1089427-83d3-4286-9f35-5af546a6eb67` | Development testing |
| Standalone 1 | `feb5b150-60fe-4441-be73-8c02a524f55a` | Standalone testing |
| Standalone 2 | `18153b17-4e27-4b58-863e-f8105b8892a2` | Standalone testing |

---

## Running E2E Tests

### List Available Test Suites

```bash
ls test/e2e/suites/
```

### Run a Specific Test Suite

```bash
./bin/aksdev e2e run \
  --azureconfig ~/work/azureconfig.yaml \
  --name "CNI Swift" \
  --deploy-env prod \
  -v
```

### Run Multiple Test Suites

```bash
./bin/aksdev e2e run \
  --azureconfig ~/work/azureconfig.yaml \
  --name "CNI Swift" \
  --name "Network Policy" \
  --deploy-env prod \
  -v
```

### Run Tests by Tag

```bash
./bin/aksdev e2e run \
  --azureconfig ~/work/azureconfig.yaml \
  --tag networking \
  --deploy-env prod \
  -v
```

### Run All Tests

```bash
./bin/aksdev e2e run \
  --azureconfig ~/work/azureconfig.yaml \
  --all \
  --deploy-env prod \
  -v
```

### Common Test Suites

| Suite | Description |
|-------|-------------|
| CNI Swift | Azure CNI with SWIFT overlay |
| Azure CNI | Standard Azure CNI networking |
| MSI | Managed Service Identity |
| Network Policy | Kubernetes NetworkPolicy |
| Shoebox Metrics | Metrics collection |

---

## Troubleshooting

### Issue 1: Token Protection Policy Error (AADSTS530084)

**Error:**
```
AADSTS530084: Access has been blocked by conditional access token protection policy
```

**Cause:** Microsoft's Token Protection Policy requires token to be bound to the device where login was initiated.

**Solutions:**
1. Use VS Code port forwarding (if on remote VM)
2. Use device code flow: `az login --use-device-code`
3. Login with Graph scope: `az login --scope https://graph.microsoft.com//.default`
4. Use a service principal

### Issue 2: Authorization Failed for Role Assignment

**Error:**
```
AuthorizationFailed: The client has an authorization with ABAC condition 
that is not fulfilled to perform action 'Microsoft.Authorization/roleAssignments/write'
```

**Cause:** You don't have Owner or User Access Administrator role on the subscription.

**Solutions:**
1. Use an E2E test subscription where standalone SP has permissions
2. Request Owner role on your subscription
3. Use a pre-configured service principal

### Issue 3: Invalid Underlay Suite ID

**Error:**
```
invalid underlay suite id
```

**Cause:** Standalone mode requires a deployed underlay cluster.

**Solution:** Use `prod` environment instead of `test`:
```bash
# In azureconfig.yaml, change:
e2e_environment: 'prod'

# And remove standalone settings:
# build_version: ...
# build_mode: standalone
```

Or regenerate config for prod environment:
```bash
./bin/aksdev e2e generate-azureconfig \
  --e2eEnvironment prod \
  --e2eManagedClusterSubscription <subscription-id> \
  --location eastus > ~/work/azureconfig.yaml
```

### Issue 4: aksdev Binary Not Found

**Error:**
```
bash: ./bin/aksdev: No such file or directory
```

**Solution:**
```bash
make build-aksdev
chmod +x ./bin/aksdev
```

### Issue 5: Go Proxy/Module Issues

**Error:**
```
go: module go.goms.io/aks/rp: reading ... 410 Gone
```

**Solution:**
```bash
source hack/aksbuilder_goproxy.sh
go clean -modcache
```

### Issue 6: Key Vault Access Denied

**Error:**
```
does not have secrets list permission on key vault 'akse2ekeyvault'
```

**Cause:** You don't have access to the E2E Key Vault.

**Solutions:**
1. Request Key Vault Secrets User role
2. Use your own service principal
3. Use prod environment without standalone mode

---

## Quick Reference

```bash
# ==================== SETUP ====================
# Clone repo
mkdir -p ~/acstor/aks-rp/go.goms.io/aks
cd ~/acstor/aks-rp/go.goms.io/aks
git clone https://msazure.visualstudio.com/CloudNativeCompute/_git/aks-rp rp
cd rp

# Build aksdev
make build-aksdev && chmod +x ./bin/aksdev

# ==================== AUTHENTICATE ====================
# Standard login
az login && az account set -s <subscription-id>

# Or with device code (for remote VMs)
az login --use-device-code

# ==================== GENERATE CONFIG ====================
mkdir -p ~/work

# For prod environment (recommended)
./bin/aksdev e2e generate-azureconfig \
  --e2eEnvironment prod \
  --e2eManagedClusterSubscription <subscription-id> \
  --location eastus > ~/work/azureconfig.yaml

# ==================== RUN TESTS ====================
./bin/aksdev e2e run \
  --azureconfig ~/work/azureconfig.yaml \
  --name "CNI Swift" \
  --deploy-env prod \
  -v

# ==================== CLEANUP ====================
./bin/aksdev cluster delete <cluster-name> \
  --azureconfig ~/work/azureconfig.yaml
```

---

## Service Principal Reference

### Pre-configured E2E Service Principals

| Property | Value |
|----------|-------|
| **Standalone SP Client ID** | `968a8391-e14a-44f8-9548-b35234cf2442` |
| **Standalone SP Object ID** | `0c335b77-8f65-4f08-818a-28f08ad9b223` |
| **E2E SP Client ID** | `13bec9da-7208-4aa0-8fc7-47b25e26ff5d` |
| **E2E SP Object ID** | `8ff738a5-abcd-4864-a162-6c18f7c9cbd9` |

### Microsoft Tenant ID

```
72f988bf-86f1-41af-91ab-2d7cd011db47
```

---

## Additional Resources

- **E2E Code:** `test/e2e/` directory
- **AKS_RP E2E Testing:** https://microsoft.sharepoint.com/:p:/r/teams/Aznet/_layouts/15/Doc.aspx?sourcedoc=%7B21EC7F02-952E-4C87-9890-A1F15AC078C1%7D&file=AKS-RP%20E2E%20Testing.pptx&action=edit&mobileredirect=true&share=IQECf-whLpWHTJiQofFawHjBAeL5W0etQauBRiq3eLL_CS0
- **E2E Test Name:** https://dev.azure.com/msazure/CloudNativeCompute/_git/aks-rp?path=/test/e2e/suites/matrixdata/data/e2ev2_checkin.test_matrix.json

---

*Last Updated: February 2026*
