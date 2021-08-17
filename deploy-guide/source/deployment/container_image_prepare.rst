.. _prepare-environment-containers:

Container Image Preparation
===========================

This documentation explains how to instruct container image preparation to do
different preparation tasks.

Choosing an image registry strategy
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Container images need to be pulled from an image registry which is reliably
available to overcloud nodes. The three common options to serve images are to
use the default registry, the registry available on the undercloud, or an
independently managed registry. During deployment the environment parameter
`ContainerImagePrepare` is used to specify any desired behaviour, including:

- Where to pull images from
- Optionally, which local repository to push images to
- How to discover the latest versioned tag for each image

In the following examples, the parameter `ContainerImagePrepare` will be
specified in its own file `containers-prepare-parameters.yaml`.

Default registry
................

By default the images will be pulled from a remote registry namespace such as
`docker.io/tripleomaster`. This is fine for development or POC clouds but is
not appropriate for production clouds due to the transfer of large amounts of
duplicate image data over a potentially unreliable internet connection.

During deployment with this default, any heat parameters which refer to
required container images will be populated with a value pointing at the
default registry, with a tag representing the latest image version.

To generate the `containers-prepare-parameters.yaml` containing these defaults,
run this command::

  openstack tripleo container image prepare default \
    --output-env-file containers-prepare-parameters.yaml

This will generate a file containing a `ContainerImagePrepare` similar to the
following::

  parameter_defaults:
    ContainerImagePrepare:
    - set:
        ceph_image: daemon
        ceph_namespace: docker.io/ceph
        ceph_tag: v4.0.0-stable-4.0-nautilus-centos-7-x86_64
        name_prefix: centos-binary-
        name_suffix: ''
        namespace: docker.io/tripleomaster
        neutron_driver: null
        tag: current-tripleo
      tag_from_label: rdo_version

During deployment, this will lookup images in `docker.io/tripleomaster` tagged
with `current-tripleo` and discover a versioned tag by looking up the label
`rdo_version`. This will result in the heat image parameters in the plan being
set with appropriate values, such as::

  DockerNeutronMetadataImage: docker.io/tripleomaster/centos-binary-neutron-metadata-agent:35414701c176a6288fc2ad141dad0f73624dcb94_43527485
  DockerNovaApiImage: docker.io/tripleomaster/centos-binary-nova-api:35414701c176a6288fc2ad141dad0f73624dcb94_43527485

.. note:: The tag is actually a Delorean hash. You can find out the versions
          of packages by using this tag.
          For example, `35414701c176a6288fc2ad141dad0f73624dcb94_43527485` tag,
          is in fact using this `Delorean repository`_.

.. _populate-local-registry-containers:

Undercloud registry
...................

As part of the undercloud install, an image registry is configured on port
`8787`.  This can be used to increase reliability of image pulls, and minimise
overall network transfers.
The undercloud registry can be used by generating the following
`containers-prepare-parameters.yaml` file::

  openstack tripleo container image prepare default \
    --local-push-destination \
    --output-env-file containers-prepare-parameters.yaml

This will generate a file containing a `ContainerImagePrepare` similar to the
following::

  parameter_defaults:
    ContainerImagePrepare:
    - push_destination: true
      set:
        ceph_image: daemon
        ceph_namespace: docker.io/ceph
        ceph_tag: v4.0.0-stable-4.0-nautilus-centos-7-x86_64
        name_prefix: centos-binary-
        name_suffix: ''
        namespace: docker.io/tripleomaster
        neutron_driver: null
        tag: current-tripleo
      tag_from_label: rdo_version

This is identical to the default registry, except for the `push_destination:
true` entry which indicates that the address of the local undercloud registry
will be discovered at upload time.

By specifying a `push_destination` value such as `192.168.24.1:8787`, during
deployment all images will be pulled from the remote registry then pushed to
the specified registry. The resulting image parameters will also be modified to
refer to the images in `push_destination` instead of `namespace`.

