# Overview of the release process:

* Part 1: Pre release tasks
* Part 2: Release to Artifactory
* Part 3: Promote to Bintray and Maven Central
  * Part 3.1: From Artifactory to Bintray
  * Part 3.2: From Bintray to Maven Central
* Part 4: Post release tasks
* Part 5: Rolling back a release

# Part 1: Pre release tasks

1.1 Get lastest changes from the upstream repository of the branch being released.

1.2 Check if `build.gradle` refers to any snapshot/milestone dependencies.

1.3 Disable the `Gradle` build plan in Bamboo:

TODO images/1.png

# Part 2: Release to Artifactory

2.1 Run a build of the `Gradle` plan in Bamboo.

2.2 Go to `Artifactory build info` and check artifacts:

TODO images/2.png

TODO images/3.png

2.3 Go to `Artifactory Release & Promotion` tab:

TODO images/4.png

2.3.1 Fill in the form with release version and VCS configuration then hit `Build and Release to Artifactory`.

2.3.2 Check uploaded jars in http://repo.spring.io/libs-staging-local/ and do a smoke test with the staged version (check the integrity of the artifacts to see if jars are not corrupted or empty, etc).

2.4 Promote the release to Artifactory:

TODO images/5.png

**NOTE:** For "Target promotion repository" field, select:

* `libs-milestone-local` for milestones
* `libs-release-local` for releases

2.5 Go to github and check if the release branch/tag are created:

TODO images/6.png

2.6 Check uploaded jars in http://repo.spring.io/libs-release-local/

**Note: The process ends here for milestone releases**

# Part 3: Promote to Bintray and Maven Central

### Part 3.1 From Artifactory to Bintray

3.1.1 Go to https://repo.spring.io and login as `buildmaster`.

3.1.2 Check if published modules are as expected:

TODO images/7.png

3.1.3 Go to `General build info` and click `Distribute`:

TODO images/8.png

First, select "spring-distributions" in "Distribution Repository" do a "Dry run". There might be 3 errors about "schema.zip", "docs.zip" and "dist.zip" because they do not match any rule for artifacts with type Maven. These 3 errors can be safely ignored:

TODO images/9.png

**Any other error should be carefully analyzed!**

Then click "Distribute". Artifacts should now be promoted to Bintray.

### Part 3.2 From Bintray to Maven Central

3.2.1 Go to https://bintray.com and login as `spring-operator`.

3.2.2 Select the corresponfing release from the home page:

TODO images/10.png

3.2.3 Go to `Maven Central` tab:

TODO images/11.png

3.2.4 Fill in the form with the credentials of the "oss.sonatype.org" account (`springsource`) from LastPass vault (since we are pushing to "oss.sonatype.org") and hit "Sync".

3.2.5 Go to https://oss.sonatype.org and login with `springsource` account.

3.2.6 In `Staging Repositories` tab, select the staged release and check its content/activity:

TODO images/12.png

After a few minutes, the staging repository will be **automatically** closed and the release will be pushed to Maven Central repository.

# Part 4: Post release tasks

4.1 Upload docs to the docs server.

4.2 Reactivate the `Gradle` build plan in Bamboo.

4.3 Generate release notes in JIRA, release the version and create next version.

4.4 Update project page on Sagan with latest release/snapshot versions.

4.5 Write release announcement blog post.

4.6 Tweet about the release (make sure to cc `@SpringCentral` until `@SpringBatch` handle is ready).

4.7 Write email to the team.

# Part 5: Rolling Back a release

If anything goes wrong before promoting to Maven Central:

1. Delete version from Bintray.
2. Delete version from Artifactory.
3. Delete release branch/tag from git locally and remotely.
4. Revert the version in `gradle.properties` and push the change to the upstream repository.
5. Restart the release process.