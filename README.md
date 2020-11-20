# Introduction
Let's get you started with Citadel by walking through this Quickstart app. You'll need a set of API keys which you can request via email team@citadelid.com.

You'll have two different API keys (`client_id`, `access_key`) and a `public_key` for initiating the widget, and we'll start in the Sandbox environment. 


# Set up the Quickstart
Once you have your API keys, it's time to run the Citadel Quickstart app locally.
You have two options - running with `docker` and `docker-compose` or starting with python and virtual env

## Quickstart with docker/docker-compose
*Requirements*: `docker` and `docker-compose` are available.

- git clone https://github.com/citadelid/quickstart
-  cd quickstart
-  make env

After you've done that, you have to put your `SECRET_KEY`, `CLIENT_ID` and `PUBLIC_KEY` in the `.env` file.

The `.env` file should look like this:
```
API_URL=https://prod.citadelid.com/v1/
API_PUBLIC_KEY=<YOUR PUBLIC KEY HERE>
API_SECRET=<YOUR SECRET KEY MUST BE HERE>
API_CLIENT_ID=<YOUR CLIENT_ID HERE>
API_PRODUCT_TYPE=employment, income or admin
```

- make python_docker

After running this command, you should see:
```
web_1  | ======================================== ENVIRONMENT ======================================== 
web_1  |  https://prod.citadelid.com/v1/ 
web_1  |  {
web_1  |     "X-Access-Secret": "<YOUR SECRET_KEY HERE>",
web_1  |     "X-Access-Client-Id": "<YOUR CLIENT_ID HERE>",
web_1  |     "Content-Type": "application/json;charset=UTF-8"
web_1  | } 
web_1  |  ============================================================================================== 
web_1  | 
web_1  |  * Serving Flask app "server" (lazy loading)
web_1  |  * Environment: development
web_1  |  * Debug mode: on
web_1  |  * Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)
web_1  |  * Restarting with stat
web_1  |  * Debugger is active!
web_1  |  * Debugger PIN: 593-914-178

```

To access the app, open http://127.0.0.1:5000/ in your browser.


## Quickstart with python and virtual env
*Requirements*: `python --version` must print into console `Python 3.8.X`

- git clone https://github.com/citadelid/quickstart
- cd quickstart
- make env

After you've done that, you have to put your `SECRET_KEY`, `CLIENT_ID` and `PUBLIC_KEY` in the `.env` file.

The `.env` file should look like this:
```
API_URL=https://prod.citadelid.com/v1/
API_PUBLIC_KEY=<YOUR PUBLIC KEY HERE>
API_SECRET=<YOUR SECRET KEY MUST BE HERE>
API_CLIENT_ID=<YOUR CLIENT_ID HERE>
API_PRODUCT_TYPE=employment, income or admin
```

- make python_local

```
======================================== ENVIRONMENT ======================================== 
 https://prod.citadelid.com/v1/ 
 {
    "X-Access-Secret": "sandbox-a6908e6a533605baddd8d8a9d9fda6cf31aa3e0a",
    "X-Access-Client-Id": "d2f856c7d0b845129feaa8e955462779",
    "Content-Type": "application/json;charset=UTF-8"
} 
 ============================================================================================== 

 * Tip: There are .env or .flaskenv files present. Do "pip install python-dotenv" to use them.
 * Serving Flask app "server" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: off
 * Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)
```

To access the app, open http://127.0.0.1:5000/ in your browser.

# Run your first verification
## Overview
The Quickstart app emulates the experience of an applicant going through a background check/income verification and visiting the applicant portal.

Before using Citadel for verification, an applicant has filled out the form. 

To streamline the process and make employment/income verification easy and instant, we "hide" the form behind the button. 

If the verification is successful via Citadel, then we show to the applicant the data that we found in their payroll account. 

If the verification isn't successful or the applicant decided to exit Citadel's widget, the applicant will see the form, fill it out and the verification can be done via an existing verification process.

## Successful verification

After opening the Quickstart app running locally, click the `Verify employment`/`Verify income` button, search for a company (e.g., `Facebook`) and select a provider.

Use the Sandbox credentials to simulate a successful login.

```
username: goodlogin
password: goodpassword
```

Once you have entered your credentials and moved to the next screen, you have succesfully done your first verification. 

The API call will be executed and the data will be loaded into the form.

## No verification

Click the `Add employer` button, search for a company (e.g., `Facebook`) and select a provider.

Click the exit icon at the top right of the widget and you'll see the empty form.

## Successful administration

If using the `admin` product, after opening the Quickstart app running locally, click the `Connect` button and select any provider. Enter `Skx8LTnyrLiw4SYk8xfkRwOt5OGQbNulypqdsqd` for the API Key and click `submit`. The API call will be executed and the data will be loaded into the drop-down input.

# What happened under the hood
## Token exchange flow

First we initiate the widget using your `public_key`. 

```
        const bridge = CitadelBridge.init({
          clientName: 'Citadel Quickstart',
          companyMappingId: null,
          key: '{{ public_key }}',
          product: '{{ product_type}}',
          trackingInfo: 'any data for tracking current user',
          onLoad: function () {
            console.log('loaded');
            successClosing = null
          },
          onEvent: function (eventType) {
            console.log('event: ', eventType);
          },
          onSuccess: async function (token) {
            console.log('token: ', token);

            successClosing = true

            const content = document.querySelector('.spinnerContainer');

            content.classList.remove('hidden');
            let verificationInfo;
            try {
              verificationInfo = await apiRequests.getVerificationInfoByToken(token);
            } catch(e) {
              console.error(e)
              content.classList.add('hidden');
              return;
            }
            content.classList.add('hidden');

            if (!verificationInfo.length) {
              return;
            }

            setUserInfo(verificationInfo[0]);

            renderEmploymentHistory(verificationInfo);
          },
          onError: function () {
            console.log('error');
          },
          onClose: function () {
            console.log('closed');
            if (successClosing !== true) {
              renderEmploymentHistory([{ company: { address: {} } }]);
            }
          },
        });
        window.bridge = bridge;
      })();
```

Once the widget is initiated, we receive a `public_token` that can be exchanged for an `access_token`, which will be used to access data via the API.  The `access_token` will be unique for this verification.

## API calls
After receiving the `access_token` we can make API calls and pass the data to the frontend to show in the app.

```
    def get_verification_info_by_token(self, access_token: str) -> Any:
        class VerificationRequest(TypedDict):
            access_token: str

        request_data: VerificationRequest = {'access_token': access_token}

        return requests.post(
            self.API_URL + 'verifications/employments/',
            json=request_data,
            headers=self.API_HEADERS,
        ).json()
```

When verification is successful, we show the form and populate it with data that we received from the backend:
```
          onSuccess: async function (token) {
            console.log('token: ', token);

            successClosing = true

            const content = document.querySelector('.spinnerContainer');

            content.classList.remove('hidden');
            let verificationInfo;
            try {
              verificationInfo = await apiRequests.getVerificationInfoByToken(token);
            } catch(e) {
              console.error(e)
              content.classList.add('hidden');
              return;
            }
            content.classList.add('hidden');

            if (!verificationInfo.length) {
              return;
            }

            setUserInfo(verificationInfo[0]);

            renderEmploymentHistory(verificationInfo);
          },
```

And if the widget was closed, we show the form that the applicant can fill out:

```
          onClose: function () {
            console.log('closed');
            if (successClosing !== true) {
              renderEmploymentHistory([{ company: { address: {} } }]);
            }
          },
```




