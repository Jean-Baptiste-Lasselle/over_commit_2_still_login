# over_commit_2_still_login
How configure a machne with overcommit_memory set to 2, and stillbe able to ssh log into it

<div id="post" class="bg-neutral-11 pbxxxl">
      <div class="post-header">
        

<div class="container">
  <ul class="list-breadcrumb">
    <li><a href="/">Home</a></li>
    
    
      <li class="current"><span>Post</span></li>
    
    
    
  </ul>
</div>

        <div class="phxxl pvl">
          <h1 class="title em-low type-dark-1 mvn">
            
            Virtual memory settings in Linux - The Problem with Overcommit
          </h1>
          <h2 class="h3 type-dark-3 em-default mvn post-summary">How to tune the Memory Overcommit settings in Linux</h2>
          <div class="type-dark-5 em-default">
            Posted on
            <span class="post-date"> Sat, Jul 2, 2016 </span>
            
            
            by
            <ul class="authors">
            
              
                <li><a href="/authors/ascherbaum">Andreas Scherbaum</a></li>
              
            
            </ul>
            <br>
            
              Categories: &nbsp;
                
                  <a href="/categories/linux">Linux</a> &nbsp;&nbsp;
                
                  <a href="/categories/greenplum-database">Greenplum Database</a> &nbsp;&nbsp;
                
                  <a href="/categories/virtual-memory">Virtual Memory</a> &nbsp;&nbsp;
                
                  <a href="/categories/overcommit">Overcommit</a> &nbsp;&nbsp;
                
                <br>
              
            
            <a href="https://github.com/pivotal-legacy/blog/edit/master/content/post/Virtual_memory_settings_in_Linux_-_The_problem_with_Overcommit.md" class="type-sm">Edit this post on GitHub.</a>
            <div class="pull-right">
              <iframe id="twitter-widget-0" scrolling="no" allowtransparency="true" class="twitter-share-button twitter-share-button-rendered twitter-tweet-button" style="position: static; visibility: visible; width: 75px; height: 28px;" title="Twitter Tweet Button" src="https://platform.twitter.com/widgets/tweet_button.2349b7ea03933b93cf1e9e9f69dac37a.en.html#dnt=false&amp;id=twitter-widget-0&amp;lang=en&amp;original_referer=http%3A%2F%2Fengineering.pivotal.io%2Fpost%2Fvirtual_memory_settings_in_linux_-_the_problem_with_overcommit%2F&amp;size=l&amp;text=%C2%B7%20Pivotal%20Engineering%20Journal&amp;time=1567642030427&amp;type=share&amp;url=http%3A%2F%2Fengineering.pivotal.io%2Fpost%2Fvirtual_memory_settings_in_linux_-_the_problem_with_overcommit%2F&amp;via=ascherbaum%20" frameborder="0"></iframe>
            </div>
          </div>
          <hr>
        </div>
      </div>
      <div class="panel-body phxxl pvn">
        

<p><a href="http://greenplum.org/">Greenplum Database</a> users sometimes add more RAM to their segment servers, but forget to adapt the Memory Overcommit settings to make use of all the new memory. After such an upgrade, at first glance, not much happens - except that the system is not using all the memory. But unless you take a close look at the memory usage on the segment hosts, you will not see that some memory goes unused.</p>

<p>To understand what’s going on, let’s first dive into the Linux Memory Management and what Overcommit actually means. Here is the background story:</p>

<h2 id="background">Background</h2>

<h3 id="ram-swap">RAM &amp; Swap</h3>

<p>In modern operating systems like Linux, you often have two types of memory available: physical memory (<a href="https://en.wikipedia.org/wiki/Random_Access_Memories">RAM</a>) and on-disk memory (<a href="https://en.wikipedia.org/wiki/Paging#Terminology">Swap</a>). Back in the old days where memory was really expensive, someone came up with the clever idea to page out currently unused memory pages to disk. If the program which allocated the memory page in the first place needed to access the page again, it was loaded from disk - and possibly other currently unused pages were swapped out in order to make some space in memory. Obviously this process is slow. Modern DRAM has access times of less than 100 Nanoseconds, where a typical spinning disk drive needs a few Milliseconds to access a block on disk. Writing out a page is even slower, and if the system is already under heavy load, other processes might require access to the precious I/O channels as well.</p>

