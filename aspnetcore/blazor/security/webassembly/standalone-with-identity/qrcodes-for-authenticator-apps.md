---
title: Enable QR code generation for TOTP authenticator apps in ASP.NET Core Blazor WebAssembly with ASP.NET Core Identity
author: guardrex
description: Learn how to configure an ASP.NET Core Blazor WebAssembly app with Identity for QR code generation with TOTP authenticator app.
ms.author: riande
monikerRange: '>= aspnetcore-8.0'
ms.date: 11/21/2024
uid: blazor/security/webassembly/standalone-with-identity/qrcodes-for-authenticator-apps
---
# Enable QR code generation for TOTP authenticator apps in ASP.NET Core Blazor WebAssembly with ASP.NET Core Identity

[!INCLUDE[](~/includes/not-latest-version-without-not-supported-content.md)]

This article explains how to configure an ASP.NET Core Blazor WebAssembly app with Identity with QR code generation for Time-based One-time Password Algorithm (TOTP) authenticator apps.

> [!NOTE]
> This article only applies standalone Blazor WebAssembly apps with a server backend web API that adopts ASP.NET Core Identity. To implement QR code generation for Blazor Web Apps, see <xref:blazor/security/qrcodes-for-authenticator-apps>.

For an introduction to two-factor authentication (2FA) using a TOTP authenticator app, see <xref:security/authentication/identity-enable-qrcodes>.

> [!WARNING]
> TOTP codes should be kept secret because it can be used to authenticate successfully multiple times before it expires.

## Namespaces and article code examples

The namespaces used by the examples in this article are:

* `Backend` for the backend server web API project ("server project" in this article).
* `BlazorWasmAuth` for the front-end client standalone Blazor WebAssembly app ("client project" in this article).

