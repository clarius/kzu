---
layout: post
title: "Smarter Visual Studio Template Authoring"
date: 2014-01-27 18:53:00 +0000
tags: [ddd, ef, eventsourcing, extensibility, gadgets, mocking, moq, msbuild, mvc, nuget, patterns, programming, t4, technology, vsx, wcf, webapi]
---

##  [Smarter Visual Studio Template Authoring](http://blogs.clariusconsulting.net/kzu/smarter-visual-studio-template-authoring/ "Smarter Visual Studio Template Authoring")

January 27, 2014 6:53 pm

From [Clarius VisualStudio Targets](https://github.com/clariuslabs/VisualStudio/wiki/Authoring-Templates) project:

The out of the box experience for authoring templates in Visual Studio is a [multi-step process](http://msdn.microsoft.com/en-us/magazine/hh456399.aspx) that isn’t very amenable to incremental improvements. An exported .zip file does not encourage rapid iteration ![;\)](https://web.archive.org/web/20210127025837im_/http://blogs.clariusconsulting.net/kzu/wp-includes/images/smilies/icon_wink.gif) .

An alternative way introduced in VS2010 was a new [project template](http://msdn.microsoft.com/en-us/library/ff527340\(v=vs.100).aspx) to generate a project (or item) template output (its compressed zip). Yup, if that sounds weird, it’s because it is. It’s a project where you don’t compile anything, but rather just work with the files with [parameter substitutions](http://msdn.microsoft.com/en-us/library/s365byhx\(v=vs.100).aspx) and the like, and where building means just zipping it. If you have just one template to distribute with your VSIX, that may be ok.

For more advanced scenarios, where reusing shared artifacts (i.e. same AssemblyInfo.cs, shared images, etc.) and having a faster build cycle is needed, [Clarius.VisualStudio](https://www.nuget.org/packages/Clarius.VisualStudio) offers much smarter template authoring. We call these templates "Smart VS Templates".

## [](https://github.com/clariuslabs/VisualStudio/wiki/Authoring-Templates#wiki-getting-started)Getting Started

The starting point after installing the [NuGet package](https://www.nuget.org/packages/Clarius.VisualStudio) is to include your template content files (you can unzip them from the inital exported template) in the VSIX project, and set the BuildAction to SmartVSTemplate on the .vstemplate file:

![](https://web.archive.org/web/20210127025837im_/https://github.com/clariuslabs/VisualStudio/wiki/media/smart-template-build-action.png)

> You may need to close and reopen the solution if the SmartVSTemplate build action does not show up as a valid BuildAction

Just make sure that the template content files which are code files aren’t added as Compile or compilation will fail (because of the parameter replacement tokens like $safeprojectname$ and the like).

Remember to add the project template’s root directory to the source.extension.vsixmanifest. If you placed your template in the `T\PT\Glass Application` directory in your project, the manifest content element would be:
    
    
    <Content>
        <ProjectTemplate>T\PT</ProjectTemplate>
    </Content>
    

> Note that this is the manifest format for VS2010 extensions. We recommend sticking to the VS2010 format, since VS2012+ use a newer schema but also understands this older (and arguably simpler) format. This keeps your extension compatible with VS2010+.
> 
> Note also that we strongly advise using abbreviated folder names for Templates\ProjectTemplates, using T (for Templates) and PT (for ProjectTemplates) or IT (for ItemTemplates). This will ensure you never hit long path errors on users’ machines.

With that in place, your template will show up under the root category for your template language (i.e. "Visual C#" root node).

![](https://web.archive.org/web/20210127025837im_/https://github.com/clariuslabs/VisualStudio/wiki/media/template-without-category.png)

## [](https://github.com/clariuslabs/VisualStudio/wiki/Authoring-Templates#wiki-template-category)Template Category

Typically you’ll want to have your own category node within the Add New dialog for your templates, especially if you provide many. Smart VS Templates makes this straightforward. Just move your content one (or many!) levels down in the folder hierarchy, and VS will make those the nested categories for your templates.

For example, if we move a `Glass Application` folder within`T\PT\Wearables\Google Glass\Glass Application` and rebuild, we will end up with the following:

![](https://web.archive.org/web/20210127025837im_/https://github.com/clariuslabs/VisualStudio/wiki/media/template-custom-category.png)

## [](https://github.com/clariuslabs/VisualStudio/wiki/Authoring-Templates#wiki-shared-artifacts)Shared Artifacts

If you deploy multiple templates, chances are that a big chunk of them are common artifacts, like AssemblyInfo.cs, maybe readme and images, etc. Rather than maintaining multiple copies of the same files and keeping them in sync, Smart VS Templates allow you to just include shared content right from the .vstemplate node. To get shared assets in the same .zip file as the template, edit the project file and add metadata to the SmartVSTemplate node as follows:
    
    
    <SmartVSTemplate Include="T\PT\Wereables\Google Glass\Google Glass Application\GlassApplication.vstemplate">
        <Include>..\..\Shared\**\*.*</Include>
    </SmartVSTemplate>
    

The paths in the new `<Include>` metadata node are relative to the .vstemplate path. Here we’re telling the Smart VS Template to include everything that’s two levels up under the Shared folder, recursively. We could also just include one single file if we wanted.

Multiple inclusions are supported, by separating the paths with semi-colon:
    
    
    <SmartVSTemplate Include="T\PT\Wereables\Google Glass\Google Glass Application\GlassApplication.vstemplate">
        <Include>..\Glass\Readme.txt;..\..\CSharp\*.cs;..\..\Shared\**\*.*</Include>
    </SmartVSTemplate>
    

Here we’re including a specific text file for (say) all Glass projects, then all code files (non-recursively) from the CSharp folder an extra level up, and then all the common shared artifacts we had before.

This is significantly more maintainable and flexible than anything the out of the box Visual Studio templates provide.

/kzu
