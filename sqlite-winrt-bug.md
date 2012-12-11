Title: SQLite on WinRT bug
Date: 2012.09.14
Summary: Got some queries that run fine, but if you add a column to the SELECT list or change the ORDER BY stop working? Got queries in a transaction that sometimes fail? Read on for the solution...

<!-- Main hero unit for a primary marketing message or call to action -->
<div class="hero-unit">
<h1>SQLite on WinRT bug</h1>
<p>Got some queries that run fine, but if you add a column to the SELECT list or change the ORDER BY stop working? Got queries in a transaction that sometimes fail? Read on for the solution...</p>
<!--<p><a class="btn btn-primary btn-large">Learn more &raquo;</a></p>-->
</div>

While working on a Windows 8 app that uses a SQLite database, I was running into a very strange bug. I would construct a query that would work fine, but when I added certain columns to the ORDER BY clause, suddenly no results would be returned. I ruled out syntax issues since the same queries worked fine when ran in [SqlSpy][]. After much debugging, I finally decided it was just a bug since I was using the Release Preview of Windows 8, beta version of VS 2012, and a pre-release version of SQLite with preliminary support for WinRT.

Fast-forward a few months and the bug cropped up again. This time, I added an additional single column to the SELECT clause and the query broke. By now, I had upgraded to the RTM version of Windows 8 and was using the official WinRT build of SQLite so I was less confident that it was just a beta bug. I decided that I needed to dig deeper.

Whenever the issue cropped up, the SQLite.Step() command would return 'CannotOpen', which [the documentation][CannotOpen] simply says means: "Unable to open the database file". A call to SQLite.GetErrorMsg() returned "library routine called out of sequence" for which the documentation [listed several possible causes][Causes], but none of them really applied. The database was definitely opened correctly and not closed unexpectedly. The calling code was isolated to a single thread, so no wonkiness there. The issue *was* occuring on the Step() command, but the statement pointer was definitely prepared properly and valid.

At this point, I could only assume the issue was deep in the bowels of the sqlite3.dll and I was not sure what to do. I googled a bunch of things on hunches, but nothing was coming up. Finally, I stumbled across [a posting][Posting] by a guy having a semi-similar problem. He was executing statements inside of a transaction and having some work and some fail. Several folks had suggestions on things to try, but nothing worked. Finally, Joe Mistachkin replied with this nuget of wisdom:

> Setting the sqlite3\_temp\_directory to the value contained in the
"Windows.Storage.ApplicationData.Current.TemporaryFolder.Path" property should clear the issue.  This can be done immediately after opening the connection using PRAGMA temp\_store\_directory command on the newly opened database connection. 

Of course! WinRT apps cant access the full file system the same way that normal desktop apps can. In fact, by default, they can only access their own installation location. Apparently, sometimes SQLite needs to write to temporary files in order to execute queries and perhaps my changes were just enough to cause the database engine to need to use such a file. Joe's advice made sense in that SQLite for WinRT would need to be told where it was allowed to store temporary files.

A quick look at the [temp\_store\_directory][TempStoreDirectory] documentation made it clear that this PRAGMA was deprecated and should not be used. (For those that dont know (like I didnt), a [PRAGMA statement][PRAGMA] is a SQLite-specific command that can be used to modify the operation of the SQLite library.) Now that I knew the cause of the issue, my googling was better focused and I found [this advice][IgnoreAdvice] that it was fine to use the temp\_store\_directory setting in this case.

So armed with a potential solution, I coded up a quick test and it did indeed solve the issue - success at last! For those using [Frank A. Krueger's sqlite-net][sqlite-net] wrapper, here are the changes required (all in the SQLiteConnection class):

	static bool isTempStoreSet;

	public SQLiteConnection(string databasePath, bool storeDateTimeAsTicks = false)
	{
	    DatabasePath = databasePath;
	    Sqlite3DatabaseHandle handle;
	    var r = SQLite3.Open(DatabasePath, out handle);
	    Handle = handle;
	    if (r != SQLite3.Result.OK)
	    {
	        throw SQLiteException.New(r, String.Format("Could not open database file: {0} ({1})", DatabasePath, r));
	    }
	    _open = true;
	
	    /* NOTE: Added to ensure that the temp directory is correctly set for WinRT apps.
	     * See: http://www.mailinglistarchive.com/html/sqlite-users@sqlite.org/2012-08/msg00238.html
	     * */
        if (!isTempStoreSet)
        {
            Execute(String.Format("PRAGMA temp_store_directory = '{0}'", Windows.Storage.ApplicationData.Current.TemporaryFolder.Path));
            tempStoreSet = true;
        }
	    /* END NOTE */
	
	    StoreDateTimeAsTicks = storeDateTimeAsTicks;
	
	    BusyTimeout = TimeSpan.FromSeconds(0.1);
	}

That was one of the longest bug-hunting expeditions I had been on for a long time, so I was glad to finally get it sorted. Thanks to the wonderful internet for being filled with folks much smarter than me for doing most of the hard work - I just wrote it up here so it was all in one place and hopefully will help somebody else out in the future.


[SqlSpy]: http://www.yunqa.de/delphi/doku.php/products/sqlitespy/index
[CannotOpen]: http://www.sqlite.org/c3ref/c_abort.html
[Causes]: http://sqlite.org/cvstrac/wiki?p=LibraryRoutineCalledOutOfSequence
[Posting]: http://sqlite.1065341.n5.nabble.com/Transaction-issues-with-WinRT-build-td63817.html
[TempStoreDirectory]: http://www.sqlite.org/pragma.html#pragma_temp_store_directory
[PRAGMA]: http://www.sqlite.org/pragma.html#pragma_temp_store_directory
[IgnoreAdvice]: http://stackoverflow.com/a/12246530/373799
[sqlite-net]: https://github.com/praeclarum/sqlite-net/blob/master/src/SQLite.cs