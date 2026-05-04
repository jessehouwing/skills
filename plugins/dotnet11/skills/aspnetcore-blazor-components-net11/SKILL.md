---
name: aspnetcore-blazor-components-net11
description: >
  Covers new .NET 11 APIs across Microsoft.AspNetCore.Components,
  Components.Web, and Components.Endpoints. Includes navigation improvements
  (GetUriWithHash, RelativeToCurrentUri), contravariant RenderFragment<in T>,
  IComponentPropertyActivator, form components (Label, DisplayName), media
  components (Image, Video, FileDownload with MediaSource), EnvironmentBoundary,
  BasePath, ITempData, and TempDataProviderType. Use when building Blazor apps
  that need navigation, forms, media rendering, environment-conditional content,
  or cross-request temporary data in .NET 11.
license: MIT
---

# ASP.NET Core Blazor Components — .NET 11 APIs

Target framework: `net11.0`

---

## Navigation Improvements
Assembly: `Microsoft.AspNetCore.Components`

### GetUriWithHash

Extension method returning the current URI with a hash fragment appended.

```csharp
public static string GetUriWithHash(this NavigationManager nav, string hash);
```

```razor
@inject NavigationManager Nav

<button @onclick="GoToSection">Jump to Section</button>

@code {
    private void GoToSection()
    {
        string uri = Nav.GetUriWithHash("#section1");
        Nav.NavigateTo(uri);
    }
}
```

### NavigationOptions.RelativeToCurrentUri

New `bool` init-only property. When `true`, the target URI resolves relative
to the current URI rather than the base URI.

```razor
@inject NavigationManager Nav

@code {
    private void GoToSettings()
    {
        // From /app/dashboard → /app/settings
        Nav.NavigateTo("../settings", new NavigationOptions
        {
            RelativeToCurrentUri = true
        });
    }
}
```

### NavLink.RelativeToCurrentUri

| Type | Default | Description |
|------|---------|-------------|
| `bool` | `false` | When `true`, matching is relative to current URI |

```razor
<NavLink href="settings/profile" RelativeToCurrentUri="true"
         Match="NavLinkMatch.Prefix">
    Profile Settings
</NavLink>
```

---

## Component Model
Assembly: `Microsoft.AspNetCore.Components`

### IComponentPropertyActivator

Interface for custom component property activation.

```csharp
public interface IComponentPropertyActivator
{
    Action<IServiceProvider, IComponent> GetActivator(Type type);
}
```

### Contravariant RenderFragment&lt;in TValue&gt;

`RenderFragment<TValue>` now uses `in` variance, enabling assignment
compatibility with more derived types.

```csharp
RenderFragment<Animal> animalFragment = (animal) => builder =>
{
    builder.AddContent(0, $"Animal: {animal.Name}");
};

// Valid due to contravariance
RenderFragment<Dog> dogFragment = animalFragment;
```

---

## Form Components
Assembly: `Microsoft.AspNetCore.Components.Web`

### Label&lt;TValue&gt;

Renders `<label>` with `for` auto-bound to the target input's `id`.

| Parameter | Type | Description |
|-----------|------|-------------|
| `For` | `Expression<Func<TValue>>` | Field expression |
| `ChildContent` | `RenderFragment?` | Label text/markup |
| `AdditionalAttributes` | `IDictionary<string, object>?` | Extra HTML attributes |

### DisplayName&lt;TValue&gt;

Renders the display name from `[Display]` or `[DisplayName]` annotations.

### InputBase&lt;TValue&gt;.IdAttributeValue

Protected property exposing the computed `id` attribute value.

### Example

```razor
<EditForm Model="@person" OnValidSubmit="Save">
    <Label For="@(() => person.Name)">Full Name:</Label>
    <InputText @bind-Value="person.Name" />

    <Label For="@(() => person.Email)">
        <DisplayName For="@(() => person.Email)" />
    </Label>
    <InputText @bind-Value="person.Email" />
</EditForm>

@code {
    private Person person = new();
    private void Save() { }

    public class Person
    {
        [Display(Name = "Email Address")]
        public string Email { get; set; } = "";
        public string Name { get; set; } = "";
    }
}
```

