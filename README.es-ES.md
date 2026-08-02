

# Laboratorio de Seguridad de la Cadena de Suministro KubeCon CloudNativeCon NA 2024

Repositorio de laboratorio/ejemplo vinculado a KubeCon CloudNativeCon NA 2024.

## Descripción general

Este laboratorio demuestra un ejemplo de seguridad en la cadena de suministro utilizando imágenes de contenedores y paquetes npm. Incluye la construcción, emisión de atestaciones y publicación de imágenes de contenedores y paquetes npm, así como la verificación de atestaciones SLSA/in-toto.

**Tecnologías utilizadas:**

- **SLSA (Supply-chain Levels for Software Artifacts):** Un marco de seguridad que proporciona una lista de verificación de estándares y controles para prevenir manipulaciones y mejorar la integridad de la cadena de suministro de software.
- **in-toto:** Un marco para garantizar la integridad de las cadenas de suministro de software.
- **Cosign:** Una herramienta para firmar, verificar y almacenar contenedores en un registro OCI.
- **Sigstore:** Un proyecto de código abierto que proporciona un estándar para firmar, verificar y proteger software.
- **GitHub Actions:** Automatiza, personaliza y ejecuta tus flujos de trabajo de desarrollo de software.

> [!NOTE]
> Este repositorio de laboratorio está vinculado a una conferencia de KubeCon/CloudNativeCon. Más información y diapositivas están disponibles en: [Enlace al evento](https://kccncna2024.sched.com/event/e835e3bc599b5a6a3a909a8cfbf2dcd5)

## Requisitos previos

- **Docker:** Para construir y ejecutar imágenes de contenedores.
- **Node.js y npm:** Para construir y publicar paquetes npm.
- **GitHub CLI (`gh`):** Para verificar atestaciones. [Guía de instalación](https://cli.github.com/manual/installation)
- **Cosign CLI:** Para verificar firmas y atestaciones. [Guía de instalación](https://docs.sigstore.dev/cosign/installation/)
- **Utilidad `jq`:** Para analizar datos JSON en la línea de comandos. [Guía de instalación](https://stedolan.github.io/jq/download/)

## Tabla de contenido

- [Descripción general](#overview)
- [Requisitos previos](#prerequisites)
- [Fase de construcción](#build-side)
  - [Imagen de contenedor](#container-image)
  - [Paquete NPM](#npm-package)
- [Fase de ejecución](#runtime-side)
  - [Verificación de imagen de contenedor](#container-image-verification)
  - [Verificación de paquete NPM](#npm-package-verification)
- [Recursos adicionales](#additional-resources)

## Fase de construcción

Dentro de este repositorio, encontrarás dos flujos de trabajo de construcción y publicación: uno para la imagen de contenedor y otro para el paquete npm.

Cada uno de estos flujos de trabajo construye, emite atestaciones y publica su resultado de compilación.

> [!CAUTION]
> Es importante generar y firmar la atestación dentro del trabajo que realiza la compilación para evitar cualquier modificación fuera de la ejecución del trabajo.

### Imagen de contenedor

Para la imagen de contenedor, puedes utilizar la función Attest/Provenance de GitHub para integrarte fácilmente con Cosign e in-toto. Dentro del flujo de trabajo `docker`, la siguiente sección maneja la atestación:

```yaml
- name: Generate container image attestation
  uses: actions/attest-build-provenance@v1
  with:
    subject-name: ghcr.io/${{ env.CONTAINER_IMAGE_NAME }}
    subject-digest: ${{ steps.build-and-push.outputs.digest }}
    push-to-registry: true
```

> [!TIP]
> Más información sobre la acción de atestación de GitHub basada en Cosign/in-toto se puede encontrar aquí: [Utilizar atestaciones de artefactos para establecer la proveniencia de las compilaciones](https://docs.github.com/en/actions/security-for-github-actions/using-artifact-attestations/using-artifact-attestations-to-establish-provenance-for-builds)

### Paquete NPM

En el flujo de trabajo de NPM, la atestación se maneja directamente mediante la CLI de npm, aprovechando el proveedor OIDC de GitHub Actions con la bandera `--provenance`:

```bash
npm publish --provenance --access public
```

> [!TIP]
> Más información sobre la proveniencia de NPM: [Generación de declaraciones de proveniencia](https://docs.npmjs.com/generating-provenance-statements#about-npm-provenance)

## Fase de ejecución

Al utilizar la imagen de contenedor o el paquete npm, puedes verificar la atestación de estos artefactos.

### Verificación de imagen de contenedor

Puedes usar `cosign` o la CLI de GitHub para verificar la atestación. Aquí te mostramos cómo hacerlo utilizando la CLI de GitHub:

```bash
gh attestation verify oci://ghcr.io/aenguerrand/kubecon-cloudnativecon-na-2024-supply-chain-security-lab@sha256:bba3fe3dd187e4a5ba2f3698e2396567326062fa --owner aenguerrand
```

Si la verificación es exitosa, verás una salida similar a:

```bash
Loaded digest sha256:a70076b4b04e9c31938e698aba676012924c22d766ad83bf9c4c28d40be6a3bc for oci://ghcr.io/aenguerrand/kubecon-cloudnativecon-na-2024-supply-chain-security-lab@sha256:bba3fe3dd187e4a5ba2f3698e2396567326062fa
Loaded 1 attestation from GitHub API
✓ Verification succeeded!

sha256:a70076b4b04e9c31938e698aba676012924c22d766ad83bf9c4c28d40be6a3bc was attested by:
REPO                                                                  PREDICATE_TYPE                  WORKFLOW                                     
AEnguerrand/kubecon-cloudnativecon-na-2024-supply-chain-security-lab  https://slsa.dev/provenance/v1  .github/workflows/docker.yaml@refs/heads/main
```

### Verificación de paquete NPM

Para el paquete NPM, puedes realizar una verificación inicial usando la CLI de npm:

1. Instala el paquete:

   ```bash
   npm install @aenguerrand/kubecon-cloudnativecon-na-2024-supply-chain-security-lab
   ```

2. Ejecuta la auditoría:

   ```bash
   npm audit signatures
   ```

Como alternativa, usa la CLI `cosign` para verificar la atestación:

1. Descarga el paquete si aún no lo has hecho:

   ```bash
   curl -O https://registry.npmjs.org/@aenguerrand/kubecon-cloudnativecon-na-2024-supply-chain-security-lab/-/kubecon-cloudnativecon-na-2024-supply-chain-security-lab-0.1.0.tgz
   ```

2. Obten la atestación desde el registro de npm:

   ```bash
   curl https://registry.npmjs.org/-/npm/v1/attestations/@aenguerrand/kubecon-cloudnativecon-na-2024-supply-chain-security-lab@0.1.0 | \
   jq '.attestations[] | select(.predicateType=="https://slsa.dev/provenance/v1").bundle' > npm-provenance.sigstore.json
   ```

3. Verifica la atestación:

   ```bash
   cosign verify-blob-attestation \
     --bundle npm-provenance.sigstore.json \
     --new-bundle-format \
     --certificate-oidc-issuer="https://token.actions.githubusercontent.com" \
     --certificate-identity="https://github.com/AEnguerrand/kubecon-cloudnativecon-na-2024-supply-chain-security-lab/.github/workflows/npm.yaml@refs/heads/main" \
     kubecon-cloudnativecon-na-2024-supply-chain-security-lab-0.1.0.tgz
   ```

Si la verificación es exitosa, verás:

```bash
Verified OK
```

Si no, recibirás un mensaje de error que indica la discrepancia. Por ejemplo:

```bash
Error: failed to verify certificate identity: no matching CertificateIdentity found, last error: expected SAN value "https://github.com/AEnguerrand/kubecon-cloudnativecon-na-2024-supply-chain-security-lab/.github/workflows/npm.yaml@refs/heads/dev", got "https://github.com/AEnguerrand/kubecon-cloudnativecon-na-2024-supply-chain-security-lab/.github/workflows/npm.yaml@refs/heads/main"
main.go:74: error during command execution: failed to verify certificate identity: no matching CertificateIdentity found, last error: expected SAN value "https://github.com/AEnguerrand/kubecon-cloudnativecon-na-2024-supply-chain-security-lab/.github/workflows/npm.yaml@refs/heads/dev", got "https://github.com/AEnguerrand/kubecon-cloudnativecon-na-2024-supply-chain-security-lab/.github/workflows/npm.yaml@refs/heads/main"
```

> [!TIP]
> Más información sobre la verificación de Cosign para diferentes tipos de registros se puede encontrar aquí: [Paquetes de verificación de Cosign](https://blog.sigstore.dev/cosign-verify-bundles/)

## Recursos adicionales

- [SLSA (Supply-chain Levels for Software Artifacts)](https://slsa.dev/)
- [in-toto](https://in-toto.io/)
- [Cosign](https://github.com/sigstore/cosign)
- [Sigstore](https://sigstore.dev/)
- [GitHub Actions: Uso de tokens OIDC](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect)
- [Atestaciones de artefactos en GitHub Actions](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions#artifact-attestations)
- [Instalación de GitHub CLI](https://cli.github.com/manual/installation)
- [Instalación de Cosign CLI](https://docs.sigstore.dev/cosign/installation/)
- [Instalación de la utilidad `jq`](https://stedolan.github.io/jq/download/)
