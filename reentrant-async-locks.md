Title: Reentrant async locks in .NET
Date: 2012.12.12
Summary: The new async/await keywords in .NET make async programming a snap, but there are some things to watch out for when synchronizing multiple threads.

<!-- Main hero unit for a primary marketing message or call to action -->
<div class="hero-unit">
<h1>Reentrant async locks in .NET</h1>
<p>The new async/await keywords in .NET make async programming a snap, but there are some things to watch out for when synchronizing multiple threads.</p>
<!--<p><a class="btn btn-primary btn-large">Learn more &raquo;</a></p>-->
</div>

So you have had your head down coding up your awesome new [Windows Store][WindowsStore] and [Windows Phone][WindowsPhone] apps, and the new <code>async/await</code> keywords are making life a lot easier. No more frankenstein code of callbacks and Invokes() littering up your beautiful codebase.

Now you need to load some data for use in your app. Could come from a webservice, or maybe a long-running local database query. Or maybe you have some initialization code that you want to ensure only runs once, even when multiple threads are involved. In the olden-days, you would wrap a <code>lock</code> statement around the critical sections and be on your merry way, but you know that <code>await</code> is [not allowed inside of a lock statement][MsdnAwait] and that [it would be a bad idea][Lippert] if they were.

There are a couple of [great async libraries][Nito] out there, and maybe you have even been reading [Stephen Toub][Toub] and written your own async locking mechanism. There are a few [well-known issues][Deadlock] (and some [lesser-known][Deadlock2] as well) that can cause deadlocks and you have been diligent to avoid them. Your app is humming along nicely until - BAM! - it suddenly it locks up. What gives?

One possible cause is that the same thread tried to obtain the lock multiple times. Unlike the <code>lock</code> keyword, async locks built on top of [SemaphoreSlim][] and other async-aware synchronization primitives **are not reentrant**.

It may seem like it would be easy to spot the culprit - just dont call <code>LockAsync()</code> inside another <code>LockAsync()</code> block, but it is often much more complicated than that. The code you had inside of your async lock may have called out to some other code (and maybe some other code, and other, etc) and eventually *that* code tries to obtain the same lock. It could be in another method or class altogether, or only on certain code paths, making it very hard to track down.

### A solution?

So if async locks are not natively reentrant, the obvious first thought would be to try to make them reentrant so that they act exactly like the <code>lock</code> keyword, which is what we are trying to mimic. This is actually a bit tricky to do, though it can be done. By tracking which threads have obtained the lock (using the [System.Environment.CurrentManagedThreadId][CurrentManagedThreadId]), you can elect to allow reentrant threads to by-pass the <code>WaitAsync</code> (or whatever mechanism you are using). Now you write some code like this:

        async void Button_Click(object sender, EventArgs e)
        {
            // wrap this critical section so we dont have file collisions
            using (new AsyncLock.LockAsync())
            {
                await WriteToFile("test");
            }
        }

        async Task WriteToFile(string text)
        {
            // wrap this critical section so we dont have file collisions
            using (new AsyncLock.LockAsync())
            {
                // write to a file or do some other long-running action
                await Task.Delay(5000);
            }
        }

Even though Button\_Click obtains the lock and then calls WriteToFile, which also tries to obtain the lock, the inner lock request will be granted and the code will work as expected. Perfect - just what we wanted.

Or is it? In reality, it turns out that we don't want to mimic the <code>lock</code> semantics exactly. Consider a simple app with two buttons and the following codebehind:

        async void Button1_Click(object sender, EventArgs e)
        {
            // wrap this critical section so we dont have file collisions
            using (new AsyncLock.LockAsync())
            {
                // write to a file or do some other long-running action
                await Task.Delay(5000);
            }
        }

        async void Button2_Click(object sender, EventArgs e)
        {
            // wrap this critical section so we dont have file collisions
            using (new AsyncLock.LockAsync())
            {
                // write to a file or do some other long-running action
                await Task.Delay(5000);
            }
        }

In the original, non-reentrant version, this code would work as expected. If the user clicks Button1, the lock will be obtained, the long-running task will start, and control will be returned to the calling method. If the user immediately clicks Button2, the lock will be contested so control will be returned immediately, and the long-running task in Button2\_Click will wait and be ran after Button1_Click is finished.

If we allow reentrancy though, Button1\_Click's task will start and when Button2\_Click tries to obtain the lock, we will bypass the lock and allow it to execute, effectively defeating the purpose of the lock altogether. So although we have allowed reentrancy, we have broken our original intent.

It turns out that allowing reentrancy is not so straight-forward. Depending on the situation, different behavior would be expected, and the complexity of tracking the reentrancy only to provide a fragile solution is not a good idea.

### So now what?

So what is the right thing to do? Well, the right thing to do is to avoid needing to lock around asynchronous operations altogether. That will completely avoid this issue and usually result in more robust code. Using the built-in classes in the [System.Collections.Concurrent][Concurrent] namespace coupled with the [Lazy&lt;T&gt;][Lazy] object can often achieve a similar effect without any (user-authored) locks at all. I will write an upcoming post with an example of how to implement this approach.

But in cases where you absolutely have to have locking behavior around asynchronous code, you really have to be extra-super-duper diligent that you never have any reentrant locking code. Using the same thread-tracking mechanisms that we talked about earlier, you could probably build a deadlock-detection scheme that would at least alert you to the issue during testing. Otherwise, you can always leave it up to your users to find for you =)


[WindowsStore]: http://www.windowsstore.com/
[WindowsPhone]: https://dev.windowsphone.com/
[MsdnAwait]: http://msdn.microsoft.com/en-us/library/vstudio/hh156528.aspx
[Lippert]: http://stackoverflow.com/a/7612714/373799
[Nito]: http://nitoasync.codeplex.com/
[Toub]: http://blogs.msdn.com/b/pfxteam/archive/2012/02/12/10266988.aspx
[Deadlock]: http://blogs.msdn.com/b/pfxteam/archive/2011/01/13/10115163.aspx
[Deadlock2]: http://nitoprograms.blogspot.com/2012/12/dont-block-in-asynchronous-code.html
[SemaphoreSlim]: http://msdn.microsoft.com/en-us/library/system.threading.semaphoreslim.aspx
[CurrentManagedThreadId]: http://msdn.microsoft.com/en-us/library/system.environment.currentmanagedthreadid.aspx
[Concurrent]: http://msdn.microsoft.com/en-us/library/system.collections.concurrent.aspx
[Lazy]: http://msdn.microsoft.com/en-us/library/dd642331.aspx