<p>Nevertheless using Swap gives the advantage of having more memory available, just in case an application needs momentarily more RAM than physically available. However for a database system it severely degrades the performance, if the data needs to be read from disk, then swapped out, and swapped in again. It is best practice not to use swap at all on a dedicated database server.</p>

<h3 id="overcommiting-memory">Overcommiting Memory</h3>

<p>In an ideal world, every application would only request as much memory as it currently needs. And frees memory instantly when it is no longer used.</p>

<p>Unfortunately that is not the case in the real world. Many applications request more memory - just in case. Or preallocate memory which in the end is never used. Ever heard of <em>Java Heap Space</em>? Or a webbrowser eating Gigabytes of memory, even though most websites are already closed? Here you go …</p>

<p>Fortunately the Operating System knows about the bad habits of applications, and overcommits memory. That is, it provisions more memory (both RAM and Swap) than it has available, betting on applications to reserve more memory pages than they actually need. Obviously this will end in an Out-of-Memory disaster if the application really needs the memory, and the kernel will go around and kill applications. The error message you see is similar to “Out of memory: Kill process …“, or in short: <a href="https://www.hawatel.com/blog/kernel-out-of-memory-kill-process">“OOM-Killer”</a>. To avoid such situations, the overcommit behavior is configurable.</p>

<p>The <a href="https://www.kernel.org/doc/Documentation/sysctl/vm.txt">two parameters</a> to modify the overcommitt settings are <em>/proc/sys/vm/overcommit_memory</em> and <em>/proc/sys/vm/overcommit_ratio</em>.</p>

<h3 id="proc-sys-vm-overcommit-memory">/proc/sys/vm/overcommit_memory</h3>

<p>This switch knows <a href="https://www.kernel.org/doc/Documentation/vm/overcommit-accounting">3 different settings</a>:</p>

<ul>
<li><strong>0</strong>: The Linux kernel is free to overcommit memory (this is the default), a heuristic algorithm is applied to figure out if enough memory is available.</li>
<li><strong>1</strong>: The Linux kernel will always overcommit memory, and never check if enough memory is available. This increases the risk of out-of-memory situations, but also improves memory-intensive workloads.</li>
<li><strong>2</strong>: The Linux kernel will not overcommit memory, and only allocate as much memory as defined in <em>overcommit_ratio</em>.</li>
</ul>

<p>The setting can be changed by a superuser:</p>

<pre><code class="hljs elixir">echo <span class="hljs-number">2</span> &gt; <span class="hljs-regexp">/proc/sys</span><span class="hljs-regexp">/vm/overcommit</span>_memory
</code></pre>

<h3 id="proc-sys-vm-overcommit-ratio">/proc/sys/vm/overcommit_ratio</h3>

<p>This setting is only used when <em>overcommit_memory = 2</em>, and defines how many percent of the physical RAM are used. Swap space goes on top of that. The default is “50”, or 50%.</p>

<p>The setting can be changed by a superuser:</p>

<pre><code class="hljs elixir">echo <span class="hljs-number">75</span> &gt; <span class="hljs-regexp">/proc/sys</span><span class="hljs-regexp">/vm/overcommit</span>_ratio
</code></pre>

<h2 id="linux-memory-allocation">Linux Memory Allocation</h2>

<p>For the purpose of this post we mainly look into <em>overcommit_memory = 2</em>, we don’t want to end up in situations where a process is killed by the OOM-killer.</p>

<p>Linux uses a simple formula to calculate how much memory can be allocated:</p>

<pre><code class="hljs nginx"><span class="hljs-attribute">Memory</span> Allocation Limit = Swap Space + RAM * (Overcommit Ratio / <span class="hljs-number">100</span>)
</code></pre>

<p>Let’s look at some some numbers:</p>

<h3 id="scenario-1">Scenario 1:</h3>

<p>4 GB RAM, 4 GB Swap, overcommit_memory = 2, overcommit_ratio = 50</p>

<ul>
<li>Memory Allocation Limit = 4 GB Swap Space + 4 GB RAM * (50% Overcommit Ratio / 100)</li>
<li>Memory Allocation Limit = 6 GB</li>
</ul>

