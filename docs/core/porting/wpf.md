---
title: Port a WPF app to .NET Core 3.0
description: Teaches you how to port a .NET Framework Windows Presentation Foundation application to .NET Core 3.0 for Windows.
author: Thraka
ms.author: adegeo
ms.date: 03/27/2019
ms.custom: 
---

# How to: Port a WPF desktop app to .NET Core

This article describes how to port your Windows Presentation Foundation-based (WPF) desktop app from .NET Framework to .NET Core 3.0. The .NET Core 3.0 SDK includes support for WPF applications. WPF is still a Windows-only framework and only runs on Windows. This example uses the .NET Core SDK CLI to create and manage your project.

In this article, various names are used to identify types of files used for migration. When migrating your project, your files will be named differently, so mentally match them to the ones listed below:

| File | Description |
| ---- | ----------- |
| **MyApps.sln** | The name of the solution file. |
| **MyWPF.csproj** | The name of the .NET Framework WPF project to port. |
| **MyWPFCore.csproj** | The name of the new .NET Core project you create. |
| **MyAppCore.exe** | The .NET Core WPF app executable. |

## Prerequisites

- [Visual Studio 2019](https://visualstudio.microsoft.com/vs/preview/?utm_medium=microsoft&utm_source=docs.microsoft.com&utm_campaign=inline+link&utm_content=wpf+core) for any designer work you want to do.

  Install the following Visual Studio workloads:
  - .NET desktop development
  - .NET cross-platform development

- A working WPF project in a solution that builds and runs without issue.
- Your project must be coded in C#. 
- Install the latest [.NET Core 3.0](https://aka.ms/netcore3download) preview.

>[!NOTE]
>**Visual Studio 2017** doesn't support .NET Core 3.0 projects. **Visual Studio 2019 Preview/RC** supports .NET Core 3.0 projects but doesn't yet support the visual designer for .NET Core 3.0 WPF projects. To use the visual designer, you must have a .NET WPF project in your solution that shares its files with the .NET Core project.

### Consider

When porting a .NET Framework WPF application, there are a few things you must consider.

01. Check that your application is a good candidate for migration.

    Use the [.NET Portability Analyzer](../../standard/analyzers/portability-analyzer.md) to determine if your project will migrate to .NET Core 3.0. If your project has issues with .NET Core 3.0, the analyzer helps you identify those problems.

01. You're using a different version of WPF.

    When .NET Core 3.0 Preview 1 was released, WPF went open-source on GitHub. The code for .NET Core WPF is a fork of the .NET Framework WPF code base. It's possible some differences exist and your app won't port.

01. The [Windows Compatibility Pack][compat-pack] may help you migrate.

    Some APIs that are available in .NET Framework aren't available in .NET Core 3.0. The [Windows Compatibility Pack][compat-pack] adds many of these APIs and may help your WPF app become compatible with .NET Core.

01. Update the NuGet packages used by your project.

    It's always a good practice to use the latest versions of NuGet packages before any migration. If your application is referencing any NuGet packages, update them to the latest version. Ensure your application builds successfully. After upgrading, if there are any package errors, downgrade the package to the latest version that doesn't break your code.

01. Visual Studio 2019 Preview/RC doesn't yet support the WPF Designer for .NET Core 3.0

    Currently, you need to keep your existing .NET Framework WPF project file if you want to use the WPF Designer from Visual Studio.

## Create a new SDK project

The new .NET Core 3.0 project you create must be in a different directory from your .NET Framework project. If they're both in the same directory, you may run into conflicts with the files that are generated in the **obj** directory. In this example, you'll create a directory named **MyWPFAppCore** in the **SolutionFolder** directory:

```
SolutionFolder
├───MyApps.sln
├───MyWPFApp
│   └───MyWPF.csproj
└───MyWPFAppCore      <--- New folder for core project
```

Next, you need to create the **MyWPFCore.csproj** project in the **MyWPFAppCore** directory. You can create this file manually by using the text editor of choice. Paste in the following XML:

```xml
<Project Sdk="Microsoft.NET.Sdk.WindowsDesktop">

  <PropertyGroup>
    <OutputType>WinExe</OutputType>
    <TargetFramework>netcoreapp3.0</TargetFramework>
    <UseWPF>true</UseWPF>
  </PropertyGroup>

</Project>
```

If you don't want to create the project file manually, you can use Visual Studio or the .NET Core SDK to generate the project. However, you must delete all other files generated by the project template except for the project file. To use the SDK, run the following command from the **SolutionFolder** directory:

```cli
dotnet new wpf -o MyWPFAppCore -n MyWPFCore
```

After you create the **MyWPFCore.csproj**, your directory structure should look like the following:

```
SolutionFolder
├───MyApps.sln
├───MyWPFApp
│   └───MyWPF.csproj
└───MyWPFAppCore
    └───MyWPFCore.csproj
```

You'll want to add the **MyWPFCore.csproj** project to **MyApps.sln** with either Visual Studio or the .NET Core CLI from the **SolutionFolder** directory:

```cli
dotnet sln add .\MyWPFAppCore\MyWPFCore.csproj
```

## Fix assembly info generation

Windows Presentation Foundation projects that were created with .NET Framework include an `AssemblyInfo.cs` file, which contains assembly attributes such as the version of the assembly to be generated. SDK-style projects automatically generate this information for you based on the SDK project file. Having both types of "assembly info" creates a conflict. Resolve this problem by disabling automatic generation, which forces the project to use your existing `AssemblyInfo.cs` file.

There are three settings to add to the main `<PropertyGroup>` node. 

- **GenerateAssemblyInfo**\
When you set this property to `false`, it won't generate the assembly attributes. This avoids the conflict with the existing `AssemblyInfo.cs` file from the .NET Framework project.

- **AssemblyName**\
The value of this property is the output binary created when you compile. The name doesn't need an extension added to it. For example, using `MyCoreApp` produces `MyCoreApp.exe`.

- **RootNamespace**\
The default namespace used by your project. This should match the default namespace of the .NET Framework project.

Add these three elements to the `<PropertyGroup>` node in the `MyWPFCore.csproj` file:

```xml
<Project Sdk="Microsoft.NET.Sdk.WindowsDesktop">

  <PropertyGroup>
    <OutputType>WinExe</OutputType>
    <TargetFramework>netcoreapp3.0</TargetFramework>
    <UseWPF>true</UseWPF>

    <GenerateAssemblyInfo>false</GenerateAssemblyInfo>
    <AssemblyName>MyCoreApp</AssemblyName>
    <RootNamespace>MyWPF</RootNamespace>
  </PropertyGroup>

</Project>
```

## Add source code
Right now, the **MyWPFCore.csproj** project doesn't compile any code. By default, .NET Core projects automatically include all source code in the current directory and any child directories. You must configure the project to include code from the .NET Framework project using a relative path. If your .NET Framework project used **.resx** files for icons and resources for your windows and controls, you'll need to include those too. 

The first `<ItemGroup>` node you need to add to your project includes the **App.xaml** file that represents the startup config and resources your app uses. The **App.xaml** file also has an accompanying **App.xaml.cs** file, but it will be automatically included in a different `<ItemGroup>`.

```xml
  <ItemGroup>
    <ApplicationDefinition Include="..\MyWPFApp\App.xaml">
      <Generator>MSBuild:Compile</Generator>
    </ApplicationDefinition>
  </ItemGroup>
```

Next, add the following `<ItemGroup>` node to your project. Each statement includes a file glob pattern that includes child directories. It includes the source code for your project, any settings files, and any resources. The **obj** directory is explicitly excluded.

```xml
  <ItemGroup>
    <Compile Include="..\MyWPFApp\**\*.cs" Exclude="..\MyWPFApp\obj\**" />
    <None Include="..\MyWPFApp\**\*.settings" />
    <EmbeddedResource Include="..\MyWPFApp\**\*.resx" />
  </ItemGroup>
```

Next, include another `<ItemGroup>` node that contains a `<Page>` entry for every **xaml** file in your project except the **App.xaml** file. These contain all of the windows, pages, and resources that are in **xaml** format. You cannot use a glob pattern here and must add an entry for every file and indicate the `<Generator>` used.

```xml
  <ItemGroup>
    <Page Include="..\MyWPFApp\MainWindow.xaml">
      <Generator>MSBuild:Compile</Generator>
    </Page>
  </ItemGroup>
```

## Add NuGet packages

Add each NuGet package referenced by the .NET Framework project to the .NET Core project. 

Most likely your .NET Framework WPF app has a **packages.config** file that contains a list of all of the NuGet packages that are referenced by your project. You can look at this list to determine which NuGet packages to add to the .NET Core project. For example, if the .NET Framework project references the `MahApps.Metro` NuGet package, add it to the project with Visual Studio. You can also add the package reference with the .NET Core CLI from the **SolutionFolder** directory:

```cli
dotnet add .\MyWPFAppCore\MyWPFCore.csproj package MahApps.Metro -v 2.0.0-alpha0262
```

The previous command would add the following NuGet reference to the **MyWPFCore.csproj** project:

```xml
  <ItemGroup>
    <PackageReference Include="MahApps.Metro" Version="2.0.0-alpha0262" />
  </ItemGroup>
```

## Problems compiling

If you have problems compiling your projects, you may be using some Windows-only APIs that are available in .NET Framework but not available in .NET Core. You can try adding the [Windows Compatibility Pack][compat-pack] NuGet package to your project. This package only runs on Windows and adds about 20,000 Windows APIs to .NET Core and .NET Standard projects.

```cli
dotnet add .\MyWPFAppCore\MyWPFCore.csproj package Microsoft.Windows.Compatibility
```

The previous command adds the following to the **MyWPFCore.csproj** project:

```xml
  <ItemGroup>
    <PackageReference Include="Microsoft.Windows.Compatibility" Version="2.0.1" />
  </ItemGroup>
```

## WPF Designer

As detailed in this article, Visual Studio 2019 Preview/RC only supports the WPF Designer in .NET Framework projects. By creating a side-by-side .NET Core project, you can test your project with .NET Core while you use the .NET Framework project to design forms. Your solution file includes both the .NET Framework and .NET Core projects. Add and design your forms and controls in the .NET Framework project, and based on the file glob patterns we added to the .NET Core projects, any new or changed files will automatically be included in the .NET Core projects.

Once Visual Studio 2019 supports the WPF Designer, you can copy/paste the content of your .NET Core project file into the .NET Framework project file. Then delete the file glob patterns added with the `<Source>` and `<EmbeddedResource>` items. Fix the paths to any project reference used by your app. This effectively upgrades the .NET Framework project to a .NET Core project.
 
## Next steps

* Read more about the [Windows Compatibility Pack][compat-pack].
* Watch a [video on porting](https://www.youtube.com/watch?v=5MomsgkWkVw&list=PLS__JrkRveTMiWxG-Lv4cBwYfMQ6m2gmt) your .NET Framework WPF project to .NET Core.

[compat-pack]: windows-compat-pack.md