### Cisco Security Suite Source Configuration ###


#### Firewalls ####
Firewalls will generate *a lot* of data.  So we will tune things a bit to try and minimise this.  However, if you need to get both *permitted _AND_ denied* connections, then you'll have to deal with the larger volumes regardless.

##### ASA #####

Replace the variables inside the LTGT symbols `<...> ` with your Splunk indexer or forwarder address.

    logging enable
    logging standby
    logging trap warnings
    logging host <logging-interface> <syslog-server-address>
    logging class auth trap errors 
    no logging message 305012
    no logging message 302010
    no logging message 302014
    no logging message 609002
    no logging message 609001
    no logging message 302016
    logging message 106015 level warnings
    logging message 313001 level warnings
    logging message 710005 level warnings
    logging message 710003 level warnings
    logging message 710002 level warnings
    logging message 302013 level warnings

Use the command `show run logging` to verify your config.  Alternatively, use `show run all logging` to see some of the "hidden" defaults.

    my-first-ASA# show run logging    

### Intrusion Detection ###

#### Sourcefire Defence Center (eStreamer) ####

##### From Defense Center #####

1.  Setup the eStreamer service in Defense Center (requires an Admin level account)

2.  Go to: `System > Local > Registration`, then clicking the `Create Client button`.

3.  Add in your Splunk server's IP address (*not hostname*), along with a password (the password is used to secure the client certificate)
_NOTE: Sourcefire recommends that you use the IP address of the Splunk server rather than its hostname_ 

4.  Once done, you should see a "success" window.  Clikc the little "download" icon to the right of the new host entry to download the client certificate (will end in .pkcs)

5.  Ensure the desired event types are selected under _eStreamer Event Configuration_ on the left-hand side (and finally, click _Save_ too)

##### From Splunk ######

