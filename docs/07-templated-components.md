# Templated components

Let's refactor some of original components and make them more reusable. Along the way we'll also create a separate library project as a home for the new components.

## Creating a component library (Visual Studio)

*If you're using the `dotnet` command line to create projects, skip ahead to the next section.*

To make a new project using **Visual Studio**, start with selecting the new ASP.NET Core Web Application options. Call the project `BlazingComponents`.

![image](https://user-images.githubusercontent.com/1430011/51865279-ac560e00-22fa-11e9-9beb-366d3723730c.png)

In the next dialog select the **Razor Class Library** option. There currently is not a dedicated Blazor Class Libary template, so we'll modify this one in the next step..

![image](https://user-images.githubusercontent.com/1430011/51865502-4a49d880-22fb-11e9-8196-bfc830edd314.png)

*If you're using **Visual Studio** you can safely proceed to the section "Modifying the library project".*

## Creating a component library (Visual Studio)

To make a new project using **dotnet** run the following commands from the directory where your solution file exists.

```
dotnet new razorclasslib -o BlazingComponents
dotnet sln add BlazingComponents
```

This should create a new project called `BlazingComponents` and add it to the solution file. There currently is not a dedicated Blazor Class Libary template, so we'll modify this one in the next step.

## Modifying the library project

Next, we can delete the existing content of the project as we won't use it. Delete the `Areas` folder and everything it contains.

Now, we will edit the project file and turn it into a Blazor library. Open the project file via *right click* -> *Edit BlazingComponents.csproj*.

Edit the `BlazingComponents.csproj` until it looks like the below:

```xml
<Project Sdk="Microsoft.NET.Sdk.Razor">

  <PropertyGroup>
    <TargetFramework>netstandard2.0</TargetFramework>
    <LangVersion>7.3</LangVersion>
    <IsPackable>true</IsPackable>
  </PropertyGroup>

  <ItemGroup>
    <!-- .js/.css files will be referenced via <script>/<link> tags; other content files will just be included in the app's 'dist' directory without any tags referencing them -->
    <EmbeddedResource Include="content\**\*.js" LogicalName="blazor:js:%(RecursiveDir)%(Filename)%(Extension)" />
    <EmbeddedResource Include="content\**\*.css" LogicalName="blazor:css:%(RecursiveDir)%(Filename)%(Extension)" />
    <EmbeddedResource Include="content\**" Exclude="**\*.js;**\*.css" LogicalName="blazor:file:%(RecursiveDir)%(Filename)%(Extension)" />
  </ItemGroup>  
  <ItemGroup>
    <PackageReference Include="Microsoft.AspNetCore.Blazor.Browser" Version="0.7.0" />
    <PackageReference Include="Microsoft.AspNetCore.Blazor.Build" Version="0.7.0" PrivateAssets="all" />
  </ItemGroup>

</Project>
```

There are a few things here worth understanding. 

Firstly, it's recommended that all Blazor project targets C# version 7.3 or newer (`<LangVersion>7.3</LangVersion>`), Blazor relies on some features added in version 7.3 to make event handlers work correctly.

Next, the `<IsPackable>true</IsPackable>` line makes it possible the create a NuGet package from this project. We won't be using this project as a package in this example, but this is a good thing to have for a class library.

Next, the lines that look like `<EmbeddedResource ... />` give the class library project special handling of content files that should be included in the project. This makes it easier to do multi-project development with static assests, and to redistibute libraries containing static assets. We'll see this in action later.

Lastly the two `<PackageReference />` elements add package references to the Blazor runtime libraries and the Blazor build support.

At this point the library is a now a fully-fledged *Blazor class library*, in the following steps we will add some components.

## Writing a templated dialog

We are going to revisit the dialog system that is part of `Index` and turn it into something that's decoupled from the application.

Let's think about how a *reusable dialog* should work. We would expect a dialog component to handle showing and hiding itself, as well as maybe styling to appear visually as a dialog. However, to be truly reusable, we need to be able to provide the content for the inside of the dialog. We call a component that accepts *content* as a parameter a *templated component*.

Blazor happens to have a feature that works for exactly this case, it's similar to how a layout works. Recall that a layout has a `Body` parameter, and the layout gets to place other content *around* the `Body`. In a layout, the `Body` parameter is of type `RenderFragment` which is a delegate type that the runtim has special handling for. The good news is that this feature is not limited layouts. Any component can declare a parameter of type `RenderFragment`.

Let's get started on this new dialog component. Create a new component file named `TemplatedDialog.cshtml` in the `BlazingComponents` project. Put the following markup inside `TemplatedDialog.cshtml`:

```html
<div class="dialog-container">
    <div class="dialog">

    </div>
</div>
```

This doesn't do anything yet because we haven't added any parameters. Recall from before the two things we want to accomplish.
1. Accept the content of the dialog as a parameter
1. Render the dialog conditionally if it is supposed to be shown

First, let's add a parameter called `ChildContent` of type `RenderFragment`. The name `ChildContent` is a special parameter name, and is used by convention when a component wants to accept a single content parameter. Next, update the markup to *render* the `ChildContent` in the middle of the markup. It should look like this:

```html
<div class="dialog-container">
    <div class="dialog">
        @ChildContent
    </div>
</div>
```

If this form looks weird to you, cross-check it with your layout file, which follows a similar pattern. Even though `RenderFragment` is a delegate type the way to *render* it not be invoking it, it's by placing the value in a normal expression so the runtime may invoke it.

Next, to give this dialog some conditional behavior, let's add a parameter of type `bool` called `Show`. After doing that, it's time to wrap all of the existing content in an `@if (Show) { ... }`. The full file should look like this:

```html
@if (Show)
{
    <div class="dialog-container">
        <div class="dialog">
            @ChildContent
        </div>
    </div>
}

@functions {
    [Parameter] RenderFragment ChildContent { get; set; }
    [Parameter] bool Show { get; set; }
}
```

Do build and make sure that everything compiles at this stage. Next we'll get down to using this new component.

## Adding a reference to the templated library

Before we can use this component in the `BlazingPizza.Client` project, we will need to add a project reference. Do this by adding a project reference from `BlazingPizza.Client` to `BlazingComponents`.

Once that's done, there's one more minor step. Open the `_ViewImports.cshtml` in the topmost directory of `BlazingPizza.Client` and add this line at the end:
```
@addTagHelper "*, BlazingComponents"
```

`@addTagHelper` is there to inform the compiler that it should consider the `BlazingComponents` library as a source for components that can be used. This is a very temporary thing that's currently needed. We expect this to be removed in the future. 

Now that the project reference has been added, do a build again to verify that every still compiles.

## Using the new dialog

We'll use this new component from `Index.cshtml`. Open the `Index.cshtml` and find the block of code that looks like:
```
@if (OrderState.ShowingConfigureDialog)
{
    <ConfigurePizzaDialog />
}
```

We are going to remove this and replace it with an invocation of the new component. Replace the block above with code like the following:
```
<TemplatedDialog Show="OrderState.ShowingConfigureDialog">
  <ConfigurePizzaDialog />
</TemplatedDialog>
```

This is wiring up our new `TemplatedDialog` component to show and hide itself based on `OrderState.ShowingConfigureDialog`. Also, we're passing in some content to the `ChildContent` paramter. Since we called the parameter `ChildContent` any content that is placed inside the `<TemplatedDialog> </TemplatedDialog>` will be captured by a `RenderFragment` delegate and passed to `TemplatedDialog`. 

note: A templated component may have multiple `RenderFragment` parameters, what we're showing here is a convenient convention when the caller wants to provide a single `RenderFragment` that represents the *main* content.

Recall that our `TemplatedDialog` contains a few `div`s. Well, this duplicates some of the structure of `ConfigurePizzaDialog`. Let's clean that up. Open `ConfigurePizzaDialog.cshtml`; it currently looks like:
```html
<div class="dialog-container">
    <div class="dialog">
        <div class="dialog-title">
        ...
        </div>
        <form class="dialog-body">
        ...
        </form>

        <div class="dialog-buttons">
        ...
        </div>
    </div>
</div>
```

We should remove the outermost two layers of `div` elements since those are now part of the `TemplatedDialog` component. After removing these it should look more like:
```html
<div class="dialog-title">
...
</div>
<form class="dialog-body">
...
</form>

<div class="dialog-buttons">
...
</div>
```

