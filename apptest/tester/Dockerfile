FROM gcr.io/cloud-marketplace-tools/testrunner:0.1.5

RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates \
    gettext \
    jq \
    uuid-runtime \
    wget \
    curl \
    && rm -rf /var/lib/apt/lists/*

RUN stable_version=$(wget -qO- https://storage.googleapis.com/kubernetes-release/release/stable.txt) \
    && wget -O /bin/kubectl "https://storage.googleapis.com/kubernetes-release/release/${stable_version}/bin/linux/amd64/kubectl" \
    && chmod 755 /bin/kubectl

COPY tests/basic-suite.yaml /tests/basic-suite.yaml
COPY --chmod=0755 tester.sh /tester.sh

WORKDIR /
ENTRYPOINT ["/tester.sh"]
