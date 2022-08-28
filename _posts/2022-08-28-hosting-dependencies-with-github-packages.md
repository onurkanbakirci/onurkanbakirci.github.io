---
layout: post
title: Hosting dependencies with GitHub Packages
#subtitle: .NET Ortamında Threading - 1
#cover-img: /assets/img/dotnet.png
thumbnail-img: /assets/img/posts/github-packages/1.png
share-img: /assets/img/posts/github-packages/1.png
tags: [github, githubpackages, nuget, dependencies]
---

In this story, we try to experience packaging and deploying our custom library to Github Packages.

First thing first, why we need Github Packages? Generally we use Docker Hub, npm, NuGet or other third party package registries to store and manage our package or distribute our public packages for other developers usage. By using GitHub Packages, we centralize our software development in one place. We can also integrate GitHub Packages with GitHub APIs, GitHub Actions, and webhooks to create an end-to-end DevOps workflow.

GitHub Packages offers different package registries such as;

- Docker registry
- RubyGems registry
- npm registry
- NuGet registry
- Apache Maven registry
- Gradle Registry

In this writing, i create simple NuGet package for demonstration purpose. Let’s create empty `C#` library project.

![](/assets/img/posts/github-packages/2.png)

After that, we define our functionalities inside of `Human.cs` file.

![](/assets/img/posts/github-packages/3.png)

After these steps, i create `.nuspec` file in order to define our package informations inside of it.

![](/assets/img/posts/github-packages/4.png)

It’s content is as following

<p><script src="https://gist.github.com/onurkanbakirci/23ca929e7bd10d592673fa69c680f417.js"></script></p>

We have to indicate that our project uses `nuspec` file. For this indication, open project file and give to `nuspec` file path.

![](/assets/img/posts/github-packages/5.png)

We need to define our Github credentials to connect to Github. For this step, let’s create `nuget.config` file in our solution path.

![](/assets/img/posts/github-packages/6.png)

Inside of `nuget.config` , we need to define our Github credentials


<p><script src="https://gist.github.com/onurkanbakirci/2642413f327d323e8379a9ef1e8e6e09.js"></script></p>

You have to add Github token generated from Github account page to `ClearTextPassword` field inside of `nuget.config` file.

After these configurations, let’s generate our custom nuget package. For that, in the project path respectively

- `dotnet build` command for creating `bin` and `obj` folders.
- `dotnet pack` command for generating nupkg known as NuGet package.(There are also options like `output` and so on.)

![](/assets/img/posts/github-packages/7.png)

After these commands, our package is generated inside bin/Debug path(because we didn’t define specific output directory).

![](/assets/img/posts/github-packages/8.png)

Let’s publish generated package to Github Packages. In solution folder, open command line.

```
dotnet nuget push ./Example.GithubPackages.NugetPackage/bin/Debug/Example.GithubPackages.NugetPackage.1.0.0.nupkg --source "github"
```

In this command, we specify where is the package and which source we use.

![](/assets/img/posts/github-packages/9.png)

As you can see, our package is pushed to Github Packages. Let’s prove that from Github Dashboard.

![](/assets/img/posts/github-packages/10.png)

{: .box-note}
Don’t push `nuget.config` file to repo because of security vulnerabilities

So, we can use this packages in our other projects. For demonstration, let’s create empty console application and add our custom package from package manager like following

![](/assets/img/posts/github-packages/11.png)

We need to change source as github (because we define our new github source in nuget.config file) to see our custom packages.

![](/assets/img/posts/github-packages/12.png)

And that’s it. Our package is ready for use. After installation, we can use `namespaces, classes, functions etc.` from our lib project.

![](/assets/img/posts/github-packages/13.png)

Source codes : https://github.com/onurkanbakirci/GithubPackages




