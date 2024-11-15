# KubeCon CloudNativeCon NA 2024 Supply Chain Security Lab

Lab/Example repository linked to KubeCon CloudNativeCon NA 2024.

## Overview

This lab demonstrates a supply chain security example using container images and npm packages. It includes building, attesting, and pushing container images and npm packages, as well as verifying SLSA/in-toto attestations.

**Technologies Used:**

- **SLSA (Supply-chain Levels for Software Artifacts):** A security framework providing a checklist of standards and controls to prevent tampering and improve software supply chain integrity.
- **in-toto:** A framework for securing the integrity of software supply chains.
- **Cosign:** A tool for container signing, verification, and storage in an OCI registry.
- **Sigstore:** An open-source project that provides a standard for signing, verifying, and protecting software.
- **GitHub Actions:** Automate, customize, and execute your software development workflows.

> [!NOTE]
> This lab repository is linked to a KubeCon/CloudNativeCon talk. More information and slides are available at: [Event Link](https://kccncna2024.sched.com/event/e835e3bc599b5a6a3a909a8cfbf2dcd5)

## Prerequisites

- **Docker:** For building and running container images.
- **Node.js and npm:** For building and publishing npm packages.
- **GitHub CLI (`gh`):** For verifying attestations. [Installation Guide](https://cli.github.com/manual/installation)
- **Cosign CLI:** For verifying signatures and attestations. [Installation Guide](https://docs.sigstore.dev/cosign/installation/)
- **`jq` utility:** For parsing JSON data in the command line. [Installation Guide](https://stedolan.github.io/jq/download/)

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Build Side](#build-side)
  - [Container Image](#container-image)
  - [NPM Package](#npm-package)
- [Runtime Side](#runtime-side)
  - [Container Image Verification](#container-image-verification)
  - [NPM Package Verification](#npm-package-verification)
- [Additional Resources](#additional-resources)

## Build Side

Inside this repository, you can find two build and publish workflows: one for the container image and one for the npm package.

Each of these workflows builds, attests, and publishes their build output.

> [!CAUTION]
> It's important to generate and sign the attestation inside the job performing the build to avoid any modification outside of the job execution.

### Container Image

For the container image, you can use GitHub's Attest/Provenance feature to easily integrate with Cosign and in-toto. Inside the `docker` workflow, the following section handles the attestation:

```yaml
- name: Generate container image attestation
  uses: actions/attest-build-provenance@v1
  with:
    subject-name: ghcr.io/${{ env.CONTAINER_IMAGE_NAME }}
    subject-digest: ${{ steps.build-and-push.outputs.digest }}
    push-to-registry: true
```

> [!TIP]
> More information about the GitHub Attestation action based on Cosign/in-toto can be found here: [Using artifact attestations to establish provenance for builds](https://docs.github.com/en/actions/security-for-github-actions/using-artifact-attestations/using-artifact-attestations-to-establish-provenance-for-builds)

### NPM Package

In the NPM workflow, attestation is handled directly by the npm CLI, leveraging the OIDC provider of GitHub Actions with the `--provenance` flag:

```bash
npm publish --provenance --access public
```

> [!TIP]
> More information about NPM provenance: [Generating provenance statements](https://docs.npmjs.com/generating-provenance-statements#about-npm-provenance)

## Runtime Side

When using the container image or the npm package, you can verify the attestation of these artifacts.

### Container Image Verification

You can use the `cosign` or the GitHub CLI to verify the attestation. Here’s how to do it using the GitHub CLI:

```bash
gh attestation verify oci://ghcr.io/aenguerrand/kubecon-cloudnativecon-na-2024-supply-chain-security-lab@sha256:bba3fe3dd187e4a5ba2f3698e2396567326062fa --owner aenguerrand
```

If the verification is successful, you will see output similar to:

```bash
Loaded digest sha256:a70076b4b04e9c31938e698aba676012924c22d766ad83bf9c4c28d40be6a3bc for oci://ghcr.io/aenguerrand/kubecon-cloudnativecon-na-2024-supply-chain-security-lab@sha256:bba3fe3dd187e4a5ba2f3698e2396567326062fa
Loaded 1 attestation from GitHub API
✓ Verification succeeded!

sha256:a70076b4b04e9c31938e698aba676012924c22d766ad83bf9c4c28d40be6a3bc was attested by:
REPO                                                                  PREDICATE_TYPE                  WORKFLOW                                     
AEnguerrand/kubecon-cloudnativecon-na-2024-supply-chain-security-lab  https://slsa.dev/provenance/v1  .github/workflows/docker.yaml@refs/heads/main
```

### NPM Package Verification

For the NPM package, you can perform an initial check using the npm CLI:

1. Install the package:

   ```bash
   npm install @aenguerrand/kubecon-cloudnativecon-na-2024-supply-chain-security-lab
   ```

2. Run the audit:

   ```bash
   npm audit signatures
   ```

Alternatively, use the `cosign` CLI to verify the attestation:

1. Download the package if you haven't already:

   ```bash
   curl -O https://registry.npmjs.org/@aenguerrand/kubecon-cloudnativecon-na-2024-supply-chain-security-lab/-/kubecon-cloudnativecon-na-2024-supply-chain-security-lab-0.1.0.tgz
   ```

2. Retrieve the attestation from the npm registry:

   ```bash
   curl https://registry.npmjs.org/-/npm/v1/attestations/@aenguerrand/kubecon-cloudnativecon-na-2024-supply-chain-security-lab@0.1.0 | \
   jq '.attestations[] | select(.predicateType=="https://slsa.dev/provenance/v1").bundle' > npm-provenance.sigstore.json
   ```

3. Verify the attestation:

   ```bash
   cosign verify-blob-attestation \
     --bundle npm-provenance.sigstore.json \
     --new-bundle-format \
     --certificate-oidc-issuer="https://token.actions.githubusercontent.com" \
     --certificate-identity="https://github.com/AEnguerrand/kubecon-cloudnativecon-na-2024-supply-chain-security-lab/.github/workflows/npm.yaml@refs/heads/main" \
     kubecon-cloudnativecon-na-2024-supply-chain-security-lab-0.1.0.tgz
   ```

If the verification is successful, you will see:

```bash
Verified OK
```

If not, you will receive an error message indicating the mismatch. For example:

```bash
Error: failed to verify certificate identity: no matching CertificateIdentity found, last error: expected SAN value "https://github.com/AEnguerrand/kubecon-cloudnativecon-na-2024-supply-chain-security-lab/.github/workflows/npm.yaml@refs/heads/dev", got "https://github.com/AEnguerrand/kubecon-cloudnativecon-na-2024-supply-chain-security-lab/.github/workflows/npm.yaml@refs/heads/main"
main.go:74: error during command execution: failed to verify certificate identity: no matching CertificateIdentity found, last error: expected SAN value "https://github.com/AEnguerrand/kubecon-cloudnativecon-na-2024-supply-chain-security-lab/.github/workflows/npm.yaml@refs/heads/dev", got "https://github.com/AEnguerrand/kubecon-cloudnativecon-na-2024-supply-chain-security-lab/.github/workflows/npm.yaml@refs/heads/main"
```

> [!TIP]
> More information about Cosign verification for different kinds of registries can be found here: [Cosign verify bundles](https://blog.sigstore.dev/cosign-verify-bundles/)

## Additional Resources

- [SLSA (Supply-chain Levels for Software Artifacts)](https://slsa.dev/)
- [in-toto](https://in-toto.io/)
- [Cosign](https://github.com/sigstore/cosign)
- [Sigstore](https://sigstore.dev/)
- [GitHub Actions: Using OIDC tokens](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect)
- [Artifact Attestations in GitHub Actions](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions#artifact-attestations)
- [Installing GitHub CLI](https://cli.github.com/manual/installation)
- [Installing Cosign CLI](https://docs.sigstore.dev/cosign/installation/)
- [Installing `jq` Utility](https://stedolan.github.io/jq/download/)