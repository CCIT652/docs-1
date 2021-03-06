---
description: This article walks you through the process of managing SSH keys by using Semaphore's secrets to authenticate to private Git repositories.
---

# Private Dependencies

Dependency mangagers like Bundler, Yarn, and Go's module system allow
specifying dependencies from private Git repositories. This makes it
easier for teams to share code without requiring separate package
hosting. Authentication typically happens over SSH. It's possible
manage SSH keys using Semaphore's [secrets][] to authenticate to
private Git repositories. This article walks you through the process.

### Create the SSH key

You'll need to generate an SSH key and associate it directly with
the project or a user who has access to that project. First, generate
a new public/private key pair on your local machine:

``` bash
ssh-keygen -t rsa -f id_rsa_semaphoreci
```

### Add the SSH key

Next, connect the SSH key to the project or user. Github [Deploy Keys][]
are the easiest way to grant access to a single project.

If you need to fetch gems from multiple repositories follow these steps:

- Create a [machine user][machine-user] with access to the repositories.
- Create a new SSH key pair for the purpose of automation.
- Add the private SSH key to the project as a secret.
- Add the public SSH key to the user settings for the machine user.

### Create the secret

Now GitHub is configured with the public key. The next step is
configure your Semaphore pipeline to use the private key. We'll use
[secret files][secrets] for this.

Use the sidebar in the web UI or `sem` CLI to create a new secret
from the existing private key in `id_rsa_semaphoreci` on your local
machine:

``` bash
sem create secret private-repo --file id_rsa_semaphoreci:/home/semaphore/.ssh/id_rsa_semaphoreci
```

This will create the file `~/.ssh/id_rsa_semaphoreci` in your Semaphore jobs.

Note: on macOS the home directory is `/Users/semaphore`.

### Use the secret in your pipeline

The last step is to add the `private-repo` secret to your Semaphore pipeline.
This will make the private key file available for use with `ssh-add`.  Here's an
example:

``` yaml
blocks:
  - name: "Test"
    task:
      secrets:
        # Mount the secret:
        - name: private-repo
      prologue:
        commands:
          - checkout
          # Correct premissions since they are too open by default:
          - chmod 0600 ~/.ssh/id_rsa_semaphoreci
          # Add the key to the ssh agent:
          - ssh-add ~/.ssh/id_rsa_semaphoreci
          # Now bundler/yarn/etc are able to pull private dependencies:
          - bundle install
      jobs:
        - name: Test
          commands:
            - rake test
```

That's all there is to it. You can use the approach to add more deploy
keys to the `private-repo` secret to cover more projects and reuse the
secret across other projects.

[secrets]: https://docs.semaphoreci.com/guided-tour/environment-variables-and-secrets/#storing-files-in-secrets
[deploy keys]: https://developer.github.com/v3/guides/managing-deploy-keys/
[machine-user]: https://developer.github.com/v3/guides/managing-deploy-keys/#machine-users
