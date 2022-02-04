---
published: false
---
# Consume ABP API from Xamarin App

If you are using the [ABP Framework ](https://abp.io "ABP Framework ") and want to authenticate and consume it in your Xamarin Forms app, please read further where I will give a short guide on how to implement the changes needed in your application.

I will not go in much detail about the actual Xamarin code implementation neither the ABP implementation, as this is a simple example of how it can be done.

Nor can I be held responsible for any bugs, breaches or unaware use of packages.

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
```chsarp
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







Next we will create a model for use in our auth. The model will hold the accessToken, which can be used in subsequent requests to the API.
The model also holds some simple methods to retrieve users, and user details for the the user requesting.

**YourApp/lib/core/models/yourapi_api.dart**

```dart
import 'dart:convert';
import 'dart:io';
import 'package:http/http.dart';
import 'package:http/http.dart' as http;


class YourProjectAPI {
  static String accessToken;
  static const String _baseUrl =
      "https://your-api/api";
  static Map<String, String> headers = {
    'Content-type': 'application/json',
    'Accept': 'application/json',
    HttpHeaders.authorizationHeader: "Bearer $accessToken",
  };

  static Future<Response> getUserDetails() async {
    final url = '$_baseUrl/identity/my-profile';
    final response = await http.get(
      url,
      headers: {HttpHeaders.authorizationHeader: "Bearer $accessToken"},
    );

    return response;
  }

  // Example API method call
  static Future<List<User>> getUsers() async {
    final response = await http.get("$_baseUrl/identity/users", headers: headers);

    
    if (response.statusCode == 200) {
      // Logic here       
    }
    return response;
  }
}

```

Next we will create a view with our auth functionality. The main things to be aware of here is the constants: "DOMAIN", "CLIENT_ID", "CLIENT_SECRET" and "REDIRECT_URI".
"CLIENT_ID", "CLIENT_SECRET" and "REDIRECT_URI" of course needs to be the same, defined previously in the API.

The views sole purpose is to handle the "Login" button clicked, fire a request towards the API's authentication endpoint and retrieve token, which needs to be parsed, saved in securestorage and used in subsequent requests.

**YourApp/lib/ui/view/auth/auth.dart**

```dart
import 'dart:convert';

import 'package:flutter/material.dart';
import 'package:flutter_appauth/flutter_appauth.dart';
import 'package:flutter_secure_storage/flutter_secure_storage.dart';
import 'package:yourapp/ui/view/auth/widgets/login.dart';
import 'package:yourapp/core/models/yourprojectapi_api.dart';

final FlutterAppAuth appAuth = FlutterAppAuth();
final FlutterSecureStorage secureStorage = const FlutterSecureStorage();

// YourProject Api details
const DOMAIN = '10.0.2.2:44349'; // 10.0.2.2 used here due to localhost cannot be resolved.
const CLIENT_ID = 'YourProjectApi_App';
const CLIENT_SECRET = "SomeSecretValue";

const REDIRECT_URI = 'com.example.app://callback';
const ISSUER = 'https://$DOMAIN';

class Auth extends StatefulWidget {
  static const route = "/newauth";

  @override
  _AuthState createState() => _AuthState();
}

class _AuthState extends State<Auth> {
  bool isBusy = false;
  bool isLoggedIn = false;
  String errorMessage;
  String name;
  String picture;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Center(
        child: isBusy
            ? CircularProgressIndicator()
            : Login(loginAction, errorMessage),
      ),
    );
  }

  Map<String, dynamic> parseIdToken(String idToken) {
    final parts = idToken.split(r'.');
    assert(parts.length == 3);

    return jsonDecode(
        utf8.decode(base64Url.decode(base64Url.normalize(parts[1]))));
  }

  getUserDetails() async {
    var profileDetails = {};
    YourProjectAPI.getUserDetails().then(
      (response) {
        if (response.statusCode == 200) {
          profileDetails = jsonDecode(response.body);
        } else {
          throw Exception('Failed to get user details');
        }
      },
    );
    return profileDetails;
  }

  Future<void> loginAction() async {
    setState(() {
      isBusy = true;
      errorMessage = '';
    });

    try {
      final AuthorizationTokenResponse result =
          await appAuth.authorizeAndExchangeCode(
        AuthorizationTokenRequest(CLIENT_ID, REDIRECT_URI,
            clientSecret: CLIENT_SECRET,
            issuer: 'https://$DOMAIN',
            scopes: ["email", "openid", "profile", "role", "phone", "address"],
            promptValues: ['login']),
      );

      // Set access token for application
      YourProjectAPI.accessToken = result.accessToken;

      final idToken = parseIdToken(result.idToken);
      final profile = await getUserDetails();
      await secureStorage.write(
          key: 'refresh_token', value: result.refreshToken);

      // This setstate is most likely not needed
      setState(() {
        isBusy = false;
        isLoggedIn = true;
        name = idToken['name'];
        picture = profile['picture'];
      });
    } catch (e, s) {
      print('login error: $e - stack: $s');
      setState(() {
        isBusy = false;
        isLoggedIn = false;
        errorMessage = e.toString();
      });
    }
  }

  @override
  void initState() {
    initAction();
    super.initState();
  }

  void initAction() async {
    final storedRefreshToken = await secureStorage.read(key: 'refresh_token');
    if (storedRefreshToken == null) return;

    setState(() {
      isBusy = true;
    });

    try {
      final response = await appAuth.token(TokenRequest(
        CLIENT_ID,
        REDIRECT_URI,
        issuer: ISSUER,
        refreshToken: storedRefreshToken,
      ));

      final idToken = parseIdToken(response.idToken);
      final profile = await getUserDetails();

      secureStorage.write(key: 'refresh_token', value: response.refreshToken);

      setState(() {
        isBusy = false;
        isLoggedIn = true;
        name = idToken['name'];
        picture = profile['picture'];
      });
    } catch (e, s) {
      print('error on refresh token: $e - stack: $s');
      setState(() {
        isBusy = false;
        isLoggedIn = false;
      });
    }
  }
}

```

**YourApp/android/app/build.gradle**

We also need the below line for Android configuration to let the app to be redirected back to

```gradle
android {
    defaultConfig {
        ...
        manifestPlaceholders = [
                'appAuthRedirectScheme': 'com.example.app'
        ]
    }
}
```

And for IOS:

```xml
<key>CFBundleURLTypes</key>
	<array>
		<dict>
			<key>CFBundleTypeRole</key>
			<string>Editor</string>
			<key>CFBundleURLSchemes</key>
			<array>
				<string>com.example.app</string>
			</array>
		</dict>
	</array>
```

## Test

With the above code implementation, we can now run our app in an Android emulator, and we should be able to have a login screen shown.

When clicking on the "Login" button, a browser should open with a our well known login screen from the API project - and then let us login and be sent back to the app!

![login](https://user-images.githubusercontent.com/265719/116081653-d134b980-a69a-11eb-90ae-0049d31e730b.gif)


## Notes
Example repository: https://github.com/Znow/ConsumeAbpFromFlutterApp

* Getting connection issues? Be sure that your SSL certificate is trusted if using https.
* If any questions/thoughts/ideas/changes - Please send me a message, and I will get back to you.
