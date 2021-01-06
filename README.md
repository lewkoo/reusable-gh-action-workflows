# Companion repository to blog post - [Re-usable and maintainable GitHub Action workflows for multiple repositories](www.livanchuk.com/dev-blog/2021/1/6/re-usable-and-maintainable-github-action-workflows-for-multiple-repositories)

This is a companion repository, containing example workflow and Dockerfiles to better illustrate the
issues and solutions described in the article.

Blog post is duplicated below for your convenience.

## Intro

While I will talk about some particulars which relate to our particular situation, the CI/CD techniques I will outline in this post will be suitable to any individual/organization that maintains a set of GitHub repositories and wants to have a *single*, *unified* GitHub Actions workflow to do whatever they want or already do with GitHub Actions. The only constants here are, basically:

* GitHub for repository hosting
* GitHub Actions for CI/CD

That's it, everything else is optional/configurable/interchangeable, etc.

This blog post also has a [companion repo](https://github.com/lewkoo/reusable-gh-action-workflows), showcasing some examples.  

Also, a word of caution: these kinds of things - devops, CI/CD, etc. - aren't my specialty, more like a side job for winter holiday season when the rest of tasks slow down. So take everything you read with a grain of salt and I would welcome any comments/additions one might have.

Let's dive in, shall we?

## Background

To give you some context, our GitHub organization maintains a lengthy list of private repositories, spanning all kinds of projects which, at some point, end up on some production systems, running either Ubuntu 16.04 (xenial) or Ubuntu 18.04 (bionic). These repositories share a build system and step to build and release our code are virtually the same for all of them.

A long time ago in a galaxy far far away, as naive as we were, we started with lengthy, manual commands to generate a Debian file and then some manual work to Slack/email those files to a person directly in charge of said production hardware to install the updated package. Of course, the process was often repeated many times if there was a need to make a quick-fix kind of change _(I still have nightmares about this today)_

Then, by the time the Empire struck back, our team had dedicated a private server to distribute these Debian (.deb) files to the production hardware and other developers who depended on it. This was a good solution in terms of distribution, but the logistics of testing/building/uploading the file to the server were still in same stage of disrepair as the Millennium Falcon was in Episode 5 - flying, but without a hyperdrive. We needed and wanted a hyperdrive to escape the clutches of the ~~Empire~~ *lots and lots* of manual labour and asking people with SSH tunnel to the APT repository server to manually upload a new version of the Debian file.

Finally, a Jedi returned and said Jedi bravely noticed a tab called *Actions* on every repository. Thus, a new era has begun in the galaxy, an era which promised a bright future where all laborious, manual tasks will be automated and quick-fixes can be tested, build and delivered to the production hardware in minutes, not hours. Automatically. By computers, using magic, no less.

## Naive Version 1

As the developer was just a padavan learner of GitHub Actions, the first GitHub Action (`alias 'GHA' 'GitHub Action'`) workflow was a collection of snippets from various tutorials and docs, trying to understand the ways of the GHA workflows. Essentially, it achieved this:
_(note,_ 🤬 _denotes a source of fatal-error in this initial attempt. The amount of such emojis indicates the gravity of the error)_  

1. Set up [strategy matrix](https://docs.github.com/en/free-pro-team@latest/actions/reference/workflow-syntax-for-github-actions#jobsjob_idstrategymatrix) to run on base-bones (🤬) Ubuntu 16.04 and 18.04 Docker containers.
2. Configure locales (🤬)
3. Install a *ton* (🤬🤬🤬) of _basic_ dependency packages in the container. (we are talking `wget` kind of stuff here) 
4. Upgrade *git* (of all things) so that `actions/checkout` would not default to REST API cloning method via a zip file and would actually create a `.git` directory. _(I still have nightmares about this today)_
5. Clone target repo (duh)
6. Create a workspace (🤬)
7. Clone repositories which are dependencies for the target repository
8. Install/upgrade dependency packages
9. Run tests
10. Build Debian file
11. Create release on GitHub, upload CHANGELOG to it
12. Upload the Debian file to the APT repository server
13. Trigger an update there
14. Check if the Debian file installs of clean container
15. Notify us in Slack

This workflow *worked* but it had these flaws:

* Workflow file itself was long and hard to read
* Workflow performed *a lot* of *repetitive* steps on every run that had *nothing* to do with the tasks it actually needs to do on every run.

While the things above are manageable or, at least, can be paid for by billable time, this GHA workflow failed on one crucial element:

*A change is this workflow needed to be propagated across all repositories* and then *tested on all of them individually*.

In short, it didn't achieve SPM, or Single Point of Maintenance.

See the [companion repo](https://github.com/lewkoo/reusable-gh-action-workflows) for a workflow where these mistakes are highlighted.

## Version 2

Let's briefly summarize the fatal-errors of the version 1:

* Duplicate container configuration steps
* Long, unreadable workflow file
* Workflow duplication across many repositories
* Difficult to change/update/upgrade in the future.

Before discussing our solution, I would like to talk about a potential solution which we rejected but it might fit into some use cases.

### Rejected idea: Workflows as submodule

One idea we had is to place our single workflow file into a new repository and register it as a submodule in all our production repositories.

This would give us SPM, or Single Point of Maintenance, but it would also come with limitations. 

While it is true that most build and release tasks are identical across all repositories, there are cases where steps are either skipped or performed in a slightly different way. 

Just having a single workflow file will limit us it terms of workflow customization with respect to the needs of every individual repo. For example, some repos do not have tests so they can skip that step.

Furthermore, we will not be able to add custom workflow steps which relate only to a single repo.

Thus, our solution was two fold:

1) Use of custom Docker images and
2) Use of a custom GitHub Action

## Docker Images: pre-baked build environment

We were already using Docker support built into GitHub Actions with its [container](https://docs.github.com/en/free-pro-team@latest/actions/reference/workflow-syntax-for-github-actions#jobsjob_idcontainer) directive as we needed to build for two (and, in the future, potentially more) systems, but our Version 1 workflow used bare-bones `ubuntu` images and needed to install a lot of dependencies on every workflow run.

Fortunately, performance penalty was relatively low - around 20 seconds of repeated billable time for every run for every repository. However, the main issue was the lack of SPM - if we needed to add a new dependency package we would need to visit every repository with this change.

Here is a snippet of some example steps which can and should be refactored into a custom docker image:

```yml
# This step installs some basic dependency packages which are needed down the line
# They should be installed when building a docker image
- name: Update git & install additional dependencies
run: |
    # Update package cache
    apt-get update
    # Install build agent and some other packages
    apt-get install -yq software-properties-common apt-utils debhelper build-essential wget
    # Upgrade git
    add-apt-repository -y -u ppa:git-core/ppa
    # Upgrade git
    apt-get install -yq git
shell: bash
```

See `repeated_conf_steps.yml` in the [companion repo](https://github.com/lewkoo/reusable-gh-action-workflows) for an example workflow.

Our solution was to create a new repo with our custom Docker images.
We then migrated *all* such configuration/install steps into a single `Dockerfile`, setting the ubuntu distro as an argument, like this:

```dockerfile
ARG _DISTRO
# Base off the basic ubuntu images
FROM ubuntu:$_DISTRO
# Install required packages and upgrade git
RUN dpkg-reconfigure debconf --frontend=noninteractive && \ 
    apt-get update && \ 
    apt-get install -yq --no-install-recommends software-properties-common apt-utils debhelper build-essential wget && \
    add-apt-repository -y -u ppa:git-core/ppa && \
    apt-get install -yq git && \
    rm -rf /var/lib/apt/lists/* && \
    apt-get purge --auto-remove && \
    apt-get clean
```

We then built and published these images to our private account on [Github container registry, or GHCR for short](https://docs.github.com/en/free-pro-team@latest/packages/guides/about-github-container-registry). You can, of course, use any other registry.

Maintaining these Docker images is easy and we also added a workflow to build and publish them to the registry, [based off this example](https://docs.docker.com/ci-cd/github-actions/). Any change to Docker images gets build and released in minutes and is therefore immediately available to all workflows that run on updated containers.

> Note on self-hosted runners

Combined with self-hosted runners and local Docker cache, you get a near-instant container initialization bonus, so that's nice.

> Note on secret data

We _did not_ store any sensitive/private data inside these container images, even though they were stored in a _private_ container registry.

All secret data was loaded into container upon startup by the workflow itself. Before you say "but that isn't SPM" - and you'd be right - secret data isn't something that gets added frequently, so we included as many things as we could think of, even if some of them aren't used now.

To store our secrets we used GitHub's secrets features on the organization level. This is a great option because we can update a secret value (say, PAT value changes at some point) and that would propagate to all repositories managed by our organization.

Secrets were loaded as pain environment values to be used inside a container later, like this:

```yml
container:
    # ...
    env:
        GH_USER: ${{ secrets.GH_USER }}
        GH_TOKEN: ${{ secrets.GH_TOKEN }}
    # ...
```

## Custom GitHub Action: refactoring common workflow steps

With pre-build install and configure steps pre-baked into a Docker image and out of the way, we were ready to implement workflow steps which are actually unique for every workflow run.

However, we needed to refactor them into a single place, preferably another repository, where we could maintain these steps. [While it is possible to trigger one workflow from another](https://blog.marcnuri.com/triggering-github-actions-across-different-repositories/), we chose instead to use a [custom GitHub Action](https://docs.github.com/en/free-pro$$-team@latest/actions/creating-actions).

First hurdle we needed to overcome was to store this custom action in a _private repository_ and still be able to use it. GitHub does not provide an explicit way of doing this, but it can be easily achieved with a `checkout` step, like this:

```yml
# Checkout the repo being built
  - uses: actions/checkout@v2
  # Checkout the action repo
  - uses: actions/checkout@v2
    with:
      repository: example-org-name/custom-action
      token: ${{ secrets.GH_TOKEN }}
      path: .github/actions/custom-action
```

After this simple step, custom private action can be run. GitHub Actions supports three types of custom actions:

1. Docker container action
2. JavaScript action
3. Composite run steps action

Exact type to use would depend on your needs. Since we needed to run a whole bunch of `bash` commands and we were already inside a configured Docker container, we chose a composite run steps action.

Composite run steps action is a fancy way of saying that it runs bash commands in order, with the ability to have inputs and outputs. This last ability - I/O - was crucial to us. I won't and I can't go into details of the actual implementations, but we ended up with a custom action that could be used in a workflow like this:

```yml
# Run the action according to the configuration
- name: 'Perform common steps'
  uses: ./.github/actions/custom-action
  id: custom-action
  with:
    step_1: true
    step_2: false
    step_3: true
    step_4: false
```

Each and every repository in our organization can pick and choose what steps it needs to execute and even under what conditions. For example, we might not want to run `step_3` on `feature` branches, but it is a must for all `release` branches.

Outputs of this action contained a JSON with results for each step. This data was then used for notifying us about anything that we want to know about regarding our workflow runs - although our notification procedures were also refactored into this custom action as just another step.

## Conclusions

With the optimizations and refactoring described above, we have achieved quite a few improvements:

1. Workflow files inside repositories were reduced from ~400 lines to ~40.
2. We achieved same work with less time - some builds with lengthy tests were reduced from 30 - 40 minutes to just 10.
3. We have a drop-in workflow file, interchangeable with all our repositories but still configurable if we need to add / remove some steps.
4. In addition to pre-building our own custom Docker containers, we ended up with, well, our own Docker containers containing a minimum setup which we can use later for other needs, like, running our code on production hardware inside Docker.
5. We achieved Single Point of Maintenance of almost all of our workflow actions - we can modify either a Docker image if we need to pre-install/configure something or a custom action itself if we need to change the behavior of the workflow.

### Acknowledgements

* [GitHub logo](https://github.com/logos) is used as allowed by GitHub - `Use the Octocat or GitHub logo in a blog post or news article about GitHub`.  
