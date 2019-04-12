# Auth0 netID Sample

![Auth0 netID Sample](img/fezbot2000-71727-unsplash.jpg)

## Introduction

### Purpose

The purpose of this sample project is to demonstrate how a web developer, using **Auth0**, can add the identity provider **netID** as an option for their end users to signup and login to his/her web application.

### About netID

**netID is a European identity provider.**

The European netID Foundation was founded in march 2018 by Mediengruppe RTL Deutschland, ProSiebenSat.1 and United Internet. Its aim is to establish the netID single sign-on as a European alternative to US providers. With netID, users can organize their consent to the use of Internet services (opt-ins) transparently and in compliance with data protection regulations. To this end, the foundation has developed an open standard that enables users from all sectors to access all internet services of partners of the European netID Foundation with the same log-in. Right from the start which focuses the initial core market Germany, 38 million accounts covering roughly about 60% of the Germany online population are ready to be used.

* https://netid.de/
* https://enid.foundation/

### About Auth0

**Auth0 is an Identity as a Platform (IDaaS) service provider.**

Auth0 is Identity made simple and secure. With Auth0, you take perhaps the riskiest endeavor your engineering organization will face, managing and securing the identities of your customers and employees, and abstract away this complexity and risk with our Identity Platform. Our standards-based Identity-as-a-Service platform is  built for developers, by developers so that your engineering organization can focus on building the solutions that delight your customers and drive your revenue. Our customers find that what typically either takes months to deliver or simply cannot be delivered via internal or external solutions, takes them only days to deliver with Auth0 due to our extensive SDKs, intuitive API, extensible architecture and our simple management dashboard. This is why Auth0 is the leader in developer-focused Identity Management, with more than 5,000 customers trusting us with billions of transactions everyday.

* https://auth0.com

## Configuration

### Prerequisites

We assume that you already have registered an Auth0 account and setup a tenant. If not, register a free account and tenant at https://auth0.com

Furthermore, we assume that you have already registered a service provider account at netID. If not, register an account at netID: https://developer.netid.de/register

### Configuration on netID side

