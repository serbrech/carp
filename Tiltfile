# -*- mode: Python -*-

# set defaults

settings = {
    "deploy_cert_manager": True,
    "preload_images_for_kind": True,
    "enable_providers": [],
    "kind_cluster_name": "kind",
}

# global settings
settings.update(read_json(
    "tilt-settings.json",
    default = {},
))

if settings.get("trigger_mode") == "manual":
    trigger_mode(TRIGGER_MODE_MANUAL)

allow_k8s_contexts(settings.get("allowed_contexts"))

default_registry(settings.get("default_registry"))

always_enable_providers = [
    "carp",
    "core",
    "azure",
    "kubeadm-bootstrap",
    "kubeadm-control-plane",
    "azure"
]

extra_args = settings.get("extra_args", {})

providers = {
    "carp": {
        "context": ".",
        "image": "juanlee/carp-controller",
        "live_reload_deps": [
            "main.go",
            "go.mod",
            "go.sum",
            "api",
            "controllers",
            "pkg",
            "config",
        ],
    },
    "azure" : {
        "context": "../../../sigs.k8s.io/cluster-api-provider-azure",
        "image": "gcr.io/k8s-staging-cluster-api-azure/cluster-api-azure-controller",
            "live_reload_deps": [
            "main.go",
            "go.mod",
            "go.sum",
            "api",
            "cloud",
            "controllers",
            "pkg",
        ],
    },
    "core": {
        "context": "../../../sigs.k8s.io/cluster-api",
        "image": "gcr.io/k8s-staging-cluster-api/cluster-api-controller",
        "live_reload_deps": [
            "main.go",
            "go.mod",
            "go.sum",
            "api",
            "cmd",
            "controllers",
            "errors",
            "third_party",
            "util",
        ],
    },
    "kubeadm-bootstrap": {
        "context": "../../../sigs.k8s.io/cluster-api/bootstrap/kubeadm",
        "image": "gcr.io/k8s-staging-cluster-api/kubeadm-bootstrap-controller",
        "live_reload_deps": [
            "main.go",
            "api",
            "controllers",
            "internal",
        ],
    },
    "kubeadm-control-plane": {
        "context": "../../../sigs.k8s.io/cluster-api/controlplane/kubeadm",
        "image": "gcr.io/k8s-staging-cluster-api/kubeadm-control-plane-controller",
        "live_reload_deps": [
            "main.go",
            "api",
            "controllers",
            "internal",
        ],
    },
}

# Reads a provider's tilt-provider.json file and merges it into the providers map. An example file looks like this:
# {
#     "name": "aws",
#     "config": {
#         "image": "gcr.io/k8s-staging-cluster-api-aws/cluster-api-aws-controller",
#         "live_reload_deps": [
#             "main.go", "go.mod", "go.sum", "api", "cmd", "controllers", "pkg"
#         ]
#     }
# }
def load_provider_tiltfiles():
    provider_repos = settings.get("provider_repos", [])

    print(provider_repos)

    for repo in provider_repos:
        file = repo + "/tilt-provider.json"
        provider_details = read_json(file, default = {})
        provider_name = provider_details["name"]
        provider_config = provider_details["config"]
        provider_config["context"] = repo
        providers[provider_name] = provider_config

tilt_helper_dockerfile_header = """
# Tilt image
FROM golang:1.14.2 as tilt-helper
# Support live reloading with Tilt
RUN wget --output-document /restart.sh --quiet https://raw.githubusercontent.com/windmilleng/rerun-process-wrapper/master/restart.sh  && \
    wget --output-document /start.sh --quiet https://raw.githubusercontent.com/windmilleng/rerun-process-wrapper/master/start.sh && \
    chmod +x /start.sh && chmod +x /restart.sh
"""

tilt_dockerfile_header = """
FROM gcr.io/distroless/base:debug as tilt
WORKDIR /
COPY --from=tilt-helper /start.sh .
COPY --from=tilt-helper /restart.sh .
COPY manager .
"""

