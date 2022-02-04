---
published: true
---
# Consume ABP API from Xamarin App

If you are using the [ABP Framework ](https://abp.io "ABP Framework ") and want to authenticate and consume it in your Xamarin Forms app, please read further where I will give a short guide on how to implement the changes needed in your application.

I will not go in much detail about the actual Xamarin code implementation neither the ABP implementation, as this is a simple example of how it can be done.

Nor can I be held responsible for any bugs, breaches or unaware use of packages.

The code shown in for the Xamarin app is by inspiration from another project that I am working on.

## Requirements
* Existing ABP API project with IdentityServer
* Existing Xamarin Forms application

## Setup API

First we need to setup our backend project, to enable another client, that is allowed to consume the API endpoints and thus make requests. Since we are using our YourProject.DbMigrator project to setup clients initially, we also need to add the new client from it's appsettings.

Remember to change "YourProject" with the right name of your project ;-)

**YourProject.DbMigrator/appsettings.json**

Add the below code to the "IdentityServer" "Clients" section:

```json
"YourProject_Xamarin": {
  "ClientId": "YourProject_Xamarin",
  "ClientSecret": "1q2w3e*",
  "RootUrl": "https://localhost:44377"
  // RootUrl = "AuthServer:Authority": "https://localhost:44349" in appsettings.json HttpApi.Host project
}
```

**YourProject.Domain/IdentityServer/IdentityServerDataSeedContributor.cs**

Add the below code after the last "Swagger Client", as shown below:

```csharp
// Xamarin Client
var xamarinClientId = configurationSection["YourProjectApi_Xamarin:ClientId"];
if (!xamarinClientId.IsNullOrWhiteSpace())
{
  var xamarinRootUrl = configurationSection["YourProjectApi_Xamarin:RootUrl"].TrimEnd('/');

  await CreateClientAsync(
    name: xamarinClientId,
    scopes: commonScopes,
    grantTypes: new[] { "authorization_code", "password" },
    secret: configurationSection["YourProjectApi_Xamarin:ClientSecret"]?.Sha256(),
    requireClientSecret: false,
    redirectUri: "xamarinformsclients://callback",
    corsOrigins: new[] { xamarinRootUrl.RemovePostFix("/") }
  );
}
```

The important part is the entry "password" in "grantTypes" array.

Be sure to run your DbMigrator project to update your database with the new client defined above. Else our Xamarin app will not be able to authenticate and use the API endpoints.

## Setup Xamarin App

Remember to change "YourProject" with the right name of your project ;-)

Let's add 2 entry fields and a login button to the MainPage.xaml:

**YourApp/MainPage.xaml**
```xml
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage xmlns="http://xamarin.com/schemas/2014/forms"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             x:Class="YourApp.MainPage">

    <StackLayout>
        <Entry Text="{Binding UserName, Mode=TwoWay}" 
               Placeholder="Email" />
            
        <Entry Text="{Binding Password, Mode=TwoWay}" 
               Placeholder="Password" 
               IsPassword="True"/>
        <Button Command="{Binding LoginCommand}" 
                Text="Login" />
    </StackLayout>

</ContentPage>
  
```

In the xaml codebehind class file please setup the BindingContext for our ViewModel

**YourApp/MainPage.xaml.cs**
```csharp
using YourApp.ViewModels;
using Xamarin.Forms;

namespace YourApp
{
    public partial class MainPage : ContentPage
    {
        private readonly MainViewModel _viewModel;
        public MainPage()
        {
            InitializeComponent();
            BindingContext = _viewModel = new MainViewModel();
        }
    }
}

```

Now let's create the MainViewModel class in "ViewModels" folder:


**YourApp/ViewModels/MainViewModel.cs**

