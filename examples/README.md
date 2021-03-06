

### [`src/BUILD`](src/BUILD)
```python
load("//terraform:def.bzl", "terraform_module", "terraform_workspace")
load("//experimental:k8s.bzl", "terraform_k8s_manifest")
load("@io_bazel_rules_docker//python:image.bzl", "py_image")

# First we build a docker image from our app's source
py_image(
    name = "py_image",
    srcs = ["server.py"],
    main = "server.py",
)

# Next, we create a Kubernetes Deployment which references the
# image we just created. Also note:
# - We have specified an 'image_chroot' which allows us to change where
#   the image is published
# - The actual image_chroot is determined at build time based on what
#   we've defined in '.bazelrc' (more on this later)
terraform_module(
    name = "hello-world_k8s",
    srcs = [
        "k8s.tf",
        "main.tf",
    ],
    embed = [":k8s-deployment"],
    visibility = ["//visibility:public"],
)

# We combine our terraform files with the Kubernetes Deployment
# to create a terraform module
terraform_module(
    name = "hello-world_k8s",
    srcs = [
        "k8s.tf",
        "main.tf",
    ],
    embed = [":k8s-deployment"],
    visibility = ["//visibility:public"],
)
```

### [`test/BUILD`](test/BUILD)
```python
load("//terraform:def.bzl", "terraform_integration_test", "terraform_workspace")

# We create a terraform workspace which uses the module we created and
# provides our module with any inputs it requires.
# - Running `bazel run :test-workspace` will run terraform init+apply
# - Running `bazel run :test-workspace.destroy` will tear down the workspace
terraform_workspace(
    name = "test-workspace",
    srcs = ["test_ws.tf"],
    modules = {
        "module": "//examples/src:hello-world_k8s",
    },
)

# Now that our workspace is up and running we can create an end-to-end
# test for it and quickly iterate on the test (without having to recreate
# all of the infrastructure) until we're happy with it. This test does require
# infrastructure though, so we tag it as 'manual' to exclude it from wildcard
# target patterns like `bazel test //...`
sh_test(
    name = "e2e_test",
    srcs = ["e2e.sh"],
    tags = ["manual"],
)

# ..But we still want an easy-to-run test that doesn't require manually spinning up
# infrastructure, so we create a 'terraform_integration_test' which will
# spin up the specified 'terraform_workspace', run the 'srctest' and then
# clean up with terraform destroy
terraform_integration_test(
    name = "e2e_integration_test",
    timeout = "short",
    srctest = ":e2e_test",
    tags = ["manual"],
    terraform_workspace = ":test-workspace",
)
```

### [`.bazelrc`](../.bazelrc)
> We've created & tested our module; now we're ready to publish it--but
what about $(IMAGE_CHROOT) from earlier? We configure that in the
workspace's `.bazelrc`. By default we upload locally, but when running
bazel with `--config=publish` we'll upload to a remote repository.
```
build         --define IMAGE_CHROOT=registry.kube-system.svc.cluster.local:80
build:publish --define IMAGE_CHROOT=index.docker.io/netchris
```

### [`release/BUILD`](release/BUILD)
```python
load("//experimental:publishing.bzl", "ghrelease", "ghrelease_assets", "ghrelease_test_suite")

VERSION = "0.2"

# To make our terraform module available to others we configure a
# 'ghrelease' which will:
# - Run "preflight-checks" (dependant test suites, etc)
# - Push associated docker images
# - Generate release notes, including a changelog from the previous
#   version
# - Increment the current tag/version's patch version (or prerelease
#   version if the `--prerelease` flag is used)
# - Create a new GitHub Release with release notes and any extra docs
# - Attach any assets to the new Release
ghrelease(
    name = "release",
    version = VERSION,
    deps = [
        ":prerelease-tests",
        ":tf-modules",
    ],
)

ghrelease_assets(
    name = "tf-modules",
    bazel_flags = ["--config=publish"],
    data = [
        "//examples/src:hello-world_ecs",
        "//examples/src:hello-world_k8s",
    ],
)

ghrelease_test_suite(
    name = "prerelease-tests",
    tests = [
        "//examples/...",
        "//examples/test:k8s-e2e_integration_test",
    ],
)

```

Check out the [releases page](https://github.com/ceason/rules_terraform/releases) to see the published modules!
