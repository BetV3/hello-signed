# hello-signed

A tiny nginx image that is built, signed and attested in GitHub Actions. It exists as the
"known good" fixture for a Kyverno `verifyImages` admission policy.

Image: `ghcr.io/betv3/hello-signed`

What the workflow (`.github/workflows/release.yml`) does on every push to `main` and on `v*` tags:

1. Builds the image from `Dockerfile` (nginx-unprivileged plus one static `index.html`) and pushes it to GHCR.
2. Signs the image digest with cosign keyless. The signing certificate comes from Fulcio using the
   GitHub Actions OIDC token, and the signature is recorded in the Rekor transparency log.
3. Generates an SPDX JSON SBOM with syft and attaches it as a cosign attestation (also keyless).
4. Verifies both the signature and the attestation in the same run.

The tag `unsigned` is a plain copy of the upstream nginx image with no signature. It is there on
purpose so the admission policy has something to reject.

## Verify locally

You need cosign (v2 or newer). No registry credentials are required; the package is public.

```sh
cosign verify \
  --certificate-identity-regexp '^https://github.com/BetV3/hello-signed/' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  ghcr.io/betv3/hello-signed:v1.0.0

cosign verify-attestation --type spdxjson \
  --certificate-identity-regexp '^https://github.com/BetV3/hello-signed/' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  ghcr.io/betv3/hello-signed:v1.0.0
```

The certificate identity for a tag build is
`https://github.com/BetV3/hello-signed/.github/workflows/release.yml@refs/tags/v1.0.0`
(for a main build the suffix is `@refs/heads/main`). To see everything attached to the image:

```sh
cosign tree ghcr.io/betv3/hello-signed:v1.0.0
```

## Kyverno keyless snippet

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-hello-signed
spec:
  validationFailureAction: Enforce
  webhookTimeoutSeconds: 30
  rules:
    - name: keyless-github-oidc
      match:
        any:
          - resources:
              kinds:
                - Pod
      verifyImages:
        - imageReferences:
            - "ghcr.io/betv3/hello-signed*"
          attestors:
            - entries:
                - keyless:
                    issuer: "https://token.actions.githubusercontent.com"
                    subjectRegExp: "^https://github.com/BetV3/hello-signed/.github/workflows/release.yml@refs/(heads/main|tags/v.*)$"
                    rekor:
                      url: "https://rekor.sigstore.dev"
```

Expected result: pods using `ghcr.io/betv3/hello-signed:v1.0.0` (or any signed digest) are admitted;
pods using `ghcr.io/betv3/hello-signed:unsigned` are rejected.
