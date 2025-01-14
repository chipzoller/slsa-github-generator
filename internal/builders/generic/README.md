# Generation of SLSA3+ provenance for arbitrary projects

This document explains how to generate SLSA provenance for projects for which
there is no language or ecosystem specific builder available.

This can be done by adding an additional step to your existing Github Actions
workflow to call a [reusable
workflow](https://docs.github.com/en/actions/using-workflows/reusing-workflows)
to generate generic SLSA provenance. We'll call this workflow the "generic
workflow" from now on.

The generic workflow differs from ecosystem specific builders (like the [Go
builder](../go)) which build the artifacts as well as generate provenance. This
project simply generates provenance as a separate step in an existing workflow.

---

- [Project Status](#project-status)
- [Benefits of Provenance](#benefits-of-provenance)
- [Generating Provenance](#generating-provenance)
  - [Getting Started](#getting-started)
  - [Supported Triggers](#supported-triggers)
  - [Workflow Inputs](#workflow-inputs)
  - [Workflow Outputs](#workflow-outputs)
  - [Provenance Format](#provenance-format)
  - [Provenance Example](#provenance-example)

---

## Project Status

This project is currently under active development. The API could change while
approaching an initial release.

## Benefits of Provenance

Using the generic workflow will generate a non-forgeable attestation to the
artifacts' digests using the identity of the GitHub workflow. This can be used
to create a positive attestation to a software artifact coming from your
repository.

That means that once your users verify the artifacts they have downloaded they
can be sure that the artifacts were created by your repository's workflow and
haven't been tampered with.

## Generating Provenance

The generic workflow uses a Github Actions reusable workflow to generate the
provenance.

### Getting Started

To get started, you will need to add some steps to your current workflow. We
will assume you have an existing Github Actions workflow to build your project.

Add a step to your workflow after you have built your project to generate a
sha256 hash of your artifacts and base64 encode it.

Assuming you have a binary called `binary-linux-amd64` you can use the
`sha256sum` and `base64` commands to create the digest. Here we use the `-w0` to
output the encoded data on one line and make it easier to use as a Github Actions
output:

```shell
$ sha256sum artifact1 artifact2 ... | base64 -w0
```

This workflow expects the `base64-subjects` input to decode to a string conforming to the expected output of the `sha256sum` command. Specifically, the decoded output is expected to be comprised of a hash value followed by a space followed by the artifact name.

After you have encoded your digest, add a new job to call the reusable workflow.

```yaml
provenance:
  permissions:
    actions: read # Needed for detection of GitHub Actions environment.
    id-token: write # Needed for provenance signing and ID
    contents: read # Needed for API access
  uses: slsa-framework/slsa-github-generator/.github/workflows/generator_generic_slsa3.yml@v1.1.0
  with:
    base64-subjects: "${{ needs.build.outputs.digest }}"
```

Here's an example of what it might look like all together.

```yaml
jobs:
  # This step builds our artifacts, uploads them to the workflow run, and
  # outputs their digest.
  build:
    outputs:
      digest: ${{ steps.hash.outputs.digest }}
    runs-on: ubuntu-latest
    steps:
      - name: "build artifacts"
        run: |
          # These are some amazing artifacts.
          echo "foo" > artifact1
          echo "bar" > artifact2
      - name: "generate hash"
        shell: bash
        id: hash
        run: |
          # sha256sum generates sha256 hash for all artifacts.
          # base64 -w0 encodes to base64 and outputs on a single line.
          # sha256sum artifact1 artifact2 ... | base64 -w0
          echo "::set-output name=digest::$(sha256sum artifact1 artifact2 | base64 -w0)"
      - name: Upload artifact1
        uses: actions/upload-artifact@3cea5372237819ed00197afe530f5a7ea3e805c8 # v3.1.0
        with:
          name: artifact1
          path: artifact1
          if-no-files-found: error
          retention-days: 5
      - name: Upload artifact2
        uses: actions/upload-artifact@3cea5372237819ed00197afe530f5a7ea3e805c8 # v3.1.0
        with:
          name: artifact2
          path: artifact2
          if-no-files-found: error
          retention-days: 5

  # This step calls the generic workflow to generate provenance.
  provenance:
    needs: [build]
    permissions:
      actions: read
      id-token: write
      contents: read
    if: startsWith(github.ref, 'refs/tags/')
    uses: slsa-framework/slsa-github-generator/.github/workflows/generator_generic_slsa3.yml@v1.1.0
    with:
      base64-subjects: "${{ needs.build.outputs.digest }}"

  # This step creates a GitHub release with our artifacts and provenance.
  release:
    needs: [build, provenance]
    runs-on: ubuntu-latest
    if: startsWith(github.ref, 'refs/tags/')
    steps:
      - name: Download artifact1
        uses: actions/download-artifact@fb598a63ae348fa914e94cd0ff38f362e927b741 # v2.1.0
        with:
          name: artifact1
      - name: Download artifact2
        uses: actions/download-artifact@fb598a63ae348fa914e94cd0ff38f362e927b741 # v2.1.0
        with:
          name: artifact2
      - name: Download provenance
        uses: actions/download-artifact@fb598a63ae348fa914e94cd0ff38f362e927b741 # v2.1.0
        with:
          # The provenance step returns an output with the artifact name of
          # our provenance.
          name: ${{needs.provenance.outputs.attestation-name}}
      - name: Create release
        uses: softprops/action-gh-release@1e07f4398721186383de40550babbdf2b84acfc5 # v0.1.14
        with:
          files: |
            artifact1
            artifact2
            ${{needs.provenance.outputs.attestation-name}}
```

### Supported Triggers

The following [GitHub trigger events](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows) are fully supported and tested:

- `schedule`
- `push` (including new tags)
- `release`
- Manual run via `workflow_dispatch`

However, in practice, most triggers should work with the exception of
`pull_request`. If you would like support for `pull_request`, please tell us
about your use case on [issue
#358](https://github.com/slsa-framework/slsa-github-generator/issues/358). If
you have an issue in all other triggers please submit a [new
issue](https://github.com/slsa-framework/slsa-github-generator/issues/new/choose).

### Workflow Inputs

The [generic workflow](https://github.com/slsa-framework/slsa-github-generator/blob/main/.github/workflows/generator_generic_slsa3.yml) accepts the following inputs:

| Name              | Required | Description                                                                                                                        |
| ----------------- | -------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| `base64-subjects` | yes      | Artifact(s) for which to generate provenance, formatted the same as the output of sha256sum (SHA256 NAME\n[...]) and base64 encoded. The encoded value should decode to, for example: `90f3f7d6c862883ab9d856563a81ea6466eb1123b55bff11198b4ed0030cac86  foo.zip` |

### Workflow Outputs

The [generic workflow](https://github.com/slsa-framework/slsa-github-generator/blob/main/.github/workflows/generator_generic_slsa3.yml) produces the following outputs:

| Name               | Description                                |
| ------------------ | ------------------------------------------ |
| `attestation-name` | The artifact name of the signed provenance |

### Provenance Format

The project generates SLSA provenance with the following values.

| Name                         | Value                                                          | Description                                                                                                                                                                                                            |
| ---------------------------- | -------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `buildType`                  | `"https://github.com/slsa-framework/slsa-github-generator@v1"` | Identifies a generic GitHub Actions build.                                                                                                                                                                             |
| `metadata.buildInvocationID` | `"[run_id]-[run_attempt]"`                                     | The GitHub Actions [`run_id`](https://docs.github.com/en/actions/learn-github-actions/contexts#github-context) does not update when a workflow is re-run. Run attempt is added to make the build invocation ID unique. |

### Provenance Example

The following is an example of the generated proveanance. Provenance is
generated as an [in-toto](https://in-toto.io/) statement with a SLSA predicate.

```json
{
  "_type": "https://in-toto.io/Statement/v0.1",
  "predicateType": "https://slsa.dev/provenance/v0.2",
  "subject": [
    {
      "name": "binary-linux-amd64",
      "digest": {
        "sha256": "2e0390eb024a52963db7b95e84a9c2b12c004054a7bad9a97ec0c7c89d4681d2"
      }
    },
  ],
  "predicate": {
    "builder": {
      "id": "https://github.com/slsa-framework/slsa-github-generator/.github/workflows/generator_generic_slsa3.yml@refs/heads/main"
    },
    "buildType": "https://github.com/slsa-framework/slsa-github-generator@v1",
    "invocation": {
      "configSource": {
        "uri": "git+https://github.com/ianlewis/actions-test@refs/heads/main.git",
        "digest": {
          "sha1": "3b5dc7cf5c0fd71c5a74c6b16cae78d49e03857c"
        },
        "entryPoint": "SLSA provenance"
      },
      "parameters": {},
      "environment": {
        "github_actor": "ianlewis",
        "github_base_ref": "",
        "github_event_name": "workflow_dispatch",
        "github_event_payload": ...,
        "github_head_ref": "",
        "github_ref": "refs/heads/main",
        "github_ref_type": "branch",
        "github_run_attempt": "1",
        "github_run_id": "2093917134",
        "github_run_number": "19",
        "github_sha1": "3b5dc7cf5c0fd71c5a74c6b16cae78d49e03857c"
      }
    },
    "metadata": {
      "buildInvocationID": "2182400786-1",
      "completeness": {
        "parameters": true,
        "environment": false,
        "materials": false
      },
      "reproducible": false
    },
    "materials": [
      {
        "uri": "git+https://github.com/ianlewis/actions-test@refs/heads/main.git",
        "digest": {
          "sha1": "3b5dc7cf5c0fd71c5a74c6b16cae78d49e03857c"
        }
      }
    ]
  }
}
```