These namespaces correspond to the projects in the `BlazorWebAssemblyStandaloneWithIdentity` sample solution in the [`dotnet/blazor-samples` GitHub repository](https://github.com/dotnet/blazor-samples). For more information, see <xref:blazor/security/webassembly/standalone-with-identity/index#sample-apps>.

If you aren't using the `BlazorWebAssemblyStandaloneWithIdentity` sample, change the namespaces in the code examples to use the namespaces of your projects.

All of the changes to the solution covered by this article take place in the `BlazorWasmAuth` project.

In this article's code examples, the code lines are artificially broken across two or more lines to eliminate or reduce horizontal scrolling of the article's code blocks. The code executes regardless of these artificial line breaks. You're welcome to condense the code in your own apps by removing the artificial line breaks after you paste the code into a project.

## Optional account confirmation and password recovery

Although apps that implement 2FA authentication often adopt account confirmation and password recovery features, 2FA authentication doesn't require it. The guidance in this article can be followed to implement 2FA without following the guidance in <xref:blazor/security/webassembly/standalone-with-identity/account-confirmation-and-password-recovery>.

## Add a QR code library to the app

QR codes for use by TOTP authenticator apps must be generated by a QR code library.

The guidance in this article uses [`nimiq/qr-creator`](https://github.com/nimiq/qr-creator), but you can use any general QR code generation library.

Download the [`qr-creator.min.js`](https://cdn.jsdelivr.net/npm/qr-creator/dist/qr-creator.min.js) library to the `wwwroot` folder of the client project. The library has no dependencies. If you intend to use the library to generate other QR codes elsewhere in the app for your own purposes, there's also a module version of the library available from the maintainer (see the [`nimiq/qr-creator`](https://github.com/nimiq/qr-creator) GitHub repository for details).

In the client project's `wwwroot/index.html` file, add the following `<script>` tag immediately after the [Blazor's `<script>` tag](xref:blazor/project-structure#location-of-the-blazor-script):

```html
<script src="qr-creator.min.js"></script>
```

## Set the TOTP organization name

Set the site name in the app settings file of the client project. Use a meaningful site name that users can identify easily in their authenticator app. Developers usually set a site name that matches the company's name. Examples: Yahoo, Amazon, Etsy, Microsoft, Zoho. We recommend limiting the site name length to 30 characters or less to allow the site name to display on narrow mobile device screens.

In the following example, the the company name is `Weyland-Yutani Corporation` (&copy;1986 20th Century Studios [*Aliens*](https://www.20thcenturystudios.com/movies/aliens)).

Added to `wwwroot/appsettings.json`:

```json
"TotpOrganizationName": "Weyland-Yutani Corporation"
```

The file after the TOTP organization name configuration is added:

```json
{
  "BackendUrl": "https://localhost:7211",
  "FrontendUrl": "https://localhost:7171",
  "TotpOrganizationName": "Weyland-Yutani Corporation"
}
```

## Add a model class for two-factor results

Add the following `TwoFactorResult` class to the `Models` folder. This class is populated by the response to a 2FA request made to the `/manage/2fa` endpoint of <xref:Microsoft.AspNetCore.Routing.IdentityApiEndpointRouteBuilderExtensions.MapIdentityApi%2A> in the server app.

`Identity/Models/TwoFactorResult.cs`:

```csharp
namespace BlazorWasmAuth.Identity.Models
{
    public class TwoFactorResult
    {
        public string SharedKey { get; set; } = string.Empty;
        public int RecoveryCodesLeft { get; set; } = 0;
        public string[] RecoveryCodes { get; set; } = [];
        public bool IsTwoFactorEnabled { get; set; }
        public bool IsMachineRemembered { get; set; }
        public string[] ErrorList { get; set; } = [];
    }
}
```

## `IAccountManagement` interface

Add the following class signatures to the `IAccountManagement` interface. The class signatures represent methods added to the cookie authentication state provider for the following client requests:

* Log in with a 2FA TOTP code (`/login` endpoint): `LoginTwoFactorCodeAsync`
* Log in with a 2FA recovery code (`/login` endpoint): `LoginTwoFactorRecoveryCodeAsync`
* Make a 2FA management request (`/manage/2fa` endpoint): `TwoFactorRequest`

`Identity/IAccountManagement.cs` (paste the following code at the bottom of the interface):

```csharp
public Task<FormResult> LoginTwoFactorCodeAsync(
    string email, 
    string password, 
    string twoFactorCode);

public Task<FormResult> LoginTwoFactorRecoveryCodeAsync(
    string email, 
    string password, 
    string twoFactorRecoveryCode);

public Task<TwoFactorResult> TwoFactorRequest(
    bool enable = false, 
    string twoFactorCode = "", 
    bool resetSharedKey = false, 
    bool resetRecoveryCodes = false, 
    bool forgetMachine = false);
```

## Update the cookie authentication state provider

Update the `CookieAuthenticationStateProvider` with features to add the following features:

* Authenticate users with either a TOTP authenticator app code or a recovery code.
* Manage 2FA in the app.

Add a class to hold deserialized response content for logging users into the app.

Add the following `HttpResponseContent` class to `Identity/CookieAuthenticationStateProvider.cs`:

```csharp
private class HttpResponseContent
{
    public string? Type { get; set; }
    public string? Title { get; set; }
    public int Status {  get; set; }
    public string? Detail {  get; set; }
}
```

The `LoginAsync` method is updated with the following logic:

* Attempts a normal login at the `/login` endpoint with an email address (user ID) and password.
* If the server responds with a success status code, the method returns a `FormResult` with the `Succeeded` property set to `true`.
* If the server responds with *401 - Unauthorized* status code and a detail code of "`RequiresTwoFactor`," a `FormResult` is returned with `Succeeded` set to `false` and the `RequiresTwoFactor` detail in the error list.

In `Identity/CookieAuthenticationStateProvider.cs`, replace the `LoginAsync` method with the following code:

```csharp
public async Task<FormResult> LoginAsync(string email, string password)
{
    try
    {
        var result = await httpClient.PostAsJsonAsync(
            "login?useCookies=true", new
            {
                email,
                password
            });

        if (result.IsSuccessStatusCode)
        {
            NotifyAuthenticationStateChanged(GetAuthenticationStateAsync());

            return new FormResult { Succeeded = true };
        }
        else if (result.StatusCode == HttpStatusCode.Unauthorized)
        {
            var responseJson = await result.Content.ReadAsStringAsync();
            var response = JsonSerializer.Deserialize<HttpResponseContent>(
                responseJson, jsonSerializerOptions);

            if (response?.Detail == "RequiresTwoFactor")
            {
                return new FormResult
                {
                    Succeeded = false,
                    ErrorList = [ "RequiresTwoFactor" ]
                };
            }
        }
    }
    catch { }

    return new FormResult
    {
        Succeeded = false,
        ErrorList = [ "Invalid email and/or password." ]
    };
}
```

A `LoginTwoFactorCodeAsync` method is added, which sends a request to the `/login` endpoint with a 2FA TOTP code (`twoFactorCode`). Otherwise, the method processes the response in a similar fashion to a normal, non-2FA login request.

Add the following method and class to `Identity/CookieAuthenticationStateProvider.cs` (paste the following code at the bottom of the class file):

```csharp
public async Task<FormResult> LoginTwoFactorCodeAsync(
    string email, string password, string twoFactorCode)
{
    try
    {
        var result = await httpClient.PostAsJsonAsync(
            "login?useCookies=true", new
            {
                email,
                password,
                twoFactorCode
            });

        if (result.IsSuccessStatusCode)
        {
            NotifyAuthenticationStateChanged(GetAuthenticationStateAsync());

            return new FormResult { Succeeded = true };
        }
    }
    catch { }

    return new FormResult
    {
        Succeeded = false,
        ErrorList = [ "Invalid email, password, or two-factor code."]
    };
}
```

A `LoginTwoFactorRecoveryCodeAsync` method is added, which sends a request to the `/login` endpoint with a 2FA recovery code (`twoFactorRecoveryCode`). Otherwise, the method processes the response in a similar fashion to a normal, non-2FA login request.

Add the following method and class to `Identity/CookieAuthenticationStateProvider.cs` (paste the following code at the bottom of the class file):

```csharp
public async Task<FormResult> LoginTwoFactorRecoveryCodeAsync(string email, 
    string password, string twoFactorRecoveryCode)
{
    try
    {
        var result = await httpClient.PostAsJsonAsync(
            "login?useCookies=true", new
            {
                email,
                password,
                twoFactorRecoveryCode
            });

        if (result.IsSuccessStatusCode)
        {
            NotifyAuthenticationStateChanged(GetAuthenticationStateAsync());

            return new FormResult { Succeeded = true };
        }
    }
    catch { }

    return new FormResult
    {
        Succeeded = false,
        ErrorList = [ "Invalid email, password, or two-factor code." ]
    };
}
```

A `TwoFactorRequest` method is added, which is used to manage 2FA for the user:

* Reset the shared 2FA key.
* Reset the user's recovery codes.
* Forget the machine, which means that a new 2FA TOTP code is required on the next login attempt.
* Enable 2FA using a 2FA code from a TOTP authenticator app.
* Obtain 2FA status with an empty request.

Add the following method and class to `Identity/CookieAuthenticationStateProvider.cs` (paste the following code at the bottom of the class file):

```csharp
public async Task<TwoFactorResult> TwoFactorRequest(
    bool enable,
    string twoFactorCode,
    bool resetSharedKey,
    bool resetRecoveryCodes,
    bool forgetMachine)
{
    string[] defaultDetail = 
        [ "An unknown error prevented two-factor authentication." ];

    try
    {
        HttpResponseMessage response;

        if (resetSharedKey)
        {
            response = await httpClient.PostAsJsonAsync("manage/2fa", 
                new { enable, resetSharedKey });
        }
        else if (resetRecoveryCodes)
        {
            response = await httpClient.PostAsJsonAsync("manage/2fa",
                new { enable, resetRecoveryCodes });
        }
        else if (forgetMachine)
        {
            response = await httpClient.PostAsJsonAsync("manage/2fa", 
                new { enable, forgetMachine });
        }
        else if (!string.IsNullOrEmpty(twoFactorCode))
        {
            response = await httpClient.PostAsJsonAsync("manage/2fa", 
                new { enable, twoFactorCode });
        }
        else
        {
            response = await httpClient.PostAsJsonAsync("manage/2fa", 
                new { });
        }

        if (response.IsSuccessStatusCode)
        {
            return await response.Content
                .ReadFromJsonAsync<TwoFactorResult>() ??
                new()
                { 
                    ErrorList = 
                        [ "There was an error processing the request." ]
                };
        }

        var details = await response.Content.ReadAsStringAsync();
        var problemDetails = JsonDocument.Parse(details);
        var errors = new List<string>();
        var errorList = problemDetails.RootElement.GetProperty("errors");

        foreach (var errorEntry in errorList.EnumerateObject())
        {
            if (errorEntry.Value.ValueKind == JsonValueKind.String)
            {
                errors.Add(errorEntry.Value.GetString()!);
            }
            else if (errorEntry.Value.ValueKind == JsonValueKind.Array)
            {
                errors.AddRange(
                    errorEntry.Value.EnumerateArray().Select(
                        e => e.GetString() ?? string.Empty)
                    .Where(e => !string.IsNullOrEmpty(e)));
            }
        }

        return new TwoFactorResult
        {
            ErrorList = problemDetails == null ? defaultDetail : [.. errors]
        };
    }
    catch { }

    return new TwoFactorResult
    {
        ErrorList = [ "There was an error processing the request." ]
    };
}
```

## Replace `Login` component

Replace the `Login` component. The following version of the `Login` component:

* Accepts a user's email address (user ID) and password for an initial login attempt.
* If login is successful (2FA is diabled), the component notifies the user that they're authenticated.
* If the login attempt results in a response indicating that 2FA is required, a 2FA input element appears to receive either a 2FA TOTP code from an authenticator app or a recovery code. Depending on which code the user enters, login is attempted again by calling either `LoginTwoFactorCodeAsync` for a TOTP code or `LoginTwoFactorRecoveryCodeAsync` for a recovery code.

`Components/Identity/Login.razor`:

```razor
@page "/login"
@using System.ComponentModel.DataAnnotations
@using BlazorWasmAuth.Identity
@using BlazorWasmAuth.Identity.Models
@inject IAccountManagement Acct
@inject ILogger<Login> Logger
@inject NavigationManager Navigation

<PageTitle>Login</PageTitle>

<h1>Login</h1>

<AuthorizeView>
    <Authorized>
        <div class="alert alert-success">
            You're logged in as @context.User.Identity?.Name.
        </div>
    </Authorized>
    <NotAuthorized>
        @foreach (var error in formResult.ErrorList)
        {
            <div class="alert alert-danger">@error</div>
        }
        <div class="row">
            <div class="col">
                <section>
                    <EditForm Model="Input" method="post" OnValidSubmit="LoginUser" 
                            FormName="login" Context="editform_context">
                        <DataAnnotationsValidator />
                        <h2>Use a local account to log in.</h2>
                        <hr />
                        <div style="display:@(requiresTwoFactor ? "none" : "block")">
                            <div class="form-floating mb-3">
                                <InputText @bind-Value="Input.Email" 
                                    id="Input.Email" class="form-control" 
                                    autocomplete="username" aria-required="true" 
                                    placeholder="name@example.com" />
                                <label for="Input.Email" class="form-label">
                                    Email
                                </label>
                                <ValidationMessage For="() => Input.Email" 
                                    class="text-danger" />
                            </div>
                            <div class="form-floating mb-3">
                                <InputText type="password" 
                                    @bind-Value="Input.Password" id="Input.Password" 
                                    class="form-control" autocomplete="current-password" 
                                    aria-required="true" placeholder="password" />
                                <label for="Input.Password" class="form-label">
                                    Password
                                </label>
                                <ValidationMessage For="() => Input.Password" 
                                    class="text-danger" />
                            </div>
                        </div>
                        <div style="display:@(requiresTwoFactor ? "block" : "none")">
                            <div class="form-floating mb-3">
                                <InputText @bind-Value="Input.TwoFactorCode" 
                                    id="Input.TwoFactorCode" class="form-control" 
                                    autocomplete="off" 
                                    placeholder="###### or #####-#####" />
                                <label for="Input.TwoFactorCode" class="form-label">
                                    2FA Authenticator Code (######) or Recovery 
                                    Code (#####-#####, dash required)
                                </label>
                                <ValidationMessage For="() => Input.TwoFactorCode" 
                                    class="text-danger" />
                            </div>
                        </div>
                        <div>
                            <button type="submit" class="w-100 btn btn-lg btn-primary">
                                Log in
                            </button>
                        </div>
                        <div>
                            <p>
                                <a href="forgot-password">Forgot password</a>
                            </p>
                            <p>
                                <a href="register">Register as a new user</a>
                            </p>
                        </div>
                    </EditForm>
                </section>
            </div>
        </div>
    </NotAuthorized>
</AuthorizeView>

@code {
    private FormResult formResult = new();
    private bool requiresTwoFactor;

    [SupplyParameterFromForm]
    private InputModel Input { get; set; } = new();

    [SupplyParameterFromQuery]
    private string? ReturnUrl { get; set; }

    public async Task LoginUser()
    {
        if (requiresTwoFactor)
        {
            if (!string.IsNullOrEmpty(Input.TwoFactorCode))
            {
                if (Input.TwoFactorCode.Length == 6)
                {
                    formResult = await Acct.LoginTwoFactorCodeAsync(
                        Input.Email, Input.Password, Input.TwoFactorCode);
                }
                else
                {
                    formResult = await Acct.LoginTwoFactorRecoveryCodeAsync(
                        Input.Email, Input.Password, Input.TwoFactorCode);
                }
            }
        }
        else
        {
            formResult = await Acct.LoginAsync(Input.Email, Input.Password);
            requiresTwoFactor = formResult.ErrorList.Contains("RequiresTwoFactor");
            Input.TwoFactorCode = string.Empty;
            formResult.ErrorList = [];
        }

        if (formResult.Succeeded && !string.IsNullOrEmpty(ReturnUrl))
        {
            Navigation.NavigateTo(ReturnUrl);
        }
    }

    private sealed class InputModel
    {
        [Required]
        [EmailAddress]
        [Display(Name = "Email")]
        public string Email { get; set; } = string.Empty;

        [Required]
        [DataType(DataType.Password)]
        [Display(Name = "Password")]
        public string Password { get; set; } = string.Empty;

        [Required]
        [RegularExpression(@"^([0-9]{6})|([A-Z0-9]{5}[-]{1}[A-Z0-9]{5})$", 
            ErrorMessage = "Must be a six-digit authenticator code (######) or " +
            "eleven-character alphanumeric recovery code (#####-#####, dash " +
            "required)")]
        [Display(Name = "Two-factor code")]
        public string TwoFactorCode { get; set; } = "123456";
    }
}
```

## Add a component to display recovery codes

Add the following `ShowRecoveryCodes` component to the app to display recovery codes to the user.

`Components/Identity/ShowRecoveryCodes.razor`:

```razor
<h3>Recovery codes</h3>

<div class="alert alert-warning" role="alert">
    <p>
        <strong>Put these codes in a safe place.</strong>
    </p>
    <p>
        If you lose your device and don't have an unused 
        recovery code, you can't access your account.
    </p>
</div>
<div class="row">
    <div class="col-md-12">
        @foreach (var recoveryCode in RecoveryCodes)
        {
            <div>
                <code class="recovery-code">@recoveryCode</code>
            </div>
        }
    </div>
</div>

@code {
    [Parameter]
    public string[] RecoveryCodes { get; set; } = [];
}
```

## Manage 2FA page

Add the following `Manage2fa` component to the app to manage 2FA for users.

If 2FA isn't enabled, the component loads a form with a QR code to enable 2FA with a TOTP authenticator app. The user adds the app to their authenticator app and then verifies the authenticator app and enables 2FA by providing a TOTP code from the authenticator app.

If 2FA is enabled, a button appears to disable 2FA.

<!-- NOTE TO REVIEWERS!

     I'm not discussing recovery code generation in this text yet
     because I'm not clear on how that's supposed to work because
     submitting such a request doesn't just re-gen the recovery 
     codes ... it also disables 2FA. That might be a bug in the
     API, but it's more likely that I don't understand the logic
     of recovery code re-gen. I've sent an email to discuss
     this point.

     What we might be doing for this is simply removing the
     recovery code re-gen (including from the cookie auth state
     provider code method for `/manage/2fa` management). 
     Leave it so that the codes are shown once when 2FA is 
     enabled, and then don't permit re-gen from the app. If the 
     user hasn't resolved their 2FA/TOTP app login situation 
     (new phone, new TOTP app, etc.) and they've used all of 
     their recovery codes, then they're simply locked out until 
     customer service/IT reactivates their account.

     BTW ... The 'forget machine' piece also doesn't seem
     particularly relevant. Machines are always remembered by
     the current API AFAICT. Perhaps, the only way to require
     TOTP every login attempt is to call for forgetting the 
     machine after every successful login ... not sure ... this
     subject will be my follow-up question on the email convo.
-->

`Components/Identity/Manage2fa.razor`:

```razor
@page "/manage-2fa"
@using System.ComponentModel.DataAnnotations
@using System.Globalization
@using System.Text
@using System.Text.Encodings.Web
@using BlazorWasmAuth.Identity
@using BlazorWasmAuth.Identity.Models
@attribute [Authorize]
@implements IAsyncDisposable
@inject IAccountManagement Acct
@inject IAuthorizationService AuthorizationService
@inject IConfiguration Config
@inject IJSRuntime JS
@inject ILogger<Manage2fa> Logger

<PageTitle>Manage 2FA</PageTitle>

<h1>Manage Two-factor Authentication</h1>
<hr />
<div class="row">
    <div class="col">
        @if (twoFactorResult is not null)
        {
            @foreach (var error in twoFactorResult.ErrorList)
            {
                <div class="alert alert-danger">@error</div>
            }

            @if (twoFactorResult.IsTwoFactorEnabled)
            {
                <div class="alert alert-success" role="alert">
                    Two-factor authentication is enabled for your account.
                </div>

                <div class="m-1">
                    <button @onclick="Disable2FA" class="btn btn-lg btn-primary">
                        Disable 2FA
                    </button>
                </div>

                @if (recoveryCodes is null)
                {
                    <div class="m-1">
                        <button @onclick="GenerateNewCodes" 
                                class="btn btn-lg btn-primary">
                            Generate New Recovery Codes
                        </button>
                    </div>
                }
                else
                {
                    <ShowRecoveryCodes 
                        RecoveryCodes="twoFactorResult.RecoveryCodes.ToArray()" />
                }
            }
            else
            {
                <h3>Configure authenticator app</h3>
                <div>
                    <p>To use an authenticator app:</p>
                    <ol class="list">
                        <li>
                            <p>
                                Download a two-factor authenticator app, such 
                                as either of the following:
                                <ul>
                                    <li>
                                        Microsoft Authenticator for
                                        <a href="https://go.microsoft.com/fwlink/?Linkid=825072">
                                            Android
                                        </a> and
                                        <a href="https://go.microsoft.com/fwlink/?Linkid=825073">
                                            iOS
                                        </a>
                                    </li>
                                    <li>
                                        Google Authenticator for
                                        <a href="https://play.google.com/store/apps/details?id=com.google.android.apps.authenticator2">
                                            Android
                                        </a> and
                                        <a href="https://itunes.apple.com/us/app/google-authenticator/id388497605?mt=8">
                                            iOS
                                        </a>
                                    </li>
                                </ul>
                            </p>
                        </li>
                        <li>
                            <p>
                                Scan the QR Code or enter this key 
                                <kbd>@twoFactorResult.SharedKey</kbd> into your 
                                two factor authenticator app. Spaces and casing 
                                don't matter.
                            </p>
                            <div @ref="qrCodeElement"></div>
                        </li>
                        <li>
                            <p>
                                Once you have scanned the QR code or input the 
                                key above, your two factor authentication app 
                                will provide you with a unique code. Enter the 
                                code in the confirmation box below.
                            </p>
                            <div class="row">
                                <div class="col-xl-6">
                                    <EditForm 
                                    Model="Input" 
                                    FormName="send-code" 
                                    OnValidSubmit="OnValidSubmitAsync" 
                                    method="post">
                                        <DataAnnotationsValidator />
                                        <div class="form-floating mb-3">
                                            <InputText 
                                                @bind-Value="Input.Code" 
                                                id="Input.Code" 
                                                class="form-control" 
                                                autocomplete="off" 
                                                placeholder="Enter the code" />
                                            <label for="Input.Code" 
                                                class="control-label form-label">
                                                Verification Code
                                            </label>
                                            <ValidationMessage 
                                                For="() => Input.Code" 
                                                class="text-danger" />
                                        </div>
                                        <button type="submit" class="w-100 btn btn-lg btn-primary">
                                            Verify
                                        </button>
                                    </EditForm>
                                </div>
                            </div>
                        </li>
                    </ol>
                </div>
            }
        }
    </div>
</div>

@code {
    private IEnumerable<string>? recoveryCodes;
    private IJSObjectReference? module;
    private TwoFactorResult twoFactorResult = new();
    private ElementReference qrCodeElement;

    [SupplyParameterFromForm]
    private InputModel Input { get; set; } = new();

    [CascadingParameter]
    private Task<AuthenticationState>? authenticationState { get; set; }

    protected override async Task OnInitializedAsync()
    {
        twoFactorResult = await Acct.TwoFactorRequest();
    }

    protected override async Task OnAfterRenderAsync(bool firstRender)
    {
        if (firstRender)
        {
            module = await JS.InvokeAsync<IJSObjectReference>("import",
                "./Components/Identity/Manage2fa.razor.js");
        }

        if (authenticationState is not null &&
            !string.IsNullOrEmpty(twoFactorResult?.SharedKey) &&
            module is not null)
        {
            var authState = await authenticationState;
            var email = authState?.User?.Identity?.Name!;

            var uri = string.Format(
                CultureInfo.InvariantCulture,
                "otpauth://totp/{0}:{1}?secret={2}&issuer={0}&digits=6",
                UrlEncoder.Default.Encode(Config["TotpOrganizationName"]!),
                email,
                twoFactorResult?.SharedKey);

            await module.InvokeVoidAsync("setQrCode", qrCodeElement, uri);
        }
    }

    private async Task Disable2FA()
    {
        twoFactorResult = 
            await Acct.TwoFactorRequest(enable: false, resetSharedKey: true);
    }

    private async Task GenerateNewCodes()
    {
        twoFactorResult = 
            await Acct.TwoFactorRequest(enable: false, resetRecoveryCodes: true);
    }

    private async Task OnValidSubmitAsync()
    {
        twoFactorResult = await Acct.TwoFactorRequest(enable: true, 
            twoFactorCode: Input.Code);
        recoveryCodes = twoFactorResult.RecoveryCodes;
    }

    private sealed class InputModel
    {
        [Required]
        [RegularExpression(@"^([0-9]{6})$", 
            ErrorMessage = "Must be a six-digit authenticator code (######)")]
        [DataType(DataType.Text)]
        [Display(Name = "Verification Code")]
        public string Code { get; set; } = string.Empty;
    }

    async ValueTask IAsyncDisposable.DisposeAsync()
    {
        if (module is not null)
        {
            await module.DisposeAsync();
        }
    }
}
```

Add the following [collocated JavaScript file](xref:blazor/js-interop/javascript-location#load-a-script-from-an-external-javascript-file-js-collocated-with-a-component) to the project. The `setQrCode` function uses [`nimiq/qr-creator`](https://github.com/nimiq/qr-creator) to create a QR code from the URI string if the `qrCode` element is rendered by the `Manage2fa` component. To customize features of the generated QR code, see the library maintainer's documentation at their GitHub repository.

`Components/Identity/Manage2fa.razor.js`:

```javascript
export function setQrCode(qrCodeElement, uri) {
  if (qrCodeElement !== null) {
    QrCreator.render({
      text: uri,
      radius: 0,
      ecLevel: 'H',
      fill: '#000000',
      background: null,
      size: 190
    }, qrCodeElement);
  }
}
```

## Link to the the Manage 2FA page

Add a link to the navigation menu for users to reach the `Manage2fa` component page.

In the `<Authorized>` content of the `<AuthorizeView>` in `Components/Layout/NavMenu.razor`, add the following markup:

```diff
<AuthorizeView>
    <Authorized>

        ...

+       <div class="nav-item px-3">
+           <NavLink class="nav-link" href="manage-2fa">
+               <span class="bi bi-key" aria-hidden="true"></span> Manage 2FA
+           </NavLink>
+       </div>

        ...

    </Authorized>
</AuthorizeView>
```

## Additional resources

* [Mandrill.net (GitHub repository)](https://github.com/feinoujc/Mandrill.net)
* [Mailchimp developer: Transactional API](https://mailchimp.com/developer/transactional/docs/fundamentals/)