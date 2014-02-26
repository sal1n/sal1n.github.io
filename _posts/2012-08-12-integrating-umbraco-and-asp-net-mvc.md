---
layout: default
title: Pity this busy monster..
category: poem
---

Scenario – you want to integrate a ASP.NET MVC application with Umbraco so that Umbraco handles all of the authentication and user management but the ASP.NET MVC app operates outside of the Umbraco framework.

Essentially we want to be able to decorate our controller actions like this:

[UmbracoAuthorize]
public ActionResult ActionThatRequireAuthentication()
{
    return View();
}
and let Umbraco handle the authentication.


This is surprisingly easy to do, rough steps are below:

1. First create your ASP.NET MVC application and add it in IIS as a virtual application somewhere under Umbraco.

2. Add the virtual app path to the “umbracoReservedPaths” AppSetting in web.config.

3. Amend your Umbraco web.config to stop it being inherited by the MVC application – to do this add:

<location path=”.” inheritInChildApplications=”false”>
…
</location>
around the whole of system.web, system.webServer and system.web.webPages.razor.

4. Now in the ASP.NET MVC app create a custom actionfilter subclassing AuthorizeAttribute:

public class UmbracoAuthorize : AuthorizeAttribute
{
  protected override bool AuthorizeCore(HttpContextBase httpContext)
  {
    // umbraco setting, you probably want these in config file
    long umbracoTicksPerMinute = 600000000L;
    long umbracoTimeOutInMinutes = 20L;

    // try and get the current umbraco context
    var cookie = httpContext.Request.Cookies.Get("UMB_UCONTEXT");

    // there is no umbraco context - force umbraco login
    if (cookie == null || String.IsNullOrEmpty(cookie.Value)) { return false; }

    // I'm using PetaPoco here as a thin ADO.NET wrapping but use whatever you want
    // ensure you have the connectionstring for the Umbraco database
    var db = new PetaPoco.Database("umbraco");

    // let's try and get the umbraco user with associated useful data
    dynamic umbracoUser = db.SingleOrDefault(PetaPoco.Sql.Builder
    .Append("SELECT contextID, timeout, userName, userEmail,
                                                  umbracoUserType.userTypeAlias")
    .Append("FROM umbracoUserLogins")
    .Append("INNER JOIN umbracoUser ON umbracoUserLogins.userID = umbracoUser.ID")
    .Append("INNER JOIN umbracoUserType ON umbracoUserType.id = umbracoUser.userType")
    .Append("WHERE contextID = @0", cookie.Value));

    // user can't be found 
    if (umbracoUser == null) { return false; }

    // update the umbraco timeout to stop you timing out on the main umbraco install
    long timeout = DateTime.Now.Ticks + (umbracoTicksPerMinute * umbracoTimeOutInMinutes);
    db.Execute(PetaPoco.Sql.Builder
    .Append("UPDATE umbracoUserLogins")
    .Append("SET timeout = @0", timeout)
    .Append("WHERE contextId = @0", cookie.Value));

    // are we already authenticated?  if so just authorize
    if (httpContext.User.Identity.IsAuthenticated) { return true; }

    // get role from umbraco - this is UserType but you could always 
    // hook into something else more fine-grained
    string[] roles = new string[1];
    roles[0] = umbracoUser.userTypeAlias;

    // check user is in correct role, if required
    if (this.Roles.Length &gt; 0 &amp;&amp; !this.Roles.Split(',').Any(httpContext.User.IsInRole))
    {
      return false;
    }

    // set the auth cookie using a generic principal
    httpContext.User = new GenericPrincipal(new GenericIdentity(umbracoUser.userName),
                                                                               roles);
    // don't need persistent cookie since all running off umbraco context
    FormsAuthentication.SetAuthCookie(umbracoUser.userName, false);

    // authorise!
    return true;
  }
}
5. Set authentication mode in web.config and ensure the loginUrl is the right place – it’s important that your name attribute IS different from the main Umbraco name else you’ll have a conflict with Umbraco Members.

6. Et voila – you can decorate controller actions with [UmbracoAuthorze] attributes and they will just work provided you are logged in as a User to Umbraco (with the relevant role). It’s a simple way of embedding a ASP.NET MVC application in Umbraco without going down the plugin route and having the extra complexity and overhead.