<h3 id="scenario-2">Scenario 2:</h3>

<p>4 GB RAM, 8 GB Swap, overcommit_memory = 2, overcommit_ratio = 50</p>

<ul>
<li>Memory Allocation Limit = 8 GB Swap Space + 4 GB RAM * (50% Overcommit Ratio / 100)</li>
<li>Memory Allocation Limit = 10 GB</li>
</ul>

<h3 id="scenario-3">Scenario 3:</h3>

<p>4 GB RAM, 2 GB Swap, overcommit_memory = 2, overcommit_ratio = 50</p>

<ul>
<li>Memory Allocation Limit = 2 GB Swap Space + 4 GB RAM * (50% Overcommit Ratio / 100)</li>
<li>Memory Allocation Limit = 4 GB</li>
</ul>

<p>Note that this is the total amount of memory which Linux will allocate. This includes all running daemons and other applications. Don’t assume that your application will be able to allocate the total limit. Linux will also provide the memory allocation limit in the field <em>CommitLimit</em> in <em>/proc/meminfo</em>.</p>

<p>In order to verify these settings and findings, I created a virtual machine and ran tests with different <em>overcommit_memory</em> and <em>overcommit_ratio</em> settings. A <a href="http://stackoverflow.com/questions/911860/does-malloc-lazily-create-the-backing-pages-for-an-allocation-on-linux-and-othe">small program</a> is used to allocate memory in 1 MB steps, until it fails because no more memory is available.</p>

<h3 id="4-gb-ram-4-gb-swap-overcommit-memory-2">4 GB RAM, 4 GB Swap, overcommit_memory = 2:</h3>

<table>
<thead>
<tr>
<th align="center">overcommit_ratio</th>
<th align="center">MemFree (kB)</th>
<th align="center">CommitLimit (kB)</th>
<th align="center">Breaks at (MB)</th>
<th align="center">Expected break (MB)</th>
<th align="center">Diff expected and actual break (MB)</th>
</tr>
</thead>

<tbody>
<tr>
<td align="center">10</td>
<td align="center">3803668</td>
<td align="center">4595144</td>
<td align="center">4200</td>
<td align="center">4488</td>
<td align="center">68</td>
</tr>

<tr>
<td align="center">25</td>
<td align="center">3802056</td>
<td align="center">5199488</td>
<td align="center">4793</td>
<td align="center">5078</td>
<td align="center">63</td>
</tr>

<tr>
<td align="center">50</td>
<td align="center">3801852</td>
<td align="center">6206724</td>
<td align="center">5771</td>
<td align="center">6062</td>
<td align="center">69</td>
</tr>

<tr>
<td align="center">75</td>
<td align="center">3802732</td>
<td align="center">7213960</td>
<td align="center">6748</td>
<td align="center">7045</td>
<td align="center">76</td>
</tr>

<tr>
<td align="center">90</td>
<td align="center">3802620</td>
<td align="center">7818300</td>
<td align="center">7340</td>
<td align="center">7636</td>
<td align="center">75</td>
</tr>

<tr>
<td align="center">100</td>
<td align="center">3802888</td>
<td align="center">8221196</td>
<td align="center">7729</td>
<td align="center">8029</td>
<td align="center">79</td>
</tr>
</tbody>
</table>

<ul>
<li><em>overcommit_ratio</em> shows the setting in /proc/sys/vm/overcommit_ratio (the VM was rebooted after every test)</li>
<li><em>MemFree</em> shows the free memory right before the test was started</li>
<li><em>CommitLimit</em> shows the entry in <em>/proc/meminfo</em></li>
<li><em>Breaks at</em> shows how much memory the program was able to allocate</li>
<li><em>Expected break</em> is the above calculation again, and should show the same number as <em>CommitLimit</em>, just in MB</li>
<li><em>Difference expected and actual break</em> shows the difference between the expected break and the actual break, but takes <em>MemFree</em> into account - that is, it calculates how much memory the test application could possibly allocate based on the free memory before running the test</li>
</ul>

<p>As you can see, the difference between what the program is expected to allocate based on the free memory, and the actual number is small. Take into account that the program itself also needs some memory for the code, loaded libraries, stack and such.</p>

