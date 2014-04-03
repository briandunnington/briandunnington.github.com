Title: Run arbitrary code on the UI thread asynchronously
Date: 2014.04.03
Summary: A gotcha and a solution for easily running async code on the UI thread from a background thread.

<div class="hero-unit">
<h1>Run arbitrary code on the UI thread asynchronously</h1>
<p>A gotcha and a solution for easily running async code on the UI thread from a background thread.</p>
</div>

With the advent of the `async` and `await` keywords in .NET, it is super easy to run arbitrary code asynchronously on a background thread. Everybody is familiar with using `Task.Run()` to do something like this:

<pre><code class="csharp">Debug.WriteLine(<span class="string">&quot;before Task.Run()&quot;</span>);
<span class="keyword">await</span> Task.Run(<span class="keyword">async</span> () =&gt;
{
    Debug.WriteLine(<span class="string">&quot;in Task.Run() - before long running process&quot;</span>);
    <span class="keyword">await</span> Task.Delay(1000);
    Debug.WriteLine(<span class="string">&quot;in Task.Run() - after long running process&quot;</span>);
});
Debug.WriteLine(<span class="string">&quot;after Task.Run()&quot;</span>);
</code></pre>

The code is straight-forward. The calling code calls `Task.Run()` and waits for it to finish, including the `await`-ed code that is being run inside of the task. As expected, the output is:

<pre>
before Task.Run()
in Task.Run() - before long running process
in Task.Run() - after long running process
after Task.Run()
</pre>

So running async code on a background thread is dead easy. What about running async code on the UI thread? It is a common scenario - you have some background thread that reads data or whatever and you need to update some UI based on that operation. The usual tool to use in this case is the CoreDispatcher's [RunAsync()][CoreDispatcher] method. If we rewrote our example above, it would look like:

<pre><code class="csharp">Debug.WriteLine(<span class="string">&quot;before Dispatcher.RunAsync()&quot;</span>);
<span class="keyword">await</span> Dispatcher.RunAsync(CoreDispatcherPriority.Normal, <span class="keyword">async</span> () =&gt;
{
    Debug.WriteLine(<span class="string">&quot;in Dispatcher.RunAsync() - before long running process&quot;</span>);
    <span class="keyword">await</span> Task.Delay(1000);
    Debug.WriteLine(<span class="string">&quot;in Dispatcher.RunAsync() - after long running process&quot;</span>);
});
Debug.WriteLine(<span class="string">&quot;after Dispatcher.RunAsync()&quot;</span>);
</code></pre>

But if you run this code, you will notice something strange. The output is:

<pre>
before Dispatcher.RunAsync()
in Dispatcher.RunAsync() - before long running process
after Dispatcher.RunAsync()
in Dispatcher.RunAsync() - after long running process
</pre>

The outer `await` returned control to the method before the inner `await` was complete. Is it a bug in the framework? Nope. The difference is in the argument type passed to `Task.Run()` and `CoreDispatcher.RunAsync()`. In `Task.Run()`, the argument is of type `Func<Task>` so the resulting `Task` is also `await`-ed. But the argument passed to `CoreDispatcher.RunAsync()` is of type [DispatchedHandler] which is a delegate that returns `void`. Since it returns `void`, the inner code block cannot be `await`-ed and returns immediately, which returns control to the outer block which continues to execute.

So now that you know about this potential pitfall, what should you do about it? Well, the entire point of using the `CoreDispatcher` in the first place was to emulate `Task.Run()`'s ability to execute a block of code on another thread (specifically the UI thread). What we really need (and wanted all along) was something like `UITask.Run()` that takes the same `Func<Task>` argument and results in the same behavior but on the UI thread instead of a background thread.

So, I present to you the `UITask` class (full source code at the end of the article). If we rewrite our example:

<pre><code class="csharp">Debug.WriteLine(<span class="string">&quot;before UITask.Run()&quot;</span>);
<span class="keyword">await</span> UITask.Run(<span class="keyword">async</span> () =&gt;
{
    Debug.WriteLine(<span class="string">&quot;in UITask.Run() - before long running process&quot;</span>);
    <span class="keyword">await</span> Task.Delay(1000);
    Debug.WriteLine(<span class="string">&quot;in UITask.Run() - after long running process&quot;</span>);
});
Debug.WriteLine(<span class="string">&quot;after UITask.Run()&quot;</span>);
</code></pre>

