---
title: Recommended Production Settings
summary: Recommended settings for production deployments.
toc: false
toc_not_nested: true
---

This page provides recommended settings for production deployments. 

<div id="toc"></div>

## Cluster Topology

- Run one node per machine. Since CockroachDB [replicates](configure-replication-zones.html) across nodes, running more than one node per machine increases the risk of data unavailability if a machine fails.

- If a machine has multiple disks or SSDs, it's better to run one node with multiple `--store` flags instead of one node per disk, because this informs CockroachDB about the relationship between the stores and ensures that data will be replicated across different machines instead of being assigned to different disks of the same machine. For more details about stores, see [Start a Node](start-a-node.html).

## Clock Synchronization

Run [NTP](http://www.ntp.org/) or other clock synchronization software on each machine. CockroachDB needs moderately accurate time; if the machines’ clocks drift too far apart, [transactions](transactions.html) will never succeed and the cluster will crash.

## File Descriptors Limit

CockroachDB can use a large number of open file descriptors, often more than is available by default. Therefore, please note the following recommendations.

For each CockroachDB node:

- At a **minimum**, the file descriptors limit must be 512 (256 per store plus 256 for networking). If the limit is below this threshold, the node will not start. 
- The **recommended** file descriptors limit is at least 10000 (5000 per store plus 5000 for networking). This higher limit ensures performance and accommodates cluster growth. 
- When the file descriptors limit is between these minimum and recommended amounts, CockroachDB will allocate 256 to networking and evenly split the rest across stores.

### Increase the File Descriptors Limit

<script>
$(document).ready(function(){

    //detect os and display corresponding tab by default
    if (navigator.appVersion.indexOf("Mac")!=-1) {
        $('#os-tabs').find('button').removeClass('current');
        $('#mac').addClass('current');
        toggleMac();
    }
    if (navigator.appVersion.indexOf("Linux")!=-1) {
        $('#os-tabs').find('button').removeClass('current');
        $('#linux').addClass('current');
        toggleLinux();
    }
    if (navigator.appVersion.indexOf("Win")!=-1) {
        $('#os-tabs').find('button').removeClass('current');
        $('#windows').addClass('current');
        toggleWindows();
    }

    var install_option = $('.install-option'),
        install_button = $('.install-button');

    install_button.on('click', function(e){
      e.preventDefault();
      var hash = $(this).prop("hash");

      install_button.removeClass('current');
      $(this).addClass('current');
      install_option.hide();
      $(hash).show();

    });

    //handle click event for os-tab buttons
    $('#os-tabs').on('click', 'button', function(){
        $('#os-tabs').find('button').removeClass('current');
        $(this).addClass('current');

        if($(this).is('#mac')){ toggleMac(); }
        if($(this).is('#linux')){ toggleLinux(); }
        if($(this).is('#windows')){ toggleWindows(); }
    });

    function toggleMac(){
        $(".mac-button:first").trigger('click');
        $("#macinstall").show();
        $("#linuxinstall").hide();
        $("#windowsinstall").hide();
    }

    function toggleLinux(){
        $(".linux-button:first").trigger('click');
        $("#linuxinstall").show();
        $("#macinstall").hide();
        $("#windowsinstall").hide();
    }

    function toggleWindows(){
        $("#windowsinstall").show();
        $("#macinstall").hide();
        $("#linuxinstall").hide();
    }
});
</script>

<div id="os-tabs" class="clearfix">
    <button id="mac" class="current" data-eventcategory="buttonClick-doc-os" data-eventaction="mac">Mac</button>
    <button id="linux" data-eventcategory="buttonClick-doc-os" data-eventaction="linux">Linux</button>
    <button id="windows" data-eventcategory="buttonClick-doc-os" data-eventaction="windows">Windows</button>
</div>

<div id="macinstall" markdown="1">

- [Yosemite and later](#yosemite-and-later)
- [Older versions](#older-versions)

#### Yosemite and later

To adjust the file descriptors limit for a single process in Mac OS X Yosemite and later, you must create a property list configuration file with the hard limit set to the recommendation mentioned [above](#file-descriptors-limit). Note that CockroachDB always uses the hard limit, so it's not technically necessary to adjust the soft limit, although we do so in the steps below.

For example, for a node with 3 stores, we would set the hard limit to at least 20000 (5000 per store and 5000 for networking) as follows: 

1.  Check the current limits:

    ~~~ shell
    $ launchctl limit maxfiles
    maxfiles    10240          10240      
    ~~~

    The last two columns are the soft and hard limits, respectively. If `unlimited` is listed as the hard limit, note that the hidden default limit for a single process is actually 10240.

2.  Create `/Library/LaunchDaemons/limit.maxfiles.plist` and add the following contents, with the final strings in the `ProgramArguments` array set to 20000:

    ~~~ xml
    <?xml version="1.0" encoding="UTF-8"?>
    <!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
      <plist version="1.0">
        <dict>
          <key>Label</key>
            <string>limit.maxfiles</string>
          <key>ProgramArguments</key>
            <array>
              <string>launchctl</string>
              <string>limit</string>
              <string>maxfiles</string>
              <string>20000</string>
              <string>20000</string>
            </array>
          <key>RunAtLoad</key>
            <true/>
          <key>ServiceIPC</key>
            <false/>
        </dict>
      </plist>
    ~~~

    Make sure the plist file is owned by `root:wheel` and has permissions `-rw-r--r--`. These permissions should be in place by default.

3.  Restart the system for the new limits to take effect.

4.  Check the current limits:

    ~~~ shell
    $ launchctl limit maxfiles
    maxfiles    20000          20000      
    ~~~

#### Older versions

To adjust the file descriptors limit for a single process in OS X versions earlier than Yosemite, edit `/etc/launchd.conf` and increase the hard limit to the recommendation mentioned [above](#file-descriptors-limit). Note that CockroachDB always uses the hard limit, so it's not technically necessary to adjust the soft limit, although we do so in the steps below.

For example, for a node with 3 stores, we would set the hard limit to at least 20000 (5000 per store and 5000 for networking) as follows:

1.  Check the current limits:

    ~~~ shell
    $ launchctl limit maxfiles
    maxfiles    10240          10240      
    ~~~

    The last two columns are the soft and hard limits, respectively. If `unlimited` is listed as the hard limit, note that the hidden default limit for a single process is actually 10240.

2.  Edit (or create) `/etc/launchd.conf` and add a line that looks like the following, with the last value set to the new hard limit:

    ~~~ shell
    limit maxfiles 20000 20000
    ~~~

3.  Save the file, and restart the system for the new limits to take effect. 

4.  Verify the new limits:

    ~~~ shell
    $ launchctl limit maxfiles
    maxfiles    20000          20000      
    ~~~

</div>

<div id="linuxinstall" markdown="1">

To adjust the file descriptors limit for a single process on Linux, enable PAM user limits and set the hard limit to the recommendation mentioned [above](#file-descriptors-limit). Note that CockroachDB always uses the hard limit, so it's not technically necessary to adjust the soft limit, although we do so in the steps below.

For example, for a node with 3 stores, we would set the hard limit to at least 20000 (5000 per store and 5000 for networking) as follows:

1.  Make sure the following line is present in both `/etc/pam.d/common-session` and `/etc/pam.d/common-session-noninteractive`:

    ~~~ shell
    session    required   pam_limits.so
    ~~~

2.  Edit `/etc/security/limits.conf` and append the following lines to the file:

    ~~~ shell
    *              soft     nofile          20000
    *              hard     nofile          20000
    ~~~

    Note that `*` can be replaced with the username that will be running the CockroachDB server.

4.  Save and close the file.

5.  Restart the system for the new limits to take effect.

6.  Verify the new limits:

    ~~~ shell
    ulimit -a
    ~~~

</div>

<div id="windowsinstall" markdown="1">

CockroachDB does not yet provide a native Windows binary. Once that's available, we will also provide documentation on adjusting the file descriptors limit on Windows.

</div>

#### Attributions

This section, "File Descriptors Limit", is a derivative of [Open File Limits](http://docs.basho.com/riak/kv/2.1.4/using/performance/open-files-limit/) by Riak, used under Creative Commons Attribution 3.0 Unported License.