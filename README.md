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
- Many of the arguments here are keywords. See the [github actions
  docs](https://docs.github.com/en/actions/learn-github-actions/understanding-github-actions)
  and the docs for specific actions (e.g.,
  [setup-python](https://github.com/actions/setup-python)) to understand them.

## Next steps

### Build matrix

In `00-ci.yml`, we ran our simple tests with python 3.9 on `ubuntu-latest`. What
if you want to test more OSs and python versions? You do that using the build
matrix, as demonstrated ina `01-ci-build-matrix.yml`.

- We've removed the `check_conda_install` job so can just focus on the build
  matrix here, but you can do the same thing for that job, if you'd like.
- If we look at the diff between the `check_pip_install` for these two files
  (`diff 00-ci.yml 01-ci-build-matrix.yml`), we see the following:

  ```diff
  21c7,11
  -     runs-on: ubuntu-latest
  ---
  +     runs-on: ${{ matrix.os }}
  +     strategy:
  +       matrix:
  +         os: [ubuntu-latest, macos-latest, windows-latest]
  +         python-version: [3.8, 3.9, '3.10']
  27c17
  -           python-version: 3.9
  ---
  +           python-version: ${{ matrix.python-version }}
  ```
  
- Instead of specifying `runs-on` and `python-version` directly, we're using
  some strange curly-bracket syntax. These are
  [contexts](https://docs.github.com/en/actions/learn-github-actions/contexts),
  which we're accessing usingGithub action's [expressions
  syntax](https://docs.github.com/en/actions/learn-github-actions/expressions).
  For our purposes, the use of the lists under `matrix` gives us a for loop over
  those values, which we access using `${{ matrix.os }}` / `${{
  matrix.python-version }}`.
- Github actions will generate all combinations here, so running this action
  will give us nine workflows which all run in parallel.

### Change frequency

So far, we've beeen running all our tests every time we push, which is probably
way too often. Let's change that with `02-ci-freq.yml`, which will build off of
`01-ci-build-matrix.yml`.

Running `diff 01-ci-build-matrix.yml 02-ci-freq.yml` highlights the following
differences at the top of the file:

```diff
3c3,9
-   push:
---
+   pull_request:
+     branches:
+     - main
+     - development
+   schedule:
+   - cron: 0 0 * * 0     # weekly
+   workflow_dispatch:
```

Now, instead of running everytime we push, we have three different triggers:
- `pull_request: branches:` Everytime we make a pull request with the specified
  branches (here, `main` and `development`), this workflow will run. This is
  useful when using [git flow](https://www.gitkraken.com/learn/git/git-flow) for
  software development to ensure all merged changes pass all tests.
- `schedule:` This specific line here specifies that we should run the workflow
  weekly (it follows the syntax of [cron](https://en.wikipedia.org/wiki/Cron), a
  standard unix utility for scheduling jobs; it can be cryptic but thankfully
  there are [tools](https://crontab.guru/) to help you generate them). This is
  useful for ensuring that a change in one of your dependencies doesn't randomly
  break your code.
- `workflow_dispatch:` This allows us to manually trigger the workflow, which
  can be handy.

These are the options I often use. See [github
docs](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#onpushpull_requestpull_request_targetpathspaths-ignore)
for more info on the possible options here.

### Tests, linters, etc.

So far, our test is super basic. What else can we do with CI? Basically,
whatever you want (with open source code). Let's look at `03-ci-tests.yml`,
which builds off of `01-ci-build-matrix.yml`.

Running `diff 01-ci-build-matrix.yml 03-ci-tests.yml` highlights the following
at the end of our code:

``` diff
6c6,30
-   check_pip_install:
---
+   lint:
+     runs-on: ubuntu-latest
+     steps:
+       - uses: actions/checkout@v3
+       - name: Install Python 3
+         uses: actions/setup-python@v4
+         with:
+           python-version: 3.9
+           cache: pip
+           cache-dependency-path: setup.py
+       - name: Install dependencies
+         run: |
+           # using the --upgrade and --upgrade-strategy eager flags ensures that
+           # pip will always install the latest allowed version of all
+           # dependencies, to make sure the cache doesn't go stale
+           pip install --upgrade --upgrade-strategy eager .
+           pip install black isort flake8
+       - name: Lint
+         run: |
+           black --check my_package/
+           isort --check my_package/
+           flake8 my_package/ --max-complexity 10
+ 
+   tests:
+     needs: lint
25a50,52
+           pip install pytest
+       - name: Run pytest
+         run: pytest tests/

```

Here, we've added a new step, `lint`, and changed `check_pip_install` to
`tests`.

- If you look at `lint` carefully, you can see that it's basically the same as
  our previous rules except:
  - we've gone back to a single OS and python version (because the outputs of
    this shouldn't depend on that)
  - we have a new `pip install` line, which installs the linters: `black`,
    `isort`, and `flake8`
  - we added a new step, `Lint`, which runs all three on a (non-existant)
    directory named `my_package/`.
- By passing the `--check` flag, any of these linters will cause the job to fail
  if our code doesn't meet their standards. This way we know our code looks like
  what we expect!
- Similarly, our `tests` step is the exact same as `check_pip_install` from
  before (including the use of the build matrix for OS and python) except:
  - we install [pytest](https://docs.pytest.org/en/7.3.x/), a nice testing
    framework for python.
  - we add a step, `Run pytest`, which runs `pytest` on a (non-existant)
    directory named `tests/`.
  - we added a `needs: lint` line. This means that the two jobs will not run in
    parallel, but that `tests` will wait for `lint` to succeed before running.
    This means we won't run tests if the linters don't pass.
- Similar to the linters, if any of the tests fail, the whole job will fail.

For both of these, we're just taking some arbitrary code that we can (and
should!) run locally and making sure it gets run everytime the workflow gets
triggered. Automating them in this way ensures all these checks are run whenever
necessary, reducing cognitive load on the developers.

But we could include any code here! See the `test_snakefile` and
`run_Freeman_check_notebook` from my
[foveated-metamers](https://github.com/billbrod/foveated-metamers/blob/main/.github/workflows/tests.yml#L35)
github (research code supporting a publication) for more complex examples. In
the first job, I run a basic
[snakemake](https://snakemake.readthedocs.io/en/stable/) command to make sure
that my automated installation and setup steps allow users to run snakemake, a
workflow manager I use in the project. In the second, I setup the environment
and run a notebook using `jupyter execute` to make sure anyone who downloads my
code can do the same.

It doesn't have to call external code either, you can simply include several
bash lines, as I do in the `no_extra_nblinks` job from
[plenoptic](https://github.com/LabForComputationalVision/plenoptic/blob/main/.github/workflows/ci.yml#L154).
Here, I'm just counting the number of `nblink` files that live under the
`docs/tutorials/` directory and ensuring that it's the same as the number of
`ipynb` files that live under the `examples/` directory (which is useful because
of how I build my docs). Any code that checks something and raises an error
status (e.g., `exit 1` in bash or `raise Exception` in python) if your
assumptions aren't met can be useful!

- Use in branch protection rules
