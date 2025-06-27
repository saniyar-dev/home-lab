Home Lab Operations RunbookThis document is the central source of truth for the architecture, configuration, and operation of this home lab.Mind Mapmindmap
  root((Home Lab))
    Container Infrastructure
      Container Registry (@1)
        Decision: Self-Hosted Docker Registry
        Implementation: Docker Container
        Configuration: /opt/registry/config.yml
      Kubernetes
        Distribution: MicroK8s
        CNI: Calico
1. Container RegistryThis section details the setup and maintenance of the local container registry.1.1. Architectural Decision Record (ADR)Decision: We will run a self-hosted container registry inside a Docker container on the host machine. This registry will serve as both a private registry for our own applications and as a pull-through cache/mirror for external registries like quay.io and docker.io.Reasoning: This decision directly addresses a 403 Forbidden error encountered when MicroK8s attempts to pull Calico CNI images from quay.io. A local registry acting as a proxy solves this issue permanently. Furthermore, as we plan to build a CI/CD pipeline, a local registry is a necessary component for storing our own built application images. This approach provides a long-term, scalable solution rather than a one-off manual fix.1.2. Pros & ConsPros:Solves Geo-Blocking: Permanently bypasses regional blocks by routing registry pulls through a configurable proxy.Enables CI/CD: Provides a local target to push custom-built application images.Improves Performance: Caches frequently used images (like base images, K8s components) on the LAN, making pod startup much faster.Increases Reliability: Reduces dependency on internet connection stability for pulling images.Cons:Slightly Higher Initial Complexity: Requires setting up a registry container, configuration file, and configuring clients (Docker, MicroK8s) to use it.Single Point of Failure: If the registry container goes down, no images can be pulled. (This is an acceptable risk for a home lab).Resource Usage: The registry itself consumes some disk space for images and a small amount of RAM/CPU.1.3. Alternative Workarounds ConsideredManual Image Loading: Manually pull required images via a proxy, save them as .tar files, and import them directly into MicroK8s's containerd using microk8s ctr image import.Critique: Effective for a one-time fix but creates significant manual toil for every new image or version update. Not a scalable solution.System-Wide Proxy for containerd: Configure the MicroK8s containerd service to use a proxy for all outgoing requests.Critique: A good solution, but running a registry mirror was chosen as it provides more long-term benefits (caching, private image storage) for roughly the same amount of setup effort.Use nerdctl with MicroK8s's containerd: Avoid installing a separate Docker daemon entirely and use the nerdctl addon to interact with the existing containerd process.Critique: This is the most resource-efficient option. The decision was made to stick with the more familiar Docker toolchain for now to prioritize getting the registry functional quickly, with the option to refactor to this more integrated approach later.1.4. Implementation DetailsHost Path for Configuration: /opt/registry/config.ymlHost Path for Image Storage: /var/registry/dataRun Command:# Ensure a consistent directory structure exists
mkdir -p /var/registry/data
touch /opt/registry/config.yml

# Run the container
docker run -d -p 5000:5000 --restart=always --name registry \
  -v /var/registry/data:/var/lib/registry \
  -v /opt/registry/config.yml:/etc/docker/registry/config.yml \
  registry:2
1.5. TroubleshootingCheck container logs:docker logs registry
Verify the registry is running:docker ps | grep registry
Test connectivity from the host:curl http://localhost:5000/v2/_catalog
A successful response for a new, empty registry is {"repositories":[]}. If you get curl: (7) Failed to connect to localhost port 5000: Connection refused, the container is likely not running or the port is not mapped correctly.1.6. Incident Log| Date | Incident Summary | Root Cause | Resolution ||  |  |  |  |
