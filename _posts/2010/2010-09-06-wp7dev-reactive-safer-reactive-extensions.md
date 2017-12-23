---
layout: post
title: "[WP7Dev][Reactive] Safer Reactive Extensions"
date: 2010-09-06 20:26:00 -0400
comments: true
category: Archive
tags: [".NET", "Windows Phone Dev"]
redirect_from: ["/post/2010/09/06/WP7Dev-Reactive-Safer-Reactive-Extensions", "/post/2010/09/06/wp7dev-reactive-safer-reactive-extensions"]
author: jay
---
<!-- more -->
<p><a href="http://blogs.codes-sources.com/jay/archive/2010/09/07/wp7dev-reactive-rendre-les-reactive-extensions-plus-stables.aspx"><em>Cet article est disponible en fran&ccedil;ais.</em></a></p>
<p>When developing .NET applications, unhandled exception in threads have the undesirable effect of terminating the current process.</p>
<p>In the following example :</p>
<p>[code:c#]</p>
<p>&nbsp;&nbsp;&nbsp; static void Main(string[] args)<br />&nbsp;&nbsp;&nbsp; {<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; var t = new Thread(ThreadMethod);<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; t.Start();<br /> <br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Console.ReadLine();<br />&nbsp;&nbsp;&nbsp; }<br /> <br />&nbsp;&nbsp;&nbsp; private static void ThreadMethod()<br />&nbsp;&nbsp;&nbsp; {<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Thread.Sleep(1000); throw new Exception();<br />&nbsp;&nbsp;&nbsp; }</p>
<p>[/code]</p>
<p>The basic exception will invariably terminate the process, and to prevent this, the exception needs to be handled properly :</p>
<p>[code:c#]</p>
<p>&nbsp;&nbsp;&nbsp; private static void ThreadMethod()<br />&nbsp;&nbsp;&nbsp; {<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; try<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; {<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Thread.Sleep(1000); throw new Exception();<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; }<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; catch (Exception e)<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; {<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; // TODO: Log and report the exception<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; }<br />&nbsp;&nbsp;&nbsp; }</p>
<p>[/code]</p>
<p>This makes classes like <a href="http://msdn.microsoft.com/en-us/library/system.threading.thread.aspx" target="_blank">System.Threading.Thread</a>, <a href="http://msdn.microsoft.com/en-us/library/system.threading.timer.aspx" target="_blank">System.Threading.Timer</a> or <a href="http://msdn.microsoft.com/en-us/library/system.threading.threadpool.aspx" target="_blank">System.Threading.ThreadPool</a> very dangerous to use if one wants to have an always running application. It is then required that no unhandled exception gets out of the custom handlers for these classes.</p>
<p>Even if it is possible to be notified when an exception has been raised and not handled properly, using the <a href="http://msdn.microsoft.com/en-us/library/system.appdomain.unhandledexception.aspx" target="_blank">AppDomain.UnhandledException</a> event, most the time this leads to the application being terminated. This termination behavior has been <a href="http://msdn.microsoft.com/en-us/library/ms228965.aspx" target="_blank">introduced in .NET 2.0</a>, to prevent unhandled exception to be silently ignored.</p>
<p>While this is a very appropriate default behavior, in an enterprise environment, I&rsquo;m usually enforcing custom static analysis or NDepend rules to prevent the use of these classes directly. This forces new code to use wrappers that provide a very wide exception handler and logs and reports the exception, but does not terminate the process. That also implies that there is still a very valid bug to be investigated, because exceptions should not be handled that late.</p>
<p>&nbsp;</p>
<h2>The case of the Reactive Framework</h2>
<p>In Silverlight for Windows Phone 7, and in any other .NET 3.5 or .NET 4.0 application that uses the <a href="http://msdn.microsoft.com/en-us/devlabs/ee794896.aspx" target="_blank">Reactive Extensions</a>, it is <a href="http://jaylee.org/post/2010/06/22/WP7Dev-Using-the-WebClient-with-Reactive-Extensions-for-Effective-Asynchronous-Downloads.aspx">very easy to switch between threads</a>.</p>
<p>Reactive operators like Timer, BufferWithTime, ObserveOn or SubscribeOn allow for specific Schedulers like ThreadPool, TaskPool or NewThread to be used, and if a subscriber does not handle exceptions properly, it ends up with a terminated application.</p>
<p>The same exemple here also terminates the application :</p>
<p>[code:c#]<br />&nbsp;&nbsp;&nbsp; static void Main(string[] args)<br />&nbsp;&nbsp;&nbsp; {<br />&nbsp;&nbsp;&nbsp;&nbsp; Observable.Timer(TimeSpan.FromSeconds(10), TimeSpan.FromSeconds(10))<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; .Subscribe(_=&gt; ThreadMethod());<br /> <br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Console.ReadLine();<br />&nbsp;&nbsp;&nbsp; }<br /> <br />&nbsp;&nbsp;&nbsp; private static void ThreadMethod()<br />&nbsp;&nbsp;&nbsp; {<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; throw new Exception();<br />&nbsp;&nbsp;&nbsp; }<br />[/code]</p>
<p>The Observable.Timer operator uses the System.Threading.Timer class and that makes it vulnerable to the same termination problems. Every subscriber needs to handle exceptions thrown in the OnNext delegate, or the application will terminate.</p>
<p>Also, do not think that the OnError delegate passed to Observable.Subscribe will handle exceptions thrown during the execution of OnNext code. OnError only notifies of errors generated by previous Reactive operators, not the current.</p>
<p>&nbsp;</p>
<h2>The IScheduler.AsSafe() extension method</h2>
<p>Unfortunately, it is not possible for now to override the default schedulers used internally by the Reactive operators. The only way to handle all unhandled exceptions properly is to use the ObserveOn operator and intercept calls to IScheduler.Schedule methods. Calls can then be decorated with appropriate exception handlers to log and report the exception without terminating the process.</p>
<p>So, to be able to generalize this logging and reporting behavior, I created the <strong>AsSafe()</strong> extension that I place at the very top of a Reactive expression :</p>
<p>[code:c#]</p>
<p>&nbsp;&nbsp;&nbsp; Observable.Timer(TimeSpan.FromSeconds(10), TimeSpan.FromSeconds(10))<br />&nbsp;&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; .ObserveOn(Scheduler.ThreadPool.AsSafe())<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; .Subscribe(_=&gt; ThreadMethod());</p>
<p>[/code]</p>
<p><br /> And here is the code of this very simple extension method :</p>
<p>[code:c#]<br />public static class SafeSchedulerExtensions<br />{<br />&nbsp;&nbsp;&nbsp; public static IScheduler AsSafe(this IScheduler scheduler)<br />&nbsp;&nbsp;&nbsp; {<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; return new SafeScheduler(scheduler);<br />&nbsp;&nbsp;&nbsp; }<br /> <br />&nbsp;&nbsp;&nbsp; private class SafeScheduler : IScheduler<br />&nbsp;&nbsp;&nbsp; {<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; private IScheduler _source;<br /> <br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; public SafeScheduler(IScheduler scheduler) {<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; this._source = scheduler;<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; }<br /> <br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; public DateTimeOffset Now { get { return _source.Now; } }<br /> <br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; public IDisposable Schedule(Action action, TimeSpan dueTime)<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; {<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; return _source.Schedule(Wrap(action), dueTime);<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; }<br /> <br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; public IDisposable Schedule(Action action)<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; {<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; return _source.Schedule(Wrap(action));<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; }<br /> <br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; private Action Wrap(Action action)<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; {<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; return () =&gt; {<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; try&nbsp; {<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; action();<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; }<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; catch (Exception e) {<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; // Log and report the exception.<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; }<br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; };<br /> <br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; }<br />&nbsp;&nbsp;&nbsp; }<br />}</p>
<p>[/code]</p>
{% include imported_disclaimer.html %}