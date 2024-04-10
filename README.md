# Poetry fails to resolve post-release versions when utilizing the default pypi provider.

Proof of concept and analysis regarding https://github.com/python-poetry/poetry/issues/9293

This project shows an example of where poetry fails to properly resolve the latest release due to an update utilizing
the [post-release](https://packaging.python.org/en/latest/specifications/version-specifiers/#post-releases) specifier which will not be reflected in the Release key of the project api as it is considered
a non-version changing release.

Example case `docutils~=0.21`. When docutils 0.21 was released there was a post-release 0.21.post1 version created which
does not appear as a release as it is still considered 0.21.

When poetry resolved the versions in via `https://pypi.org/pypi/docutils/json` it found `0.21.post1` however when it
goes for `https://pypi.org/pypi/docutils/0.21.post1/json` that will not be there because 0.21 is the correct version.

Poetry is trying to hit the [release api](https://warehouse.pypa.io/api-reference/json.html#release) of a version that will not exist.

Poetry should instead look for the `0.21` version instead of `0.21.postX` version.

# Explanation of workarounds
Many people will report adding a non-priority indicated pypi fixes this issue. For example adding:

```
[[tool.poetry.source]]
name = "pypi-public"
url = "https://pypi.org/simple/"
```

There are two reasons why this workaround "works":
  1. As found in the poetry repositories docs about [an undefined source being considered a primary source](https://github.com/python-poetry/poetry/blob/2ad0d938854fae3c5e3e8b49e4414cef4687fb60/docs/repositories.md?plain=1#L145), the 
    lack of priority will make this the first repository searched. 
  2. The [repository_pool.package](https://github.com/python-poetry/poetry/blob/eb74d6209ecda2840ea1c0c2375cb7f6485a39f9/src/poetry/repositories/repository_pool.py#L204) resolution will utilize the non-prioritized source.
  3. Because the name bypasses the [LegacyRepository pypi name guard](https://github.com/python-poetry/poetry/blob/c2d506916a8634c3d7a7f721b177484198e65c95/src/poetry/repositories/legacy_repository.py#L37-L38) to causes the [CachedRepository](https://github.com/python-poetry/poetry/blob/4e3cddd5fe31cc29ecbafbde0cb881660637b881/src/poetry/repositories/cached_repository.py#L36) 
    _get_release_info to utilize the [legacy_repository._get_release_info](https://github.com/python-poetry/poetry/blob/c2d506916a8634c3d7a7f721b177484198e65c95/src/poetry/repositories/legacy_repository.py#L106) which relies on the 
    SimpleRepositoryPage's link cache that'll have the link expected.

# Where the bug is
The bug occurs when a post release is in pypi _and_ there isn't a `poetry.lock` specifying what to resolve.

The [pypi_repository._get_release_info](https://github.com/python-poetry/poetry/blob/c2d506916a8634c3d7a7f721b177484198e65c95/src/poetry/repositories/pypi_repository.py#L133-L135) will end up throwing [PackageNotFound due to the json_data](https://github.com/python-poetry/poetry/blob/c2d506916a8634c3d7a7f721b177484198e65c95/src/poetry/repositories/pypi_repository.py#L133-L135) not 
resolving for the post release version.

# Suggested Points of Fixing
There are several ways in which one could fix this.

* Allow [pypi_repository._find_packages](https://github.com/python-poetry/poetry/blob/37028af45787af736b50c081589532034642469d/src/poetry/repositories/pypi_repository.py#L85-L103)
to go ahead and utilize [add_package](https://github.com/python-poetry/poetry/blob/925424a4be70b3f442387f4627a39b8906b082c3/src/poetry/repositories/repository.py#L69-L70)
since all the data is already in the [package of the _find_packages](https://github.com/python-poetry/poetry/blob/925424a4be70b3f442387f4627a39b8906b082c3/src/poetry/repositories/repository.py#L40)
response.

or

* Utilize `version.release` instead of `version` during the [_get_release_info](https://github.com/python-poetry/poetry/blob/37028af45787af736b50c081589532034642469d/src/poetry/repositories/pypi_repository.py#L129-L133)
 call of the pypi_repository.

# Why is this hard to find
If you ever get a `poetry.lock` file, then the dependencies will not be resolved, and it's hard to track this issue.

I found this because a colleague mentioned the issue, and it appears the dependency was updated April 9th and today
is April 10th making timing crucial in being able to sniff this out.

# Examples
The example projects can each utilize poetry where release-pypi will not resolve the dependency however simple-pypi
will.

Simply run `poetry install -vvv --dry-run`.
You may have to clear your cache `poetry cache clear --all .` after it installs once, and you have a lock file.

# Etc
Download details showing post1 of sdist.
![Screenshot 2024-04-10 at 1.43.06 PM.png](Screenshot%202024-04-10%20at%201.43.06%E2%80%AFPM.png)
Release history doesn't include post1, as expected.
![Screenshot 2024-04-10 at 1.43.22 PM.png](Screenshot%202024-04-10%20at%201.43.22%E2%80%AFPM.png)

* The [response](response-docutils-0.21-json.json) to https://pypi.org/pypi/docutils/0.21/json shows correct sdist with
  post version.

* Legacy call will find expected 0.21.post1 in the [response](response-simple-docutils.json).

  `curl --header "Accept: application/vnd.pypi.simple.v1+json"  https://pypi.org:443/simple/docutils/ | jq`

* https://pypi.org/pypi/docutils/json includes correct references in the [response](response-docutils-json.json).