1. Login into the netID Developer Portal
2. Create a `Dienst (service)` for which you want to leverage the netID single sign-on. Select `Dienst` in the menu, click `Dienst hinzufügen` (Add service) and fill in the required details in the following screen. Detailed documentation can be found [here](https://developerzone.netid.de/developerportal/dienste/dienst-anlegen.html#indexlink_help_developerportal_dienste). In the end, it should look similar to this:
![Placeholder](img/netid-created-service-1.png)
3. Create a `Client` for this service by clicking on `Client hinzufügen` (Add client) - Detailed documentation can be found [here](https://developerzone.netid.de/developerportal/clients/client-anlegen.html#indexlink_help_developerportal_clients) - it's critical to set the proper Auth0 callback url for your tenant. The Auth0 Callback URL is `https://<tenant>.auth0.com/login/callback` where `<tenant>` needs to be replaced with yours.
In the end it should look similar to this:
![Placeholder](img/netid-create-client-2.png)
4. Once the client is created it will initially run in a sandboxed mode, which means it can only be used with whitelisted netID-Accounts (email addresses). In order to successfully run through the initial integration add a
test user (mail address) to the whitelist by selecting `Testnutzer hinzufügen` (Add Test-Account), outlined [here](https://developerzone.netid.de/developerportal/testnutzer/testnutzer-anlegen.html). Up to 10 accounts can be whitelisted per sandboxed client.
5. Fetch **Client ID** and **netID Token - Sandbox** (later refered to as **client secret**) for the Auth0 configuration by expanding the client details      
![Placeholder](img/netid-client-details-1.png)

### Configuration on Auth0 side

1. Log into the Auth0 Dashboard.
2. Go to `Extensions` in the left-hand menu.
3. Make sure that the `Custom Social Connections` extension is installed. If not, install it by clicking on it.
![Placeholder](img/custom-social-connection-1.png)
4. Once installed, click on the extension to configure it.
5. The upcoming screen shows an overview of pre-configured but disabled custom connections. Press the orange button labeled `+ New Connection` to create and configure a new connection.
![Placeholder](img/custom-social-connections-2.png)

6. In the upcoming configuration screen, enter the following values:

**Name**: `netID`

**Client ID**: `<enter the ClientID you retrieved from netID>`

**Client Secret**: `<enter the Client Secret you retrieved from netID>`

**Fetch User Profile Script**:

```javascript
function(accessToken, ctx, cb) {
  request.get('https://broker.netid.de/userinfo', {
    headers: {
      'Authorization': 'Bearer ' + accessToken
    }
  }, function(e, r, b) {
    if (e) return cb(e);
    if (r.statusCode !== 200) return cb(new Error('StatusCode: ' + r.statusCode));
    var profile = JSON.parse(b);
    cb(null, {
      name: profile.given_name + ' ' + profile.family_name,
      family_name: profile.family_name,
      given_name: profile.given_name,
      email: profile.email,
      email_verified: profile.email_verified,
      birthdate: profile.birthdate,
      gender: profile.gender,
      user_id: profile.sub,
      phone_number: profile.phone_number,
      phone_verified: profile.phone_number_verified
    });
  });
}
```

**Authorization URL**:

```
https://broker.netid.de/authorize?claims={"userinfo":{"birthdate":{"essential":true},"gender":{"essential":true},"email": {"essential":true},"email_verified": {"essential":true},"given_name": {"essential":true},"family_name":{"essential":true},"phone_number":{"essential":true},"phone_number_verified":{"essential":true}}}
```

In this url, you notice the explicit parameter `claims`. This parameter tells Auth0 which claims to ask for from netID. **Please note** - netID will ignore any data requested via standard `scopes` (_name, address, ..._) as well as optional/voluntary claims.

Every OpenID Connect call must always request the openid scope. In addition, the required master data must be formulated as `essential` claims.

netID supports the following claims

* `gender` - gender of the end user
* `given_name` - first name(s) of the end user
* `family_name` - Last name(s) of the end user
* `birthdate` - Date of birth of the end user
* `email` - email address of the end user
* `email_verified` - verification status of the email address of the end user
* `postal_code` - the postal code of the end user's address.
* `locality` - city of the address of the end user
* `street_address` - street and house number of the end user's address
* `country` - country of the end user's address
* `phone_number` - phone number of the end user
* `phone_number_verified` - verification status of the end user's phone number

You can adjust the claims parameter according to your needs.

**Token URL**: `https://broker.netid.de/token`

**Scope**: `openid`

**Custom Headers**:

Since netID requires *Basic Authentication* for authentication, but the *Auth0 Custom Social Login* extension submits the credentials via post body per default, we need to create a custom header for the Authorization and add it to the configuration.

The hash is created by Base64-encoding the `clientID` and `clientSecret` concenated with a ``:``. So the pseudo code to generate this would be:

`base64encode(clientId + ":" + clientSecret)`

so an example could then look like:

```
{
  "Authorization": "Basic NDQyNWM3NGItN2VhOS00MDI0LTk4OGQtY2JmZmRiZjQwNGM1OkhISEh6bHQzZ1JORHV5TGkzbHJmRC1YMWpjTnpfUG5wd1RjYTloc1lySHo4cFlMblVSNVpucmxaMEpKSko="
}
```

In the end, it should look similar to this:

![Placeholder](img/Screen_Shot_2019-03-12_at_11.47.13_AM.png)

You can try out the connection by pressing the `Try` button and, upon success, get a screen like this:

![Placeholder](img/Screen_Shot_2019-03-12_at_11.50.01_AM.png)

Under the `Apps` tab, make sure that your client/application is enabled for this custom social connection, otherwise enable the checkbox.

![Placeholder](img/Screen_Shot_2019-03-25_at_4.11.09_PM.png)

After hitting the `Save` button and returning to the overview of custom social connections, you should see the newly added one listed and active here:

![Placeholder](img/custom-social-connections-1.png)


7. At last, adjust the `Universal Login Page` configuration and add a few lines of custom css to make the netID button appear with the respective logo and coloring.
In the Auth0 dashboard, go to `Universal Login Page` > `Login` (https://manage.auth0.com/#/login_page) and enable the `Customize Login Page` checkbox. Adjust the source code of the login page replacing the `theme` section in the JSON object with the following lines:

```
      theme: {
        logo:            config.icon,
        primaryColor:    colors.primary ? colors.primary : 'green',
        authButtons: {
          "netID": {
            displayName: "netID",
            primaryColor: "#74b82b",
            foregroundColor: "#fff",
            icon: "https://raw.githubusercontent.com/auth0-blog/auth0-netid-sample/master/img/favicon_trans.png"
          }
        }
      },
```

Before:

![Placeholder](img/login_before.png)

After:

![Placeholder](img/login_after.png)


## Screenflow

### Screencast

A screencast of the authentication flow is available here: [1080.mov](mov/1080.mov)

### Authentication

When pressing the login button within the client application, Auth0 provides a login/signup widget with options to login/signup via username and password or via social login, such as netID.

![Placeholder](img/Screen_Shot_2019-03-12_at_10.50.50_AM.png)

When choosing netID (we're assuming the end user already has an account at netID), he will be asked to login at netID, providing email address... **Please note** - if your client is running in sandboxed mode, as described here, it will only function with the whitelisted email addresses.

![Placeholder](img/Screen_Shot_2019-03-12_at_10.52.52_AM.png)

... and password:

![Placeholder](img/Screen_Shot_2019-03-12_at_10.53.23_AM.png)

Upon successful login, he will be asked give consent to netID to grant Auth0 access to his personal information:

![Placeholder](img/Screen_Shot_2019-03-12_at_10.53.48_AM.png)

Furthermore, the demo client asks for consent to access the Auth0 tenant:

![Placeholder](img/Screen_Shot_2019-03-12_at_10.54.13_AM.png)

After consent is given, the browser redirects to the application, having received an ID token and information about the user:

![Placeholder](img/Screen_Shot_2019-03-12_at_10.54.43_AM.png)

### Auth0 Backend

In the Auth0 backend, the client application is registered as follows:

![Placeholder](img/applications.png)

### Auth0 Dashboard: Users

Looking at the user list and into a user's detail after he authenticated:

![Placeholder](img/Screen_Shot_2019-03-12_at_11.51.47_AM.png)

The user's primary identity provider is shown as `oauth2`...

![Placeholder](img/Screen_Shot_2019-03-12_at_11.52.01_AM.png)

... and the raw json of this user contains the netID identity of the user:

![Placeholder](img/Screen_Shot_2019-03-12_at_11.52.12_AM.png)


### netID Backend

The `Meine Daten` (`My Data`) section for the user within the netID backend:

 ![Placeholder](img/Screen_Shot_2019-03-12_at_10.48.22_AM.png)

The user can see an overview of approved applications/services under `Meine Dienste` (`My Services`).

 ![Placeholder](img/Screen_Shot_2019-03-12_at_10.47.58_AM.png)

The user can always review and revoke his consent that grants the service access to his personal information.

![Placeholder](img/Screen_Shot_2019-03-12_at_10.52.06_AM.png)

## Photo Credits
[Photo by Fezbot2000 on Unsplash](https://unsplash.com/photos/V4CU3OBsrEk)
