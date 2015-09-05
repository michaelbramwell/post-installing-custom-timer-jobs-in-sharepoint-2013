Installing custom Timer Jobs in SharePoint 2013
===============================================

Abstract
--------
There are plenty of articles which more or less detail the title of this post, [msdn for example](https://msdn.microsoft.com/en-us/library/office/hh528519(v=office.14).aspx), this is just what made sense for myself in the context of a SharePoint 2013 Timer Job that requires Web Application Feature scope. Some of the code contained within is morphed from this [article](http://www.codeproject.com/Tips/634208/Create-and-Deploy-Custom-Timer-Job-Definition-in-S)

Extend SPJobDefinition
----------------------
First thing you need to do is create a new class that extends SPJobDefinition.
Then create your three constructors and override execute method.

<pre>
public class CustomTimerJobExecution : SPJobDefinition
{
  public CustomTimerJobExecution() { }

  public CustomTimerJobExecution(string jobName, SPService service)
  : base(jobName, service, null, SPJobLockType.None)
  {
    this.Title = jobName;
  }

  public CustomTimerJobExecution(string jobName, SPWebApplication webapp)
  : base(jobName, webapp, null, SPJobLockType.ContentDatabase)
  {
    this.Title = jobName;
  }

  public override void Execute(Guid contentDbId)
  {
    try
    {
      // execute your logic here
    }
    catch (Exception ex)
    {
      // do your logging here
      throw;
    }
  }
}
</pre>

In this case we will be creating a feature with Web Application scope. Notice that the second constructor takes a parameter of type SPService, this is what you might use if you had given your feature farm scope. In this instance we are interested in the third constructor as it takes a type of SPWebApplication as one of its parameters. We can then do something like the following in the above Execute method to derive a site url using the SPWebApplication context.

<pre>
SPSecurity.RunWithElevatedPrivileges(delegate()
{
  SPWebApplication webApplication = this.Parent as SPWebApplication;  
  SPContentDatabase contentDb = webApplication.ContentDatabases[contentDbId];

  string siteUrl = string.Empty;

  foreach (SPSite site in contentDb.Sites)
  {

    if (site.RootWeb.Title == "Your Root Web Title")
    {
      using (SPSite site = new SPSite(site.RootWeb.Url))
      using (SPWeb web = site.OpenWeb())
      {
        // web context in here
      }

      break;

    }
  }
});
</pre>

Create the Event Receiver
-------------------------
Right click on your feature and select the option to create an event receiver class.

<pre>
[Guid("c876fbb3-6255-44e2-86ba-f9f7465ca816")]
public class CustomTimerJobEventReceiver : SPFeatureReceiver
{
  public override void FeatureActivated(SPFeatureReceiverProperties properties)
  {
    try
    {
      SPSecurity.RunWithElevatedPrivileges(delegate()
      {
          SPWebApplication parentWebApp = (SPWebApplication)properties.Feature.Parent;
          SPSite site = properties.Feature.Parent as SPSite;
          DeleteExistingJob(parentWebApp);
          CreateJob(parentWebApp);
      });
    }
    catch (Exception ex)
    {
      // do your logging here
      throw ex;
    }
  }

  private void CreateJob(SPWebApplication site)
  {
    try
    {
      CustomTimerJobExecution job = new CustomTimerJobExecution("Custom Timer Job", site);
      job.Schedule = new SPDailySchedule
      {
        BeginHour = 02,
        BeginMinute = 00,
        BeginSecond = 0,
        EndHour = 02,
        EndMinute = 30,
        EndSecond = 0,
      };

      job.Update();
    }
    catch (Exception ex)
    {
      // do your logging here
      throw;
    }
  }

  public void DeleteExistingJob(SPWebApplication site)
  {
    try
    {
      foreach (SPJobDefinition job in site.JobDefinitions)
      {
        if (job.Name == "Custom Timer Job")
        {
          job.Delete();
        }
      }
    }
    catch (Exception ex)
    {
      // do your logging here
      throw;
    }
  }

  public override void FeatureDeactivating(SPFeatureReceiverProperties properties)
  {
    try
    {
      SPSecurity.RunWithElevatedPrivileges(delegate()
      {
        SPWebApplication parentWebApp = (SPWebApplication)properties.Feature.Parent;
        DeleteExistingJob(parentWebApp);
        });
      }
      catch (Exception ex)
      {
        // do your logging here
        throw ex;
      }
    }
  }
</pre>

Extending SPFeatureReceiver we need to override two methods; FeatureActivated and FeatureDeactivating. Starting with FeatureActivated we can derive the SPWebApplication context from properties.Feature.Parent, properties being a parameter of type SPFeatureReceiverProperties.

We can now call DeleteExistingJob method to remove the timer job if it already exists. While we on this method lets quickly jump to FeatureDeactivating method which simply calls DeleteExistingJob to remove the timer job. Back to FeatureActivated, the timer job can now be created which is achieved by calling CreateJob method which creates an instance of our CustomTimerJobExecution class. Inherited from SPJobDefinition is the property Schedule which in this case we create a new SPDailySchedule. Speaking of scheduling we could have also chosen to [schedule by the minute, week, month or a one of event](https://msdn.microsoft.com/en-us/library/office/microsoft.sharepoint.spschedule(v=office.15).aspx).

Mind the Manifest
-----------------
Your feature's manisfest file contains configuration data such as the Receiver Assembly name and importantly the Receiver Class, that is the feature's event receiver. If at anytime you say change the class name of the event receiver Visual Studio will not automatically update the receiver class, which unless you manually correct, will cause the timer job installation and/or execution of it to fail.
While we are in configuration land make sure the feature's properties; 'Activate On Default' is set to false and 'Always Force Install' is set to true.

Installing
----------
Once your projects Package contains the timer job feature you can deploy via visual studio or PowerShell. If the timer job feature is not hidden in the features properties configuration you will be able enable and disable it from SharePoint Central Administration under the web application's list of features. However PowerShell is better:

<pre>
Enable-SPFeature -Identity "CustomTimerJobFeature" -Url {url of your web application} -confirm:$false
</pre>

It often helps to then reboot all TimerService instances as your projects assembly files will often be cached by the Time Jobs OWSTIMER process.

<pre>
$farm = Get-SPFarm
$farm.TimerService.Instances | foreach{$_.Stop();$_.Start();}
</pre>

Debugging with ULS
------------------
During your feature install it is advisable to be watching the tail on your  [ULS](https://msdn.microsoft.com/en-us/library/office/ff512738(v=office.14).aspx) logs. This will help identify the source of any exceptions. If your exception is complaining about your project assembly missing then it is likely you have an incorrect assembly mapping in you feature's manifest (see above).

Once your Timer Job feature is successfully installed you will also want to watch for exceptions from the execution of the Timer Job. So launch your job from with SharePoint Central Admin; central admin -> monitoring -> timer jobs, and pretty soon you should see your job kick off under the OWSTIMER process, any exception raised by your code executed from with the Execute override of SPJobDefinition method (see above) will now be apparent. As an alternative to ULS logging; you can of course attach the Visual Studio debugger to the OWSTIMER process, this however for reason's unknown to me does not always work in my local development environment.
