---
layout: post
title: "Python package distribution can be easy"
date: 2020-01-26 11:26:00 +0100
categories: python
tags: python package distribution
permalink: python-package-distribution-can-be-easy
---

There have been so [many blog posts and presentations about Python packaging](https://www.youtube.com/watch?v=AQsZsgJ30AE&feature=youtu.be&t=50) that I'm reluctant to write what could be interpreted as _yet another one_, but I'll give it a go. Why? Because up until recently, I was working on several interconnected libraries, each of which had a more ridiculous build process than the next.

Needless to say, each had its own deployment procedure. Some were documented, others were left to the author's interpretation. Dozens of commands to execute. Some had to be built locally because the CI was apparently "too low on resources". All of this for pure Python packages. No C extensions. It's like saying "our machine doesn't have the resources to create a ZIP file".

Not to mention libraries that had themselves listed as build dependencies, or `setup.py` reading the virtual environment's environment variables to determine which version of a particular dependency should be installed. Or git branching workflows so complicated that made resolving merge conflicts a routine, with the changelog being the worst of it.

If you're a pragmatic developer who has better things to do with its time than watching broken processes in action, you would have searched for a better way. And you would have found one after a quick search and a few mouse clicks away because these issues have already been addressed and solved by the community.

## Summary

When I started looking for a solution, I wanted to achieve three things:

1. Have a single command for performing all the package validity checks: build, test suite, linting, documentation, etc.
2. Have a single command for deployment where the developer specifies the next version and it magically appears on PyPI;
3. Make the aforementioned processes easy to debug, modify and replace if a better option arises.

This led me to [`tox`](https://github.com/tox-dev/tox), an automation tool primarily used for running test suites against multiple version of dependencies. However, it's also useful for automating all sorts of different tasks, even the ones requiring Python 3 when you're still using Python 2. To give you a taste of where this leads, the following things were achieved:

1. The test suite was executed _against the installed distribution_ and multiple versions of dependencies, thus ensuring the distribution is valid and you support multiple versions of your dependencies. This is achieved by executing `tox` in the command line to run the whole test suite, or `tox -e <env>` if you want to test a specific use case.
2. When one wants to create a new release, one has to execute `tox -e release` to create a feature release, or `tox -e release -- patch` to create a patch release when [semantic versioning](https://semver.org/) is used. The release process comprises several other `tox` processes, such as `changelog` or `manifest`, each of which can be executed individually.
3. The deployment was executed in the CI environment, but it could also be executed locally. This ensured the distributions are created in a controlled and stable environment, not on someone's machine with some random settings.

This process was propagated into other packages and into the internal [`cookiecutter`](https://github.com/cookiecutter/cookiecutter) project to make it available to new packages as well. This means that developers had a unified, standardized workflow which enabled everyone to initiate a new release almost instantly, only to have it appear on PyPI in a matter of minutes.

This post will summarize my experiences and point you to other useful posts covering this subject matter so you don't waste time on the same issues. I'll spare you the intricate details of producing the package as there is already enough material on that subject online.

However, if you need a reference where everything is already confgured, take a look at the [`centerline`](https://github.com/fitodic/centerline) package. I've created this package during college as an exercise, but nowadays I use it primarily for testing packaging and distribution. The main files you should be focusing on are [`setup.cfg`](https://github.com/fitodic/centerline/blob/master/setup.cfg), [`pyproject.toml`](https://github.com/fitodic/centerline/blob/master/pyproject.toml), [`.bumpversion.cfg`](https://github.com/fitodic/centerline/blob/master/.bumpversion.cfg), [`.travis.yml`](https://github.com/fitodic/centerline/blob/master/.travis.yml), and last but not least, [`tox.ini`](https://github.com/fitodic/centerline/blob/master/tox.ini).

## A bit of history

Python has a rather complicated build and distribution mechanism when it comes to third party libraries. It's not rocket science, just a large amount of legacy that you have to navigate. The initial idea was to have a _batteries included_ approach where the Python's core had all the things you would ever need.

However, that process inhibited reuse of code that was considered useful, but not useful enough to be included into Python's core. And so [PyPI](https://pypi.org) was born, a place where people could upload their Python code and share it with the world. Now you have the best of both worlds, right?

Well, yes and no. In hindsight, its easy to criticize the decisions that were made at the time that led us to a point where we have [multiple ways to build and distribute libraries](https://discuss.python.org/t/developing-a-single-tool-for-building-developing-projects/2584) and [call Python's standard library a dead end](https://pyfound.blogspot.com/2019/05/amber-brown-batteries-included-but.html). The point is, although things are not ideal, there is no point complaining about it if you're not going to do anything about it.

To correct the course of the entire Python ecosystem, a large amount of time, energy and money is needed. That's something the open-source community is traditionally scarce at, but [things are improving](https://discuss.python.org/t/developing-a-single-tool-for-building-developing-projects/2584).

On the other hand, if you are experiencing issues with it, and you want to start somewhere, you can start in your backyard (so to speak). Review the tooling choices that are currently made available to you, standardize your libraries' maintenance process, obfuscate the internals by providing simple high level APIs or access points to get the job done.

During this process you'll familiarize yourself with the ins and outs of the build system, the [terminology](https://packaging.python.org/glossary/), and you'll be able to make informed decisions based on your needs, and spare your colleagues who are less interested in the package distribution.

With that in mind, I would like to present to you one of the ways that helped us resolve our differences and enabled us to get past the semi-automated phase. It's important to note that this is not _the_ way. If it doesn't suit you or your team's needs, fine. This also won't cast blame on the packaging ecosystem, but show you an approach that improved the collaboration on various libraries by adopting the following changes:

* [Switching to the `src/` layout](#the-src-layout)
* [Declarative configuration](#declarative-configuration)
* [Trunk based development](#trunk-based-development)
* [Changelog and version management](#changelog-and-version-management)
* [Release automation](#release-automation)

## The `src` layout

When dealing with Python libraries that are sometimes called packages (i.e. a directory with the `__init__.py` file) because they mostly consist of packages that need to be distributed, there are two most-common ways of organizing your codebase:

1. [Namespace layout](https://packaging.python.org/tutorials/packaging-projects/#creating-the-package-files) (I'm not sure it's the correct term, but let's go with it for now):

``` BASH
├── mypackage
│   ├── __init__.py
│   └── mod1.py
├── tests
├── setup.py
└── setup.cfg
```

2. [`src/` layouts](https://setuptools.readthedocs.io/en/latest/setuptools.html#using-a-src-layout):

``` BASH
├── src
│   └── mypackage
│       ├── __init__.py
│       └── mod1.py
├── tests
├── setup.py
└── setup.cfg
```

You may be wondering what's the difference? The difference is how the Python's `import` system treats these files during development and testing, and that has an effect on the build and distribution processes.

### Development mode

Before we get into details, please bear in mind that some people use symlinks to connect packages to projects where they are being used, whereas others use [development mode](https://setuptools.readthedocs.io/en/latest/setuptools.html#development-mode) enabled by [`pip`'s `editable installs`](https://pip.pypa.io/en/stable/reference/pip_install/#install-editable). I personally prefer the latter which goes something along the lines of this:

``` BASH
$ mkproject centerline  # provided by `virtualenvwrapper`
$ git clone https://github.com/fitodic/centerline .
$ pip install -e .[dev,test,docs]
```

Why do I prefer this? Because from my experience, it's the most stable and reliable development approach when developing and distributing packages. It also drastically simplifies the package's setup, both in testing and production environments.

On the other hand, I found `symlink`s impractical because not all packages are meant to be used as dependencies in other projects, for example Django packages. And I prefer to have them tested immediately as a standalone codebase. Which brings us to the crux of the problem.

### Testing packages

When using namespace layouts, i.e. the package is in the project's root directory where `setup.py` is located, Python will implicitly include the current directory in `sys.path`. This means that when you run your test suite, the tests will be ran against the code you have in your current working directory, _not the installed distribution_.

If you don't think thats a big deal, let me give you an example. Let's say you have a library that has a test suite that is executed in the CI (Continuous Integration) environment. Someone introduces a change to the library's configuration (e.g. `setup.py`) or forgets to include some static files in the `MANIFEST.in` file (they are not picked up by default). All the tests pass and you create a new release.

The release is successfully deployed to PyPI and installed by your users who start getting `ImportError`s. The library is clearly installed, but its empty. It has its metadata so everything looks OK from `pip`'s point of view, **but the code is missing** (or parts of it).

How can this scenario be avoided? By placing your code in the `src/` directory and configuring `setuptools` (or whichever build system you are using) to search for packages in the `src/` directory. **This forces you to install the distribution that would be shipped to your users and running the test suite against it.** That way, if there are any configuration errors, you'll catch them immediately.

You may have also noticed that in the `src/` layout pictured above, the `tests` directory is located on the same level as is the `src/` directory. The `pytest` documentation has a chapter on [good integration practices](https://docs.pytest.org/en/latest/goodpractices.html) which I recommend you read. The approach described above is in line with these practices for exactly these reasons, to improve the reliability of the entire library development process.

If you are interested in more details, I highly recommend the following blog posts:

* [Rehashing the `src` layout](https://blog.ionelmc.ro/2017/09/25/rehashing-the-src-layout/)
* [Python packaging pitfalls](https://blog.ionelmc.ro/2014/06/25/python-packaging-pitfalls/)
* [Packaging a Python library](https://blog.ionelmc.ro/2014/05/25/python-packaging/)
* [Testing & packaging](https://hynek.me/articles/testing-packaging/#src)

## Declarative configuration

To make the package usable, you have to configure it before you deploy it. This configuration comprises package metadata, the list of dependencies that need to be installed, what to include into the distribution, the build system, etc. Unfortunately, at this moment, there are a number of files you have to configure to achieve this.

Let's start with the build phase. [PEP-518](https://www.python.org/dev/peps/pep-0518/) introduced the [`pyproject.toml`](https://snarky.ca/clarifying-pep-518/) which enables users to use either [`setuptools`](https://github.com/pypa/setuptools), [`flit`](https://github.com/takluyver/flit) or [`poetry`](https://github.com/python-poetry/poetry) to build the distribution from source.

Why is this important? Long story short, when `pip` installs the package from an [`sdist` (source distribution)](https://packaging.python.org/glossary/), it executes the `setup.py` file in order to build a `wheel` (binary distribution). When executing `setup.py`, it assumes its only dependencies are `setuptools` and `wheel`, and it needs `setuptools` to do that. What if this is deployed in a restrained environment without `setuptools` installed beforehand? How do you specify which package to download in order to install your own package? Especially when you need `setuptools` to read the configuration in the first place.

`pyproject.toml` solves this issue by allowing you to specify the build dependencies, and you only need `pip`. You could have specified the build dependencies in `setup.py` before that, but then again, you had to have `setuptools` installed to read it.

There is also another thing that is problematic with `setup.py`. You could find all manner of customized "stuff" in them, some even requiring dependencies that were yet to be installed. For instance, the `setup.py` imported the library's version from `myproject/__init__.py`, which also contained imports to dependencies. But how did we come to that? I suppose package developers treated packages as regular projects and kept using `requirements.txt` files, custom build scripts and various other procedures to "make it work".

The [declarative configuration](https://setuptools.readthedocs.io/en/latest/setuptools.html#configuring-setup-using-setup-cfg-files) is a way to limit the scope of abuse. By declaring everything in `setup.cfg`, or one day `pyproject.toml` if it supports it, there aren't many ways to "hack" your way around it. I hope.

Even though I prefer `setup.cfg`, that doesn't mean you cannot put all of this in `setup.py` to achieve the same result. It's a matter of preference, although [more](https://coverage.readthedocs.io/en/coverage-5.0.3/changes.html#version-5-0b1-2019-11-11) and [more](https://github.com/psf/black#pyprojecttoml) tools have started adding support for reading their configuration from `pyproject.toml`. In my opinion, one file would ideally hold the entire project's configuration, but then again, it's a matter of preference.

### Dependencies

Libraries declare, or _should_ declare, their dependencies in spans to support multiple versions simultaneously, unlike projects like Web applications, where you would want to specify the exact version that's being deployed. Furthermore, you would also want to list you extra dependencies that users may or may not install, depending on their needs. You can use the same extra dependencies for setting up your development and testing environment as well:

```ini
install_requires =
	Fiona>=1.7.0
	Shapely>=1.5.13
	numpy>=1.10.4
	scipy>=0.16.1
	Click>=7.0

[options.extras_require]
dev =
	tox
gdal =
	GDAL>=2.3.3
lint =
	flake8
	isort
	black
test =
	pytest>=4.0.0
	pytest-cov
	pytest-sugar
	pytest-runner
docs =
	sphinx
```

## Trunk based development

Once the library's configuration is in place, other developers will most likely join in and couldn't care less about the package's structure and deployment, as long as it works. But collaboration on a package isn't just run the test suite -> code review -> merge changes. Packages often provide functionality that builds on its dependencies.

As with all dependencies, there are deprecation periods and sometimes, its just not possible to continue supporting all versions. The Python 2 to 3 migration is one such example where packages simultaneously supported both version of Python, and then dropped Python 2. These types of changes impact their users the most, so they should be executed with care.

The difficult part is supporting multiple incompatible versions at the same time. A `compat.py` module, or the module where all the compatibility edge-cases are hidden, can only get you so far. At some point in time, you'll want to create a release that will only continue receiving security patches, while most of the development and innovation carries on.

This is where release management from the version control system's perspective kicks in. There are several available options, such as [Git Flow](https://www.gitflow.com/) or [Trunk based development](https://trunkbaseddevelopment.com/) (TBD). I've used both of them when working on libraries and can safely say that TBD produces more satisfying results.

In a nutshell, TBD requires developers to create short-lived branches from the `master` branch that will be merged back into the `master` branch. Every once in a while, a release branch is created from the `master` branch that carries a version designation, such as `release_1_11`. That branch is the source for creating 1.11.X releases until the version or branch is deprecated. Meanwhile, all the security patches that are merged into the `master` branch are [`cherry-pick`ed](https://git-scm.com/docs/git-cherry-pick) into the `release` branch. If you're lucky and the developers working on the project make [sensible commits and commit messages](https://chris.beams.io/posts/git-commit/), it's even easier to manage releases, especially around bugfixes and security patches.

This approach enables the continuation of feature development according to the project's roadmap, without worrying too much about backwards compatibility. After all, that's what the `release` branches are for.

## Changelog and version management

One of the reasons why TBD was adopted in the first place was changelog and version management. My team inherited a customized GitFlow workflow that introduced more frustration when someone introduced an ill-advised method for handling multiple releases simultaneously. There were practically two codebases whose functionality was more or less the same, apart from the code that handled compatibility issues between two versions of a certain dependency.

This brought even more frustration when a new release had to be made. Various merge requests were issued, slow pipelines executed, merge conflicts resolved (mostly around the changelog), and so forth. There had to be a better way.

After some research, I stumbled upon [TBD](#trunk-based-development) and everything just clicked as described in the previous chapter. Furthermore, I dropped the custom built changelog and versioning script in favor of [`towncrier`](https://github.com/hawkowl/towncrier) and [`bumpversion`](https://github.com/peritus/bumpversion). I don't think there is a need to go into too many details as their documentations are preety straightforward. [After some minor issues](https://github.com/tox-dev/tox/issues/1081), these two libraries were successfully integrated into `tox` workflow.

## Release automation

The last step to set up was the release process. This is also a `tox` environment, named `release` that built the changelog using `towncrier`, bumped the package version using `bumpversion`, tagged it and pushed the changes to `origin`.

From there on, the CI would run the test suite one more time, again using `tox` so developers can easily reproduce issues if they arise, build the standard (`sdist`) and binary (`wheel`) distributions, and upload them to PyPI using [`twine`](https://github.com/pypa/twine). If you are using Travis CI, you can use [its own mechanism for uploading distributions](https://docs.travis-ci.com/user/deployment/pypi/).

## Conclusion

As mentioned earlier, this is one approach that may or may not suit you and your team. Each library used here has an alternative and it also comes down to a matter of preference.

If you are starting out or need to create a new library, I would advise you to use one fo the existing Python cookiecutters, such as [`cookiecutter-pylibrary`](https://github.com/ionelmc/cookiecutter-pylibrary). You can also build your own as I did. Beats copy/pasting coded from an existing project or documentation.
