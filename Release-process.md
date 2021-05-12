# Overview of the release process:

* Part 1: Pre release tasks
* Part 2: Release to Artifactory
* Part 3: Promote to Maven Central
  * Part 3.1: Stage on Sonatype
  * Part 3.2: Release to Maven Central
* Part 4: Post release tasks
* Part 5: Rolling back a release

# Part 1: Pre release tasks

1.1 Get latest changes from the upstream repository of the branch being released.

1.2 Check if `build.gradle` / `pom.xml` refers to any snapshot/milestone dependencies.

1.3 Disable the `Gradle` / `Maven` build plan in Bamboo:

<img alt="disable-build-plan" src="https://raw.githubusercontent.com/wiki/spring-projects/spring-batch/images/release-process/disable-build-plan.png">

# Part 2: Release to Artifactory

2.1 Run a build of the `Gradle` / `Maven` plan in Bamboo.

2.2 Go to `Artifactory build info` and check artifacts:

<img alt="check-build-info" src="https://raw.githubusercontent.com/wiki/spring-projects/spring-batch/images/release-process/check-build-info.png">

<img alt="check-artifacts" src="https://raw.githubusercontent.com/wiki/spring-projects/spring-batch/images/release-process/check-artifacts.png">

2.3 Go to `Artifactory Release & Promotion` tab:

<img alt="artifactory-build-release" src="https://raw.githubusercontent.com/wiki/spring-projects/spring-batch/images/release-process/artifactory-build-release.png">

2.3.1 Fill in the form with release version and VCS configuration then hit `Build and Release to Artifactory`. *NB:* Uncheck the `Use Release Branch` in the previous screenshot.

2.3.2 Check uploaded jars in http://repo.spring.io/libs-staging-local/ and do a smoke test with the staged version (check the integrity of the artifacts to see if jars are not corrupted or empty, etc).

2.4 Promote the release to Artifactory:

<img alt="artifactory-promote" src="https://raw.githubusercontent.com/wiki/spring-projects/spring-batch/images/release-process/artifactory-promote.png">

**NOTE:** For "Target promotion repository" field, select:

* `libs-milestone-local` for milestones
* `libs-release-local` for releases

2.5 Go to github and check if the release branch/tag are created:

<img alt="github" src="https://raw.githubusercontent.com/wiki/spring-projects/spring-batch/images/release-process/github.png">

2.6 Check uploaded jars in http://repo.spring.io/libs-release-local/

**Note: The process ends here for milestone releases**

# Part 3: Promote to Maven Central

### Part 3.1 Stage the release on Sonatype

3.1.1 Go to Github Actions: https://github.com/spring-projects/spring-batch/actions.

3.1.2 Run the "Maven Central Staging" workflow:

<img alt="github-action" src="https://raw.githubusercontent.com/wiki/spring-projects/spring-batch/images/release-process/github-action.png">

Fill in the form with the build name and build number from artifactory:

<img alt="build-info" src="https://raw.githubusercontent.com/wiki/spring-projects/spring-batch/images/release-process/build-info.png">

3.1.3 Check the staging repository ID from the workflow logs:

<img alt="staging-repo-id" src="https://raw.githubusercontent.com/wiki/spring-projects/spring-batch/images/release-process/staging-repo-id.png">

This staging repository Id is needed for the next step.

### Part 3.2 Release to Maven Central

3.2.1 Go to https://s01.oss.sonatype.org/ and login with `springsource` account (Credentials in LastPass).

3.2.2 In the `Staging Repositories` tab, select the staging repository with the ID from previous step and check its content/activity:

<img alt="sonatype" src="https://raw.githubusercontent.com/wiki/spring-projects/spring-batch/images/release-process/sonatype.png">

If everything looks good, the staging repository can be closed and released from the UI. Otherwise, it should be dropped.

# Part 4: Post release tasks

4.1 Update the `current` and `current-SNAPSHOT` symbolic links on the docs server.

4.2 Reactivate the `Gradle` / `Maven` build plan in Bamboo.

4.3 Generate release notes and create a release on Github.

4.4 Update project page on Sagan with latest release/snapshot versions.

4.5 Write release announcement blog post.

4.6 Tweet about the release using the `@SpringBatch` handle.

4.7 Post a message in the `#spring-release` slack channel.

4.8 Update `build.gradle` / `pom.xml` with next snapshot versions of Spring Projects.

# Part 5: Rolling Back a release

If anything goes wrong before promoting to Maven Central:

1. Delete version from Artifactory.
2. Delete release branch/tag from git locally and remotely.
3. Revert the version in `gradle.properties` / `pom.xml` and push the change to the upstream repository.
4. Restart the release process.