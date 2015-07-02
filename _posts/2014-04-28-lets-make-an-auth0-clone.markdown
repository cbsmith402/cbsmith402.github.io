---
layout: post
title: "Let's make an Auth0 clone with ASP.NET Identity"
date: 2014-04-28 12:00:00
categories: aspnet jwt authentication sheetrock
---

The Problem
-----------
What's the least fun part of building an app, web or otherwise? For me, it's always authentication. Registration screens, email confirmations, password hashing, OAuth voodoo... It's a pain, and most of it is boilerplate, but there's always enough small differences that I end up re-inventing the wheel with every project I do. I just want to get to the MEAT of the application, not write boring user management stuff!

Then I discovered Auth0. It seemed perfect: Just create an application in their dashboard, add a couple lines of code to your project and you can authenticate users! Awesome!

I want to say before I get into the rest of this that I absolutely love the concept of Auth0. Delegating authentication and possibly authorization to a vendor who knows what they're doing seems like a good idea. They have a solid API, a beautiful widget that works both on the web and mobile, and scads of API and platform integrations. Their support is top-notch too. But they recently changed their pricing model and that made it less attractive for me. I strongly dislike tiered pricing and prefer pay-what-you-use models much more. That's probably why I'm such a huge fan of Azure and Cloudant, but I digress.

So with Auth0's pricing changes, as well as the fact that I don't "own" the user data on my site, I got to thinking about alternatives. Without finding any decent ones, I thought "Am I not a DEVELOPER?!?" and decided to build my own.

Enter Sheetrock
---------------
I'm going to call it Sheetrock because it was partially inspired by the wonderful Drywall starter kit for Node.js projects. Get it? Sheetrock, drywall... 

Anyway.

Let's talk about what this thing should do. It should handle logging in with both a username/password or social providers, email confirmation, password reset and a basic dashboard for viewing my users. Basically, all the stuff we need to do but hate doing. I'll be blogging my progress with creating and improving it. My goal is to get to the same basic level of functionality as Auth0, albeit without some of the more esoteric login providers and "enterprise" scenarios. I just want to log in to a web app and a mobile app for now. I'd also like it to work with Azure Mobile Services, since that's an awesome part of Azure. That means we need to generate the JWT tokens that AMS expects.

JWT
---
What's a JWT?

Well, it stands for Javascript Web Token, and it's my favorite replacement for cookies. It's basically an encrypted JSON object with a specific structure. I could get into all the reasons why they're awesome, but the Auth0 guys have already done that for me here: <a href="https://auth0.com/blog/2014/01/27/ten-things-you-should-know-about-tokens-and-cookies/">https://auth0.com/blog/2014/01/27/ten-things-you-should-know-about-tokens-and-cookies/</a>

Another thing is that since all you need to decrypt a JWT is the master key, we can host the login service on a different domain and not have to worry about cookie domains and whatnot. In fact, for one mobile-only app I'm working on, I'll put the login service on Azure and just host it at the https://blah.azurewebsites.net domain. Free SSL cert!

But this all sounds like a lot, and it is, but most of it has already been done for us!


ASP.NET Identity to the rescue
------------------------------
Thankfully, Microsoft has built most of the moving parts of these requirements into the various ASP.NET OWIN modules, namely the ASP.NET Identity ones. In fact, they've also created a template for a basic website that does login, user management AND role management. However, it's only half the puzzle as it only handles cookie authentication, not JWTs. We can fix that pretty easily, though, as I'll show you.

The first thing we'll want to do is create an empty ASP.NET project in Visual Studio. And by empty I mean, COMPLETELY empty:

![Empty ASP.NET Project]({{ site.url }}/assets/Lets-make-an-Auth0-clone-with-aspnet-identity/Sheetrock_Empty_AspNet.png)

This is because this next Nuget package will essentially set up all the controllers, views and code for us and we don't want anything getting in its way. That package would be the Microsoft.AspNet.Identity.Samples one (you may need to enable Prerelease packages in Nuget to find it). 

![Adding Microsoft.AspNet.Identity.Samples]({{ site.url }}/assets/Lets-make-an-Auth0-clone-with-aspnet-identity/Sheetrock_Add_Identity_Samples_Nuget.png)

This will add all the boilerplate and scaffolding for logging in and managing users. We'll just let it use the built-in LocalDB for now, but let's revisit that in a later blog post.

![Lots of stuff being added]({{ site.url }}/assets/Lets-make-an-Auth0-clone-with-aspnet-identity/Sheetrock_Add_Identity_Samples.png)

Let's run it and make sure we can register and log in.

