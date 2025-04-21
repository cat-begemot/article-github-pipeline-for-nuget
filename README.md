# Automating NuGet Package Verification and Publishing Using GitHub Actions

In this article, I’ll walk through a practical example of how to configure [CI/CD](https://github.com/resources/articles/devops/ci-cd) using [GitHub Actions](https://github.com/features/actions) to validate and publish a NuGet package — starting with a minimal useful pipeline and gradually expanding it to fully automated the required processes.

## Table of Contents

* [Environment, Process, and Goals](#environment-process-and-goals)
* [Manual Steps Required for Publishing](#manual-steps-required-for-publishing)
* [A Brief Overview of GitHub Actions Automation and YAML File Structure](#a-brief-overview-of-github-actions-automation-and-yaml-file-structure)
* [1. MVP Pipeline](#1-mvp-pipeline)
	* [1.1. Initial Configuration](#11-initial-configuration)
	* [1.2. Creating a Pipeline with a Trigger](#12-creating-a-pipeline-with-a-trigger)
	* [1.3. Adding the Package Build Job](#13-adding-the-package-build-job)
	* [1.4. Adding the Package Publishing Job](#14-adding-the-package-publishing-job)
	* [1.5. Result](#15-result)
* [2. Adding a Test Verification Step](#2-adding-a-test-verification-step)
* [3. Adding a Check for the Current Project Version](#3-adding-a-check-for-the-current-project-version)
* [4. Adding a Tag with the Version to the Current Release Commit](#4-adding-a-tag-with-the-version-to-the-current-release-commit)
* [5. Creating a Release in the GitHub Repository](#5-creating-a-release-in-the-github-repository)
* [6. Managing Job Dependencies and Execution Order](#6-managing-job-dependencies-and-execution-order)
* [7. Final Pipeline](#7-final-pipeline)
* [Conclusion](#conclusion)
* [References](#references)

## Environment, Process, and Goals

Let’s assume we’re developing a library using the `C#/.NET` stack and plan to make it publicly available as a `NuGet` package. At this stage, we have a .NET solution in a local Git repository. For remote version control, we’re using the `GitHub` service.

To understand what we’ll be automating later, let’s walk through the manual NuGet package deployment process. We can divide the work into two parts: the initial environment and project configuration, and the repetitive manual steps required for each NuGet publication.

## Manual Steps Required for Publishing

Whenever a new version is ready for release, the following steps need to be executed each time to deliver the new NuGet version to users:

1. Run unit tests
2. Increment the [version number](https://learn.microsoft.com/en-us/nuget/create-packages/package-authoring-best-practices#package-version) in the project configuration file: `<Version>1.0.1</Version>`
3. Merge commits into the main branch and add a `Git` tag with the release version pointing to the commit
4. Build the release version of the library: `dotnet pack --configuration Release`
5. Publish the release version to `nuget.org` for public access: `dotnet nuget push {NUGET_NAME_WITH_VERSION} --api-key {API_KEY} --source https://api.nuget.org/v3/index.json`
6. Create a [release](https://docs.github.com/en/repositories/releasing-projects-on-github/about-releases) in the `GitHub` repository with a description of the changes included in the current version

We’ll take these steps as the requirements to be automated.

## A Brief Overview of GitHub Actions Automation and YAML File Structure

[GitHub Actions](https://github.com/features/actions) is a [CI/CD](https://github.com/resources/articles/devops/ci-cd) tool used to automate workflows for building and publishing software. Workflows can include things like managing branches during pull requests, code reviews and merges, as well as building, testing, and publishing results.

Many of the publishing steps are repetitive and routine, and can be automated — optionally with added flexibility via parameters or conditions. In other words, the process can be automated by writing a script in a YAML file, which `GitHub` interprets as a set of automation instructions.

A YAML file includes the following elements:

1. Triggers for launching automation (e.g., a commit to a specific branch or a pull request)
2. Environment in which the commands will be run (e.g., OS type and version, or container)
3. Command definitions. These are defined as `steps` within a single `job`. A `pipeline` can contain `multiple jobs`.

You can read more about how GitHub Actions works on the [official documentation site](https://docs.github.com/en/actions/about-github-actions/understanding-github-actions).

The documentation for writing workflows is available [here](https://docs.github.com/en/actions/writing-workflows).

## 1. MVP Pipeline

### 1.1. Initial Configuration

1. We’ll use [nuget.org](https://www.nuget.org) as the NuGet package host. If you don’t already have an account, [create one](https://learn.microsoft.com/en-us/nuget/nuget-org/individual-accounts#add-a-new-individual-account).
2. [Generate an API key](https://learn.microsoft.com/en-us/nuget/nuget-org/publish-a-package#create-an-api-key) in your NuGet account. This will be required to publish your package.
3. Add the necessary [metadata](https://learn.microsoft.com/en-us/nuget/create-packages/package-authoring-best-practices) to the `<PropertyGroup>` and `<ItemGroup>` sections to the `*.csproj` project configuration file for publication.

```xml
<PropertyGroup>

  <!-- Nuget package unique identifier and version -->
  <PackageId>Library.UsefulPackage</PackageId>
  <Version>1.0.1</Version>

  <!-- License information -->
  <PackageLicenseExpression>MIT</PackageLicenseExpression>

  <!--  Author, description, icon, documentation -->
  <Authors>Software Developer</Authors>
  <Title>Short description</Title>
  <Description>Project description</Description>
  <PackageIcon>logo.png</PackageIcon>
  <PackageReadmeFile>README.md</PackageReadmeFile>
  <GenerateDocumentationFile>True</GenerateDocumentationFile>

  <!-- Repository reference -->
  <RepositoryUrl>https://github.com/software-developer/useful-library</RepositoryUrl>

  <!-- Tags describing the project for indexing and searching -->
  <PackageTags>dotnet, useful, lib, etc</PackageTags>
</PropertyGroup>

<ItemGroup>

   <!-- PropertyGroup necessary references -->
   <None Include="..\..\images\logo.png" Pack="true" PackagePath="\"/>
   <None Include="..\..\LICENSE" Pack="true" PackagePath="LICENSE"/>
   <None Include="..\..\README.md" Pack="true" PackagePath="\"/>

</ItemGroup>
```

### 1.2. Creating a Pipeline with a Trigger

Let’s add a `./github/workflows/release-and-publish.yml` file to the project repository, specifying the name of the pipeline and the [event that triggers it](https://docs.github.com/en/actions/writing-workflows/choosing-when-your-workflow-runs/events-that-trigger-workflows). In our case, it will be a commit to the master branch of the remote repository.

```yaml
# 1.1. Creating a Pipeline with a Trigger
# Pipeline name
name: Create release and publish NuGet

# Trigger condition. In this case it is a commit into the remote master branch
on:
  push:
    branches:
      - "master"

# Jobs will be added here according to the requirements...
jobs:
```

To create a minimally functional and useful pipeline, we'll add a job to build the NuGet package and a job to publish the artifact to nuget.org.

### 1.3. Adding the Package Build Job

From this point on, I'll only show the delta of the changes to the pipeline. The final version can be found [at the end of the article]().

```yaml
# 1.2. Adding the Package Build Job
# Unique job identifier that can be used as a reference
create_nuget:
  # User-friendly job name for the UI purposes
  name: Create NuGet
  # Environment definition. Each job is executed in a separate, isolated environment
  runs-on: ubuntu-24.04
  # Save path to the NuGet directory in the environment variable
  env:
    NuGetDirectory: ${{ github.workspace}}/nuget
  # List of commands to be run sequentially
  steps:
    # Checkout on a branch commit to access the source code
    - name: Checkout repository
      uses: actions/checkout@v4

    # Install SDK
    - name: Setup .NET
      uses: actions/setup-dotnet@v4

    # Build and pack package
    - name: Pack
      shell: pwsh
      run: dotnet pack .\src\UsefulPackage --configuration Release --output ${{ env.NuGetDirectory }}

    # Uploading an artifact to the repository for access from other jobs
    - uses: actions/upload-artifact@v4
      with:
        name: nuget
        if-no-files-found: error
        retention-days: 7
        path: ${{ env.NuGetDirectory }}/*.nupkg
```

As a result, we’ll have an uploaded artifact stored with the name UsefulPackage.1.0.1.nupkg, where the version number comes from the .csproj file of the project. Don’t forget to increment the version number with every release, as you won’t be able to publish the same version twice on nuget.org.

### 1.4. Adding the Package Publishing Job

```yaml
# 1.3. Adding the Package Publishing Job
deploy:
  name: Deploy NuGet
  runs-on: ubuntu-24.04
  # A ready artifact is required before publishing
  # The job waits for the create_nuget job to complete
  needs: create_nuget
  # This jobs runs if create_nuget succeeds
  if: success()
  # Save path to the NuGet directory in the environment variable
  env:
    NuGetDirectory: ${{ github.workspace}}/nuget
  steps:
    # Download the contents of the repository
    - name: Download artifact
      uses: actions/download-artifact@v4
      with:
        name: nuget
        path: ${{ env.NuGetDirectory }}

    - name: Setup .NET Core
      uses: actions/setup-dotnet@v4

    # Publish the NuGet package using the dotnet utility
    - name: Publish NuGet package
      shell: pwsh
      run: |
        foreach($file in (Get-ChildItem "${{ env.NuGetDirectory }}" -Recurse -Include *.nupkg)) {
            dotnet nuget push $file --api-key "${{ secrets.NUGET_APIKEY }}" --source https://api.nuget.org/v3/index.json --skip-duplicate
        }
```

In the publishing step, we iterate through the contents of the uploaded artifact storage (including subdirectories), and every file with the `*.nupkg` extension is uploaded to the `nuget.org`.

To avoid exposing your NuGet host access key, [store it securely in the repository settings](https://docs.github.com/en/actions/security-for-github-actions/security-guides/using-secrets-in-github-actions) and reference it in the pipeline like this: `"${{ secrets.NUGET_APIKEY }}"`.

### 1.5. Result

In the end, we have a minimally working pipeline:

<img width="600" src="./assets/Pic-1.png"/>

In the following sections, we’ll expand the current functionality to fully meet the original requirements:

- Ensure the pipeline only proceeds if tests pass
- Prevent release if the project version is unchanged or incorrectly updated
- Tag the release commit
- Create a GitHub release

## 2. Adding a Test Verification Step

To ensure that code with failing tests is never published, we’ll add a job to run and check unit tests.

```yaml
# 2. Adding a Test Verification Step
run_test:
  name: Run tests
  runs-on: ubuntu-24.04
  steps:
    - name: Checkout repository
      uses: actions/checkout@v4

    - name: Setup .NET
      uses: actions/setup-dotnet@v4

    - name: Run tests
      shell: pwsh
      run: dotnet test --configuration Release .\src\UsefulPackage.UnitTests
```

## 3. Adding a Check for the Current Project Version

At this stage, we want to make sure the project version in the `*.csproj` file is higher than the last released version, which we’ll extract from the tag on the most recent release commit. In other words, this check ensures we didn’t forget to bump the project version before pushing to the `master` branch.

```yaml
# 3. Adding a Check for the Current Project Version
check_version:
  name: Check project version
  runs-on: ubuntu-24.04
  outputs:
    # The job returns the result of the check in the variable is_valid
    # From other jobs the result can be obtained using the expression needs.check_version.outputs.is_valid
    is_valid: ${{ steps.compare_versions.outputs.is_valid }}
  steps:
    - name: Checkout repository
      uses: actions/checkout@v4

    - name: Get project version from .csproj
      shell: bash
      run: |
        # Get a project version from the *.csproj file
        VERSION=$(grep -oPm1 "(?<=<Version>)[^<]+" ./src/UsefulPackage/UsefulPackage.csproj)
        echo "Project version is $VERSION"
        # Save the result into the variable VERSION
        echo "VERSION=$VERSION" >> $GITHUB_ENV

    - name: Get latest tag
      id: tag
      run: |
        # Get the latest release version according to the latest version tag from the repository
        git fetch --tags
        LATEST_TAG=$(git tag -l "v*" --sort=-v:refname | head -n 1)
        echo "Latest tag: $LATEST_TAG"
        # Save the result into the variable LATEST_TAG
        echo "LATEST_TAG=$LATEST_TAG" >> $GITHUB_ENV

    - name: Compare Strings
      id: compare_versions
      run: |
        # Find the max version by comparing VERSION and LATEST_TAG and save the result into the variable GREATER_VERSION
        GREATER_VERSION=$(printf "%s\n%s" "$VERSION" "${LATEST_TAG#v}" | sort -V | tail -n 1)
        if [[ "$VERSION" == "$GREATER_VERSION" && "$VERSION" != "${LATEST_TAG#v}" ]]; then
          # If the version in the project configuration file is higher than the tag version, then the check is passed.
          echo "The new release version is ${LATEST_TAG#v}"
          echo "is_valid=true" >> $GITHUB_OUTPUT
        else
          # Otherwise it is signaled about the error
          echo "The project version is not incremented"
          echo "is_valid=false" >> $GITHUB_OUTPUT
        fi
```


## 4. Adding a Tag with the Version to the Current Release Commit

Having a Git tag with the release version is useful for at least three reasons:
- It helps to quickly find and check out the commit for a specific release version in the git log
- This tag is used by the version check in the previous step
- GitHub Releases are linked to the corresponding tag

```yaml
# 4. Adding a Tag with the Version to the Current Release Commit
tag_and_push:
  name: Create and push release tag
  runs-on: ubuntu-24.04
  steps:
    - name: Checkout repository
      uses: actions/checkout@v4

    # Change git configuration for the current job environment for committing the tag under the author's name and email
    - name: Set up Git
      run: |
        git config --global user.name "${{ secrets.GIT_USER_NAME }}"
        git config --global user.email "${{ secrets.GIT_USER_EMAIL }}"

    # Using the same code a second time, the best practice is to put it in a separate action
    # To simplify the example this step is skipped
    - name: Get project version from .csproj
      shell: bash
      run: |
        VERSION=$(grep -oPm1 "(?<=<Version>)[^<]+" ./src/UsefulPackage/UsefulPackage.csproj)
        echo "Project version is $VERSION"
        echo "VERSION=$VERSION" >> $GITHUB_ENV

    - name: Fetch the latest changes from the remote repository
      run: |
        git fetch --tags

    - name: Create and push tag
      run: |
        NEW_TAG="v$VERSION"
        git tag $NEW_TAG
        echo "Tag created: $NEW_TAG"
        git push origin $NEW_TAG 
```

## 5. Creating a Release in the GitHub Repository

Ideally, users should have access to [release documentation](https://docs.github.com/en/repositories/releasing-projects-on-github/about-releases) for each version outlining relevant changes.

```yaml
# 5. Creating a Release in the GitHub Repository
release:
  name: Create release
  runs-on: ubuntu-24.04
  env:
    # Temporary token for workflow authentication to create a release
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  steps:
    - name: Checkout repository
      uses: actions/checkout@v4

    - name: Create GitHub release
      run: |
        git fetch --tags
        NEW_TAG=$(git describe --tags --abbrev=0 origin/master)
        echo "Latest tag on master: $NEW_TAG"
        gh release create $NEW_TAG \
            --repo="$GITHUB_REPOSITORY" \
            --title="${NEW_TAG#v}" \
            --generate-notes \
            --generate-notes \
            --verify-tag \
            --latest
```

## 6. Managing Job Dependencies and Execution Order

By automating all the requirements, we end up with a sequence of actions:

<img width="600" src="./assets/Pic-2.png"/>

But the issue is that jobs run independently, regardless of whether other jobs pass or fail. However, in our case, we have clear dependencies and a required order. For instance, there’s no point running the whole workflow if the tests fail or the version is invalid.

To manage dependencies and conditions, we use `needs` and `if` tags in jobs. These let us refer to other jobs or their outputs by their unique IDs.

```yaml
# This job waits for the completion of the specified jobs
needs: [run_test, check_version]
# Create a tag only if unit tests pass and version check returns valid
if: ${{ success() && needs.check_version.outputs.is_valid == 'true' }}
```

With correctly configured dependencies and execution order, the workflow will look like this:

<img width="1200" src="./assets/Pic-3.png"/>

You can see how I achieved this by checking the final YAML file mentioned earlier.

## 7. Final Pipeline

```yaml
# 1.1. Creating a Pipeline with a Trigger
# Pipeline name
name: Create release and publish NuGet

# Trigger condition. In this case it is a commit into the remote master branch
on:
  push:
    branches:
      - "master"

jobs:
  # 2. Adding a Test Verification Step
  run_test:
    name: Run tests
    runs-on: ubuntu-24.04
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4

      - name: Run tests
        shell: pwsh
        run: dotnet test --configuration Release .\src\UsefulPackage.UnitTests
  
  # 3. Adding a Check for the Current Project Version
  check_version:
    name: Check project version
    runs-on: ubuntu-24.04
    outputs:
      # The job returns the result of the check in the variable is_valid
      # From other jobs the result can be obtained using the expression needs.check_version.outputs.is_valid
      is_valid: ${{ steps.compare_versions.outputs.is_valid }}
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Get project version from .csproj
        shell: bash
        run: |
          # Get a project version from the *.csproj file
          VERSION=$(grep -oPm1 "(?<=<Version>)[^<]+" ./src/UsefulPackage/UsefulPackage.csproj)
          echo "Project version is $VERSION"
          # Save the result into the variable VERSION
          echo "VERSION=$VERSION" >> $GITHUB_ENV

      - name: Get latest tag
        id: tag
        run: |
          # Get the latest release version according to the latest version tag from the repository
          git fetch --tags
          LATEST_TAG=$(git tag -l "v*" --sort=-v:refname | head -n 1)
          echo "Latest tag: $LATEST_TAG"
          # Save the result into the variable LATEST_TAG
          echo "LATEST_TAG=$LATEST_TAG" >> $GITHUB_ENV

      - name: Compare Strings
        id: compare_versions
        run: |
          # Find the max version by comparing VERSION and LATEST_TAG and save the result into the variable GREATER_VERSION
          GREATER_VERSION=$(printf "%s\n%s" "$VERSION" "${LATEST_TAG#v}" | sort -V | tail -n 1)
          if [[ "$VERSION" == "$GREATER_VERSION" && "$VERSION" != "${LATEST_TAG#v}" ]]; then
            # If the version in the project configuration file is higher than the tag version, then the check is passed
            echo "The new release version is ${LATEST_TAG#v}"
            echo "is_valid=true" >> $GITHUB_OUTPUT
          else
            # Otherwise it is signaled about the error
            echo "The project version is not incremented"
            echo "is_valid=false" >> $GITHUB_OUTPUT
          fi

  # 4. Adding a Tag with the Version to the Current Release Commit
  tag_and_push:
    name: Create and push release tag
    runs-on: ubuntu-24.04
    # Waiting for completion the following jobs...
    needs: [run_test, check_version]
    # A tag is created after only if the unit tests have passed and the version check has been completed successfully
    if: ${{ success() && needs.check_version.outputs.is_valid == 'true' }}
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      # Change git configuration for the current job environment for committing the tag under the author's name and email
      - name: Set up Git
        run: |
          git config --global user.name "${{ secrets.GIT_USER_NAME }}"
          git config --global user.email "${{ secrets.GIT_USER_EMAIL }}"

      # Using the same code a second time, the best practice is to put it in a separate action
      # To simplify the example this step is skipped
      - name: Get project version from .csproj
        shell: bash
        run: |
          VERSION=$(grep -oPm1 "(?<=<Version>)[^<]+" ./src/UsefulPackage/UsefulPackage.csproj)
          echo "Project version is $VERSION"
          echo "VERSION=$VERSION" >> $GITHUB_ENV

      - name: Fetch the latest changes from the remote repository
        run: |
          git fetch --tags

      - name: Create and push tag
        run: |
          NEW_TAG="v$VERSION"
          git tag $NEW_TAG
          echo "Tag created: $NEW_TAG"
          git push origin $NEW_TAG

  # 5. Creating a Release in the GitHub Repository
  release:
    name: Create release
    runs-on: ubuntu-24.04
    needs: tag_and_push
    if: success()
    env:
      # Temporary token for workflow authentication to create a release
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Create GitHub release
        run: |
          git fetch --tags
          NEW_TAG=$(git describe --tags --abbrev=0 origin/master)
          echo "Latest tag on master: $NEW_TAG"
          gh release create $NEW_TAG \
              --repo="$GITHUB_REPOSITORY" \
              --title="${NEW_TAG#v}" \
              --generate-notes \
              --generate-notes \
              --verify-tag \
              --latest

  # 1.2. Adding the Package Build Job
  # Unique job identifier that can be used as a reference
  create_nuget:
    # User-friendly job name for the UI purposes
    name: Create NuGet
    # Environment definition. Each job is executed in a separate, isolated environment
    runs-on: ubuntu-24.04
    # Save path to the NuGet directory in the environment variable
    needs: tag_and_push
    if: success()
    env:
      NuGetDirectory: ${{ github.workspace}}/nuget
    # List of commands to be run sequentially
    steps:
      # Checkout on a branch commit to access the source code
      - name: Checkout repository
        uses: actions/checkout@v4

      # Install SDK
      - name: Setup .NET
        uses: actions/setup-dotnet@v4

      # Build and pack package
      - name: Pack
        shell: pwsh
        run: dotnet pack .\src\UsefulPackage --configuration Release --output ${{ env.NuGetDirectory }}

      # Uploading an artifact to the repository for access from other jobs
      - uses: actions/upload-artifact@v4
        with:
          name: nuget
          if-no-files-found: error
          retention-days: 7
          path: ${{ env.NuGetDirectory }}/*.nupkg

  # 1.3. Adding the Package Publishing Job
  deploy:
    name: Deploy NuGet
    runs-on: ubuntu-24.04
    # A ready artifact is required before publishing
    # The job waits for the create_nuget job to complete
    needs: create_nuget
    # This jobs runs if create_nuget succeeds
    if: success()
    # Save path to the NuGet directory in the environment variable
    env:
      NuGetDirectory: ${{ github.workspace}}/nuget
    steps:
      # Download the contents of the repository
      - name: Download artifact
        uses: actions/download-artifact@v4
        with:
          name: nuget
          path: ${{ env.NuGetDirectory }}

      - name: Setup .NET Core
        uses: actions/setup-dotnet@v4

      # Publish the NuGet package using the dotnet utility
      - name: Publish NuGet package
        shell: pwsh
        run: |
          foreach($file in (Get-ChildItem "${{ env.NuGetDirectory }}" -Recurse -Include *.nupkg)) {
              dotnet nuget push $file --api-key "${{ secrets.NUGET_APIKEY }}" --source https://api.nuget.org/v3/index.json --skip-duplicate
          }
```

## Conclusion

In this article, I shared my experience creating a GitHub Actions pipeline to automate routine NuGet publishing steps, which lets you focus more on the actual functionality and less on repetitive release processes.

Merging new features with the version set in the master branch triggers an automated workflow for validating and publishing the NuGet package to nuget.org, making it available shortly after via NuGet Package Manager or any other NuGet client.

Don’t forget to update the release notes section on GitHub after the pipeline completes.

<img width="350" src="./assets/Pic-4.png"/> <img width="600" src="./assets/Pic-5.png"/>

## References

1. [Автоматизация верификации и публикации NuGet пакета с помощью GitHub actions](https://github.com/cat-begemot/article-pipeline-for-nuget/blob/main/RU.md)
