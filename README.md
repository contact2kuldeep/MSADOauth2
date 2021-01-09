# MSADOauth2

I started to study how to do a simple Azure AD Oauth2 authentication without the need to maintain the session and in overall no need for additional bells and whistles. First I of course ended up testing magium/active-directory , but it is a huge 10Mb package of code that is hard to validate and you need Composer to even install it properly that adds even more overhead that I just didn’t want to carry. After doing enough googling and concluding that there is no good example for PHP how to implement a single page authentication with Oauth2 like specified here, I decided to write my own targeted especially for Azure AD integration.

### Before going into my example, I need to state that there are nice examples for many programming languages available from Microsoft and other tools and documentation that I find useful:

- https://docs.microsoft.com/en-us/azure/active-directory/develop/sample-v2-code
- https://docs.microsoft.com/en-us/azure/active-directory/develop/v2-oauth2-auth-code-flow
- https://docs.microsoft.com/en-us/graph/permissions-reference
- https://developer.microsoft.com/en-us/graph/graph-explorer
- https://docs.microsoft.com/en-us/rest/api/azure/ (in case you wonder the possibilities of Oauth2 with Azure)
- https://code.visualstudio.com/#alt-downloads (really good code editor for multiple operating systems)

The code above for sure is not implementing everything defined by Oauth2 standard, but it seems to do its job. If you plan to use it for something else than just testing, please remove the unnecessary var_dumps and echo “< pre >” from the beginning of the script and of course add the things needed for your application.


```
{
    "@odata.context": "https://graph.microsoft.com/v1.0/$metadata#users/$entity",
    "businessPhones": [
        "+1 412 555 0109"
    ],
    "displayName": "Megan Bowen",
    "givenName": "Megan",
    "jobTitle": "Auditor",
    "mail": "MeganB@M365x214355.onmicrosoft.com",
    "mobilePhone": null,
    "officeLocation": "12/1110",
    "preferredLanguage": "en-US",
    "surname": "Bowen",
    "userPrincipalName": "MeganB@M365x214355.onmicrosoft.com",
    "id": "48d31887-5fad-4d73-a9f5-3c356e68a038"
}
```

#### At Azure AD you need to make an app registration that matches to your application. Here is how to do that:

- Just open https://aad.portal.azure.com or https://portal.azure.com and open “Azure Active Directory” there.
- From left menu under Manage section open “App registrations”.
- Next click “+ New registration” from the top of the view you just opened.
- Now you can enter the name of your application and select is your app Single tenant or Multitenant app. And this selection of course depends how publicly you mean to share this application. If you are unsure, select Single tenant to be at the safe side. You can change this later if needed from “Authentication” page. The most important thing in this view is to give the Redirect URI to your authentication page (the page that contains my example code). This needs to be secured with HTTPS, so don’t even bother trying with just http://, since it will not work. However this URL does not need to be publicly available since it is accessed by your browser, not by Azure itself, so even localhost will work as long as you have https:// connection to it.
- Since you now have the app registration created and you are in “Overview” page, please copy your Application (client) ID and Directory (tenant) ID, since you will need those with my example code.
- And now you are almost ready! There is only the one final thing to do, which is creating the “Client secret” for your registered app. Click “Certificates & secrets” from left menu and from the page that opens click “+New client secret” button. Now you can give some description if you like, but the main thing here is to select how long your secret is valid. After you have selected, just click “Add” button and there you have it. Please note that Azure will not show the secret to you afterwards, so you need to copy it now to a safe place or to create a new one if you lost it.

Since I’m too lazy to take screenshots and blurring out the sensitive content, here is Microsoft’s documentation how to create the App registration, this covers bullets 1-5 from above: https://docs.microsoft.com/en-us/azure/active-directory/develop/quickstart-register-ap
The only thing this guide does not show is the Client Secret creation, but that is only couple of clicks and instructed in the last bullet (6).

#### If you need to get also user groups listed, that is rather easy with:

```
  $options = array(
    "http" => array( //Use "http" even if you send the request with https
      "method" => "GET",
      "header" => "Accept: application/json\r\n" .
        "Authorization: Bearer " . $authdata["access_token"] . "\r\n"
    )
  );
  $context = stream_context_create($options);
  $json = file_get_contents("https://graph.microsoft.com/v1.0/me/memberOf", false, $context);
  if ($json === false) errorhandler(array("Description" => "Error received during user group data fetch.", "PHP_Error" => error_get_last(), "\$_GET[]" => $_GET, "HTTP_msg" => $options), $error_email);
  $groupdata = json_decode($json, true);  //This should now contain your logged on user memberOf (groups) information
  if (isset($groupdata["error"])) errorhandler(array("Description" => "Group data fetch contained an error.", "\$groupdata[]" => $groupdata, "\$authdata[]" => $authdata, "\$_GET[]" => $_GET, "HTTP_msg" => $options), $error_email);
```

Locate the above code for example after $userdata section. Please notice that you will need to add slightly more permissions to your app registration or else you will get “empty” array as return:


- Open your app registration and API permissions page.
- Click “+Add a permission” button.
- Select “Microsoft Graph” and then “Delegated permissions”.
- Next you need to expand “Group” section and select “Group.Read.All” permission. Click “Add permissions” button from the bottom.
- The final thing needed is to grant admin consent to your application, where you obviously need high enough permissions to your Azure Active Directory to do so. Note that the admin consent is only granted for this specific permission you added at step 4.


When mapping the user groups to your application, likely the best attributes for that are onPremisesSecurityIdentifier or securityIdentifier. At least these should not break up easily during the time. If you need to see an example output from https://graph.microsoft.com/v1.0/me/memberOf, just use the Graph Explorer to get it.


#### Quick way to inject the login data into your $_SESSION variables:

```
  //This replaces all previous data in your session, but during login process you likely want to do that 
  $_SESSION = $userdata;
  $_SESSION["oauth_bearer"] = $authdata["access_token"];
  $_SESSION["groups"] = $groupdata; 
```
