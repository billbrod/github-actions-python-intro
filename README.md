# github-actions-python-intro

Quick introduction for using github actions for python objects.

The readme will walk through some information, and then the included `##-ci*.yml`
files show workflows of increasing complexity. To use these for your own
project, copy into the `.github/workflows/` folder (creating it if it doesn't
exist) and modify as needed.

## Background

Github actions are one way of performing [continuous
integration](https://en.wikipedia.org/wiki/Continuous_integration) (CI), which
runs automatic tests regularly ("continuous") before making any changes to the
main codebase ("integration"). There are other platforms for doing so (Travis
CI, Circle CI, etc.), but Github actions has the benefit of being built into
Github. They all work in a fairly similarly: define a bunch of steps to run and
when you want them to run, generally in a `.yml` file (see
[here](https://en.wikipedia.org/wiki/YAML) for more details). The platforms
described above provide servers ([github-hosted
runners](https://docs.github.com/en/actions/using-github-hosted-runners/about-github-hosted-runners#supported-runners-and-hardware-resources)
as github calls them) to run the tests on, which are free for open source
projects (though they have limited resources and never include GPUs). Many of
these services can also be self-hosted, and thus run on your own server (e.g.,
Flatiron uses [Jenkins](https://www.jenkins.io/), which allows us to use
Flatiron clusters for GPUs).

### Why would you do this?

CI commonly includes tests and
[linters](https://en.wikipedia.org/wiki/Lint_(software)) to make sure the code
runs and meets a desired code style.

For software development, CI is often used to make sure that any new changes
don't break old code and to enforce a uniformity of style, which makes
understanding the code easier. It is thus *incredibly* useful (I would argue,
necessary).

For research software, it is not as obviously necessary. However, I've used CI
for research code to ensure that my code is installable and some basic
functionality runs. This means I know someone else will be able to use it (e.g.,
that I included all necessary dependencies) and means I'm aware when a change in
one of my dependencies (or my own code!) breaks something basic.

It can also be used in more complex cases. The [Journal of Open Source
Software](https://joss.theoj.org/) uses it to build papers from markdown, using
the Github issues framework to centralize the review process. Here's
[fRAT](https://github.com/openjournals/joss-reviews/issues/5200), a package for
analyzing functional MRI data that I reviewed.

I've heard of people using it to generate all the figures for their scientific
papers, but my analyses have always required too much time and memory to be
feasible for this.

## Basic set up

Okay, all that aside, how does one do this? Github's
[documentation](https://docs.github.com/en/actions) on its actions are quite
thorough, and a bit intimidating. As CI's generally used by software engineers,
most of what you'll find is targeted at them, rather than scientists, and can be
frustrating to approach. Here, we'll provide some basic examples that are
hopefully useful.

Let's look at the first workflow, `00-ci.yml`. This simple workflow runs every
time we push to the repository and checks that we're installable with `conda`
(requires an `environment.yml` file) and `pip` (requires `setup.py` or something
similar).

Several things are going on here:
- we specify the name of the workflow and how often the CI runs at the top.
- we have two separate jobs, which will run in parallel. `yml` files are
  hierarchical, based on indent level. `check_pip_install` and
  `check_conda_install` are both one level under `jobs` and thus are independent
  jobs.
- Each job specifies the operating system it runs on (in this case,
  `ubuntu-latest`) and then the steps required.
- The steps are enumerated using `-` and run in sequence! If one of them fails,
  the whole job will fail and quit out.
- The three steps here (`checkout`, `setup-python` / `setup-conda`, and `Install
  dependencies` / `Create environment`) will be required for every job you
  create: `checkout` checks out your repo from github, the following sets up
  either python or conda (which includes python) to run your code, and the third
  actually installs it. Note this mirrors what your users will do!
- Note that the setup steps include some configuration including, critically,
  the python version. We'll return to that later.

## Next steps

