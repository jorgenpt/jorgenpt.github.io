---
layout: post
title: "Flutter & GitHub Workflows: Deploying to TestFlight"
comments: true
categories:
    - GitHub
    - iOS
    - Flutter
draft: true
---

When I wanted my personal [Flutter](https://flutter.dev/) app to distribute iOS builds from a [GitHub workflow](https://docs.github.com/en/actions) using [TestFlight](https://developer.apple.com/testflight/), I couldn't find a good cohesive guide to setting it up. The process was a bit nuanced and time consuming to get right.

In the end I created a GitHub workflow that automatically publishes whenever I push a versioned tag to GitHub -- it builds the project, signs it, and publishes that build to TestFlight.

Below I've provided the comprehensive set of steps you need to set this up from scratch, which I reproduced [using a public sample repository for your reference](https://github.com/jorgenpt/flutter_github_example/tree/publish-to-testflight).

<!-- more -->


## Requirements

In order to deploy builds to [TestFlight](https://developer.apple.com/testflight/), Apple requires that you have an [Apple Developer Program](https://developer.apple.com/programs/enroll/) account -- these cost $99/year.

I'm also assuming you've created a private or public repository on GitHub that you'll be using for version control where you can run the GitHub Workflow we'll set up.


## Setup with Apple

These are the steps to configure your app in Apple's systems.

1. Create a Bundle ID on [Apple Developer portal](https://developer.apple.com/account/resources/identifiers/add/bundleId) -- it can be an arbitrary bundle ID, and it doesn't require any specific Capabilities or App Services. Typically these are reverse-domain names, for my example I used `no.tjer.HelloWorld`.
1. Create an iOS App for your Bundle ID in [App Store Connect](https://appstoreconnect.apple.com/apps) by clicking the <i class="fa fa-plus-circle" title="circled plus icon"></i> next to _Apps_
1. Configure TestFlight on [App Store Connect](https://appstoreconnect.apple.com/apps), by navigating to your app and selecting the _TestFlight_ tab.
    1. Click the <i class="fa fa-plus-circle" title="circled plus icon"></i> next to _Internal Testing_ to create a new internal group
    1. Name it whatever you'd prefer (e.g. Developers)
    1. Click the new group in the sidebar and add yourself by clicking the <i class="fa fa-plus-circle" title="circled plus icon"></i> next to _Tester (0)_


## Create & set up a Flutter project for iOS

First we set up a project with Flutter with the iOS platform enabled and configure the project's Bundle ID

1. Create a new Flutter project with iOS enabled, if you haven't already:
    
    ```sh
    flutter create --platforms ios --org no.tjer --project-name hello_world --description "Test for iOS deploy on GH" flutter_github_example
    ```
1. Navigate to your project directory (e.g. `flutter_github_example` in the above example) and open `ios/Runner.xcworkspace` in Xcode.
1. Make sure the Bundle Identifier for the project matches the one you created above by navigating to the Runner project in the left pane, clicking the Runner target, and then updating the _Bundle Identifier_ under the _General_ tab.


## Configure code signing

To help managing code signing certificates and provisioning profiles, as well as the process of building & signing, we'll use the excellent open source project [fastlane](https://fastlane.tools/). We'll be using a **private** GH repository to distribute the signing certificate to the builder (and to the team, if wanted).


### Setting up `fastlane` for your project

Please refer to the [fastlane setup docs](https://docs.fastlane.tools/getting-started/ios/setup/) for more details, but here's a quick overview of what to do to configure Fastlane.

1. Create a text-file called `ios/Gemfile` with the following content: 

    ```rb
    source "https://rubygems.org"

    gem "fastlane"
    gem "cocoapods"
    ```
1. From a shell:

    ```sh
    brew install rbenv ruby-build # Set up rbenv to control the ruby version
    rbenv init
    ```
1. Close your terminal and re-open it for `rbenv` to be initialized, then navigate to your Flutter project and run:

    ```sh
    rbenv install -l 2>&1 | grep '^\d' | sort -n | tail -n 1 > .ruby-version # Pick the most recent stable Ruby -- at the time of writing this is 3.1.1.
    rbenv install # Install the requested version of Ruby
    gem install bundler
    cd ios
    bundle lock --add-platform x86_64-darwin-19 # Make sure the platform list includes the GH Runner platform
    bundle exec fastlane init # Start the initial configuration of fastlane
    ```
    - Choose _🚀 Automate App Store distribution_ when `fastlane init` prompts you for what you're configuring, and sign in to your Apple Developer account.
1. Make sure to add the following files to git:
    - `.ruby-version` -- the Ruby version we're using for fastlane
    - `ios/fastlane/Appfile` -- configuration for the iOS app store
    - `ios/fastlane/Fastfile` -- where we keep our fastlane recipe
    - `ios/Gemfile` -- the dependencies for fastlane
    - `ios/Gemfile.lock` -- specific version informatin for the gems specified in the Gemfile


### Configuring App Store Connect API access for fastlane

1. Set up an App Store Connect API key [on the _Users & Access_ section of the site](https://appstoreconnect.apple.com/access/users) by going to the _Keys_ tab and then:
    1. Click the <i class="fa fa-plus-circle" title="circled plus icon"></i> next to _Active (0)_.
    1. Give it an identifying name (E.g. _Flutter GH Deploy_ for the example project
    1. Choose _Developer_ as the Access for the key, and click Generate
1. Refresh the page (navigating back to _Keys_) and choose to _Download API key_ on the right side of the newly added key
1. Add the API key to your GitHub secrets -- this allows GitHub Workflows to publish new TestFlight builds:
    1. Navigate to your Flutter project's GitHub page (e.g. https://github.com/jorgenpt/flutter_github_example/ for my example repo).
    1. Go to Settings, and select Secrets > Actions on the left side.
    1. Add a _New repository secret_ named `APP_STORE_CONNECT_API_KEY_KEY` (yes, `KEY` should be there twice) and paste in the contents of the API key file you downloaded from App Store Connect (the file is named something like `AuthKey_KEYID.p8`).


### Configuring certificates & provisioning profiles

First, create a private repository [on GitHub](https://github.com/new) and mark it as _Private_. In my case, I named it `certificates`. Don't initialize it with any files or licenses. Make a note of the SSH url for your repository, in my case that'll be `git@github.com:jorgenpt/certificates`.

1. Navigate to your Flutter project in a shell and then run these commands:

    ```sh
    # These all need to be run from the ios platform of your Flutter project
    cd ios

    # For the following command, give `match init` the SSH URL for your repository (e.g.
    # git@github.com:myuser/certificates) and generate a random password (though make a note
    # of it for the next step).
    # You should add the generated Matchfile to Git.
    bundle exec fastlane match init 

    # Create an appstore distribution certificate & provisioning profile
    bundle exec fastlane match appstore
    # Create a development certificate & provisioning profile
    bundle exec fastlane match development
    ```
1. Make sure that your Xcode project is configured to use the same certificates:
    1. Open `ios/Runner.xcworkspace` in Xcode
    1. Navigate to the Runner project in the left pane, click the Runner target, and switch to the _Signing & Capabilities_ tab.
    1. Uncheck _Automatically manage signing_.
    1. Select _Release_ in the top bar and set the provisioning profile to the one that starts with _match AppStore_.
    1. Select _Debug_ in the top bar and set the provisioning profile to the one that starts with _match Development_, and repeat this for _Profile_ as well.
    {% img /images/flutter-github-testflight-configured-signing.png %}
1. Create a new SSH key that your workflow can use to access your GitHub certificates repository:

    ```sh
    ssh-keygen -C github.com:myuser/my_flutter_project_name -f ~/Desktop/id_rsa_build
    ```

1. Add the password you generated above & the SSH key to your GitHub secrets -- this allows GitHub Workflows to sign your build:
    1. Navigate to your Flutter project's GitHub page (e.g. https://github.com/jorgenpt/flutter_github_example/ for my example repo).
    1. Go to Settings, and select Secrets > Actions on the left side.
    1. Add a _New repository secret_ named `MATCH_PASSWORD` and paste in the password you generated above.
    1. Add a second _New repository secret_ named `SSH_PRIVATE_KEY` and paste in the contents of `id_rsa_build` on your Desktop.


## Putting it all together

Your GitHub repository's secrets should have three different secrets configured:

{% img /images/flutter-github-testflight-configured-secrets.png 640 %}

Now all that remains is to set up a fastlane _lane_ that runs our build for us, and configure GitHub Workflows to invoke it:

1. Create `.github/workflows/publish_ios.yml` in your Flutter repository from the [reference publish_ios.yml in the example project](https://raw.githubusercontent.com/jorgenpt/flutter_github_example/blogpost-testflight/.github/workflows/publish_ios.yml).
1. Create `ios/fastlane/Fastfile` in your Flutter repository from the [reference Fastfile in the example project](https://raw.githubusercontent.com/jorgenpt/flutter_github_example/blogpost-testflight/ios/fastlane/Fastfile).
1. Update `ios/fastlane/Fastfile` with your project details:
    1. `APP_IDENTIFIER` is the _Bundle Identifier_ we created in _Setup with Apple_.
    1. `APPSTORECONNECT_ISSUER_ID` is the _Issuer ID_ from the _Keys_ section of [_Users and Access_ on App Store Connect](https://appstoreconnect.apple.com/access/users).
    1. `APPSTORECONNECT_KEY_ID` is the _Key ID_ from the specific key that we created in _Configuring App Store Connect API access for fastlane_, which you can look up in the _Keys_ section of [_Users and Access_ on App Store Connect](https://appstoreconnect.apple.com/access/users).

That's it! Now just create a git tag and push it to GitHub to start a build:

```sh
git tag v0.1.0 # This accepts any tag name  starting with "v"
git push --tags
```

You can see an example run [in the Actions tab of `flutter_github_example`](https://github.com/jorgenpt/flutter_github_example/actions/workflows/publish_ios.yml).

These builds will by default be available to _Internal Testers_ -- you can use the App Store Connect app or website to release one of these builds to _External Testers_.