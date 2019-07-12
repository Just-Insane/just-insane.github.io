---
title: The Importance of Security Testing
published: true
categories:
  - 'Blog'
  - 'Security'
tag:
  - 'Keycloak'
  - 'cybersecurity'
  - 'authentication'
canonical_url: https://medium.com/@just_insane/the-importance-of-security-testing-60e8856856c9
---

While setting up a new Keycloak client in my lab over the weekend, I discovered something odd, a number of users in my Keycloak database who should not have been there.

#### Background:

In my homelab environment, I have been using [Keycloak](https://www.keycloak.org/) for securing my services, either directly for applications like Nextcloud, Confluence, and Jira, or indirectly, via a authenticated reverse proxy, for things that don’t such as single page applications like Hashicorp’s [Consul](https://www.consul.io/) or [Jupyter](https://jupyter.org/). This has been a fine setup, and has allowed me to access my internal applications easily and securely from anywhere, with 2FA and seamless access when moving between services. Or so I thought.

#### Discovery:

After noticing some odd users in my Keycloak user’s list, I decided to do some digging. I knew I had initially allowed sign-ups to Keycloak to very locked down services, like read only access to parts of Confluence, and my internal issue tracker on Jira. These users were put into a default role in Keycloak called External-Users, which had very limited access to services on my network.

I decided to create a new user and see what all could be accessed, for the hell of it. I opened a new private window so that I could ensure I was not using cookies from my authenticated session, and went to work creating a new user. I was happily greeted with a [Duo](https://duo.com/) page (Duo is a 2FA provider that you can integrate with to protect many different services), until I realized there was an option to enroll myself as a user in Duo. That is when I realized there was an issue.

#### Breach:

After registering myself as a new user in Duo, I started to poke around, going first to Confluence and Jira, and seeing that I still had my limited permissions, was a bit relived. Then I went to Consul, and the page opened right to the main app, showing all my services humming away happily. Next I went to Jupyter, and immediately was greeted with the option of creating a new shell session.

#### Containment:

As soon as I was able to get access to Jupyter, I disabled signups on Keycloak, cleaned out the unknown users (some of which used legitimate emails), and set Duo to not allow users to enroll themselves. I also ensured there were no active sessions in Keycloak and started locking my network down to ensure there was nobody still inside the perimeter.

#### Root Cause:

The root cause of this issue was my reverse proxy. Knowing that most of my applications were secured behind it, especially the ones which do not handle their own authentication, I tore it apart. I didn’t have to look far to see the error in my ways.

Almost all of my proxied services had this block in the server section (private information omitted):

```
access_by_lua '
 local opts = {
 discovery = "$URL/.well-known/openid-configuration",
 redirect_uri_path = "/redirect_uri",
 redirect_uri_scheme = "https",
 client_id = "OpenResty",
 ssl_verify = "no",
 client_secret = "$ClientSecret",
 session_contents = {id_token=true}
 }
 local res, err = require("resty.openidc").authenticate(opts)

if err then
 ngx.status = 403
 ngx.say(err)
 ngx.exit(ngx.HTTP_FORBIDDEN)
 end
';
```

Let’s break that down a bit, shall we?

My reverse proxy is OpenResty, a modified Nginx server setup to run Lua. On CentOS7 you have to build Nginx from source to add an OIDC authentication plugin, with OpenResty, it is built in.

First, we define some options for the OIDC authentication:

1. The URL of the openid configuration information
2. The path to redirect to, this is a URI that does not exist in the backend application that should be redirected to after authentication with Keycloak so the proxy can check the authentication
3. The redirect scheme
4. The Keycloak Client ID
5. If SSL for Keycloak should be verified (this should be set to yes)
6. The client secret that you get from Keycloak
7. The session contents, in this case, we are just looking for the id\_token
8. The authentication check, more on this below
9. What to do if there is an error, in this case, send a 403, print the error, and set HTTP\_FORBIDDEN, then exit

Looking at the authentication check, do you see the problem? It is only checking that the person accessing the protected resource is authenticated to Keycloak. It does not check for their group, roles, or even to ensure they have a valid email address for the domain.

Since anyone could sign up to Keycloak, and anyone could enroll themselves in Duo, they would be authenticated, and would therefore pass this check automatically, giving anyone who tried access to these internal applications.

#### It Gets Worse:

Knowing that I had setup external Identity Providers in Keycloak, such as Facebook and Google (adding external Identity Providers in Keycloak lets people sign in with other services which handle their own authentication), I went back and tested these out.

I closed out my current private window, and opened a new one, going back to my Consul URL, and clicking on the login button for Google. When prompted, I entered a spare email address and password, and once I got through the Google verification, I was redirected back to Keycloak, and then to the main Consul page.

After checking in Keycloak, I didn’t see a new user created with this email. Since authentication was handled by Google, all Keycloak was caring about was if the user successfully authenticated with Google, and just passed the user back to the proxy as being authenticated in Keycloak.

Not only could anyone sign up for an account in Keycloak and add themselves to 2FA, they could completely bypass these steps without raising suspicion and not creating any user records.

#### Lessons Learned:

The lessons I learned through this were many, the first is to always double check everything, especially when it is security related. If I had checked these authentication flows when initially setting everything up, I would have caught these issues much sooner and this would not have been a problem.

Second, it is important to understand and read the documentation fully. This could have been easily mitigated or completely prevented by adding more checks to OpenResty, disabling sign up or third party authentication in Keycloak, or preventing user self-enrollment in Duo.

Third, check everything. If you enable a feature, ensure it does exactly what you are expecting it to do. If not, it is best to disable it, or get clarification as to why it is doing what it is.

#### Conclusion:

I messed up, spectacularly. Luckily, it was in a test environment and not production, and there was nothing overly sensitive on my network.

This is a mistake that I will not forget, and that I will bring with me in my career.

There is no inherent issue with the tools that I used, or even a combination of tools, it was purely a configuration issue on my part. I would still 100% recommend all of the tools mentioned in this post, and intend to keep on using them. I have gained more respect for them, and intend to put them to use to further secure my network in the future.

#### Future Thoughts:

As I rebuild my lab with a focus on Kubernetes and Zero Trust network design, the lessons I learned here will guide me in ensuring that everything is as secure as possible.