```csharp
using System.Threading.Tasks;
using YourApp.ViewModels;
using YourApp.Services.Identity;
using Xamarin.Forms;

namespace YourApp.ViewModels
{
    public class MainViewModel : BaseViewModel
    {
        
        private string _userName;
        private string _password;
        private bool _loading;
        
        IIdentityService IdentityService => DependencyService.Get<IIdentityService>();
        
        public Command LoginCommand { get; }

        public MainViewModel()
        {
            LoginCommand = new Command(async () => await OnLoginClicked());
        }
        
        public string UserName
        {
            get => _userName;
            set => SetProperty(ref _userName, value);
        }

        public string Password
        {
            get => _password;
            set => SetProperty(ref _password, value);
        }
        
        private async Task OnLoginClicked()
        {
            var loginResult = await IdentityService.LoginAsync(UserName, Password);
            if (!loginResult)
            {
                return;
            }

            // await Shell.Current.GoToAsync($"//{nameof(StartPage)}");
        }
    }
}
```


Our viewmodels inherit from a BaseViewModel which some generic helper methods to use:

**YourApp/ViewModels/BaseViewModel.cs**

```csharp
using System;
using System.Collections.Generic;
using System.ComponentModel;
using System.Runtime.CompilerServices;
using System.Threading.Tasks;
using Xamarin.Essentials;

namespace YourApp.ViewModels
{
    public class BaseViewModel : INotifyPropertyChanged
    {
        bool isBusy = false;
        public bool IsBusy
        {
            get { return isBusy; }
            set { SetProperty(ref isBusy, value); }
        }

        string title = string.Empty;
        public string Title
        {
            get { return title; }
            set { SetProperty(ref title, value); }
        }

        protected bool SetProperty<T>(ref T backingStore, T value,
            [CallerMemberName] string propertyName = "",
            Action onChanged = null)
        {
            if (EqualityComparer<T>.Default.Equals(backingStore, value))
                return false;

            backingStore = value;
            onChanged?.Invoke();
            OnPropertyChanged(propertyName);
            return true;
        }

        public async Task<bool> IsGrantedAsync(string permissionName)
        {
            bool.TryParse(await SecureStorage.GetAsync(permissionName), out bool result);
            return result;
        }

        public async Task<bool> IsAuthenticated()
        {
            return !string.IsNullOrEmpty(await SecureStorage.GetAsync("user_id")) || !string.IsNullOrEmpty(await SecureStorage.GetAsync("access_token"));
        }

        #region INotifyPropertyChanged
        public event PropertyChangedEventHandler PropertyChanged;
        protected void OnPropertyChanged([CallerMemberName] string propertyName = "")
        {
            var changed = PropertyChanged;
            if (changed == null)
                return;

            changed.Invoke(this, new PropertyChangedEventArgs(propertyName));
        }
        #endregion
    }
}

```

Now we need to add the interfaces and classes for making HTTP requests towards our ABP API:

**YourApp/Services/IHttpClientService.cs**
This interface contains a method for making a Auth POST request towards our API:

```csharp
using System.Collections.Generic;
using System.Net.Http;
using System.Threading.Tasks;

namespace YourApp.Services.Http
{
    public interface IHttpClientService<T, C> where T : class where C : class
    {
        Task<T> AuthPostAsync(string uri, string userName, string password);
    }
}

```

**YourApp/Services/HttpClientService.cs**
And the implementation of the interface as shown below. 

