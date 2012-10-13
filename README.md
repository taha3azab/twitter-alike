# twitter-alike

Using RavenDB in an ASP.NET MVC website

## Using RavenDB from your ASP.NET MVC website is actually quite simple.

First install RavenDB to your project from nuget or the distribution packages. 
A detailed guide on how to do that can be found [here](http://ravendb.net/docs/intro/quickstart/adding-ravendb-to-your-application).

Once you have RavenDB installed, there are only 2 things you need to do: creating a singleton document store, and creating way for RavenDB sessions to be created and closed.

## Creating the document store

The DocumentStore object is not cheap to create, and therefore needs to be created only once per application.

First, declare a static IDocumentStore instance in your application. The easiest way to do that is to add the following somewhere in your global.asax.cs file:


	public static DocumentStore Store;

### In the Application_Start() method of your global.asax.cs file, have the following:


	Store = new DocumentStore { ConnectionStringName = "RavenDB" };
	Store.Initialize();
     
	IndexCreation.CreateIndexes(Assembly.GetCallingAssembly(), Store);


The first 2 lines initialize the DocumentStore using a connection string named RavenDB, 
and the third line automatically creates all indexes that are declared in your code.

If you installed RavenDB from nuget, the connection string was already added for you. 
If not, you will need to add the following to your web.config, 
or insert the connection parameters in the code itself:

	<connectionStrings>
		<add name="RavenDB" connectionString="Url=http://localhost:8080" />
	</connectionStrings>
	
This connection string expects a RavenDB server to be listening on the local 8080 port. 
You can execute the Raven.Server.exe executable from your RavenDB installation folder.

For Embedded mode you will need to configure a DataDir instead.

Now lets look at configuring the session lifecycle.

Managing sessions

Sessions are very cheap to create, but we want to avoid writing unnecessary code in our Controllers. For that end, we will create a RavenDB session when an HTTP session starts, and dispose of it once it ends. Before disposing of the session we will call SaveChanges(), and that will assure all changes will be persisted.

All it takes is the following code in the abstract base Controller which all your application Controllers inherit from:

	public IDocumentSession RavenSession { get; protected set; }
	 
	protected override void OnActionExecuting(ActionExecutingContext filterContext)
	{
		RavenSession = MvcApplication.Store.OpenSession();
	}
 
	protected override void OnActionExecuted(ActionExecutedContext filterContext)
	{
		if (filterContext.IsChildAction)
			return;
	 
		using (RavenSession)
		{
			if (filterContext.Exception != null)
				return;
	 
			if (RavenSession != null)
				RavenSession.SaveChanges();
		}
	}
	
To make things even cleaner, create an abstract controller named RavenController having this code, 
and derive all your Controllers from it. Everything will be handled seamlessly for you, 
and you will now have RavenSession available to query or persist with.

## Routes

The convention for document IDs in RavenDB is to have both the collection name and an integer value separated by a slash, 
like this: {collection name}/{integer}. 
This may cause issues with ASP.NET MVC routing for actions like "Read" or "Delete" requiring a document ID, 
mainly because of the slash.

The easiest way to get around this is to have your controllers use the version of session.Load<>() 
accepting an integer. This function will automatically translate the integer value to the fully qualified document ID, 
and then use it to perform the load operation. 
This will allow you to write a controller accepting a non-string ID, 
and therefore just be able to use the default routing rules of ASP.NET MVC:


	public ActionResult Read(int id)
	{
		 var obj = RavenSession.Load<MyClass>(id);
	 
		 return View(obj);
	}