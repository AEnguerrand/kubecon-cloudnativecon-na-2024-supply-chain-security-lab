# kubecon-cloudnativecon-na-2024-supply-chain-security-lab
Lab/Example - Link to the KubeCon CloudNativeCon NA 2024

## Overview

This lab demonstrates a supply chain security example using Docker and npm packages. It includes building, attesting, and pushing container images and npm packages, as well as verifying SLSA/in-toto attestations.


> [!NOTE]
> This lab repository is link to a KubeCon/CloudNative Con talk, more inforations and slide are avaible: https://kccncna2024.sched.com/event/e835e3bc599b5a6a3a909a8cfbf2dcd5

## On the build side

Inside this repositories, you can find 2 build and publish workflow, one for the container image, one for the npm package.

Each of this workflows are building, attesting and publishing their build output.

> [!CAUTION]
> It's important to generate and sign the attestion inside the job doing the build, to avoid any modification outside of the job excution.


### Container image

For the container, it's possible to use the Github Attest / Provenance of Github, to have an easy integration with Cosign and in-toto, inside the workflow `docker`, this part if doing the attestation:
```bash
- name: Generate container image attestation
  uses: actions/attest-build-provenance@v1
  with:
    subject-name: ghcr.io/${{ env.CONTAINER_IMAGE_NAME }}
    subject-digest: ${{ steps.build-and-push.outputs.digest }}
    push-to-registry: true
```


> [!TIP]
> More information about the Github Attestion action based on Cosign/In-toto: https://docs.github.com/en/actions/security-for-github-actions/using-artifact-attestations/using-artifact-attestations-to-establish-provenance-for-builds

### NPM Package

Inside the NPM workflow, the attestation is handle dirrectly by the npm CLI, an based on the OIDC provider of Github Action Workflow, with the flag `--provenance`:
```bash
run: npm publish --provenance --access public
```

> [!TIP]
> More information about the NPM provenance: https://docs.npmjs.com/generating-provenance-statements#about-npm-provenance

## On the runtime side

When you are using the conainer image or the npm package you can check the attestation of theses artefacts

### Container image

For the container, you can use the cosign or the Github CLI in this particular case:
```bash
gh attestation verify oci://ghcr.io/aenguerrand/kubecon-cloudnativecon-na-2024-supply-chain-security-lab:sha-bba3fe3dd187e4a5ba2f3698e2396567326062fa --owner aenguerrand
```

If all is correct, you are going to have this output:
```bash
gh attestation verify oci://ghcr.io/aenguerrand/kubecon-cloudnativecon-na-2024-supply-chain-security-lab:sha-bba3fe3dd187e4a5ba2f3698e2396567326062fa --owner aenguerrand
Loaded digest sha256:a70076b4b04e9c31938e698aba676012924c22d766ad83bf9c4c28d40be6a3bc for oci://ghcr.io/aenguerrand/kubecon-cloudnativecon-na-2024-supply-chain-security-lab:sha-bba3fe3dd187e4a5ba2f3698e2396567326062fa
Loaded 1 attestation from GitHub API
✓ Verification succeeded!

sha256:a70076b4b04e9c31938e698aba676012924c22d766ad83bf9c4c28d40be6a3bc was attested by:
REPO                                                                  PREDICATE_TYPE                  WORKFLOW                                     
AEnguerrand/kubecon-cloudnativecon-na-2024-supply-chain-security-lab  https://slsa.dev/provenance/v1  .github/workflows/docker.yaml@refs/heads/main
```

### NPM Package

For the NPM package, you can have a first check with the NPM CLI, you need to first install it:
```bash
npm install @aenguerrand/kubecon-cloudnativecon-na-2024-supply-chain-security-lab
```
And then:

```bash
npm audit signatures
```

But you can also the `cosign` CLI to check the attestation, in order to do that, first you need to download your package (if you d'ont have yet the package):
```bash
curl https://registry.npmjs.org/@aenguerrand/kubecon-cloudnativecon-na-2024-supply-chain-security-lab/-/kubecon-cloudnativecon-na-2024-supply-chain-security-lab-0.1.0.tgz > kubecon-cloudnativecon-na-2024-supply-chain-security-lab-0.1.0.tgz  
```

Then, the attestation from npmjs.com registry:
```bash
curl https://registry.npmjs.org/-/npm/v1/attestations/@aenguerrand/kubecon-cloudnativecon-na-2024-supply-chain-security-lab@0.1.0 | jq '.attestations[]|select(.predicateType=="https://slsa.dev/provenance/v1").bundle' > npm-provenance.sigstore.json
```

At the end, you can verificy the attestation, based on the attestation, your package and by defining the indentiy of the signer, in this case a Github Action Workflow.
```bash
cosign verify-blob-attestation --bundle npm-provenance.sigstore.json --new-bundle-format --certificate-oidc-issuer="https://token.actions.githubusercontent.com" --certificate-identity="https://github.com/AEnguerrand/kubecon-cloudnativecon-na-2024-supply-chain-security-lab/.github/workflows/npm.yaml@refs/heads/main" kubecon-cloudnativecon-na-2024-supply-chain-security-lab-0.1.0.tgz 
```

So, if all is correct, you are going to have:
```bash
cosign verify-blob-attestation --bundle npm-provenance.sigstore.json --new-bundle-format --certificate-oidc-issuer="https://token.actions.githubusercontent.com" --certificate-identity="https://github.com/AEnguerrand/kubecon-cloudnativecon-na-2024-supply-chain-security-lab/.github/workflows/npm.yaml@refs/heads/main" kubecon-cloudnativecon-na-2024-supply-chain-security-lab-0.1.0.tgz 

Verified OK
```

If not, you are going to have an error, for example, ifthe identity is not matching:
```bash
cosign verify-blob-attestation --bundle npm-provenance.sigstore.json --new-bundle-format --certificate-oidc-issuer="https://token.actions.githubusercontent.com" --certificate-identity="https://github.com/AEnguerrand/kubecon-cloudnativecon-na-2024-supply-chain-security-lab/.github/workflows/npm.yaml@refs/heads/dev" kubecon-cloudnativecon-na-2024-supply-chain-security-lab-0.1.0.tgz 

Error: failed to verify certificate identity: no matching CertificateIdentity found, last error: expected SAN value "https://github.com/AEnguerrand/kubecon-cloudnativecon-na-2024-supply-chain-security-lab/.github/workflows/npm.yaml@refs/heads/dev", got "https://github.com/AEnguerrand/kubecon-cloudnativecon-na-2024-supply-chain-security-lab/.github/workflows/npm.yaml@refs/heads/main"
main.go:74: error during command execution: failed to verify certificate identity: no matching CertificateIdentity found, last error: expected SAN value "https://github.com/AEnguerrand/kubecon-cloudnativecon-na-2024-supply-chain-security-lab/.github/workflows/npm.yaml@refs/heads/dev", got "https://github.com/AEnguerrand/kubecon-cloudnativecon-na-2024-supply-chain-security-lab/.github/workflows/npm.yaml@refs/heads/main"
```

> [!TIP]
> More information about the Cosign verification for different kind of registry: https://blog.sigstore.dev/cosign-verify-bundles/