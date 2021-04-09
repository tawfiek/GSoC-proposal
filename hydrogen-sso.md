# SSO In Hydrogen

## Introduction

Hydrogen is a Matrix chat client, built to provide seamless mobile first experience on mobile web browsers.

Single Sign-ON (SSO) is one of the most important authentication scheme in the industry, since it becomes with a great improvement of user experience in apps.

Matrix provides bunch of choses to authenticate users using SSO, that has been also implemented into all other matrix chat clients, so it a key feature to be implemented into hydrogen as well.
## Login in Hydrogen

In `src/platform/web/ui/login` we have all views related to the login view.<br>
`LoginView` is the main view that represent the basic email-password inputs and the home server url input that make the user able to sign into the app.

![Image 2](./assets/signIn-page.png)

Login view are rendered into `RootView`, well in the `LoginView` we just start login through the `LoginViewModel` through `SessionContainer` which is the Model that will handles all our session data, pressing login button will call `login()` method passing user inputs username, password and homeServer URL which is setting up the `SessionLoadViewModel` and tracking it then  call the `startWithLogin()` method from a new object of the `SessionContainer`, then the session container start a password login by making a new object from `HomeServerAPI` and call password login method. Hmm , lets simplify it with diagram.

![Image 3](./assets/login-process.png)

lets call it the "login flow diagram" as we gonna referee to it in next sections and modify it.
## Login Flows in Matrix

Matrix main home server is depend on varies authentication flows one of these flows is SSO, all supported authentication flows are provided into the `homeServer` in this [Docs](https://matrix.org/docs/spec/client_server/latest#id204).

the return of the login flows API call should be something like that

``` JSON
[
  {
    "type": "m.login.sso",
    "identity_providers": [
      {
        "id": "oidc-github",
        "name": "GitHub",
        "icon": "mxc://matrix.org/sVesTtrFDTpXRbYfpahuJsKP",
        "brand": "github"
      },
      {
        "id": "oidc-google",
        "name": "Google",
        "icon": "mxc://matrix.org/ZlnaaZNPxtUuQemvgQzlOlkz",
        "brand": "google"
      },
      {
        "id": "oidc-gitlab",
        "name": "GitLab",
        "icon": "mxc://matrix.org/MCVOEmFgVieKFshPxmnejWOq",
        "brand": "gitlab"
      },
      {
        "id": "oidc-facebook",
        "name": "Facebook",
        "icon": "mxc://matrix.org/nsyeLIgzxazZmJadflMAsAWG",
        "brand": "facebook"
      },
      {
        "id": "oidc-apple",
        "name": "Apple",
        "icon": "mxc://matrix.org/QQKNSOdLiMHtJhzeAObmkFiU",
        "brand": "apple"
      }
    ],
  },
  {
    "type": "m.login.token"
  },
  {
    "type": "m.login.password"
  },
  {
    "type": "uk.half-shot.msc2778.login.application_service"
  }
]
```

As we can see here, the return of `loginFlow` End point based on what the `homeServer` supports.

Well, lets have a deeper dive into `Hydrogen` trying to figure out how we can impalement this feature.

## Implementation

>NOTE: All codes in this document is just my thoughts about the implementation, The purpose of writing this code is rounding out the idea and simplifying it for the reader, and it is not considered as a production ready code.

### Adding login flows call into `HomeServer` class

To add SSO in Hydrogen, first we need call the login flow end point, adjust the view, then make the SSO authentication process.

So, to call the end point the home server URL should be provided first, to do that we need to add this new API call into our `HomeServer` service  to be something like that.

``` javascript
  getSupportedLoginMethods (options = null) {
    return this._get("/login", null, null, options);
  }
```
Now the `HomeServer` class has the functionality to do the HTTP call to get the login flows mentioned above form the home server.

We need to call this method in a proper context to loginFlow which we can move forward.

### Session Container
In session container we should add the `requestSupportedLoginFlows` method which is making a new object of `HomeServer` and call the new  method we added above to make the call then set the status of the container

```javascript
  async requestSupportedLoginFlows(homeServer) {
    try {
        const request = this._platform.request;
        const clock = this._platform.clock;
        const hsApi = new HomeServerApi({
            homeServer,
            request,
            createTimeout: clock.createTimeout,
        });
        this._supportedLoginFlows =  await hsApi.getSupportedLoginMethods().response();
        this._status.set(LoadStatus.LoginFlowsLoaded.find());
    } catch (err) {
        this._error = new Error('This home serve URL is not valid');
        console.error(err);
        this._status.set(LoadStatus.Error);
    }
  }
```

### `LoginView`

In the Login view currently we just support the password login flow, we need to adjust this view to wait for the login flows call to be fulfilled and then map another two new views ( or more in the future ) specific for every flow.

So again, we need to split the login view into two views one for password login flow lets call it `PasswordView` and the other for sso login flow `SSOLoginFlow`.

The initial `LoginView` should be something like this

![Image 3](./assets/login-view-1.png)

And after loading the flows and map the new views it should be something like

![Image 3](./assets/login-view-2.png)

The request fo the login flow should be done from the `LoginModelView` which it handles our domain and logic, in `LoginModelView` we should make an event handler that handles the change in homeserver input change event, by loading the login views, it should be something like that

``` javascript
  _requestLoginFlows (homeServer) {
    //  Here we make new object of the Loading flow view model
    // This View should make a new loader spinner to indicates the login flow is loading
    // This view creates new session container load the login flow through it ,
    //  and when it's done the ready method should be called to map the login view to add the login flows supported as shown in the above views.

    this._ladLoginFlowViewModel  = this.track(new LoadLoginFlowViewModel({
        createAndStartSessionContainer: () => {
            this._sessionContainer = this._createSessionContainer();
            this._sessionContainer.requestSupportedLoginFlows(homeServer);
            return this._sessionContainer;
        },
        ready: sessionContainer => {
            this._sessionContainer = null;
            this._supportedLoginFlows = _supportedLoginFlows
        },
        homeServer,
    }));

    this._loadLoginFlowViewModel.start();
    this.emitChange('loadLoginFlowViewModel');
  }
```
The ready method here should be called by the `LoadLoginFlowViewModel` to set the incoming supported login flows.

In the login view the new views should be mapped using mapView method in the template.

For SSO login flow view for example:

```javascript
  t.mapView(
    (vm) => vm.loadLoginFlowViewModel,
    (loadLoginFlowViewModel) =>
        loadLoginFlowViewModel
            ? getSSOViewIfSupported()
            : null
  ),


  function getSSOViewIfSupported () {
    if (vm.supportedLoginFlows.find(f => f.type === 'm.login.sso' )) {
      return new SSOLoginViewModel ()
    } else {
      return;
    }
  }

```

and so on for all login flows.

 