![Sheetrock First Run]({{ site.url }}/assets/Lets-make-an-Auth0-clone-with-aspnet-identity/Sheetrock_First_Run.png)

Exxxcellent. Now let's register, just to make sure it all works. I'll use my email address and super-secret password.

![Registering myself]({{ site.url }}/assets/Lets-make-an-Auth0-clone-with-aspnet-identity/Sheetrock_Cbsmith_Register.png)

And as you can see, it's going to require an email confirmation. Just another one of the nice things we get for free with ASP.NET Identity.

![Confirm account page]({{ site.url }}/assets/Lets-make-an-Auth0-clone-with-aspnet-identity/Sheetrock_Cbsmith_Confirm.png)

Since I haven't set up a SMTP server for it to send from, it's just going to give me a confirmation link. We'll need to fix this for a production system, but let's keep going. I'll just click on that link to confirm my account.

![Account confirmed]({{ site.url }}/assets/Lets-make-an-Auth0-clone-with-aspnet-identity/Sheetrock_Cbsmith_Confirmed.png)

Let's explore this app a little from an admin's perspective. By default, it creates an admin@admin.com user with a password of "Admin@123456" so you can bootstrap yourself to admin status. Let's peek around at the various screens.

Users list

![Sheetrock Users List]({{ site.url }}/assets/Lets-make-an-Auth0-clone-with-aspnet-identity/Sheetrock_Users_List.png)

