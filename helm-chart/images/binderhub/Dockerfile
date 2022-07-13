# We use a build stage to package binderhub and pycurl into a wheel which we
# then install by itself in the final image which is relatively slimmed.
ARG DIST=buster


# The build stage
# ---------------
FROM gitlab-registry.cern.ch/linuxsupport/cs9-base:latest as build-stage
# ARG DIST is defined again to be made available in this build stage's scope.
# ref: https://docs.docker.com/engine/reference/builder/#understand-how-arg-and-from-interact
ARG DIST

# Install node as required to package binderhub to a wheel
RUN dnf install -y nodejs python3-pip

# Copy the whole git repository to /tmp/binderhub
COPY . /tmp/binderhub
WORKDIR /tmp/binderhub

# Build the binderhub python library into a wheel and save it to the ./dist
# folder. There are no pycurl or ruamel.yaml.clib wheels so we build our own in
# the build stage.
RUN python -mpip install build && python -mbuild --wheel .
RUN pip wheel --wheel-dir ./dist \
       ruamel.yaml.clib

# We download tini from here were we have wget available.

# The final stage
# ---------------
FROM gitlab-registry.cern.ch/linuxsupport/cs9-base:latest
WORKDIR /

# The slim version doesn't include git as required by binderhub
RUN yum install -y git curl python3-pycurl python3-pip && yum clean all

# We use tini as an entrypoint to not loose track of SIGTERM signals as sent
# before SIGKILL when "docker stop" or "kubectl delete pod" is run. By doing
# that the pod can terminate very quickly.
 
RUN curl  -L --output /tini "https://github.com/krallin/tini/releases/download/v0.19.0/tini-amd64" \
 && chmod +x /tini

# Copy the built wheels from the build-stage. Also copy the image
# requirements.txt built from the binderhub package requirements.txt and the
# requirements.in file using the ./dependency script.
COPY --from=build-stage /tmp/binderhub/dist/*.whl pre-built-wheels/
COPY helm-chart/images/binderhub/requirements.txt .

# Install pre-built wheels and the generated requirements.txt for the image.
RUN pip install --no-cache-dir \
        pre-built-wheels/*.whl \
        -r requirements.txt

# When using the ./dependency script to output a frozen environment, we do it
# from within this container. So below we conditionally install pip-tools for
# use by the ./dependency script.
ARG PIP_TOOLS=
RUN test -z "$PIP_TOOLS" || pip install --no-cache pip-tools==$PIP_TOOLS

ENTRYPOINT ["/tini", "--", "python3", "-m", "binderhub"]
CMD ["--config", "/etc/binderhub/config/binderhub_config.py"]
ENV PYTHONUNBUFFERED=1
EXPOSE 8585