```csharp
using System;
using System.Net.Http;
using System.Net.Http.Headers;
using System.Text;
using System.Threading.Tasks;
using Xamarin.Forms;
using YourApp.Services.Identity;
using System.Text.Json;

namespace YourApp.Services.Http
{
    public class HttpClientService<T, C> : IHttpClientService<T, C> where T : class where C : class
    {
        public IIdentityService IdentityService => DependencyService.Get<IIdentityService>();
        Lazy<HttpClient> _httpClient;
        private async Task<string> GetAccessTokenAsync() => await IdentityService.GetAccessTokenAsync();

        private JsonSerializerOptions Options => new JsonSerializerOptions
        {
            WriteIndented = true,
            PropertyNameCaseInsensitive = true // this is the point
        };

        public async Task<T> AuthPostAsync(string uri, string userName, string password)
        {
            _httpClient = await GetHttpClientAsync();

            var data = $"grant_type=password";
            data += $"&username={userName}";
            data += $"&password={password}";
            data += $"&client_id={Global.Settings.IdentityServer.ClientId}";
            data += $"&client_secret={Global.Settings.IdentityServer.ClientSecret}";
            data += $"&scope={Global.Settings.IdentityServer.Scope}";
            
            var content = new StringContent(data, Encoding.UTF8, "application/x-www-form-urlencoded");

            var response = await _httpClient.Value.PostAsync(uri, content);
            response.EnsureSuccessStatusCode();

            var stringResult = await response.Content.ReadAsStringAsync();
            var result = JsonSerializer.Deserialize<T>(stringResult, Options);
            return result;
        }

        private HttpClientHandler GetHttpClientHandler()
        {
           ///////////////////////////////////////
            // EXCEPTION : Javax.Net.Ssl.SSLHandshakeException: 'java.security.cert.CertPathValidatorException: Trust anchor for certification path not found.'
            // SOLUTION :

            var httpClientHandler = new HttpClientHandler
            {
                ServerCertificateCustomValidationCallback = (message, cert, chain, errors) => { return true; }
            };

            return httpClientHandler;
        }

        private async Task<Lazy<HttpClient>> GetHttpClientAsync()
        {
            var accessToken = await GetAccessTokenAsync();
            var httpClient = new Lazy<HttpClient>(() => new HttpClient(GetHttpClientHandler()));
            httpClient.Value.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", accessToken ?? "");
            return httpClient;
        }
    }
}
```


**YourApp/Services/IIdentityService.cs**

```csharp
using System.Threading.Tasks;

namespace YourApp.Services.Identity
{
    public interface IIdentityService
    {
        Task<string> GetAccessTokenAsync();
        Task<bool> LoginAsync(string userName, string password);
        Task<bool> LogoutAsync();
    }
}

```

**YourApp/Services/IdentityService.cs**

```csharp
using YourApp.Services.Http;
using System;
using System.Collections.Generic;
using System.IdentityModel.Tokens.Jwt;
using System.Net.Http;
using System.Text;
using System.Text.Json;
using System.Threading.Tasks;
using Xamarin.Essentials;
using Xamarin.Forms;

namespace YourApp.Services.Identity
{
    public class IdentityService : IIdentityService
    {
        private const string AccessToken = "access_token";
        private const string IdentityToken = "identity_token";
        private const string UserId = "user_id";

        // Get User by email
        public IHttpClientService<IdentityUserDto, IdentityUserDto> HttpClient => DependencyService.Get<IHttpClientService<IdentityUserDto, IdentityUserDto>>();

        public IHttpClientService<IdentityDto, IdentityDto> AuthHttpClient => DependencyService.Get<IHttpClientService<IdentityDto, IdentityDto>>();

        public async Task<bool> LoginAsync(string userName, string password)
        {
            var loginResult = await AuthHttpClient.AuthPostAsync(Global.Settings.Api.TokenUri, userName, password);
            
            if (!string.IsNullOrEmpty(loginResult.error))
            {
                return false;
            }
            
            await SecureStorage.SetAsync(AccessToken, loginResult.access_token);
            
            var customClaims = ExtractCustomClaims(loginResult.access_token);
            
            return true;
        }

        public JsonDocument ExtractCustomClaims(string accessToken)
        {
            var base64payload = accessToken.Split('.')[1];
            base64payload =
                base64payload.PadRight(base64payload.Length + (base64payload.Length * 3) % 4, '='); // add padding
            var bytes = Convert.FromBase64String(base64payload);
            var jsonPayload = Encoding.UTF8.GetString(bytes);
            var claimsFromAccessToken = JsonDocument.Parse(jsonPayload);
            return claimsFromAccessToken;
        }

        public async Task<string> GetAccessTokenAsync()
        {
            var accessToken = await SecureStorage.GetAsync(AccessToken);
            if (!string.IsNullOrWhiteSpace(accessToken))
            {
                var handler = new JwtSecurityTokenHandler();
                var jwtToken = handler.ReadJwtToken(accessToken);
                var validTo = jwtToken.ValidTo;
                if (validTo <= DateTime.Now.AddMinutes(5))
                {
                    // await LoginAsync();
                }
                else
                {
                    return accessToken;
                }
            }
            else
            {
                // await LoginAsync();
            }

            return await SecureStorage.GetAsync(AccessToken);
        }

        public async Task<bool> LogoutAsync()
        {
            SecureStorage.Remove(AccessToken);
            var idTokenHint = await SecureStorage.GetAsync(IdentityToken);
            SecureStorage.Remove(IdentityToken);
            SecureStorage.Remove(UserId);
            SecureStorage.Remove("email");
           
            return true;
        }
    }

    // Should be in it's own file (SRP)
    public class IdentityDto
    {
        public string access_token { get; set; }
        public int expires_in { get; set; }
        public string token_type { get; set; }
        public string scope { get; set; }
        public string error { get; set; }
        public string error_description { get; set; }
    }

    // Should be in it's own file (SRP)
    public class IdentityUserDto
    {
        //public ExtraProperties extraProperties { get; set; }
        public string id { get; set; }
        public DateTime? creationTime { get; set; }
        public string creatorId { get; set; }
        public DateTime? lastModificationTime { get; set; }
        public string lastModifierId { get; set; }
        public bool isDeleted { get; set; }
        public string deleterId { get; set; }
        public DateTime? deletionTime { get; set; }
        public string tenantId { get; set; }
        public string userName { get; set; }
        public string name { get; set; }
        public string surname { get; set; }
        public string email { get; set; }
        public bool emailConfirmed { get; set; }
        public string phoneNumber { get; set; }
        public bool phoneNumberConfirmed { get; set; }
        public bool lockoutEnabled { get; set; }
        public DateTime? lockoutEnd { get; set; }
        public string concurrencyStamp { get; set; }
    }
}
```

