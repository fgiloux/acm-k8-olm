# The base image is expected to contain
# /bin/opm (with a serve subcommand) and /bin/grpc_health_probe
# it does not work with the opm version coming with the latest registry image. An older one needs to be used instead 
# FROM registry.redhat.io/openshift4/ose-operator-registry@sha256:169a287c49142c993cafbce265183faf67fdf4b337d3bab0abdb7a25b034baad
FROM registry.redhat.io/openshift4/ose-operator-registry:v4.9

# Configure the entrypoint and command
ENTRYPOINT ["/bin/opm"]
CMD ["serve", "/configs"]

# Copy declarative config root into image at /configs
ADD compliance-operator-index /configs

# Set DC-specific label for the location of the DC root directory
# in the image
LABEL operators.operatorframework.io.index.configs.v1=/configs