Roles (there's only one for now)

![Sheetrock Roles List]({{ site.url }}/assets/Lets-make-an-Auth0-clone-with-aspnet-identity/Sheetrock_Roles_List.png)

User Detail

![Admin user details]({{ site.url }}/assets/Lets-make-an-Auth0-clone-with-aspnet-identity/Sheetrock_User_Details.png)

Nice. It's not perfect, but it's not supposed to be. We can build on this.

So, now we can log in on the web and do some basic user management, but that's not really much of a replacement for Auth0. How do I get a JWT token? And what do I do with it?

What we'll need is a controller action that generates the token from the currently logged in user. I'll use John Sheehan's aptly named JWT Nuget package for that. 

![JWT Nuget package]({{ site.url }}/assets/Lets-make-an-Auth0-clone-with-aspnet-identity/Sheetrock_Add_JWT.png)

And now add the following to our AccountController.cs file:

	```csharp
	[Authorize]
	public ActionResult Token()
	{
		var sha256 = new SHA256Managed();
		var secretBytes = System.Text.Encoding.UTF8.GetBytes("seeeeectretz" + "JWTSig");
		byte[] signingKey = sha256.ComputeHash(secretBytes);

		var issueTime = DateTime.Now;
		var utc0 = new DateTime(1970, 1, 1, 0, 0, 0, 0, DateTimeKind.Utc);
		var exp = (int)issueTime.AddDays(30).Subtract(utc0).TotalSeconds;

		Dictionary<string, object> claims = new Dictionary<string, object>()
		{
			{ "iss", "urn:microsoft:windows-azure:zumo"},
			{ "aud", "urn:microsoft:windows-azure:zumo"},
			{ "uid", User.Identity.GetUserId()},
			{ "exp", exp },
			{ "ver", "2"}
		};

		return RedirectToAction("Success", new { token = JWT.JsonWebToken.Encode(claims, signingKey, JWT.JwtHashAlgorithm.HS256) });
	}

	public ActionResult Success()
	{
		return View();
	}
	```

OK, so there's a lot going on here. Let's step through it.

First notice that I've marked the Token action with an [Authorize] attribute. That way someone has to log in first before they can get a token, duh! This also has a nice side effect that I'll show you in a minute.

After that, we're going to create a hash of the master key ("seeeeectretz"). Why did I just concatenate two string constants there? Mostly so I could swap out the hard-coded key while I was testing. We'll rectify that little bit of ugliness later and make the key configurable. Right now let's just keep moving.

Next, we calculate an expiration time for the token. In this example, we'll make it expire in 30 days.

Since a JWT token is just a collection of key/value pairs that make up claims, we'll populate a Dictionary with the claims we need to have to authenticate. Those first two items are the "Issuer" ("iss") and "Audience" ("aud") claims. What's with the "microsoft:windows-azure:zumo" stuff? That'll come in when we connect to AMS :)

Then we add the user's ID ("uid") with the currently logged in user, the expiration date of the token ("exp") and the version ("ver") of 2. Again, AMS is the reason for including the version of 2.

Finally, we redirect to the "Success" action, adding the encoded token as a query string value.

In the Success action, we simply return a View, which you'll need to add. Let's add a View under the Views/Account folder called (you guessed it) "Success.cshtml":

	```csharp
	@@{
		ViewBag.Title = "Success";
	}

	<h2>Login successful!</h2>
	```

Yep, all it does it says "Login successful", but it's important for the next part of our little test.

Let's go ahead and run it and see what happens. Try going to http://localhost:&lt;port&gt;/Account/Token URL.

![Sheetrock Token Url]({{ site.url }}/assets/Lets-make-an-Auth0-clone-with-aspnet-identity/Sheetrock_Token_Url_Not_LoggedIn.png)

If you had already logged in to the site, it should take you directly to the Success action and you should see the token in the query string, like this:

![Sheetrock Token Success]({{ site.url }}/assets/Lets-make-an-Auth0-clone-with-aspnet-identity/Sheetrock_Success_Token.png)

If not, you should be redirected to the login page:

![Sheetrock Token Login]({{ site.url }}/assets/Lets-make-an-Auth0-clone-with-aspnet-identity/Sheetrock_Login_ReturnUrl_Token.png)

Notice how the return URL is /Account/Token? That will automatically send the user to the token handler once they successfully log in. This way, in our app, we just send them to /Account/Token and if they're not logged in, it'll prompt for credentials. If not (or, importantly, if they've checked "Remember me"), it'll go directly to the Success page with the token!

OK, so we have a token. Wonderful. What do I do with it? Well, let's try doing a round trip and authenticating to ourselves first. To do that, we'll need to head to Nuget again and grab another nice Microsoft package called Microsoft.Owin.Security.Jwt.

![Adding Microsoft.Owin.Security.Jwt]({{ site.url }}/assets/Lets-make-an-Auth0-clone-with-aspnet-identity/Sheetrock_OWIN_JWT_Nuget.png)

OK, cool. Now, let's look in the App_Start/Startup.Auth.cs. As you can see, it's already set up to check for cookies to authenticate:

![Startup.Auth cookie handler code]({{ site.url }}/assets/Lets-make-an-Auth0-clone-with-aspnet-identity/Sheetrock_VS2013_StartupAuth.png)

It's actually as simple as adding a JWT handler. Put this right above that cookie code:

	```csharp
	var sha256 = new SHA256Managed();
	var secretBytes = System.Text.Encoding.UTF8.GetBytes("seeeeectretz" + "JWTSig");
	byte[] signingKey = sha256.ComputeHash(secretBytes);

	app.UseJwtBearerAuthentication(
		new JwtBearerAuthenticationOptions
		{
			AuthenticationMode = Microsoft.Owin.Security.AuthenticationMode.Active,
			AllowedAudiences = new[] { "urn:microsoft:windows-azure:zumo" },
			IssuerSecurityTokenProviders =
				new[]
			{
			    new SymmetricKeyIssuerSecurityTokenProvider("urn:microsoft:windows-azure:zumo", signingKey)
			}
	});
	```

Notice that there's some repetitive code there :) Let's go through this one too.

First, we create the same signing key as the Token action. Again, we'll fix this later. Then, we call the UseJwtBearerAuthentication extension method on the app. That will tell the OWIN pipeline to look for JWT bearer tokens. Notice that we do keep the cookie code, though, so we'll be able to use either method to log in.

AuthenticationMode tells it whether it can modify the response if the token is invalid, or whether the rest of the app will handle it. Basically, we're telling it that we want the JWT module to handle 401s.

Then we tell it which audiences we will accept in the token. We'll use that goofy AMS one again, but we could also add our own if we needed to differentiate. And finally, we'll tell it to use a SymmetricKeyIssuerSecurityTokenProvider to decode the token, giving the issuer name (again, same as AMS) and the signing key.

So, to test that this works, let's add a simple Web API to this and see if it'll take a token instead of a cookie. First, add a Web API v2 controller called TestApiController.

![Adding a test Web API controller]({{ site.url }}/assets/Lets-make-an-Auth0-clone-with-aspnet-identity/Sheetrock_Add_TestApiController.png)

Then, since we started with an empty ASP.NET app, it doesn't have the scaffolding for Web API controller routing, so we'll need to add it. Create a file called WebApiConfig.cs under App_Start and put the following in it:
	
	```csharp
	using System.Web.Http;

	namespace Sheetrock.App_Start
	{
		public static class WebApiConfig
		{
			public static void Register(HttpConfiguration config)
			{
				// Web API configuration and services

				// Web API routes
				config.MapHttpAttributeRoutes();

				config.Routes.MapHttpRoute(
					name: "DefaultApi",
					routeTemplate: "api/{controller}/{id}",
					defaults: new { id = RouteParameter.Optional }
				);
			}
		}
	}
	```

Now we also need to modify the Global.asax.cs to call it:

	```csharp
	AreaRegistration.RegisterAllAreas();
	GlobalConfiguration.Configure(WebApiConfig.Register);
	FilterConfig.RegisterGlobalFilters(GlobalFilters.Filters);
	RouteConfig.RegisterRoutes(RouteTable.Routes);
	BundleConfig.RegisterBundles(BundleTable.Bundles);
	```

And finally, let's just keep the basic Web API, but add an Authorize attribute to the Get action:
	
	```csharp
	// GET api/<controller>
	[Authorize]
	public IEnumerable<string> Get()
	{
		return new string[] { "value1", "value2" };
	}
	```

This is just a simple test, but it should validate that we're accepting tokens. To pass JWTs in via an HTTP call, the standard is to create an HTTP header with the Authorization attribute that looks like this:

	Authorization: Bearer <token goes here>

The easiest way to do this is to use Fiddler to manually create a request to our local site with that header:

![Fiddler with Authorization Bearer header]({{ site.url }}/assets/Lets-make-an-Auth0-clone-with-aspnet-identity/Sheetrock_Fiddler_Added_Header.png)

Click Execute and...

![Fiddler with Authorization Bearer header response]({{ site.url }}/assets/Lets-make-an-Auth0-clone-with-aspnet-identity/Sheetrock_API_Test_Success.png)

Woohoo! So, if we add the Authorization header, the OWIN pipeline will process it and automatically turn it into an authenticated user! And all we had to do was add some code in the Startup.Auth and an Authorize attribute on our actions. Sweet!

Cleaning Up
-----------
Alright, so hard-coding the key in the code is going to make my coder OCD go crazy, so let's fix that before we go any further.

I'd really like to put my master key in the Web.config and load it from one place. Let's create a static class called JwtMasterKeyManager in App_Code to handle that for us:
	
	```csharp
	using System;
	using System.Collections.Generic;
	using System.Linq;
	using System.Security.Cryptography;
	using System.Web;
	using System.Web.Configuration;

	namespace Sheetrock
	{
		public static class JwtMasterKeyManager
		{
			public static byte[] GetEncodedMasterKey()
			{
				var sha256 = new SHA256Managed();
				var secretBytes = System.Text.Encoding.UTF8.GetBytes(WebConfigurationManager.AppSettings["MasterKey"] + "JWTSig");
				byte[] signingKey = sha256.ComputeHash(secretBytes);

				return signingKey;
			}
		}
	}
	```

Now in Startup.Auth, replace the part we added earlier with this:
	
	```csharp
	app.UseJwtBearerAuthentication(
		new JwtBearerAuthenticationOptions
		{
			AuthenticationMode = Microsoft.Owin.Security.AuthenticationMode.Active,
			AllowedAudiences = new[] { "urn:microsoft:windows-azure:zumo" },
			IssuerSecurityTokenProviders =
				new[]
			{
			    new SymmetricKeyIssuerSecurityTokenProvider("urn:microsoft:windows-azure:zumo", Sheetrock.JwtMasterKeyManager.GetEncodedMasterKey())
			}
	});
	```

And our Token action:
	
	```csharp
	[Authorize]
	public ActionResult Token()
	{
		var issueTime = DateTime.Now;
		var utc0 = new DateTime(1970, 1, 1, 0, 0, 0, 0, DateTimeKind.Utc);
		var exp = (int)issueTime.AddDays(30).Subtract(utc0).TotalSeconds;
		var nbf = (int)issueTime.AddDays(-1).Subtract(utc0).TotalSeconds;

		Dictionary<string, object> claims = new Dictionary<string, object>()
		{
			{ "iss", "urn:microsoft:windows-azure:zumo"},
			{ "aud", "urn:microsoft:windows-azure:zumo"},
			{ "uid", User.Identity.GetUserId()},
			{ "exp", exp },
			{ "ver", "2"}
		};

		return RedirectToAction("Success", new { token = JWT.JsonWebToken.Encode(claims, Sheetrock.JwtMasterKeyManager.GetEncodedMasterKey(), JWT.JwtHashAlgorithm.HS256) });
	}
	```

Ahh, much cleaner. Now let's add our "seeeeecretz" token to the web.config:

	```xml
	<appSettings>
		<add key="owin:AppStartup" value="IdentitySample.Startup,Sheetrock" />
		<add key="webpages:Version" value="3.0.0.0" />
		<add key="webpages:Enabled" value="false" />
		<add key="ClientValidationEnabled" value="true" />
		<add key="UnobtrusiveJavaScriptEnabled" value="true" />
		<add key="MasterKey" value="seeeeekretz" />
	</appSettings>
	```

Azure Mobile Services
---------------------
OK, so I've been dangling this over your head up to this point. We can authenticate to the API on our little website here which is all fine and dandy, but what if we want to authenticate to AMS? Maybe I want to have my API there too for some stuff, right?

Well, since we added those "zumo" claims, it's actually pretty easy! Let's take it step by step.

First, if you haven't already, go ahead and create a new Mobile Service in your Azure portal.

![Creating an Azure Mobile Service]({{ site.url }}/assets/Lets-make-an-Auth0-clone-with-aspnet-identity/Sheetrock_Azure_Dashboard.png)

Next, let's create a simple table API. I called mine "posts".

![Adding a table called "posts" in the Azure dashboard]({{ site.url }}/assets/Lets-make-an-Auth0-clone-with-aspnet-identity/Sheetrock_Azure_Add_Table.png)

(Notice how I set it so that only authenticated users can write to the table)

Let's modify the Insert handler to add the user's ID to the item that's posted:

![Modifying the table script to handle inserts]({{ site.url }}/assets/Lets-make-an-Auth0-clone-with-aspnet-identity/Sheetrock_Azure_Table_Script.png)

Now, go back to the dashboard for the mobile service, and down at the bottom click on the Manage Keys button. You should get a window showing you your Application Key and your Master Key. Click the Copy button to copy the Master Key to your clipboard. 

![Azure Mobile Services keys]({{ site.url }}/assets/Lets-make-an-Auth0-clone-with-aspnet-identity/Sheetrock_Azure_Access_Keys.png)

If you're quick, you'll see where this is headed. Just replace "seeeeekretz" in the Web.config with the AMS master key:
	
	```xml
	<appSettings>
		<add key="owin:AppStartup" value="IdentitySample.Startup,Sheetrock" />
		<add key="webpages:Version" value="3.0.0.0" />
		<add key="webpages:Enabled" value="false" />
		<add key="ClientValidationEnabled" value="true" />
		<add key="UnobtrusiveJavaScriptEnabled" value="true" />
		<add key="MasterKey" value="<Master Key Goes Here>" />
	</appSettings>
	```

Let's go back to Fiddler again, but this time we're going to do the HTTP headers a little differently:

![Fiddler with header for Azure Mobile Services]({{ site.url }}/assets/Lets-make-an-Auth0-clone-with-aspnet-identity/Sheetrock_Fiddler_Added_AMS.png)

Notice how instead of "Authorization: Bearer ...", I added a "X-ZUMO-AUTH" header with the token instead. Can anyone guess what the code name for AMS was? :) Now, for the URL, it's just https:&lt;servicename&gt;.azure-mobile.net/tables/&lt;table name&gt;. We'll do a POST to that URL with some simple JSON that just has a Title and Body properties. Hit Execute and...

![Fiddler with successful response from Azure Mobile Services]({{ site.url }}/assets/Lets-make-an-Auth0-clone-with-aspnet-identity/Sheetrock_Fiddler_AMS_Response.png)

Success! AMS has accepted our token since we used the same Master Key. Now let's go look at the data in the portal...

![Azure dashboard showing Mobile Services data we inserted]({{ site.url }}/assets/Lets-make-an-Auth0-clone-with-aspnet-identity/Sheetrock_Azure_Table_Data.png)

Well, look at that. The CreatedBy column has a GUID in it. And that GUID...

![Visual Studio 2013 data explorer showing our users table]({{ site.url }}/assets/Lets-make-an-Auth0-clone-with-aspnet-identity/Sheetrock_VS2013_User_Data.png)

Yep, it's the user ID from our website's user table! So we logged in to one place and a completely separate service authenticated us. Not bad for a few minutes' work (or less), eh?

Summing Up
----------
So, let's see what we've accomplished here:

1. Created a basic website with username/password authentication, password reset, roles, user management, etc.
2. Created an action to generate JWT tokens
3. Set up our website so that APIs can be authenticated with said tokens
4. Set up Azure Mobile Services to accept our tokens as well

We're definitely not done here, though. I've posted the code so far on Github (<a href="https://github.com/cbsmith402/sheetrock">https://github.com/cbsmith402/sheetrock</a>) so you can try it out yourself. In future posts I'll be setting up multi-factor auth, a mobile login component and social providers. Until then, I accept pull requests!