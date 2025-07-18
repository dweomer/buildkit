---
title: SLSA definitions
---

BuildKit supports the [creation of SLSA Provenance](./slsa-provenance.md) for
builds that it runs.

The provenance format generated by BuildKit is defined by the
SLSA Provenance format (supports both [v0.2](https://slsa.dev/spec/v0.2/provenance)
and [v1](https://slsa.dev/spec/v1.1/provenance)).

This page describes how BuildKit populate each field, and whether the field gets
included when you generate attestations `mode=min` and `mode=max`.

## SLSA v1

### `buildDefinition.buildType`

* Ref: https://slsa.dev/spec/v1.1/provenance#buildType
* Included with `mode=min` and `mode=max`.

The `buildDefinition.buildType` field is set to `https://github.com/moby/buildkit/blob/master/docs/attestations/slsa-definitions.md`
and can be used to determine the structure of the provenance content.

```json
    "buildDefinition": {
      "buildType": "https://github.com/moby/buildkit/blob/master/docs/attestations/slsa-definitions.md",
      ...
    }
```

### `buildDefinition.externalParameters.configSource`

* Ref: https://slsa.dev/spec/v1.1/provenance#externalParameters
* Included with `mode=min` and `mode=max`.

Describes the config that initialized the build.

```json
    "buildDefinition": {
      "externalParameters": {
        "configSource": {
          "uri": "https://github.com/moby/buildkit.git#refs/tags/v0.11.0",
          "digest": {
            "sha1": "4b220de5058abfd01ff619c9d2ff6b09a049bea0"
          },
          "path": "Dockerfile"
        },
        ...
      },
    }
```

For builds initialized from a remote context, like a Git or HTTP URL, this
object defines the context URL and its immutable digest in the `uri` and
`digest` fields. For builds using a local frontend, such as a Dockerfile, the
`path` field defines the path for the frontend file that initialized the build
(`filename` frontend option).

### `buildDefinition.externalParameters.request`

* Ref: https://slsa.dev/spec/v1.1/provenance#externalParameters
* Partially included with `mode=min`.

Describes build inputs passed to the build.

```json
    "buildDefinition": {
      "externalParameters": {
        "request": {
          "frontend": "gateway.v0",
          "args": {
            "build-arg:BUILDKIT_CONTEXT_KEEP_GIT_DIR": "1",
            "label:FOO": "bar",
            "source": "docker/dockerfile-upstream:master",
            "target": "release"
          },
          "secrets": [
            {
              "id": "GIT_AUTH_HEADER",
              "optional": true
            },
            ...
          ],
          "ssh": [],
          "locals": []
        },
        ...
      },
    }
```

The following fields are included with both `mode=min` and `mode=max`:

- `locals` lists any local sources used in the build, including the build
  context and frontend file.
- `frontend` defines type of BuildKit frontend used for the build. Currently,
  this can be `dockerfile.v0` or `gateway.v0`.
- `args` defines the build arguments passed to the BuildKit frontend.

  The keys inside the `args` object reflect the options as BuildKit receives
  them. For example, `build-arg` and `label` prefixes are used for build
  arguments and labels, and `target` key defines the target stage that was
  built. The `source` key defines the source image for the Gateway frontend, if
  used.

The following fields are only included with `mode=max`:

- `secrets` defines secrets used during the build. Note that actual secret
  values are not included.
- `ssh` defines the ssh forwards used during the build.

### `buildDefinition.internalParameters.buildConfig`

* Ref: https://slsa.dev/spec/v1.1/provenance#internalParameters
* Only included with `mode=max`.

Defines the build steps performed during the build.

BuildKit internally uses LLB definition to execute the build steps. The LLB
definition of the build steps is defined in the
`buildDefinition.internalParameters.buildConfig.llbDefinition` field.

Each LLB step is the JSON definition of the
[LLB ProtoBuf API](https://github.com/moby/buildkit/blob/v0.10.0/solver/pb/ops.proto).
The dependencies for a vertex in the LLB graph can be found in the `inputs`
field for every step.

```json
    "buildDefinition": {
      "internalParameters": {
        "buildConfig": {
          "llbDefinition": [
            {
              "id": "step0",
              "op": {
                "Op": {
                  "exec": {
                    "meta": {
                      "args": [
                        "/bin/sh",
                        "-c",
                        "go build ."
                      ],
                      "env": [
                        "PATH=/go/bin:/usr/local/go/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
                        "GOPATH=/go",
                        "GOFLAGS=-mod=vendor",
                      ],
                      "cwd": "/src",
                    },
                    "mounts": [...]
                  }
                },
                "platform": {...},
              },
              "inputs": [
                "step8:0",
                "step2:0",
              ]
            },
            ...
          ]
        },
      }
    }
```

### `buildDefinition.internalParameters.builderPlatform`

* Ref: https://slsa.dev/spec/v1.1/provenance#internalParameters
* Included with `mode=min` and `mode=max`.

```json
    "buildDefinition": {
      "internalParameters": {
        "builderPlatform": "linux/amd64"
        ...
      },
    }
```

BuildKit sets the `builderPlatform` of the build machine. Note that this is not
necessarily the platform of the build result that can be determined from the
`in-toto` subject field.

### `buildDefinition.resolvedDependencies`

* Ref: https://slsa.dev/spec/v1.1/provenance#resolvedDependencies
* Included with `mode=min` and `mode=max`.

Defines all the external artifacts that were part of the build. The value
depends on the type of artifact:

- The URL of Git repositories containing source code for the image
- HTTP URLs if you are building from a remote tarball, or that was included
  using an `ADD` command in Dockerfile
- Any Docker images used during the build

The URLs to the Docker images will be in
[Package URL](https://github.com/package-url/purl-spec) format.

All the build materials will include the immutable checksum of the artifact.
When building from a mutable tag, you can use the digest information to
determine if the artifact has been updated compared to when the build ran.

```json
    "buildDefinition": {
      "resolvedDependencies": [
        {
          "uri": "pkg:docker/alpine@3.17?platform=linux%2Famd64",
          "digest": {
            "sha256": "8914eb54f968791faf6a8638949e480fef81e697984fba772b3976835194c6d4"
          }
        },
        {
          "uri": "https://github.com/moby/buildkit.git#refs/tags/v0.11.0",
          "digest": {
            "sha1": "4b220de5058abfd01ff619c9d2ff6b09a049bea0"
          }
        },
        ...
      ],
      ...
    }
```

### `runDetails.builder.id`

* Ref: https://slsa.dev/spec/v1.1/provenance#builder.id
* Included with `mode=min` and `mode=max`.

The field is set to the URL of the build, if available.

```json
    "runDetails": {
      "builder": {
        "id": "https://github.com/docker/buildx/actions/runs/3709599520"
        ...
      },
      ...
    }
```

> [!NOTE]
> This value can be set using the `builder-id` attestation parameter.

### `runDetails.metadata.invocationID`

* Ref: https://slsa.dev/spec/v1.1/provenance#invocationId
* Included with `mode=min` and `mode=max`.

Unique identifier for the build invocation. When building a multi-platform image
with a single build request, this value will be the shared by all the platform
versions of the image.

```json
    "runDetails": {
      "metadata": {
        "invocationID": "rpv7a389uzil5lqmrgwhijwjz",
        ...
      },
      ...
    }
```

### `runDetails.metadata.startedOn`

* Ref: https://slsa.dev/spec/v1.1/provenance#startedOn
* Included with `mode=min` and `mode=max`.

Timestamp when the build started.

```json
    "runDetails": {
      "metadata": {
        "startedOn": "2021-11-17T15:00:00Z",
        ...
      },
      ...
    }
```

### `runDetails.metadata.finishedOn`

* Ref: https://slsa.dev/spec/v1.1/provenance#finishedOn
* Included with `mode=min` and `mode=max`.

Timestamp when the build finished.

```json
    "runDetails": {
      "metadata": {
        "finishedOn": "2021-11-17T15:01:00Z",
        ...
      },
    }
```

### `runDetails.metadata.buildkit_metadata`

* Ref: https://slsa.dev/spec/v1.1/provenance#extension-fields
* Partially included with `mode=min`.

This extension field defines BuildKit-specific additional metadata that is not
part of the SLSA provenance spec.

```json
    "runDetails": {
      "metadata": {
        "buildkit_metadata": {
          "source": {...},
          "layers": {...},
          "vcs": {...},
        },
        ...
      },
    }
```

#### `source`

Only included with `mode=max`.

Defines a source mapping of LLB build steps, defined in the
`buildDefinition.internalParameters.buildConfig.llbDefinition` field, to their
original source code (for example, Dockerfile commands). The `source.locations`
field contains the ranges of all the Dockerfile commands ran in an LLB step.
`source.infos` array contains the source code itself. This mapping is present
if the BuildKit frontend provided it when creating the LLB definition.

#### `layers`

Only included with `mode=max`.

Defines the layer mapping of LLB build step mounts defined in
`buildDefinition.internalParameters.buildConfig.llbDefinition` to the OCI
descriptors of equivalent layers. This mapping is present if the layer data was
available, usually when attestation is for an image or if the build step pulled
in image data as part of the build.

#### `vcs`

Included with `mode=min` and `mode=max`.

Defines optional metadata for the version control system used for the build. If
a build uses a remote context from Git repository, BuildKit extracts the details
of the version control system automatically and displays it in the
`buildDefinition.externalParameters.configSource` field. But if the build uses
a source from a local directory, the VCS information is lost even if the
directory contained a Git repository. In this case, the build client can send
additional `vcs:source` and `vcs:revision` build options and BuildKit will add
them to the provenance attestations as extra metadata. Note that, contrary to
the `buildDefinition.externalParameters.configSource` field, BuildKit doesn't
verify the `vcs` values, and as such they can't be trusted and should only be
used as a metadata hint.

### `runDetails.metadata.buildkit_hermetic`

* Ref: https://slsa.dev/spec/v1.1/provenance#extension-fields
* Included with `mode=min` and `mode=max`.

This extension field is set to true if the build was hermetic and did not access
the network. In Dockerfiles, a build is hermetic if it does not use `RUN`
commands or disables network with `--network=none` flag.

```json
    "runDetails": {
      "metadata": {
        "buildkit_hermetic": true,
        ...
      },
    }
```

### `runDetails.metadata.buildkit_completeness`

* Ref: https://slsa.dev/spec/v1.1/provenance#extension-fields
* Included with `mode=min` and `mode=max`.

This extension field defines if the provenance information is complete. It is
similar to `metadata.completeness` field in SLSA v0.2.

`buildkit_completeness.request` is true if all the build arguments are included
in the `buildDefinition.externalParameters.request` field. When building with
`min` mode, the build arguments are not included in the provenance information
and request is not complete. Request is also not complete on direct LLB builds
that did not use a frontend.

`buildkit_completeness.resolvedDependencies` is true if
`buildDefinition.resolvedDependencies` field includes all the dependencies of
the build. When building from un-tracked source in a local directory, the
dependencies are not complete, while when building from a remote Git repository
all dependencies can be tracked by BuildKit and
`buildkit_completeness.resolvedDependencies` is true.

```json
    "runDetails": {
      "metadata": {
        "buildkit_completeness": {
          "request": true,
          "resolvedDependencies": true
        },
        ...
      },
    }
```

### `runDetails.metadata.buildkit_reproducible`

* Ref: https://slsa.dev/spec/v1.1/provenance#extension-fields
* Included with `mode=min` and `mode=max`.

This extension field defines if the build result is supposed to be byte-by-byte
reproducible. It is similar to `metadata.reproducible` field in SLSA v0.2. This
value can be set by the user with the `reproducible=true` attestation parameter. 

```json
    "runDetails": {
      "metadata": {
        "buildkit_reproducible": false,
        ...
      },
    }
```

## SLSA v0.2

### `builder.id`

* Ref: https://slsa.dev/spec/v0.2/provenance#builder.id
* Included with `mode=min` and `mode=max`.

The field is set to the URL of the build, if available.

```json
    "builder": {
      "id": "https://github.com/docker/buildx/actions/runs/3709599520"
    },
```

> [!NOTE]
> This value can be set using the `builder-id` attestation parameter.

### `buildType`

* Ref: https://slsa.dev/spec/v0.2/provenance#buildType
* Included with `mode=min` and `mode=max`.

The `buildType` field is set to `https://mobyproject.org/buildkit@v1` and can be
used to determine the structure of the provenance content.

```json
    "buildType": "https://mobyproject.org/buildkit@v1",
```

### `invocation.configSource`

* Ref: https://slsa.dev/spec/v0.2/provenance#invocation.configSource
* Included with `mode=min` and `mode=max`.

Describes the config that initialized the build.

```json
    "invocation": {
      "configSource": {
        "uri": "https://github.com/moby/buildkit.git#refs/tags/v0.11.0",
        "digest": {
          "sha1": "4b220de5058abfd01ff619c9d2ff6b09a049bea0"
        },
        "entryPoint": "Dockerfile"
      },
      ...
    },
```

For builds initialized from a remote context, like a Git or HTTP URL, this
object defines the context URL and its immutable digest in the `uri` and
`digest` fields. For builds using a local frontend, such as a Dockerfile, the
`entryPoint` field defines the path for the frontend file that initialized the
build (`filename` frontend option).

### `invocation.parameters`

* Ref: https://slsa.dev/spec/v0.2/provenance#invocation.parameters
* Partially included with `mode=min`.

Describes build inputs passed to the build.

```json
    "invocation": {
      "parameters": {
        "frontend": "gateway.v0",
        "args": {
          "build-arg:BUILDKIT_CONTEXT_KEEP_GIT_DIR": "1",
          "label:FOO": "bar",
          "source": "docker/dockerfile-upstream:master",
          "target": "release"
        },
        "secrets": [
          {
            "id": "GIT_AUTH_HEADER",
            "optional": true
          },
          ...
        ],
        "ssh": [],
        "locals": []
      },
      ...
    },
```

The following fields are included with both `mode=min` and `mode=max`:

- `locals` lists any local sources used in the build, including the build
  context and frontend file.
- `frontend` defines type of BuildKit frontend used for the build. Currently,
  this can be `dockerfile.v0` or `gateway.v0`.
- `args` defines the build arguments passed to the BuildKit frontend.

  The keys inside the `args` object reflect the options as BuildKit receives
  them. For example, `build-arg` and `label` prefixes are used for build
  arguments and labels, and `target` key defines the target stage that was
  built. The `source` key defines the source image for the Gateway frontend, if
  used.

The following fields are only included with `mode=max`:

- `secrets` defines secrets used during the build. Note that actual secret
  values are not included.
- `ssh` defines the ssh forwards used during the build.

### `invocation.environment`

* Ref: https://slsa.dev/spec/v0.2/provenance#invocation.environment
* Included with `mode=min` and `mode=max`.

```json
    "invocation": {
      "environment": {
        "platform": "linux/amd64"
      },
      ...
    },
```

The only value BuildKit currently sets is the `platform` of the current build
machine. Note that this is not necessarily the platform of the build result that
can be determined from the `in-toto` subject field.

### `materials`

* Ref: https://slsa.dev/spec/v0.2/provenance#materials
* Included with `mode=min` and `mode=max`.

Defines all the external artifacts that were part of the build. The value
depends on the type of artifact:

- The URL of Git repositories containing source code for the image
- HTTP URLs if you are building from a remote tarball, or that was included
  using an `ADD` command in Dockerfile
- Any Docker images used during the build

The URLs to the Docker images will be in
[Package URL](https://github.com/package-url/purl-spec) format.

All the build materials will include the immutable checksum of the artifact.
When building from a mutable tag, you can use the digest information to
determine if the artifact has been updated compared to when the build ran.

```json
    "materials": [
      {
        "uri": "pkg:docker/alpine@3.17?platform=linux%2Famd64",
        "digest": {
          "sha256": "8914eb54f968791faf6a8638949e480fef81e697984fba772b3976835194c6d4"
        }
      },
      {
        "uri": "https://github.com/moby/buildkit.git#refs/tags/v0.11.0",
        "digest": {
          "sha1": "4b220de5058abfd01ff619c9d2ff6b09a049bea0"
        }
      },
      ...
    ],
```

### `buildConfig`

* Ref: https://slsa.dev/spec/v0.2/provenance#buildConfig
* Only included with `mode=max`.

Defines the build steps performed during the build.

BuildKit internally uses LLB definition to execute the build steps. The LLB
definition of the build steps is defined in `buildConfig.llbDefinition` field.

Each LLB step is the JSON definition of the
[LLB ProtoBuf API](https://github.com/moby/buildkit/blob/v0.10.0/solver/pb/ops.proto).
The dependencies for a vertex in the LLB graph can be found in the `inputs`
field for every step.

```json
  "buildConfig": {
    "llbDefinition": [
      {
        "id": "step0",
        "op": {
          "Op": {
            "exec": {
              "meta": {
                "args": [
                  "/bin/sh",
                  "-c",
                  "go build ."
                ],
                "env": [
                  "PATH=/go/bin:/usr/local/go/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
                  "GOPATH=/go",
                  "GOFLAGS=-mod=vendor",
                ],
                "cwd": "/src",
              },
              "mounts": [...]
            }
          },
          "platform": {...},
        },
        "inputs": [
          "step8:0",
          "step2:0",
        ]
      },
      ...
    ]
  },
```

### `metadata.buildInvocationId`

* Ref: https://slsa.dev/spec/v0.2/provenance#buildInvocationId
* Included with `mode=min` and `mode=max`.

Unique identifier for the build invocation. When building a multi-platform image
with a single build request, this value will be the shared by all the platform
versions of the image.

```json
    "metadata": {
      "buildInvocationID": "rpv7a389uzil5lqmrgwhijwjz",
      ...
    },
```

### `metadata.buildStartedOn`

* Ref: https://slsa.dev/spec/v0.2/provenance#buildStartedOn
* Included with `mode=min` and `mode=max`.

Timestamp when the build started.

```json
    "metadata": {
      "buildStartedOn": "2021-11-17T15:00:00Z",
      ...
    },
```

### `metadata.buildFinishedOn`

* Ref: https://slsa.dev/spec/v0.2/provenance#buildFinishedOn
* Included with `mode=min` and `mode=max`.

Timestamp when the build finished.

```json
    "metadata": {
      "buildFinishedOn": "2021-11-17T15:01:00Z",
      ...
    },
```

### `metadata.completeness`

* Ref: https://slsa.dev/spec/v0.2/provenance#metadata.completeness
* Included with `mode=min` and `mode=max`.

Defines if the provenance information is complete.

`completeness.parameters` is true if all the build arguments are included in the
`parameters` field. When building with `min` mode, the build arguments are not
included in the provenance information and parameters are not complete.
Parameters are also not complete on direct LLB builds that did not use a
frontend.

`completeness.environment` is always true for BuildKit builds.

`completeness.materials` is true if `materials` field includes all the
dependencies of the build. When building from un-tracked source in a local
directory, the materials are not complete, while when building from a remote Git
repository all materials can be tracked by BuildKit and `completeness.materials`
is true.

```json
    "metadata": {
      "completeness": {
        "parameters": true,
        "environment": true,
        "materials": true
      },
      ...
    },
```

### `metadata.reproducible`

* Ref: https://slsa.dev/spec/v0.2/provenance#metadata.reproducible
* Included with `mode=min` and `mode=max`.

Defines if the build result is supposed to be byte-by-byte reproducible. This
value can be set by the user with the `reproducible=true` attestation parameter.

```json
    "metadata": {
      "reproducible": false,
      ...
    },
```

### `metadata.https://mobyproject.org/buildkit@v1#hermetic`

Included with `mode=min` and `mode=max`.

This extension field is set to true if the build was hermetic and did not access
the network. In Dockerfiles, a build is hermetic if it does not use `RUN`
commands or disables network with `--network=none` flag.

```json
    "metadata": {
      "https://mobyproject.org/buildkit@v1#hermetic": true,
      ...
    },
```

### `metadata.https://mobyproject.org/buildkit@v1#metadata`

Partially included with `mode=min`.

This extension field defines BuildKit-specific additional metadata that is not
part of the SLSA provenance spec.

```json
    "metadata": {
      "https://mobyproject.org/buildkit@v1#metadata": {
        "source": {...},
        "layers": {...},
        "vcs": {...},
      },
      ...
    },
```

#### `source`

Only included with `mode=max`.

Defines a source mapping of LLB build steps, defined in the
`buildConfig.llbDefinition` field, to their original source code (for example,
Dockerfile commands). The `source.locations` field contains the ranges of all
the Dockerfile commands ran in an LLB step. `source.infos` array contains the
source code itself. This mapping is present if the BuildKit frontend provided it
when creating the LLB definition.

#### `layers`

Only included with `mode=max`.

Defines the layer mapping of LLB build step mounts defined in
`buildConfig.llbDefinition` to the OCI descriptors of equivalent layers. This
mapping is present if the layer data was available, usually when attestation is
for an image or if the build step pulled in image data as part of the build.

#### `vcs`

Included with `mode=min` and `mode=max`.

Defines optional metadata for the version control system used for the build. If
a build uses a remote context from Git repository, BuildKit extracts the details
of the version control system automatically and displays it in the
`invocation.configSource` field. But if the build uses a source from a local
directory, the VCS information is lost even if the directory contained a Git
repository. In this case, the build client can send additional `vcs:source` and
`vcs:revision` build options and BuildKit will add them to the provenance
attestations as extra metadata. Note that, contrary to the
`invocation.configSource` field, BuildKit doesn't verify the `vcs` values, and
as such they can't be trusted and should only be used as a metadata hint.