<p>Also important: <strong>Using the default settings the system will use around 6 GB of combined RAM and Swap.</strong></p>

<p>Let’s look at two more examples.</p>

<h3 id="4-gb-ram-no-swap-overcommit-memory-2">4 GB RAM, no Swap, overcommit_memory = 2:</h3>

<table>
<thead>
<tr>
<th align="center">overcommit_ratio</th>
<th align="center">MemFree (kB)</th>
<th align="center">CommitLimit (kB)</th>
<th align="center">Breaks at (MB)</th>
<th align="center">Expected break (MB)</th>
<th align="center">Diff expected and actual break (MB)</th>
</tr>
</thead>

<tbody>
<tr>
<td align="center">10</td>
<td align="center">3803964</td>
<td align="center">402892</td>
<td align="center">243</td>
<td align="center">394</td>
<td align="center">69</td>
</tr>

<tr>
<td align="center">25</td>
<td align="center">3802532</td>
<td align="center">1007236</td>
<td align="center">803</td>
<td align="center">984</td>
<td align="center">40</td>
</tr>

<tr>
<td align="center">50</td>
<td align="center">3799844</td>
<td align="center">2014472</td>
<td align="center">1756</td>
<td align="center">1968</td>
<td align="center">12</td>
</tr>

<tr>
<td align="center">75</td>
<td align="center">3803580</td>
<td align="center">3021708</td>
<td align="center">2708</td>
<td align="center">2951</td>
<td align="center">23</td>
</tr>

<tr>
<td align="center">90</td>
<td align="center">3805424</td>
<td align="center">3626048</td>
<td align="center">3276</td>
<td align="center">3542</td>
<td align="center">48</td>
</tr>

<tr>
<td align="center">100</td>
<td align="center">3804236</td>
<td align="center">4028944</td>
<td align="center">3653</td>
<td align="center">3935</td>
<td align="center">63</td>
</tr>
</tbody>
</table>

<p>If you leave the <em>overcommit_ratio = 50</em>, the test application can only allocate half the memory (50%). Technically, <strong>all applications and daemons on the system can only use 2 GB RAM altogether</strong>. Only if <em>overcommit_ratio</em> is changed to <em>100</em> in this scenario, the entire memory is used.</p>

<h3 id="8-gb-ram-4-gb-swap-overcommit-memory-2">8 GB RAM, 4 GB Swap, overcommit_memory = 2:</h3>

<table>
<thead>
<tr>
<th align="center">overcommit_ratio</th>
<th align="center">MemFree (kB)</th>
<th align="center">CommitLimit (kB)</th>
<th align="center">Breaks at (MB)</th>
<th align="center">Expected break (MB)</th>
<th align="center">Diff expected and actual break (MB)</th>
</tr>
</thead>

<tbody>
<tr>
<td align="center">10</td>
<td align="center">7921080</td>
<td align="center">5008020</td>
<td align="center">4607</td>
<td align="center">4891</td>
<td align="center">53</td>
</tr>

<tr>
<td align="center">25</td>
<td align="center">7922144</td>
<td align="center">6231680</td>
<td align="center">5797</td>
<td align="center">6086</td>
<td align="center">59</td>
</tr>

<tr>
<td align="center">50</td>
<td align="center">7920776</td>
<td align="center">8271108</td>
<td align="center">7784</td>
<td align="center">8078</td>
<td align="center">63</td>
</tr>

<tr>
<td align="center">75</td>
<td align="center">7922520</td>
<td align="center">10310536</td>
<td align="center">9758</td>
<td align="center">10069</td>
<td align="center">81</td>
</tr>

<tr>
<td align="center">90</td>
<td align="center">7922820</td>
<td align="center">11534192</td>
<td align="center">10946</td>
<td align="center">11264</td>
<td align="center">89</td>
</tr>

<tr>
<td align="center">100</td>
<td align="center">7921180</td>
<td align="center">12349964</td>
<td align="center">11747</td>
<td align="center">12061</td>
<td align="center">83</td>
</tr>
</tbody>
</table>

<p>The memory is doubled, and now the application will use around 8 GB RAM if <em>overcommit_ratio</em> is set to 50%.</p>

<p>Two more examples, this time with different overcommit_memory settings:</p>