Now we get the expected output:

<pre>
before UITask.Run()
in UITask.Run() - before long running process
in UITask.Run() - after long running process
after UITask.Run()
</pre>

Behind the scenes, the `UITask` class still relies on the `CoreDispatcher` to marshal the call back to the UI thread, but it wraps it in a [TaskCompletionSource] so we can wait until the inner code block is actually done. Note that because it still relies on the `CoreDispatcher`, you need to intialize `UITask` with an instance of the `CoreDispatcher` when you app starts up.

Full source for `UITask`:

<pre><code class="csharp"><span class="keyword">using</span> System;
<span class="keyword">using</span> System.Threading.Tasks;
<span class="keyword">using</span> Windows.UI.Core;

<span class="keyword">namespace</span> UITaskSample
{
    <span class="keyword">public</span> <span class="keyword">static</span> <span class="keyword">class</span> UITask
    {
        <span class="keyword">static</span> CoreDispatcher dispatcher;

        <span class="keyword">public</span> <span class="keyword">static</span> <span class="keyword">void</span> Initialize(CoreDispatcher dispather)
        {
            UITask.dispatcher = dispather;
        }

        <span class="keyword">public</span> <span class="keyword">static</span> Task Run(Action action)
        {
            <span class="keyword">if</span> (dispatcher == <span class="keyword">null</span>)
                <span class="keyword">throw</span> <span class="keyword">new</span> InvalidOperationException(<span class="string">&quot;UITask must be initialized with a dispatcher before calling Run()&quot;</span>);

            <span class="keyword">var</span> tcs = <span class="keyword">new</span> TaskCompletionSource&lt;<span class="keyword">bool</span>&gt;();
            <span class="keyword">var</span> ignore = dispatcher.RunAsync(CoreDispatcherPriority.Normal, () =&gt;
            {
                <span class="keyword">try</span>
                {
                    action();
                    tcs.TrySetResult(<span class="keyword">true</span>);
                }
                <span class="keyword">catch</span> (Exception ex)
                {
                    tcs.TrySetException(ex);
                }
            });
            <span class="keyword">return</span> tcs.Task;
        }

        <span class="keyword">public</span> <span class="keyword">static</span> Task Run(Task task)
        {
            <span class="keyword">return</span> Run(() =&gt; task);
        }

        <span class="keyword">public</span> <span class="keyword">static</span> Task Run(Func&lt;Task&gt; taskFunc)
        {
            <span class="keyword">if</span> (dispatcher == <span class="keyword">null</span>)
                <span class="keyword">throw</span> <span class="keyword">new</span> InvalidOperationException(<span class="string">&quot;UITask must be initialized with a dispatcher before calling Run()&quot;</span>);

            <span class="keyword">var</span> tcs = <span class="keyword">new</span> TaskCompletionSource&lt;<span class="keyword">bool</span>&gt;();
            <span class="keyword">var</span> ignore = dispatcher.RunAsync(CoreDispatcherPriority.Normal, <span class="keyword">async</span> () =&gt;
            {
                <span class="keyword">try</span>
                {
                    <span class="keyword">await</span> taskFunc();
                    tcs.TrySetResult(<span class="keyword">true</span>);
                }
                <span class="keyword">catch</span> (Exception ex)
                {
                    tcs.TrySetException(ex);
                }
            });
            <span class="keyword">return</span> tcs.Task;
        }
    }
}
</code></pre>


[CoreDispatcher]: http://msdn.microsoft.com/en-us/library/windows/apps/windows.ui.core.coredispatcher.runasync.aspx
[DispatchedHandler]: http://msdn.microsoft.com/en-us/library/windows/apps/windows.ui.core.dispatchedhandler.aspx
[TaskCompletionSource]: http://msdn.microsoft.com/en-us/library/dd449174(v=vs.110).aspx