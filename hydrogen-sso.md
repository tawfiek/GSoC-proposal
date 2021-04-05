# SSO In Hydrogen

## Introduction

Hydrogen is a Matrix chat client, built to provide seamless mobile first experience on mobile web browsers.

Single Sign-ON (SSO) is one of the most important authentication scheme in the industry, since it becomes with a great improvement of user experience in apps.

Matrix provides bunch of choses to authenticate users using SSO, that has been also implemented into all other matrix chat clients, so it a key feature to be implemented into hydrogen as well.

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

As we can see here, the return of `loginFlow` End point based on what the `homeServer` supports. <br> So here returns with it's data that needed to adjust the UI for the user and to use it into SSO flow as well.

Well, lets have a deeper dive into `Hydrogen` trying to figure out how we can impalement this feature.

## Hydrogen Architecture

Hydrogen follow MVVM Design patterns.

![Image 1](./assets/MVVM_Diagram.png)

MVVM is stands for Model View View-Model.

`Model`: here is usually class that holds the application data and business logic . <br>
`ViewModel`: is abstraction of the view exposing public properties, it managing the binding data  between Model and View <br>
`View`: The view is the user interface that mange the view components animations changing state and so on.

In hydrogen all ModelViews are collected into `src/domain` directory, we will interact with some ModelView in this project like `LoginViewModel` and `SessionViewModel`. <br>
All ViewModels in Hydrogen are a classes that extends `ViewModel` class, This class describes the basic functionality that ModelView should provide in every `ViewModel` to be functional as expected in all ViewModels.

### `ViewModel` class

This class extends `EventEmitter` class and add some functionality to keep track of disposables to manage state, handling the internationalization (i18n), and also provide View and Model some usable information about platform ..etc.

### `TemplateView` class

Hydrogen also has a main class called `TemplateView` that has all functionality needed to mount and umount view, It also uses  `TemplateViewBuilder` to build the desired components out of `render()` functions object using XHTML namespace.
## Login in Hydrogen