---

## Media Components
Assembly: `Microsoft.AspNetCore.Components.Web`
Namespace: `Microsoft.AspNetCore.Components.Web.Media`

### MediaSource

Wraps binary data with a MIME type and cache key.

| Member | Type | Description |
|--------|------|-------------|
| Constructor | `(byte[], string, string)` | Data, MIME type, cache key |
| Constructor | `(Stream, string, string)` | Stream, MIME type, cache key |
| `Stream` | `Stream` | Underlying data stream |
| `MimeType` | `string` | MIME type |
| `CacheKey` | `string` | Client-side cache key |
| `Length` | `long` | Data length in bytes |

### Image, Video, FileDownload Components

- **`Image`** — Renders `<img>` from a `MediaSource`
- **`Video`** — Renders `<video>` from a `MediaSource`
- **`FileDownload`** — Triggers browser file download

| Parameter (FileDownload) | Type | Description |
|--------------------------|------|-------------|
| `Source` | `MediaSource` (required) | File data |
| `FileName` | `string` (required) | Download file name |
| `Text` | `string?` | Button/link text |

### Example: Image from Byte Array

```razor
@using Microsoft.AspNetCore.Components.Web.Media

<Image Source="@imageSource" alt="Profile photo" width="200" />

@code {
    private MediaSource? imageSource;

    protected override async Task OnInitializedAsync()
    {
        byte[] bytes = await LoadImageBytesAsync();
        imageSource = new MediaSource(bytes, "image/png", "profile-photo");
    }
}
```

### Example: FileDownload

```razor
<FileDownload Source="@reportSource" FileName="report.pdf"
              Text="Download Report" />

@code {
    private MediaSource? reportSource;

    protected override void OnInitialized()
    {
        byte[] data = GenerateReport();
        reportSource = new MediaSource(data, "application/pdf", "monthly-report");
    }
}
```

---

## EnvironmentBoundary Component
Assembly: `Microsoft.AspNetCore.Components.Web`

Conditionally renders content based on the hosting environment.

| Parameter | Type | Description |
|-----------|------|-------------|
| `Include` | `string?` | Comma-separated environments to render in |
| `Exclude` | `string?` | Comma-separated environments to hide in |
| `ChildContent` | `RenderFragment?` | Content to conditionally render |

```razor
<EnvironmentBoundary Include="Development">
    <div class="alert alert-info">Debug Mode: diagnostics enabled.</div>
</EnvironmentBoundary>

<EnvironmentBoundary Exclude="Production">
    <p>This content is hidden in production.</p>
</EnvironmentBoundary>
```

---

## BasePath & TempData
Assembly: `Microsoft.AspNetCore.Components.Endpoints`

### BasePath Component
A sealed component that sets the Blazor app base path in SSR.

```razor
<head>
    <BasePath />
    <HeadOutlet />
</head>
```

### ITempData Interface
Dictionary-based temporary data that survives a single subsequent request.

| Member | Description |
|--------|-------------|
| `this[string key]` | Gets or sets a temp data value |
| `Get(string key)` | Gets and marks for deletion |
| `Peek(string key)` | Gets without marking for deletion |
| `Keep(string key)` | Retains value for next request |

### TempDataProviderType Enum
```csharp
public enum TempDataProviderType { Cookie, SessionStorage }
```

### RazorComponentsServiceOptions
| Property | Type | Description |
|----------|------|-------------|
| `TempDataCookie` | `CookieBuilder` | Cookie configuration |
| `TempDataProviderType` | `TempDataProviderType` | Storage mechanism |

### Example: Configure TempData

```csharp
builder.Services.AddRazorComponents(options =>
{
    options.TempDataProviderType = TempDataProviderType.SessionStorage;
}).AddInteractiveServerComponents();
```

### Example: Use ITempData in Components

```razor
@page "/checkout"
@inject ITempData TempData

@code {
    private void SaveAndNavigate()
    {
        TempData["OrderId"] = "ORD-12345";
        TempData["Message"] = "Order placed successfully!";
        Nav.NavigateTo("/confirmation");
    }
}
```
