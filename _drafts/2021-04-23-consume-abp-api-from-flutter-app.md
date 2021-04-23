---
published: false
---
## Consume ABP API from Flutter App

If you are using the [ABP Framework ](https://abp.io "ABP Framework ") and want to consume it in your Flutter application, please read further where I will give a guide on how to implement the changes needed in your application.

### Setup API

First we need to setup our backend project, to enable another client, that is allowed to consume the API endpoints and thus make requests. Since we are using our DbMigrator to setup clients initially, we also need to add the new client from it's appsettings.

_YourProject.DbMigrator/appsettings.json_

Add the below code to the "IdentityServer" "Clients" section:

```json
"YourProjectApi_App": {
  "ClientId": "YourProjectApi_App",
  "ClientSecret": "SomeSecretValue",
  "RootUrl": "https://your-api"
  // RootUrl = "AuthServer:Authority": "https://localhost:44349" in appsettings.json HttpApi.Host project
}
```

_YourProject.Domain/IdentityServer/IdentityServerDataSeedContributor.cs_

Add the below code after the last "Swagger Client", as shown below:

```csharp
 Flutter App Client
var appClientId = configurationSection["YourProjectApi_App:ClientId"];
if (!appClientId.IsNullOrWhiteSpace())
{
  var appRootUrl = configurationSection["YourProjectApi_App:RootUrl"].TrimEnd('/');

  await CreateClientAsync(
    name: appClientId,
    scopes: commonScopes,
    grantTypes: new[] { "authorization_code", "client_credentials" },
    secret: configurationSection["YourProjectApi_App:ClientSecret"]?.Sha256(),
    requireClientSecret: false,
    redirectUri: "com.example.app://callback",
    corsOrigins: new[] { appRootUrl.RemovePostFix("/") }
  );
}
```

### Setup Flutter App

We need to add the "flutter_appauth" package from this repository https://github.com/MaikuB/flutter_appauth in the dependencies section in the pubspec file, as shown below:

_YourApp/pubspec.yaml_

```yaml
dependencies:
  flutter:
    sdk: flutter
  flutter_appauth: ^0.9.1
```

Next we will create a view with our auth functionality:

_YourApp/lib/auth.dart_

```dart
import 'dart:convert';

import 'package:flutter/material.dart';
import 'package:flutter_appauth/flutter_appauth.dart';
import 'package:flutter_secure_storage/flutter_secure_storage.dart';
import 'package:stockup/ui/view/auth/widgets/login.dart';
import 'package:stockup/core/models/global.dart';

final FlutterAppAuth appAuth = FlutterAppAuth();
final FlutterSecureStorage secureStorage = const FlutterSecureStorage();

// YourProject Api details
const DOMAIN = 'your-api';
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

            // TODO Get picture and score from API
      Globals.profile = ProfileInfo(
          uuid: "123",
          name: profile["userName"],
          picture: null,
          score: 0,
          priceTargets: pricetargetMap);
      Navigator.pushReplacementNamed(context, StockOverview.route);

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

