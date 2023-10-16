---
title: 1. Using NuGet packages as plugins
images: []
---

# 1. Using NuGet packages as plugins

This post will be describing my journey to get a sensible plugin orchestration and delivery mechanism in place for the Stride game engine.
My main goal is to use NuGet architecture as much as possible and deliver a user experience that doesn't require you to be a genius, nor has you copy-pasting things around too much. And because plugins are only meant for build time, and may target something else than your game, they cannot be directly referenced the usual way.

Stride integrates with MSBuild in a few ways. The GameStudio is acting a bit like Visual Studio in that it allows you to add projects to your solution,
builds the projects for you when you make code changes, etc.
So when thinking about a plugin architecture I immediately thought about using NuGet package references as a way to deliver plugins and resolve their dependencies.
Stride already performs NuGet restore and parses the `package.assets.json` file to browse the dependencies of your projects.
What is needed now is a system that allows us to detect plugins and load them into memory of the editor.

## Detecting something is a plugin

Initially I thought it may be enough to have the user mark a PackageReference with `StridePlugin` property.
However, this requires the user to explicitly know what is a plugin and what isn't.
And there's an issue with my ease-of-use goal. If the user references a runtime package, say `Stride.Physics`, we would expect that
the plugin package `Stride.Physics.Design` would be automatically referenced as it is necessary to include it to process physics assets.