1.  Download and install the [Cisco eStreamer App](https://apps.splunk.com/app/1629) if you haven't already

2.  Ensure you have [all the pre-requisite Perl modules](https://apps.splunk.com/app/1629/documentation) on your system.  You can use `perldoc -l <pkg name>` or `perl -e "use <pkg name>"`or just running the `estreamer_client.pl` command to see if any errors popup.
Using Ubuntu for example you can check via Perl directly:
 ```
 perl -e "use IO::Socket::SSL"
 ```
 And if any errors pop up, just use CPAN or your favorite package manager to install those modules.  Using Ubuntu for example you can install either via CPAN:
 ```
 cpan IO::Socket::SSL
 ```
 Or if using your system's package manager, you can generally achieve a more graceful installation.  Using Ubuntu as an example with apt-get, we can knock them all out at once with any quirky OS-specific dependencies resolved along the way.
 ```
 apt-get install libio-socket-inet6-perl libio-socket-ssl-perl libnetaddr-ip-perl libstorable-perl libsocket6-perl libsocket-linux-perl
 ```
If you see the usage statement, then you are golden:
```
# ./estreamer_client.pl 
Usage:  estreamer_client.pl [options]
...
```
 
3.  Upload the client certificate file to your Splunk server (remember, likely a Forwarder for distributed deployments).  I use SCP like follows:
`scp Downloads/10.67.41.37.pkcs12 awurster@10.67.41.37:~/downloads/` 

4.  Place your client cert into the eStreamer Splunk App directory (_$SPLUNK_HOME/etc/apps/eStreamer/bin/_). I choose to place it in the local directory, although it's configurable anywhere. 
`mv ~/downloads/10.67.41.37.pkcs12 /opt/splunk/etc/apps/eStreamer/bin/`
_Depending on how you're running Splunk or copying the cert file, you may need to *chown* or *chmod* it to make it readable by the script._

5.  Navigate to the eStreamer app directory (_$SPLUNK_HOME/etc/apps/eStreamer/bin/_), and run the eStreamer client again to validate that it all works.
```
$ cd /opt/splunk/etc/apps/eStreamer/bin/
$ ./estreamer_client.pl 
Usage:  estreamer_client.pl [options]
...
```

6.  Restart Splunk at this point, and then access the GUI to begin setting up the eStreamer inputs.
`/opt/splunk/bin/splunk restart`

7.  If restarting right after the App Installation process, you should be prompted to configure the app further, otherwise navigate to the "Settings" page from within the eStreamer App menu.

8.  Now, configure the *"Defense Center Connection Details"* section.  Be sure to specify a full path to your certificate file as well.  And if you enabled any of the advanced options in Defense Center, then you'll want to do the same on the eStreamer App config page too.

9. Finally, uncheck the *"Disable eStreamer"* tick box at the top of the page once you're ready, and *Save* the changes.

10.  Navigate back to the eStreamer App Overview page to ensure your events have begin rolling in.


Follow the [Cisco eStreamer App Configuration Docs](https://apps.splunk.com/app/1629/documentation) for more detail and tips.


#### Content Security ####
ESA and WSA logs are quite different, although they both run AsyncOS, hence the same section.  

Within the Cisco AS Cyber Range, we log using Syslog over TCP, but many folks will recommend you use SCP for various reasons such as robustness, etc.

If you are using SCP to upload the files, I'd recommend using the following rollover options:
* Maximum File Size: __10M__
* Rollover by Custom Time: __5m__

Log subscription details below will be the same regardless of your transfer type.  We advise to *duplicate* the log subscriptions, rather than *changing the defaults*.

##### Web Security (WSA) #####
For WSA's, you'll need to send out the __accesslogs__ to Splunk.  These store everything about a given web transaction.  It can be anything from an HTML page, to an image, to an API call.  A simple transaction like below illustrates my user ID __awurster__ going to the website `www.cisco.com` from a test PC:

    1387350521.491 763 192.168.100.100 TCP_MISS/200 7140 GET http://www.cisco.com/ "CYBERCISCO\awurster@CyberRange" DIRECT/www.cisco.com text/html MONITOR_CUSTOMCAT_12-CyberRange_Custom_AW-CyberRange_Inside_ADAuth-NONE-NONE-NONE-DefaultGroup <C_Cisc,6.5,0,"-",0,0,0,1,"-",-,-,-,"-",1,-,"-","-",-,-,IW_comp,-,"Unknown","-","Unknown","Unknown","-","-",74.86,0,-,"Unknown","-"> -

This log message is based on the _Squid_ access log type.  You can tell where the transaction is from (__192.168.100.100__), where it's going (`http://www.cisco.com/`), how big it was (__7140 bytes__), whether it succeeded (__TCP_MISS/200__), and so on.

This particular message was generated by  AsyncOS 7.7. From time to time, new fields or variations may be added in future releases for extra functionality, etc.

And of course, nothing is easy in life, so we'll need to add some extra fields to the subscription to get all the nice correlation and added benefit within Splunk.  Add in the following "custom" fields:

    %u %<Referer: %k %XF %q

Once you add, all those in, you get something a little bit more verbose:

    1387350521.491 763 192.168.100.100 TCP_MISS/200 7140 GET http://www.cisco.com/ "CYBERCISCO\awurster@CyberRange" DIRECT/www.cisco.com text/html MONITOR_CUSTOMCAT_12-CyberRange_Custom_AW-CyberRange_Inside_ADAuth-NONE-NONE-NONE-DefaultGroup <C_Cisc,6.5,0,"-",0,0,0,1,"-",-,-,-,"-",1,-,"-","-",-,-,IW_comp,-,"Unknown","-","Unknown","Unknown","-","-",74.86,0,-,"Unknown","-"> - "Mozilla/5.0 (Windows NT 6.1; WOW64; Trident/7.0; rv:11.0) like Gecko" - 173.37.145.84 "Cisco Internal" 3379

_*Note that once it gets into Splunk, you'll see an extra syslog style header if using Syslog transport.*_

Use something intelligent like `accesslogs_splunk_tcp` or something when naming your subscription, to denote that it's A) not the stock subscription, and B) to tell a bit about where it's going and how it gets there.  As you can imagine, yes, that's what we call ours.

So in summary, you'll create the following subscription:

Log Type:           `Squid`

Custom Fields:      `%u %<Referer: %k %XF %q`

Rerieval Method:    `Syslog Push`


##### Email Security (ESA) #####
For ESA's you need to send your `mail_logs` subscription to Splunk.  Similar to the WSA above, we should call it something like `mail_logs_splunk_tcp`.

Not as many options as in the WSA, so just leave all else defaults besides the transport.

mail_logs within the ESA describe the mail transactions performed by the system, whether it's mail being sent, received, dropped, etc.  So once properly configured, you'll see a stream similar to:

    Thu Oct 24 19:15:33 2013 Info: Start MID 120821 ICID 1227005
    Thu Oct 24 19:15:33 2013 Info: MID 120821 ICID 1227005 From: <hacker@cyber-range-cisco.com>
    Thu Oct 24 19:15:33 2013 Info: MID 120821 ICID 1227005 RID 0 To: <XojMyqmheth@cisco-training-sydney.com>
    Thu Oct 24 19:15:33 2013 Info: MID 120821 Message-ID '<KNMpYzPYdOUdZJzL@cyber-range-cisco.com>'
    Thu Oct 24 19:15:33 2013 Info: MID 120821 Subject 'Please check Attached Web Page'
    Thu Oct 24 19:15:33 2013 Info: MID 120821 ready 786 bytes from <hacker@cyber-range-cisco.com>
    Thu Oct 24 19:15:33 2013 Info: MID 120821 matched all recipients for per-recipient policy DEFAULT in the outbound table
    Thu Oct 24 19:15:33 2013 Info: MID 120821 using engine: CASE spam negative
    Thu Oct 24 19:15:33 2013 Info: MID 120821 interim AV verdict using Sophos CLEAN
    Thu Oct 24 19:15:33 2013 Info: MID 120821 antivirus negative 
    Thu Oct 24 19:15:33 2013 Info: MID 120821 Outbreak Filters: verdict positive

###### Logging Embedded URLs ######

Another interesting item to log (which is currently not supported by ESAs) are the raw URLs from within a Message Body.  This is potentially effective against spearphishing or other stealthy campaigns such as "snowshoe" techniques which become harder to detect.  A script or addon within Splunk could then easily analyse the URL pieces or contents themselves.  You might check for URL length, number of punctionations, whether the domain / host is blacklisted, etc.

In the example below, since we use `k=v` format, Splunk automatically extracts the URL into a field `message_url` (yay! no tuning required on your search head!).  *NOTE* that this filter will potentially generate a significant amount of logging noise and/or false positives; best to test these on a limited-scope first (i.e. also match against your own personal recipient address to start).

Enter the filter (CLI only operation) as such:

    ESAv-01.cybercisco.com> filters
        ...
    Choose the operation you want to perform:
    - NEW - Create a new filter.
        ...
    []> new
        ...
    Enter filter script.  Enter '.' on its own line to end.
    
    LogBodyURLs: if only-body-contains("http(s)?:\\/\\/[a-zA-Z0-9_\\-\\.]+(:\\d+)?([^\\s\\<\\\"]+)?", 1) {
                     log-entry("Filter LogBodyURLs fired: message_url=$MatchedContent");
                     archive ("LogBodyURLArchive");
                 }
    .
    
    1 filters added.
        ...
    
    ESAv-01.cybercisco.com> commit
        ...

A sample raw log might then look like:

    Tue Aug 19 18:52:04 2014 Info: MID 248517 Custom Log Entry: Filter LogBodyURLs fired: message_url=http://ihaveabadreputation.com/phishme.do

##### Cyber Threat Defense (Lancope) #####
CTD / Lancope config is *very* difficult to locate and somewhat counterintuitive, so hang on tight.

Within the StealthWatch Mngement Console - choose the `Configuration` menu, select `Response Management...` and you should be shown the `Syslog Formats` screen (`Configuration > Response Management... > Syslog Formats`).

Select `Add` to create a new format scheme, which Lancope will use to send events into Splunk via syslog.

Give it a nice name and description - we use `Splunk Syslog`.  Leave `Facility` and `Severity` in place as the defaults - unless you know what you're doing and need to tweak them for some reason.

Now, you need to construct the format from scratch.  It's really cool that you can completely define the whole schema, but a bit tedious and of course daunting to figure out all the fields.  We've done that for you though already!

Enter in the following text verbatim to the `MSG Part` window:

    alarm_category_name="{alarm_category_name}", alarm_severity_name="{alarm_severity_name}", alarm_status="{alarm_status}", alarm_type_name="{alarm_type_name}", details="{details}", source_ip="{source_ip}", source_host_group_names="{source_host_group_names}", source_device_type "{source_device_type}", source_mac_address="{source_mac_address}", source_hostname="{source_hostname}", target_ip="{target_ip}", target_host_group_names="{target_host_group_names}", target_device_type="{target_device_type}", target_mac_address="{target_mac_address}", target_hostname="{target_hostname}", target_url="{target_url}"

_Of course - if you'd like to add more fields - feel free to do so at this point (removing is not recommended, as this could break some of the Splunk dashboards).  Use the `Help` button to access the online SMC docs and reference the formats, and the `Test` button to check what your format looks like._

Next, select `Actions` and configure Splunk as the Syslog server to receive this data.

Choose `Add` to enter in your Splunk Forwarder or Indexer address, and choose the formatter your just created using the `Syslog Message` dropdown for `Format`.  At this point, you can trigger actual data to be sent to Splunk using the `Test` message on the screen.

Close the screen to complete the CTD logging setup.

### Sizing and Baselining ###

*WIP* 