<h3 id="4-gb-ram-4-gb-swap-overcommit-memory-0">4 GB RAM, 4 GB Swap, overcommit_memory = 0:</h3>

<table>
<thead>
<tr>
<th align="center">overcommit_ratio</th>
<th align="center">MemFree (kB)</th>
<th align="center">CommitLimit (kB)</th>
<th align="center">Breaks at (MB)</th>
</tr>
</thead>

<tbody>
<tr>
<td align="center">10</td>
<td align="center">3802128</td>
<td align="center">4595144</td>
<td align="center">7802</td>
</tr>

<tr>
<td align="center">25</td>
<td align="center">3800896</td>
<td align="center">5199488</td>
<td align="center">7806</td>
</tr>

<tr>
<td align="center">50</td>
<td align="center">3802776</td>
<td align="center">6206724</td>
<td align="center">7794</td>
</tr>

<tr>
<td align="center">75</td>
<td align="center">3798900</td>
<td align="center">7213960</td>
<td align="center">7805</td>
</tr>

<tr>
<td align="center">90</td>
<td align="center">3804468</td>
<td align="center">7818300</td>
<td align="center">7808</td>
</tr>

<tr>
<td align="center">100</td>
<td align="center">3803396</td>
<td align="center">8221196</td>
<td align="center">7798</td>
</tr>
</tbody>
</table>

<h3 id="4-gb-ram-4-gb-swap-overcommit-memory-1">4 GB RAM, 4 GB Swap, overcommit_memory = 1:</h3>

<table>
<thead>
<tr>
<th align="center">overcommit_ratio</th>
<th align="center">MemFree (kB)</th>
<th align="center">CommitLimit (kB)</th>
<th align="center">Breaks at (MB)</th>
</tr>
</thead>

<tbody>
<tr>
<td align="center">10</td>
<td align="center">3803952</td>
<td align="center">4595144</td>
<td align="center">7794</td>
</tr>

<tr>
<td align="center">25</td>
<td align="center">3804852</td>
<td align="center">5199488</td>
<td align="center">7804</td>
</tr>

<tr>
<td align="center">50</td>
<td align="center">3804564</td>
<td align="center">6206724</td>
<td align="center">7800</td>
</tr>

<tr>
<td align="center">75</td>
<td align="center">3805032</td>
<td align="center">7213960</td>
<td align="center">7800</td>
</tr>

<tr>
<td align="center">90</td>
<td align="center">3802900</td>
<td align="center">7818300</td>
<td align="center">7803</td>
</tr>

<tr>
<td align="center">100</td>
<td align="center">3801392</td>
<td align="center">8221196</td>
<td align="center">7748</td>
</tr>
</tbody>
</table>

<p>In both scenarios the OS will allocate as much memory as possible, even using swap - which is a no-go for a database server. That is the reason why <em>overcommit_memory</em> should always be set to <em>2</em> if you run a dedicated server.</p>

<h3 id="running-several-applications-in-parallel">Running several applications in parallel</h3>

<p>One last example, showing that the OS is taking all applications into account when calculating the memory for our test application:</p>

<p>Host memory is 8 GB, overcommit_ratio = 100, overcommit_memory = 2. Same scenario as the third table, last line.
This time, two applications are running: one is allocating 4 GB of memory, then the other application is started.</p>

<p>The test application breaks at 7641 MB. This number plus the 4096 MB from the other application together is 11737 MB - just 10 MB difference from the 11747 MB in the last test in the table.</p>

<h2 id="conclusions">Conclusions</h2>

<p>If you expand the memory in a system, always check the <em>overcommit_ratio</em> setting. Else you will end up not using all the memory you just spent money on, or even worse your server will swap memory pages to disk.</p>

<p>Here is the formula for not using Swap, but using all RAM:</p>

<pre><code class="hljs nginx"><span class="hljs-attribute">Overcommit</span> Ratio = <span class="hljs-number">100</span> * ((RAM - Swap Space) / RAM)
</code></pre>

<h2 id="emc-dca">EMC DCA</h2>

<p><a href="https://www.emc.com/">EMC</a> offers a <a href="https://pivotal.io/big-data/emc-dca">Data Computing Appliance</a> (DCA), which is a preconfigured <a href="http://greenplum.org/">Greenplum Database</a> system running on commodity hardware. The v2 of the DCA, by default, has 64 GB RAM and 32 GB Swap on every server:</p>