So I went online and I found [PackageType](https://learn.microsoft.com/en-us/nuget/create-packages/set-package-type?tabs=dotnet#custom-package-types)
concept which enables us to declare something is a Stride plugin when creating the NuGet package.
It's great and it also gives us the ability to filter NuGet.org for packages with the specified type - e.g. <https://www.nuget.org/packages?packageType=StridePlugin> - which is a huge win for a mechanism of discovering plugins.

This is great, however, currently package type is only used by NuGet SDK when installing package first time into your project to detect that for example your not referencing a .NET tool as dependency and it doesn't preserve this information for later.
So I have opened an issue on NuGet repo to ask for adding support for it ([#12934](https://github.com/NuGet/Home/issues/12934)).
In the meantime I foresee a potential workaround where a target in `Stride.Core` checks PackageType of your project and if it matches the plugin type
it emits a new file into your package for which we can look after your package is unpacked in the nuget cache.

Ok, so given a dependency tree where some deps are plugins we can probably detect the plugins and select them for loading.

## Circular dependencies

So I mentioned earlier how the author of `Stride.Physics` should be able to declare that plugin package `Stride.Physics.Design` needs to be included for the user.
How do the dependencies look like?

```
User project -> Stride.Physics -?-> Stride.Physics.Design -> Stride.Physics
```

Ok, so in 90% of plugins I'm aiming at, there will be a runtime lib and a design lib and the design lib has to reference runtime lib to generate data for the runtime lib to consume.

NuGet doesn't allow circular dependencies. You won't even create the package (if using regular tooling).

So I started thinking. And the idea was good but ultimately doesn't work. But this blog post is about exploring ideas so let's go.

### Indirect reference idea

Let's simplify names - `X` is runtime lib, `X.D` is design time lib.

* `X.D` references `X` via ProjectReference which will be converted into a package dependency when packed.
* `X.D` marks itself as a Stride plugin.
* `X` notes `<RequiredStridePlugin Include="Plugin" Version="1.0.0" />` in its project
* When `X` is packed we create a `build/X.targets` file which will be included in the user project that declares items `IncludeStridePlugin` matching the requirement.
* When user project referencing `X` is built we convert `IncludeStridePlugin` into `PackageReference`

I learned a bit more about MSBuild while working on this. Especially trying to answer the question how to select the maximum version from the possibly duplicate containing list of `IncludeStridePlugin` items. And the answer was [target batching](https://learn.microsoft.com/en-us/visualstudio/msbuild/item-metadata-in-target-batching?view=vs-2022).

Here's my code which was placed in `Stride.Core` targets:

```xml
<!--
First the creator of a plugin will include the following in their runtime project:
  
<ItemGroup>
  <RequiredStridePlugin Include="MyPlugin" Version="1.0.0" />
</ItemGroup>
  
This target will generate a RequiredStridePlugin.targets file in the buildTransitive folder of the package,
so that the consumers of the runtime package will have the following added to their project:
  
<ItemGroup>
  <IncludeStridePlugin Include="MyPlugin" Version="1.0.0" />
</ItemGroup>
-->
<Target Name="_StrideConfigureRequiredPluginsInPackage" BeforeTargets="Build" Condition="@(RequiredStridePlugin->Count()) &gt; 0">
  <ItemGroup>
    <_StrideRequiredPlugin Include="@(RequiredStridePlugin)" />
  </ItemGroup>
    
  <WriteLinesToFile
    File="$(IntermediateOutputPath)$(PackageId).targets"
    Lines="&lt;Project&gt;;@(_StrideRequiredPlugin->'&lt;ItemGroup&gt;&lt;IncludeStridePlugin Include=&quot;%(Identity)&quot; Version=&quot;%(Version)&quot; /&gt;&lt;/ItemGroup&gt;');&lt;/Project&gt;"
    Overwrite="true" />
    
  <ItemGroup>
    <!-- TODO: how can we allow the user to add their own targets? Using PackageId as name of the file is required for it to be loaded.
      see https://github.com/NuGet/docs.microsoft.com-nuget/blob/main/docs/reference/errors-and-warnings/NU5129.md -->
    <None Include="$(IntermediateOutputPath)$(PackageId).targets" PackagePath="build\" Pack="true" />
    <None Include="$(IntermediateOutputPath)$(PackageId).targets" PackagePath="buildTransitive\" Pack="true" />
  </ItemGroup>
</Target>

<!--
Based on IncludeStridePlugin declared we will try to load the latest version of the plugin.
This only matters if two referenced library require the same plugin but with different versions.
  
<IncludeStridePlugin Include="X" Version="1.2.0" />
<IncludeStridePlugin Include="X" Version="1.4.0" />
becomes
<_StridePluginReference Include="X" Version="1.4.0" />
-->
<!-- We use the Inputs and Outputs to allow Target batching and group IncludeStridePlugin by package name -->
<Target Name="_StrideIncludePluginReferencesByHighestVersion" BeforeTargets="CollectPackageReferences" Inputs="@(IncludeStridePlugin)" Outputs="__%(Identity)">
  <!-- for each Version update _TempStridePluginVersion if Version > _TempStridePluginVersion -->
  <CreateProperty Value="%(IncludeStridePlugin.Version)" Condition="'$(_TempStridePluginVersion)' == '' OR $([System.Version]::Parse(%(IncludeStridePlugin.Version))) &gt; $([System.Version]::Parse($(_TempStridePluginVersion)))">
    <Output TaskParameter="Value" PropertyName="_TempStridePluginVersion" />
  </CreateProperty>
  <!-- include the plugin reference with the greatest version -->
  <ItemGroup>
    <_StridePluginReference Include="%(IncludeStridePlugin.Identity)" Version="$(_TempStridePluginVersion)" />
  </ItemGroup>
  <!-- unset property for the next target invocation -->
  <CreateProperty Value="">
    <Output TaskParameter="Value" PropertyName="_TempStridePluginVersion" />
  </CreateProperty>
</Target>

<!--
For each _StridePluginReference we will add a PackageReference if not already set by the user (allows user to override the version).
Needs to be invoked before CollectPackageReferences target for the packages to be picked up for build correctly.
We only include build assets from the package to avoid the plugin being used for compilation.
-->
<Target Name="_StrideIncludePluginReferencesFromPackages" AfterTargets="_StrideIncludePluginReferencesByHighestVersion" BeforeTargets="CollectPackageReferences" Condition="@(_StridePluginReference->Count()) &gt; 0">
  <ItemGroup>
    <PackageReference Include="@(_StridePluginReference)" Exclude="@(PackageReference)" IncludeAssets="build;buildTransitive" />
  </ItemGroup>
</Target>
```

So why doesn't it work?

Well, the issue is simple - you can't run targets from other packages during Restore because they're not restored yet.
And if you cannot add the `PackageReference` before the restore then it's not restored, it's not in the dependency graph, not populated in `project.assets.json` and not taken into account during build.

Moreover, I found [this](https://github.com/NuGet/docs.microsoft.com-nuget/blob/main/docs/concepts/MSBuild-props-and-targets.md#guidance-for-the-content-of-msbuild-props-and-targets)

> There are a few things that must not be done in packages' .props and .targets, such as not specifying properties and items that affect restore, as those will be automatically excluded.
> - Some examples of items that must not be added or updated: PackageReference, PackageVersion, PackageDownload, etc.

So it looks like we probably can't get away with indirectly including package references just yet.

### Making plugins not have dependencies

Something I haven't considered too much but crossed my mind - what if we enforce `PrivateAssets=all` on any dependency of the plugin, thus making it not have dependencies at all, thus none of them will be circular? Will need to think more.

## Next?
This post will be updated over time. For the main issue spawning these ideas look at [Stride Plugin RFC](https://github.com/stride3d/stride/issues/1120).
