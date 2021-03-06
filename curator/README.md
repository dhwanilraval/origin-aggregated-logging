# Curator

Curator allows the admin to remove old indices from Elasticsearch on a per-project
basis.  The pod will read its configuration from a mounted yaml file that
is structured like this:

    PROJECT_NAME:
      ACTION:
        UNIT: VALUE

    PROJECT_NAME:
      ACTION:
        UNIT: VALUE          
     ...      

* PROJECT\_NAME - the actual name of a project - "myapp-devel"
** For operations logs, use the name `.operations` as the project name
* ACTION - the action to take - currently only "delete"
* UNIT - one of "days", "weeks", or "months" 
* VALUE - an integer for the number of units 
* `.defaults` - use `.defaults` as the PROJECT\_NAME to set the defaults for
projects that are not specified
** runhour: NUMBER - hour of the day in 24 hour format at which to run the 
curator jobs
** runminute: NUMBER - minute of the hour at which to run the curator jobs
** timezone: STRING - String in tzselect(8) or timedatectl(1) format - the
   default timezone is `UTC`

For example, using::

    myapp-dev:
     delete:
       days: 1
    
    myapp-qe:
      delete:
        weeks: 1
    
    .operations:
      delete:
        weeks: 8
    
    .defaults:
      delete:
        days: 31
      runhour: 0
      runminute: 0
      timezone: America/New_York
    ...

Every day, curator will run, and will delete indices in the myapp-dev project
older than 1 day, and indices in the myapp-qe project older than 1 week.  All
other projects will have their indices deleted after they are 31 days old.  The
curator jobs will run every day at midnight in the `America/New_York` timezone,
regardless of geographical location where the pod is running, or the timezone
setting of the pod, host, etc.

*WARNING*: Using `months` as the unit

When you use month based trimming, curator starts counting at the _first_ day of
the current month, not the _current_ day of the current month.  For example, if
today is April 15, and you want to delete indices that are 2 months older than
today (`delete: months: 2`), curator doesn't delete indices that are dated
older than February 15, it deletes indices older than _February 1_.  That is,
it goes back to the first day of the current month, _then_ goes back two whole
months from that date.
If you want to be exact with curator, it is best to use `days` e.g. `delete: days: 31`
[Curator issue](https://github.com/elastic/curator/issues/569)

To create the curator configuration, you can just edit the current
configuration in the deployed configmap:

    $ oc edit configmap/logging-curator

If this does not redeploy automatically, redeploy manually:

    $ oc deploy --latest logging-curator

For scripted deployments, do this:

    $ create /path/to/mycuratorconfig.yaml
    $ oc delete configmap logging-curator ; sleep 1
    $ oc create configmap logging-curator --from-file=config.yaml=/path/to/mycuratorconfig.yaml ; sleep 1

Then redeploy as above.

You can also specify default values for the run hour, run minute, and age in
days of the indices when processing the curator template.  Use
`CURATOR_RUN_HOUR` and `CURATOR_RUN_MINUTE` to set the default runhour and
runminute, `CURATOR_RUN_TIMEZONE` to set the run timezone, and use
`CURATOR_DEFAULT_DAYS` to set the default index age in days. These are only
used if not specified in the config file.