.. admonition:: Stein and newer
   :class: stein

   Prior to Stein, Docker Registry v2 (provided by "Docker
   Distribution" package), was the service running on tcp 8787.
   Since Stein it has been replaced with an Apache vhost called
   "image-serve", which serves the containers on tcp 8787 and
   supports podman or buildah pull commands. Though podman or buildah
   tag, push, and commit commands are not supported, they are not
   necessary because the same functionality may be achieved through
   use of the "sudo openstack tripleo container image prepare"
   commands described in this document.


Running container image prepare
...............................
The prepare operations are run at the following times:

#. During ``undercloud install`` when `undercloud.conf` has
   `container_images_file=$HOME/containers-prepare-parameters.yaml` (see
   :ref:`install_undercloud`)
#. During ``overcloud deploy`` when a `ContainerImagePrepare` parameter is
   provided by including the argument `-e
   $HOME/containers-prepare-parameters.yaml`
   (see :ref:`overcloud-prepare-container-images`)
#. Any other time when ``sudo openstack tripleo container image prepare`` is run

As seen in the last of the above commands, ``sudo openstack tripleo
container image prepare`` may be run without ``default`` to set up an
undercloud registry without deploying the overcloud. It is run with
``sudo`` because it needs to write to `/var/lib/image-serve` on the
undercloud.


Options available in heat parameter ContainerImagePrepare
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

To do something different to the above two registry scenarios, your custom
environment can set the value of the ContainerImagePrepare heat parameter to
result in any desired registry and image scenario.

Discovering versioned tags with tag_from_label
..............................................

If you want these parameters to have the actual tag `current-tripleo` instead of
the discovered tag (in this case the Delorean hash,
`35414701c176a6288fc2ad141dad0f73624dcb94_43527485` ) then the `tag_from_label`
entry can be omitted.

Likewise, if all images should be deployed with a different tag, the value of
`tag` can be set to the desired tag.

Some build pipelines have a versioned tag which can only be discovered via a
combination of labels. For this case, a template format can be specified
instead::

      tag_from_label: {version}-{release}

It's possible to use the above feature while also disabling it only
for a subset of images by using an `includes` and `excludes` list as
described later in this document. This is useful when using the above
but also using containers from external projects which doesn't follow
the same convention like Ceph.

Copying images with push_destination
....................................

By specifying a `push_destination`, the required images will be copied from
`namespace` to this registry, for example::

  ContainerImagePrepare:
  - push_destination: 192.168.24.1:8787
    set:
      namespace: docker.io/tripleomaster
      ...

This will result in images being copied from `docker.io/tripleomaster` to
`192.168.24.1:8787/tripleomaster` and heat parameters set with values such as::

  DockerNeutronMetadataImage: 192.168.24.1:8787/tripleomaster/centos-binary-neutron-metadata-agent:35414701c176a6288fc2ad141dad0f73624dcb94_43527485
  DockerNovaApiImage: 192.168.24.1:8787/tripleomaster/centos-binary-nova-api:35414701c176a6288fc2ad141dad0f73624dcb94_43527485

.. note:: Use the IP address of your undercloud, which you previously set with
    the `local_ip` parameter in your `undercloud.conf` file. For these example
    commands, the address is assumed to be `192.168.24.1:8787`.

By setting different values for `namespace` and `push_destination` any
alternative registry strategy can be specified.

Ceph and other set options
..........................

The options `ceph_namespace`, `ceph_image`, and `ceph_tag` are similar to
`namespace` and `tag` but they specify the values for the ceph image. It will
often come from a different registry, and have a different versioned tag
policy.

The values in the `set` map are used when evaluating the file
`/usr/share/openstack-tripleo-common/container-images/tripleo_containers.yaml.j2`
as a Jinja2 template. This file contains the list of every container image and
how it relates to TripleO services and heat parameters.

Authenticated Registries
........................

If a container registry requires a username and password, then those
values may be passed using the following syntax::

  ContainerImagePrepare:
  - push_destination: 192.168.24.1:8787
    set:
      namespace: quay.io/...
      ...
  ContainerImageRegistryCredentials:
    'quay.io': {'<your_quay_username>': '<your_quay_password>'}

.. note:: If the `ContainerImageRegistryCredentials` contain the credentials
    for a registry whose name matches the `ceph_namespace` parameter, those
    credentials will be extracted and passed to ceph-ansible as the
    `ceph_docker_registry_username` and `ceph_docker_registry_password` parameters.

Layering image preparation entries
..................................

Since the value of `ContainerImagePrepare` is a list, multiple entries can be
specified, and later entries will overwrite any earlier ones. Consider the
following::

  ContainerImagePrepare:
  - tag_from_label: rdo_version
    push_destination: true
    excludes:
    - nova-api
    set:
      namespace: docker.io/tripleomaster
      name_prefix: centos-binary-
      name_suffix: ''
      tag: current-tripleo
  - push_destination: true
    includes:
    - nova-api
    set:
      namespace: mylocal
      tag: myhotfix

This will result in the following heat parameters which shows a `locally built
<build_container_images>`
and tagged `centos-binary-nova-api` being used for `DockerNovaApiImage`::

  DockerNeutronMetadataImage: 192.168.24.1:8787/tripleomaster/centos-binary-neutron-metadata-agent:35414701c176a6288fc2ad141dad0f73624dcb94_43527485
  DockerNovaApiImage: 192.168.24.1:8787/mylocal/centos-binary-nova-api:myhotfix

The `includes` and `excludes` entries can control the resulting image list in
addition to the filtering which is determined by roles and containerized
services in the plan. `includes` matches take precedence over `excludes`
matches, followed by role/service filtering. The image name must contain the
value within it to be considered a match.

The `includes` and `excludes` list is useful when pulling OpenStack
images using `tag_from_label: '{version}-{release}'` while also
pulling images which are not tagged the same way. The following
example shows how to do this with Ceph::

  ContainerImagePrepare:
  - push_destination: true
    set:
      namespace: docker.io/tripleomaster
      name_prefix: centos-binary-
      name_suffix: ''
      tag: current-tripleo
    tag_from_label: '{version}-{release}'
    excludes: [ceph]
  - push_destination: true
    set:
      ceph_image: ceph
      ceph_namespace: docker.io/ceph
      ceph_tag: latest
    includes: [ceph]

Modifying images during prepare
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

It is possible to modify images during prepare to make any required changes,
then immediately deploy with those changes. The use-cases for modifying images
include:

- As part of a Continuous Integration pipeline where images are modified with
  the changes being tested before deployment
- As part of a development workflow where local changes need to be deployed for
  testing and development
- When changes need to be deployed but are not available through an image
  build pipeline (proprietary addons, emergency fixes)

The modification is done by invoking an ansible role on each image which needs
to be modified. The role takes a source image, makes the requested changes,
then tags the result. The prepare can then push the image and set the heat
parameters to refer to the modified image. The modification is done in
the undercloud registry so it is not possible to use this feature when
using the Default registry, where images are pulled directly from a
remote registry during deployment.

The ansible role `tripleo-modify-image`_ conforms with the required role
interface, and provides the required behaviour for the modify use-cases. Modification is controlled via modify-specific keys in the
`ContainerImagePrepare` parameter:

- `modify_role` specifies what ansible role to invoke for each image to modify.
- `modify_append_tag` is used to append to the end of the
  source image tag. This makes it obvious that the resulting image has been
  modified. It is also used to skip modification if the `push_destination`
  registry already has that image, so it is recommended to change
  `modify_append_tag` whenever the image must be modified.
- `modify_vars` is a dictionary of ansible variables to pass to the role.

The different use-cases handled by role `tripleo-modify-image`_ are selected by
setting the `tasks_from` variable to the required file in that role. For all of
the following examples, see the documentation for the role
`tripleo-modify-image`_ for the other variables supported by that `tasks_from`.

While developing and testing the `ContainerImagePrepare` entries which modify
images, it is recommended to run prepare on its own to confirm it is being
modified as expected::

  sudo openstack tripleo container image prepare \
    -e ~/containers-prepare-parameters.yaml

Updating existing packages
..........................

The following entries will result in all packages being updated in the images,
but using the undercloud host's yum repository configuration::

  ContainerImagePrepare:
  - push_destination: true
    ...
    modify_role: tripleo-modify-image
    modify_append_tag: "-updated"
    modify_vars:
      tasks_from: yum_update.yml
      compare_host_packages: true
      yum_repos_dir_path: /etc/yum.repos.d
    ...

Install RPM files
.................

It is possible to install a directory of RPM files, which is useful for
installing hotfixes, local package builds, or any package which is not
available through a package repository. For example the following would install
some hotfix packages only in the `centos-binary-nova-compute` image::

  ContainerImagePrepare:
  - push_destination: true
    ...
    includes:
    - nova-compute
    modify_role: tripleo-modify-image
    modify_append_tag: "-hotfix"
    modify_vars:
      tasks_from: rpm_install.yml
      rpms_path: /home/stack/nova-hotfix-pkgs
    ...

Modify with custom Dockerfile
.............................

For maximum flexibility, it is possible to specify a directory containing a
`Dockerfile` to make the required changes. When the role is invoked, a
`Dockerfile.modified` is generated which changes the `FROM` directive and adds
extra `LABEL` directives. The following example runs the custom
`Dockerfile` on the `centos-binary-nova-compute` image::

  ContainerImagePrepare:
  - push_destination: true
    ...
    includes:
    - nova-compute
    modify_role: tripleo-modify-image
    modify_append_tag: "-hotfix"
    modify_vars:
      tasks_from: modify_image.yml
      modify_dir_path: /home/stack/nova-custom
    ...

An example `/home/stack/nova-custom/Dockerfile` follows. Note that after any
`USER root` directives have been run, it is necessary to switch back to the
original image default user::

    FROM docker.io/tripleomaster/centos-binary-nova-compute:latest

    USER root

    COPY customize.sh /tmp/
    RUN /tmp/customize.sh

    USER "nova"

..  _Delorean repository: https://trunk.rdoproject.org/centos7-master/ac/82/ac82ea9271a4ae3860528eaf8a813da7209e62a6_28eeb6c7/
..  _tripleo-modify-image: https://github.com/openstack/ansible-role-tripleo-modify-image


Modify with Python source code installed via pip from OpenDev Gerrit
....................................................................


If you would like to build an image and apply your patch in a Python project in
OpenStack, you can use this example::

  ContainerImagePrepare:
  - push_destination: true
    ...
    includes:
    - heat-api
    modify_role: tripleo-modify-image
    modify_append_tag: "-devel"
    modify_vars:
      tasks_from: dev_install.yml
      source_image: docker.io/tripleomaster/centos-binary-heat-api:current-tripleo
      refspecs:
        -
          project: heat
          refspec: refs/changes/12/1234/3
    ...

It will produce a modified image with Python source code installed via pip.

Building hotfixed containers
............................

The `tripleoclient` OpenStack plugin provides a command line interface which
will allow operators to apply packages (hotfixes) to running containers. This
capability leverages the **tripleo-modify-image** role, and automates its
application to a set of containers for a given collection of packages.

Using the provided command line interface is simple. The interface has very few
required options. The noted options below inform the tooling which containers
need to have the hotfix(es) applied, and where to find the hotfixed package(s).

============ =================================================================
   option       Description
============ =================================================================
--image       The `--image` argument requires the use fully qualified image
              name, something like *localhost/image/name:tag-data*. The
              `--image` option can be used more than once, which will inform
              the tooling that multiple containers need to have the same
              hotfix packages applied.
--rpms-path   The `--rpms-path` argument requires the full path to a
              directory where RPMs exist. The RPMs within this directory will
              be installed into the container, producing a new layer for an
              existing container.
--tag         The `--tag` argument is optional, though it is recommended to
              be used. The value of this option will append to the tag of the
              running container. By using the tag argument, images that have
              been modified can be easily identified.
============ =================================================================

With all of the required information, the command to modify existing container
images can be executed like so.

.. code-block:: shell

    # The shell variables need to be replaced with data that pertains to the given environment.
    openstack tripleo container image hotfix --image ${FULLY_QUALIFIED_IMAGE_NAME} \
                                             --rpms-path ${RPM_DIRECTORY} \
                                             --tag ${TAG_VALUE}

When this command completes, new container images will be available on the
local system and are ready to be integrated into the environment.

You should see the image built on your local system via buildah CLI:

.. code-block:: shell

   # The shell variables need to be replaced with data that pertains to the given environment.
   sudo buildah images | grep ${TAG_VALUE}

Here is an example on how to push it into the TripleO Container registry:

.. code-block:: shell

   # ${IMAGE} is in this format: <registry>/<namespace>/<name>:<tag>
   sudo openstack tripleo container image push --local \
        --registry-url 192.168.24.1:8787 ${IMAGE}

.. note::

    Container images can be pushed to the TripleO Container registry or
    a Docker Registry (using basic auth or the bearer token auth).

Now that your container image is pushed into a registry, you can deploy it
where it's needed. Two ways are supported:

* (Long but persistent): Update Container$NameImage where $Name is the name of
  the service we update (e.g. ContainerNovaComputeImage). The parameters
  can be found in TripleO Heat Templates. Once you update it into your
  environment, you need to re-run the "openstack overcloud deploy" command
  again and the necessary hosts will get the new container.

  Example::

    parameter_defaults:
        # Replace the values by where the image is stored
        ContainerNovaComputeImage: <registry>/<namespace>/<name>:<tag>

* (Short but not persistent after a minor update): Run Paunch or Ansible
  to update the container on a host. The procedure is already documented
  in the :doc:`./tips_tricks` manual.


Once the hotfixed container image has been deployed, it's very important to
check that the container is running with the right rpm version.
For example, if the nova-compute container was updated with a new hotfix image,
we want to check that the right nova-compute rpm is installed:

.. code-block:: shell

    sudo podman exec -ti -u root nova_compute rpm -qa | grep nova-compute

It will return the version of the openstack-nova-compute rpm and we can compare
it with the one that was delivered via rpm. If the version is not correct (e.g.
older), it means that the hotfix image is wrong and doesn't contain the rpm
provided to build the new image. The image has to be rebuilt and redeployed.
