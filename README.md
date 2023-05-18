# azure-login

This example uses Node and Nest.js - 2023

* Login to Microsoft Azure Console and search for App registrations
<img width="892" alt="Screenshot 2023-05-18 at 7 30 36 AM" src="https://github.com/mkabatek/azure-login/assets/1764486/506ed7c1-365e-46fa-b11b-d2216c09f1a8">
* Cea a new App, New Registration

* <img width="816" alt="Screenshot 2023-05-18 at 7 32 04 AM" src="https://github.com/mkabatek/azure-login/assets/1764486/9ef3f82f-a71f-43d5-87c5-e4050ff810a7">

For Supported account type select: 

```Accounts in any organizational directory (Any Azure AD directory - Multitenant) and personal Microsoft accounts (e.g. Skype, Xbox)```

For Redirect URI select `Web` and create your redirect URI: `http://localhost:5173/auth/microsoft` You will be created this route on the Frontend of your application.

Under `Management` go to `Certificates & secrets`

<img width="265" alt="Screenshot 2023-05-18 at 7 39 19 AM" src="https://github.com/mkabatek/azure-login/assets/1764486/35ff3812-8d30-4259-b1d5-1c686c10086c">

Click New Client Secret to generate a new client secret. Copy the secret and paste it somewhere because you will only be able to view it once.

After you have generated a secret go back to the overview section of app registrations click the `Endpoints` button. Here you will find the two endpoints we will need to complete the flow.

```
OAuth 2.0 authorization endpoint (v2)
https://login.microsoftonline.com/common/oauth2/v2.0/authorize
OAuth 2.0 token endpoint (v2)
https://login.microsoftonline.com/common/oauth2/v2.0/token
```

Also in the overview section you will need the `Application (client) ID`. If you want add or change the redirect URIs, you can find that under Management on the left.

Create a route in your web application on the frontend. This route will hit your server that generates and returns URL using the `OAuth 2.0 authorization endpoint (v2)` above. You will simply add the following query string paramters to the URL above:

```
redirect_uri: `${this.envSvc.get('FRONTEND_ENDPOINT')}/auth/microsoft`, // The redirect URL you setup earlier
client_id: this.envSvc.get('MICROSOFT_CLIENT_ID'), // The client ID from the overview page
response_type: 'code', // Response type will just be a string 'code'
prompt: 'consent',
response_mode: 'query',
scope: ['https://graph.microsoft.com/User.Read'].join(' '), // Here you can define the permission you want to access from the user
```

Below is a node JS example generateing the URL.

```
// Get google auth token
@Get('/microsoft-auth-url')
async microsoftAuthUrl() {
  const url = `https://login.microsoftonline.com/common/oauth2/v2.0/authorize`;
  const options = {
    redirect_uri: `${this.envSvc.get('FRONTEND_ENDPOINT')}/auth/microsoft`,
    client_id: this.envSvc.get('MICROSOFT_CLIENT_ID'),
    response_type: 'code',
    prompt: 'consent',
    response_mode: 'query',
    scope: ['https://graph.microsoft.com/User.Read'].join(' '),
  };

  return `${url}?${querystring.stringify(options)}`;
}
```

When you return the URL to the frontend, use the URL to navigate to the generated URL. This will start the login flow, the user can type in their Microsfot email, and password. If they login succesfully this will return them to the `redirect_uri` specified in both the Azure console in your app registration, and in the generated autherization endpoint you generated above.

The login flow will redirect the user back to your frontend, with a `code` in the query paramters e.g.:

```http://localhost:5173/auth/microsoft?code=<THE CODE WILL BE HERE>```

Use this rediect URL on your frontend to hit your backend server and pass the code to the backend. Below is an example in React/Typescript of the frontend code to hit the server.

```
/**
 * Fetch Microsoft Token/User
 */
export const fetchMicrosoftToken: (code: string) => Promise<any> = (
  code: string
) => {
  return new Promise<any>(async (resolve, reject) => {
    const response = await fetch(SERVICE_ENDPOINT + `/microsoft`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      credentials: "include",
      body: JSON.stringify({ code: code }),
    });
    if (!response.ok) {
      return reject();
    }
    resolve(response.json());
  });
};
```

The above code simply send a `POST` request to the server to a route I have created called `/microsoft`


In your backend route using the `code` you obtained above you will get a `token` using the `OAuth 2.0 token endpoint (v2)`: 
`https://login.microsoftonline.com/common/oauth2/v2.0/token`. Your server will send a `POST` request with the following paramters as form data:

```
code: body.code, // The code you received from the user logging in
client_id: this.envSvc.get('MICROSOFT_CLIENT_ID'), // The client ID of your app created in the azure console
client_secret: this.envSvc.get('MICROSOFT_CLIENT_SECRET'), // The secret you generate at the beggining
redirect_uri: `${this.envSvc.get('FRONTEND_ENDPOINT')}/auth/microsoft`, // Your redirect URI from before
grant_type: 'authorization_code', // This string 'authorization_code'
scope: ['https://graph.microsoft.com/User.Read'].join(' '), // The permissions
```


In the backend my route looks like this:

```
// Get Microsoft auth token/fetch Microsoft user
@Post('/microsoft')
async microsoft(@Body() body: { code: string }) {
  console.log(body.code);
  const url = `https://login.microsoftonline.com/common/oauth2/v2.0/token`;
  const values = {
    code: body.code,
    client_id: this.envSvc.get('MICROSOFT_CLIENT_ID'),
    client_secret: this.envSvc.get('MICROSOFT_CLIENT_SECRET'),
    redirect_uri: `${this.envSvc.get('FRONTEND_ENDPOINT')}/auth/microsoft`,
    grant_type: 'authorization_code',
    scope: ['https://graph.microsoft.com/User.Read'].join(' '),
  };

  const tokens = await firstValueFrom(
    this.httpService.post(url, querystring.stringify(values), {
      headers: {
        'Content-Type': 'application/x-www-form-urlencoded',
      },
    }),
  );

  // Fetch the user's profile with the access token and bearer
  const microsoftUser = await lastValueFrom(
    this.httpService.get(`https://graph.microsoft.com/v1.0/me/`, {
      headers: {
        Authorization: `Bearer ${tokens.data.access_token}`,
      },
    }),
  );

  let existingAccount = await noThrow(() => this.accountRepo.findByEmail(microsoftUser.data.mail));
  if (existingAccount) {
    // If user has existing account, log them in.

  } else {
    // Create new account if logging in with microsoft for the first time.

  }
  return existingAccount;
}
```

You can see above after we get the token (it can ben found in the response `token.data.access_token`), we use that token to send a GET request to `https://graph.microsoft.com/v1.0/me/`, with the token set in the headers as `Authorization: Bearer ${tokens.data.access_token}`. This GET request will return the user's data: email address etc. This will indicate a succeful login and then you can use this login information to either log the user into your service or create a new account for them. THE END.