<pre><code class="hljs dts">[gpadmin@seg1 ~]$ cat <span class="hljs-meta-keyword">/proc/</span>meminfo
<span class="hljs-symbol">MemTotal:</span>     <span class="hljs-number">65886676</span> kB (<span class="hljs-number">64342</span> MB)
<span class="hljs-symbol">SwapTotal:</span>    <span class="hljs-number">33553400</span> kB (<span class="hljs-number">32766</span> MB)
</code></pre>

<p>Let’s do the math:</p>

<pre><code class="hljs nginx"><span class="hljs-attribute">Overcommit</span> Ratio = <span class="hljs-number">100</span> * ((<span class="hljs-number">64</span> GB - <span class="hljs-number">32</span> GB) / <span class="hljs-number">64</span> GB)
Overcommit Ratio = <span class="hljs-number">50</span>
</code></pre>

<p>And sure enough:</p>

<pre><code class="hljs elixir">[gpadmin<span class="hljs-variable">@seg1</span> ~]<span class="hljs-variable">$ </span>cat /proc/sys/vm/overcommit_ratio
<span class="hljs-number">50</span>
</code></pre>

<p>That’s fine. Now let’s assume the system is expanded to 256 GB RAM, without reimaging the server - that is, still 32 GB Swap Space.</p>

<p>Based on the <em>Allocation Limit Formula</em>, Linux will not allocate all the memory:</p>

<pre><code class="hljs matlab">Memory Allocation Limit = Swap Space + RAM * (Overcommit Ratio / <span class="hljs-number">100</span>)
Memory Allocation Limit = <span class="hljs-number">32</span> GB + <span class="hljs-number">256</span> GB * (<span class="hljs-number">50</span><span class="hljs-comment">% / 100)</span>
Memory Allocation Limit = <span class="hljs-number">160</span> GB
</code></pre>

<p>Ooops! The system was expanded with 256 GB RAM, but is only using nearly as much as half of it. That is clearly not good.</p>

<pre><code class="hljs nginx"><span class="hljs-attribute">Overcommit</span> Ratio = <span class="hljs-number">100</span> * ((RAM - Swap Space) / RAM)
Overcommit Ratio = <span class="hljs-number">100</span> * ((<span class="hljs-number">256</span> GB - <span class="hljs-number">32</span> GB) / <span class="hljs-number">256</span> GB)
Overcommit Ratio = <span class="hljs-number">87</span>.<span class="hljs-number">5</span>
</code></pre>

<p>The new setting for <em>/proc/sys/vm/overcommit_ratio</em> must be <em>87.5</em> in order to utilize all available RAM, but not use the Swap.</p>

      </div>
      <figure class="center fig-responsive">
        <iframe id="twitter-widget-1" scrolling="no" allowtransparency="true" class="twitter-share-button twitter-share-button-rendered twitter-tweet-button" style="position: static; visibility: visible; width: 75px; height: 28px;" title="Twitter Tweet Button" src="https://platform.twitter.com/widgets/tweet_button.2349b7ea03933b93cf1e9e9f69dac37a.en.html#dnt=false&amp;id=twitter-widget-1&amp;lang=en&amp;original_referer=http%3A%2F%2Fengineering.pivotal.io%2Fpost%2Fvirtual_memory_settings_in_linux_-_the_problem_with_overcommit%2F&amp;size=l&amp;text=%C2%B7%20Pivotal%20Engineering%20Journal&amp;time=1567642030433&amp;type=share&amp;url=http%3A%2F%2Fengineering.pivotal.io%2Fpost%2Fvirtual_memory_settings_in_linux_-_the_problem_with_overcommit%2F&amp;via=ascherbaum%20" frameborder="0"></iframe>
        <script>!function(d,s,id){var js,fjs=d.getElementsByTagName(s)[0],p=/^http:/.test(d.location)?'http':'https';if(!d.getElementById(id)){js=d.createElement(s);js.id=id;js.src=p+'://platform.twitter.com/widgets.js';fjs.parentNode.insertBefore(js,fjs);}}(document, 'script', 'twitter-wjs');</script>
      </figure>
    </div>
