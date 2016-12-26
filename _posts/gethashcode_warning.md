Title: UWP GetHashCode warning
Date: 2016.12.26
Summary: A cautionary tale about making assumptions

<div class="hero-unit">
<h1>UWP GetHashCode warning</h1>
<p>GetHashCode may not work the way you think it does in UWP apps, and it could be breaking your app.</p>
</div>

I have a few small apps in the Windows Store that I developed several years ago. For the most part, they just hum along and I 
don't have to think about them very much. But the other day, I noticed a user review that said [riverFLOW][] was no longer
working. Since I had not updated the app at all since 2012, it was not suprising. The fix was relatively simple (turns out that
the USGS, where I get the source data, had changed over to mandatory HTTPS connections and my app was still using the HTTP urls).

However, when preparing to submit an updated package, I noticed that the app was behaving strangely on Windows 10 Mobile devices.
Favortie rivers were not persisting across app restarts, and pinned rivers were causing the app to crash when launched. I dug into
it and discovered something shocking (at least to me, at the time): string.GetHashCode was returning different values for the same
input each time the app was run!

Since the app dated all the way back to the Windows Phone 7 days, it was using the [AgFx][] library which used GetHashCode-generated
names for the data stored in isolated storage. Since the hash codes were no longer stable across app sessions, the data was effectively
unretrievable. I found [another post][627] by a developer using the same library with the same issues that confirmed my findings.

A quick trip to [MSDN][] made it clear that GetHashCode should not have been relied upon in the first place:

> The hash code itself is not guaranteed to be stable. Hash codes for identical strings can differ across versions of the .NET Framework and across platforms (such as 32-bit and 64-bit) for a single version of the .NET Framework. **In some cases, they can even differ by application domain**.
>
> As a result, hash codes should never be used outside of the application domain in which they were created, they should never be used as key fields in a collection, and **they should never be persisted**. 

That was probably always the case, but since GetHashCode had always returned the same value for the same input (at least on the same platforms),
it was surprising to me to see this new behavior. In my case, I didn't even need the filename hashing feature of AgFx since my data already
had unique names, but I did find [this alternative][alternative] hashing function if you need a stable hash replacement.

Hopefully you all are smarter than me and did not rely on assumptions about implementation details. Just because something has always worked
one way, don't assume it always will.

[riverFLOW]: https://www.microsoft.com/en-us/store/p/riverflow/9nblggh0k7d4
[AgFx]: https://github.com/shawnburke/AgFx
[627]: https://social.msdn.microsoft.com/Forums/sqlserver/en-US/74f88da4-1dec-4700-ad6c-c4b19c8165dc/stringgethashcode-now-returns-different-values-every-time-the-app-is-run-on-a-windows-10-device?forum=wpdevelop
[MSDN]: https://msdn.microsoft.com/en-us/library/system.string.gethashcode(v=vs.110).aspx
[alternative]: https://gfkeogh.blogspot.com/2016/07/windows-universal-gethashcode.html