Now let's add a class that holds some basic settings that we can use in our previously shown class implementations:

**YourApp/Global.cs**

```csharp
using System.Collections.Generic;

namespace YourApp
{
    public class Global
    {
        public IdentityServer IdentityServer { get; }
        public Api Api { get; }

        public Global(IdentityServer identityServer, Api api)
        {
            IdentityServer = identityServer;
            Api = api;
        }

        public static Global Settings { get; } = new Global(new IdentityServer(), new Api());
    }

    // Should be in it's own file (SRP)
    public class IdentityServer 
    {
        public readonly string Authority = "https://localhost:44377";

        public readonly string ClientId = "YourApp_Xamarin";
        public readonly string Scope = "email openid profile role phone address YourApp";
        public readonly string ClientSecret = "1q2w3e*";
        public readonly string RedirectUri = "xamarinformsclients://callback";
    }

    // Should be in it's own file (SRP)
    public class Api
    {
        private readonly string _apiEndpoint = "https://localhost:44377/api/";
        public readonly string RegularEndpoint = "https://localhost:44377";
        public string TokenUri => RegularEndpoint + "/connect/token";
    }
}

```

Lastly we need to register our service dependencies in the main App file:

**YourApp/App.xaml.cs**

```csharp
using YourApp.Services.Http;
using YourApp.Services.Identity;
using Xamarin.Forms;

namespace YourApp
{
    public partial class App : Application
    {
        public App()
        {
            InitializeComponent();
            
            DependencyService.Register<IdentityService>();
            DependencyService.Register<HttpClientService<IdentityUserDto, IdentityUserDto>>();
            DependencyService.Register<HttpClientService<IdentityDto, IdentityDto>>();

            MainPage = new MainPage();
        }

        protected override void OnStart()
        {
        }

        protected override void OnSleep()
        {
        }

        protected override void OnResume()
        {
        }
    }
}

```


## Test

With the above code implementation, we can now run our app in an Android emulator, and we should be able to have our MainPage shown with 2 entry fields and a login button.

Try to use the standard "admin" login credentials from ABP.

![login](https://user-images.githubusercontent.com/265719/116081653-d134b980-a69a-11eb-90ae-0049d31e730b.gif)


## Notes
Example repository: https://github.com/Znow/ConsumeAbpFromXamarinApp

* Getting connection issues? Be sure that your SSL certificate is trusted if using https.
* If any questions/thoughts/ideas/changes - Please send me a message, and I will get back to you.
