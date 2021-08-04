## Spring Batch Support Policy

Spring Batch follows the [VMware Tanzu OSS support policy](https://tanzu.vmware.com/support/oss) for critical bugs and security issues.

* Major versions will be supported for at least 3 years from the release date (but you must run a supported minor version).
* Minor versions will be supported for at least 12 months.

[Commercial support is also available](https://tanzu.vmware.com/spring-runtime) from VMware which offers an extended support period.

All Spring Batch releases are publicly available from Maven Central and https://repo.spring.io.
We do not have a private repository reserved only for paying customers.

## End of Life

Spring Batch releases are marked as "end of life" when they are no longer supported or released in any form.
If you are running an EOL version, you should upgrade as soon as possible.

Please note that a version can be out of support before it is end of life.
During this time you should only expect releases for critical bugs or security issues.

A Spring Batch release is supported as long as the last Spring Boot version that brings it is supported.

## Releases

### Release Schedule

Spring Batch currently has a yearly-based release cadence and typically follows the same release schedule as Spring Framework for minor and major versions.

Patch releases are published as necessary.

As much as possible, we recommend that all users migrate to the latest supported release.

### Released Versions

The following releases are actively maintained:

| Version | Released | Spring Boot version | Expected End of Life|
|:------- |:---------|:--------------------|:--------------------|
| 4.3.x   | October 2020 | 2.5.x | February 2023|
| 4.2.x   | October 2019 | 2.3.x | February 2022 |

The following releases are end of life:

| Version | Released | End of Life|
|:------- |:---------|:-----------|
| 4.1.x | October 2018 | April 2020 |
| 3.0.x | May 2014 | January 2019 |

All releases prior to 3.x are also not supported anymore.