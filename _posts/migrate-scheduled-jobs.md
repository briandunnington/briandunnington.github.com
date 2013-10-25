Title: Migrating Scheduled Task Jobs
Date: 2013.10.25
Summary: Finally retiring that old Windows Server 2003 machine? Need to migrate your scheduled tasks? Here is an easy solution.

<div class="hero-unit">
<h1>Migrating Scheduled Task Jobs</h1>
<p>Finally retiring that old Windows Server 2003 machine? Need to migrate your scheduled tasks? Here is an easy solution.</p>
</div>

One of our clients is finally migrating from Windows Server 2003 to Windows Server 2012 (don't judge) and I ran into an interesting problem. They have a bunch of jobs set up in the Task Scheduler that need to be migrated over, but the old machine uses a deprecated binary .job file format to store the job data, while the newer machine uses an xml-based file format.

Of course, there is a built-in solution for this problem: the [`schtasks`][schtasks] program now has an `/XML` switch to export an xml file. However, that version of `schtasks` only works on the newer OSes, so you can easily convert the scripts on your old machine.

Surely Google can help, right? It didnt take long to find several solutions [like this][remotemachine] that let you specify the remote host name of the older OS. In that case, you can use the newer `schtasks` (with xml export) but point it at the older machine and everything is easy-peasy.

But that only works if the two machines are on the same domain and can see each other. In my case, we were moving from one virtual hosting company to another, so the two machines could not interact in this way.

Back to Google and I find a [long-winded but working solution][solution] that requires you to copy the old version of `schtasks` to the new machine, pipe some directory output to text file, manually create an Excel spreadsheet that gets saved as a `.bat` file and eventually execute `schtasks` on a copy of the old jobs. Whew - there had to be a better way! So I whipped one up.

<pre><code class="csharp"><span class="keyword">class</span> Program
    {
        <span class="keyword">static</span> <span class="keyword">void</span> Main(<span class="keyword">string</span>[] args)
        {
            Console.WriteLine(<span class="string">&quot;username:&quot;</span>);
            <span class="keyword">var</span> username = Console.ReadLine();
            Console.WriteLine(<span class="string">&quot;password:&quot;</span>);
            <span class="keyword">var</span> password = Console.ReadLine();
            <span class="keyword">var</span> files = Directory.GetFiles(@<span class="string">&quot;c:\windows\tasks&quot;</span>, <span class="string">&quot;*.job&quot;</span>);
            <span class="keyword">if</span> (files == <span class="keyword">null</span> || files.Count() == 0)
            {
                Console.WriteLine(<span class="string">&quot;No .job files found to process&quot;</span>);
            }
            <span class="keyword">else</span>
            {
                <span class="keyword">foreach</span> (<span class="keyword">var</span> file <span class="keyword">in</span> files)
                {
                    <span class="keyword">var</span> fileNameWithoutExtension = Path.GetFileNameWithoutExtension(file);
                    <span class="keyword">var</span> arguments = String.Format(<span class="string">&quot;/change /TN {0} /RU {1} /RP {2}&quot;</span>, fileNameWithoutExtension, username, password);
                    <span class="keyword">var</span> psi = <span class="keyword">new</span> ProcessStartInfo(@<span class="string">&quot;schtasksXP&quot;</span>, arguments);
                    psi.UseShellExecute = <span class="keyword">false</span>;
                    Console.WriteLine(psi.FileName + <span class="string">&quot; &quot;</span> + psi.Arguments);
                    <span class="keyword">var</span> p = Process.Start(psi);
                    p.WaitForExit();
                }
            }
            Console.WriteLine(<span class="string">&quot;DONE&quot;</span>);
            Console.WriteLine(<span class="string">&quot;Press any key to continue...&quot;</span>);
            Console.ReadKey();
        }
    }
</code></pre>

To use it:

- Copy your old .job files to `C:\Windows\Tasks` on the new machine
- Download the .zip file and unzip the contents to a folder
- Run `TaskScheduleJobConverter.exe` which will prompt you for a username & password to be used to execute the imported jobs

When the script completes, all of the old jobs will have been converted and imported into the new machine's Task Scheduler.

[Download the program][download]

[schtasks]: http://msdn.microsoft.com/en-us/library/windows/desktop/bb736357%28v=vs.85%29.aspx
[remotemachine]: http://jon.netdork.net/2011/03/08/powershell-and-exporting-windows-scheduled-tasks/
[solution]: http://superuser.com/a/596401
[download]: /downloads/TaskScheduleJobConverter.zip