# Configures a provider by doing the following:
#
# 1. Enables a local_resource go build of the provider's manager binary
# 2. Configures a docker build for the provider, with live updating of the manager binary
# 3. Runs kustomize for the provider's config/ and applies it
def enable_provider(name):
    p = providers.get(name)

    context = p.get("context")

    # Prefix each live reload dependency with context. For example, for if the context is
    # test/infra/docker and main.go is listed as a dep, the result is test/infra/docker/main.go. This adjustment is
    # needed so Tilt can watch the correct paths for changes.
    live_reload_deps = []
    for d in p.get("live_reload_deps", []):
        live_reload_deps.append(context + "/" + d)

    # Set up a local_resource build of the provider's manager binary. The provider is expected to have a main.go in
    # manager_build_path. The binary is written to .tiltbuild/manager.
    local_resource(
        name + "_manager",
        cmd = "cd " + context + ';mkdir -p .tiltbuild;CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -ldflags \'-extldflags "-static"\' -o .tiltbuild/manager',
        deps = live_reload_deps,
    )

    additional_docker_helper_commands = p.get("additional_docker_helper_commands", "")
    additional_docker_build_commands = p.get("additional_docker_build_commands", "")

    dockerfile_contents = "\n".join([
        tilt_helper_dockerfile_header,
        additional_docker_helper_commands,
        tilt_dockerfile_header,
        additional_docker_build_commands,
    ])

    # Set up an image build for the provider. The live update configuration syncs the output from the local_resource
    # build into the container.
    entrypoint = ["sh", "/start.sh", "/manager"]
    provider_args = extra_args.get(name)
    if provider_args:
        entrypoint.extend(provider_args)

    docker_build(
        ref = p.get("image"),
        context = context + "/.tiltbuild/",
        dockerfile_contents = dockerfile_contents,
        target = "tilt",
        entrypoint = entrypoint,
        only = "manager",
        live_update = [
            sync(context + "/.tiltbuild/manager", "/manager"),
            run("sh /restart.sh"),
        ],
    )

    # Apply the kustomized yaml for this provider
    yaml = str(kustomize(context + "/config"))
    substitutions = settings.get("kustomize_substitutions", {})
    for substitution in substitutions:
        value = substitutions[substitution]
        yaml = yaml.replace("${" + substitution + "}", value)
    k8s_yaml(blob(yaml))

# Prepull all the cert-manager images to your local environment and then load them directly into kind. This speeds up
# setup if you're repeatedly destroying and recreating your kind cluster, as it doesn't have to pull the images over
# the network each time.
def deploy_cert_manager():
    registry = "quay.io/jetstack"
    version = "v0.14.2"
    images = ["cert-manager-controller", "cert-manager-cainjector", "cert-manager-webhook"]

    if settings.get("preload_images_for_kind"):
        for image in images:
            local("docker pull {}/{}:{}".format(registry, image, version))
            local("kind load docker-image --name {} {}/{}:{}".format(settings.get("kind_cluster_name"), registry, image, version))

    local("kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/{}/cert-manager.yaml".format(version))

    # wait for the service to become available
    local("kubectl wait --for=condition=Available --timeout=300s apiservice v1alpha3.cert-manager.io")

# Users may define their own Tilt customizations in tilt.d. This directory is excluded from git and these files will
# not be checked in to version control.
def include_user_tilt_files():
    user_tiltfiles = listdir("tilt.d")
    for f in user_tiltfiles:
        include(f)

# Enable core cluster-api plus everything listed in 'enable_providers' in tilt-settings.json
def enable_providers():
    user_enable_providers = settings.get("enable_providers", [])
    union_enable_providers = {k: "" for k in user_enable_providers + always_enable_providers}.keys()
    for name in union_enable_providers:
        enable_provider(name)

##############################
# Actual work happens here
##############################
include_user_tilt_files()

# load_provider_tiltfiles()

if settings.get("deploy_cert_manager"):
    deploy_cert_manager()

enable_providers()