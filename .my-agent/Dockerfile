# Toolchain for the GNUmakefile and CI: Go, Terraform for docs generation,
# golangci-lint, yq for `make lintcheck-new`, and rumdl for the docs lint.
# Bump a version together with both of its checksums.

USER root

RUN apt-get update \
    && apt-get install -y --no-install-recommends make unzip \
    && rm -rf /var/lib/apt/lists/*

ENV GO_VERSION=1.26.5 \
    GO_SHA256_AMD64=5c2c3b16caefa1d968a94c1daca04a7ca301a496d9b086e17ad77bb81393f053 \
    GO_SHA256_ARM64=fe4789e92b1f33358680864bbe8704289e7bb5fc207d80623c308935bd696d49 \
    TERRAFORM_VERSION=1.15.8 \
    TERRAFORM_SHA256_AMD64=d25ce7b6902013ad905db3d2eab0be4cd905887fe88b81a6171b8d5503c31f3d \
    TERRAFORM_SHA256_ARM64=8891e9dcedc9e3b8950bc6af9d4d8af1f4cfade3062f53b9dc403a89f6ce8c9c \
    GOLANGCI_LINT_VERSION=2.12.2 \
    GOLANGCI_LINT_SHA256_AMD64=8df580d2670fed8fa984aac0507099af8df275e665215f5c7a2ae3943893a553 \
    GOLANGCI_LINT_SHA256_ARM64=44cd40a8c76c86755375adfeea52cfd3533cb43d7bd647771e0ae065e166df3a \
    YQ_VERSION=4.53.3 \
    YQ_SHA256_AMD64=fa52a4e758c63d38299163fbdd1edfb4c4963247918bf9c1c5d31d84789eded4 \
    YQ_SHA256_ARM64=578648e463a11c1b6db6010cbf41eafed6bee79466fcffa1bb446672cf7945ea \
    RUMDL_VERSION=0.2.41 \
    RUMDL_SHA256_AMD64=f2a6e5f11b2bae89b3319ad909a5df76b79a8e23a6ec9394c3e738b71ef11a74 \
    RUMDL_SHA256_ARM64=51161dbb0f70c8c9adfa154aca62d4dea09166db15636096acedd70cd54baca1

RUN ARCH="$(dpkg --print-architecture)" \
    && case "$ARCH" in \
        amd64) RUST_ARCH=x86_64; \
               GO_SHA256="$GO_SHA256_AMD64"; TERRAFORM_SHA256="$TERRAFORM_SHA256_AMD64"; \
               GOLANGCI_LINT_SHA256="$GOLANGCI_LINT_SHA256_AMD64"; YQ_SHA256="$YQ_SHA256_AMD64"; \
               RUMDL_SHA256="$RUMDL_SHA256_AMD64" ;; \
        arm64) RUST_ARCH=aarch64; \
               GO_SHA256="$GO_SHA256_ARM64"; TERRAFORM_SHA256="$TERRAFORM_SHA256_ARM64"; \
               GOLANGCI_LINT_SHA256="$GOLANGCI_LINT_SHA256_ARM64"; YQ_SHA256="$YQ_SHA256_ARM64"; \
               RUMDL_SHA256="$RUMDL_SHA256_ARM64" ;; \
        *) echo "unsupported arch: $ARCH" >&2; exit 1 ;; \
       esac \
    && cd /tmp \
    && curl -fsSL "https://go.dev/dl/go${GO_VERSION}.linux-${ARCH}.tar.gz" -o go.tar.gz \
    && echo "$GO_SHA256  go.tar.gz" | sha256sum -c - \
    && tar -xzf go.tar.gz -C /usr/local \
    && curl -fsSL "https://releases.hashicorp.com/terraform/${TERRAFORM_VERSION}/terraform_${TERRAFORM_VERSION}_linux_${ARCH}.zip" -o terraform.zip \
    && echo "$TERRAFORM_SHA256  terraform.zip" | sha256sum -c - \
    && unzip -q -o terraform.zip terraform -d /usr/local/bin \
    && curl -fsSL "https://github.com/golangci/golangci-lint/releases/download/v${GOLANGCI_LINT_VERSION}/golangci-lint-${GOLANGCI_LINT_VERSION}-linux-${ARCH}.tar.gz" -o golangci-lint.tar.gz \
    && echo "$GOLANGCI_LINT_SHA256  golangci-lint.tar.gz" | sha256sum -c - \
    && tar -xzf golangci-lint.tar.gz -C /usr/local/bin --strip-components=1 "golangci-lint-${GOLANGCI_LINT_VERSION}-linux-${ARCH}/golangci-lint" \
    && curl -fsSL "https://github.com/mikefarah/yq/releases/download/v${YQ_VERSION}/yq_linux_${ARCH}" -o yq \
    && echo "$YQ_SHA256  yq" | sha256sum -c - \
    && install -m 0755 yq /usr/local/bin/yq \
    && curl -fsSL "https://github.com/rvben/rumdl/releases/download/v${RUMDL_VERSION}/rumdl-v${RUMDL_VERSION}-${RUST_ARCH}-unknown-linux-musl.tar.gz" -o rumdl.tar.gz \
    && echo "$RUMDL_SHA256  rumdl.tar.gz" | sha256sum -c - \
    && tar -xzf rumdl.tar.gz -C /usr/local/bin rumdl \
    && rm -f go.tar.gz terraform.zip golangci-lint.tar.gz yq rumdl.tar.gz \
    && /usr/local/go/bin/go version \
    && terraform version \
    && golangci-lint version \
    && yq --version \
    && rumdl --version

ENV GOPATH=/opt/go \
    PATH=/usr/local/go/bin:/opt/go/bin:$PATH

RUN install -d -o agent -g agent /opt/go /opt/provider

USER agent

COPY --chown=agent:agent go.mod go.sum /opt/provider/
COPY --chown=agent:agent tools/go.mod tools/go.sum /opt/provider/tools/
RUN cd /opt/provider && go mod download \
    && cd tools && go mod download

USER root
RUN find / -xdev -perm /6000 -type f -exec chmod -s {} +
USER agent
