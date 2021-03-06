# misp42splunk
A Splunk app to use MISP as a backend (lookup and store events)
If you have TheHive installed, you also may create alerts

# Credits
This app is largely inspired by https://github.com/xme/splunk/tree/master/getmispioc and the associated blog https://blog.rootshell.be/2017/10/31/splunk-custom-search-command-searching-misp-iocs/ for MISP interactions.

The alert_action for TheHive is inpired by [this Splunk app](https://splunkbase.splunk.com/app/3642/)

# Prerequisites
1. Install Python 3 on the Splunk Search Head.
2. Check that python3 is at /usr/bin/python3.

    + if not, you may create a symbolic link to the python3.x binary.
    + alternatively you may edit misp42splunk/bin/misp-get-ioc.py and adjust the path to your environment.

3. Install PyMISP (see https://github.com/CIRCL/PyMISP).
4. Check that your Splunk SH can connect to the MISP instance. 

# Installation
This App is designed to run on Splunk Search Head(s)
1. Download the ZIP file and install the app on your Splunk Search Head(s) (you may remove -master from file name)
    * Please note that this app come with a copy of Splunk SDK under misp42splunk/bin
2. A custom endpoint has been defined so you need to restart Splunk (for later updates, you may skip this step)
3. At next logon, you should be invited to configure the app (if not go to Manage Apps > TA-MISP 42 Splunk > Set up) 
    - For MISP
        - provide the url to your MISP instance;
        - provide the authkey;
        - check the certificate of the MISP server.
    - For TheHive
        - provide the url to the API of your instance;
        - provide the authkey.

# Use Cases

Here some activities you may carry out more easily with this app.
## Hunting in Splunk logs
Fresh IOC from MISP > saved searches in Splunk > on match create an alert on [TheHive](https://thehive-project.org/) or (later) any SIR platform of your choice.

## Creating events based on automated sandboxing
If you have output of analysis pushed to Splunk you may automate the creation of events
Log on sandboxing output > saved search to qualify, sanitize (dedup remove top Alexa, etc.)  and prepare the table > set a splunk alert to create event(s) in MISP

# Usage
## custom command mispgetioc
This custom command must be the first of a search (or a sub-search). The results are displayed in a table.
The command syntax is as follow:

    |mispgetioc ( [eventid=id] | [last=interval] )
                [onlyids=y|n]
                [category=string]
                [type=string]
                [server=https://host:port] 
                [authkey=misp-authorization-key]
                [sslcheck=y|n]                  
                

- You must set either eventid or last parameters
- last interval is a number followed by d(ays), h(ours) or m(inutes)
- you may filter the results using type and category parameters
- you may overwrite the misp server parameters for this search

## Alert sent to TheHive
When you create a Splunk alert, you may add an alert action to create alerts in TheHive
### collect results in Splunk
#### search with a column by artifact type
you may build a search returning some values for these fields

    autonomous-system
    domain
    file
    filename
    fqdn
    hash
    ip
    mail
    mail_subject
    other
    regexp
    registry
    uri_path
    url
    user-agent

and one field to group rows.
For example

    | eval id = md5(some common key in rows belonging to the same alert)
    | table id, autonomous-system, domain, file, filename, fqdn, hash, ip, mail, mail_subject, other, regexp, registry, uri_path, url, user-agent

Values may be empty for some fields; they will be dropped gracefully. You may add any other columns, they will be passed as elements but only fields above are imported as observables when you create/update a case.

#### search with 2 columns: type & value
You may also build a search with one artifact by row. You use field id to group several rows together    
For example: 

    | mispgetioc last=1d
    | eval id = md5(eventid)
    | table id, type, value


### create the alert action "Alert to create THEHIVE alert(s)"
Fill in fields. If value is not provided, default will be provided if needed.

* Alert overall description
    - Case Template: The case template to use for imported alerts.
    - Type: The alert type. Defaults to "alert".
    - Source: The alert source. Defaults to "splunk".
    - Unique ID: A field name that contains a unique identifier specific to the source event. You may use the field value to group artifacts under the same alert.
    - Title: The title to use for created alerts.
    - Description: The description to send with the alert.
    - Tags: Use single comma-separated string without quotes for multiple tags (ex. "badIP,spam").
    - Severity: Change the severity of the created alert.
    - TLP: Change the TLP of the created alert. Default is TLP:AMBER
* TheHive API parameters (optional if they have been defined in general setup)
    - URL: The URL to submit alerts to e.g. http://hive.example.com/api/alert.
    - API KEY: The API KEY for authentication

## Alert to create MISP event(s)    
When you create an alert, you may add an alert action to directly create events in MISP based on search results

### collect results in Splunk
At the moment you need to search and prepare the results as a table with the following command
    | table _time type value to_ids eventkey info category

* Mandatory fields:
    - type: the type of attribute. It must use MISP attribute names
    - value: the value of the attribute - you should check that the value complies with the type
 
* Optional fields:
    - _time: the timestamp will be converted to YYYY-MM-DD for event date. if not provided, set to localtime
    - to_ids: if not defined, set to False
    - category: if not defined, set to None and populated in relation with the type of attribute
    - eventkey: This string/id is used to group several rows of the results belonging to the same event (e.g. attributes of type email-src, email-subject). The actual value is not pushed to MISP. If not specified by row, this value might be overall defined for the alert - see below
    - info: This string will be set in the Info field of MISP event. This value might be overall defined for the alert - see below

### create the alert and add alert_action to create events
Save your search as alert. Select "Alert to create MISP event(s)" as action
Fill in the form to tune your alert to your needs.


Global event parameters
Info
$description$ The default Info field for the MISP events if not provided in results.
Tags
Use single comma-separated string without quotes for multiple tags (ex. "badIP,spam").


* Alert overall description: this section is for Splunk documentation
    - Title: The title of this alert.
    - Description: The description to send with the alert.
* Global event parameters: the parameters will apply for all events created by this alert unless overwritten (see above)
    - Unique ID: indicate the field containing the unique id to group several rows under a single event. If not defined an default eventkey will be generated and all results will be added to the same event.
    - Info: This string will be set in the Info field of MISP event. If not defined, the Info field will contain 'malspam'. By default, it takes a copy of the description (token $description$)
    - Distribution: Change the Distribution. Defaults to Your organisation only
    - Threat Level: Change the Threat Level. Defaults to Undefined
    - Analysis: Change Analysis status. Default to Initial
    - TLP: Change the TLP of the created alert. Defaults to TLP-Amber
    - tags: comma-separated list of tags (not implemented yet)
* Specific alert parameters for MISP serve: If specified, URL and auth key will superseede the config file (misp.conf)
Using those fields you may search in one MISP instance and create events in another one.
    - URL: MISP URL (leave blank to use default settings).
    - Auth Key: The Authkey to submit alerts to (leave blank to use default settings).

# Todo
- implement sslcheck boolean for specific misp_alert settings.
- implement event tagging in misp_alert_create_event
- implement misp_alert with a column per type of attributes
- store some saved searches and lookups as examples

# Licence
This app misp42splunk is licensed under the GNU Lesser General Public License v3.0.
