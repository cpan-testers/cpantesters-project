
# CPAN Testers Project Roadmap

## Operations and Documentation

* Status: <span style="color: yellow">In Development</span>

### Repositories

* [cpantesters-project](https://github.com/cpan-testers/cpantesters-project)
    * Documenting the project plan and ongoing issues
* [cpantesters-deploy](https://github.com/cpan-testers/cpantesters-deploy)
    * Documenting the current application and deployment process
* [cpantesters-schema](https://github.com/cpan-testers/cpantesters-schema)
    * Documenting the current database

### Requirements

* Monitoring
    * XXX
* Statistics
    * XXX
* Deployment
    * Deployment should be automated
    * A QA/Staging environment should be set up
    * The deployment should be able to create a development server in
      a VM

### Current Problems

* Monitoring
    * Various users have various monitoring scripts of their own
      in-place, but CPAN Testers does not have its own monitoring
    * CPAN Testers has a lot of moving parts that each need monitoring
      for stability
    * Possible Technologies
        * Icinga2
            * Based on / forked from Nagios, it has a
              well-documented set of configuration, easy command line
              tools for automation and deployment, and a nice web GUI
              application for viewing system status
        * Others. Need to evaluate.
    * Opportunities
        * Build a Rex plugin to deploy the monitoring tool and automate
          building the monitoring configuration.

* Metrics and Statistics
    * The statistics pages on
      <http://stats.cpantesters.org/osmatrix-month.html> are not being
      updated.
    * More research and documentation on this problem is needed
        * How are statistics being generated?
        * Where are statistics being stored?
        * How are statistics reports being generated?
    * Possible Technologies
        * Graphite
            * Python based statistics database
            * Easy configuration and maintenance
            * Remote access over the network
            * Performs well enough for large organizations (which we are
              not)
            * Works with Kibana for dashboards and graphs
            * Supports automated aggregations
            * Does not support tagging metrics for later aggregation
              queries
        * RRDTool
            * Fast, native round-robin databases
            * Very simple data layer
            * Will need a model layer on top to route incoming stats to
              the right database file
            * Will need a remote access layer on top to allow
              centralized statistics storage
            * RRDTool's limitations is why Graphite exists
        * OpenTSDB
            * Java/Hadoop-based time series database
            * Very fast
            * Works with Kibana for dashboards and graphs
            * Supports tagging metrics for running queries on different
              tags
    * Opportunities
        * Build a Rex plugin to deploy the metrics database and automate
          building the configuration

## Backend: Data Processing (ETL)

* Status: <span style="color: blue">Planning</span>

### Requirements

The Backend must read incoming data from the Metabase and process it
into useful statistics:

* Distribution Name
* Distribution Version
* Perl Version
* Perl Architecture
* OS Name
* OS Version
* Status (pass/fail/unknown/na)

These statistics are made available to generate reports via APIs and the
web app.

The backend should also store summary statistics containing only test
status (pass/fail/unknown/na) aggregated by distribution and
distribution version, for efficient consumption by other systems.

### Current Problems

* Performance
    * Users expect to see reports very shortly after they release their
      distribution, or at least, very shortly after a report is
      delivered to the Metabase. If a report is in the Metabase, but not
      visible on the web app, they get frustrated.
        * For example, the [Matrix](http://matrix.cpantesters.org) app
          has a corresponding [Fast
          Matrix](http://fast-matrix.cpantesters.org) which loads data
          from the Metabase log, bypassing the CPAN Testers backend

### Future Plans

* Messaging API / Job workers
    * Using a message queue to get pushed updates from the Metabase to
      an array of backend workers can improve performance and
      reliability
    * The API could also be consumed by external users to allow
      real-time updates on the status of their reports
    * Possible Technologies
        * [Minion](http://metacpan.org/pod/Minion)
            * Minion is a job queue / worker model based on Mojolicious
            * Using a job-runner platform will increase scalability
            * Using existing tech will decrease development time
            * Integrates well with [Mojolicious](http://mojolicious.org)
              web framework
        * [Mercury](http://metacpan.org/pod/Mercury)
            * Mercury is a message broker using WebSockets which
              supports a worker job distribution pattern
            * Written by preaction, so bias is likely
            * Using a pure-Perl messaging solution will simplify
              distribution and installation as opposed to ZeroMQ or
              nanomsg
                * ZeroMQ is installable from CPAN as long as there's
                  a compiler available
            * Using a pure-Perl message broker will simplify
              distribution and installation as opposed to ActiveMQ,
              RabbitMQ, or Kafka
            * Mercury currently has no authentication, which would be
              necessary to ensure only our job workers are available in
              the worker pool

* ETL Framework
    * It would be nice if there was an existing ETL framework we could
      use for our processing to reduce the amount of code we have to
      write ourselves.
    * I also know this has been a strange gap in CPAN, having done a lot
      of ETL at various workplaces. Their ETL frameworks have never
      escaped into the wild.
    * Since our problem is basically "Copy data from this JSON blob into
      these fields in a SQL database", it seems like an ETL framework
      should be able to do this in a few lines of configuration.
    * Possible Technologies
        * [Catmandu](http://librecat.org) <http://metacpan.org/pod/Catmandu>
            * Mature ETL with command-line utilities and a Perl API
            * Supports a simple transformation language
            * Supports MongoDB, ElasticSearch, and DBI
            * Logging with Log::Any
            * Good documentation at <http://librecat.org/Catmandu/>
        * [Data::Tubes](http://github.polettix.it/Data-Tubes)
          <http://metacpan.org/pod/Data::Tubes>
            * Pretty new, author warns some option names may change
            * Pure Perl. Easy to understand for Perl developers
            * Really easy to get started and add new functionality to
            * Some questionable API parts that may lead to strange
              interactions
                * `tube()` in scalar context may return a single item or
                  an arrayref if there are multiple items. No way to
                  know which is happening
                * Package names in a tube are subject to some confusing
                  rules regarding what they resolve to:
                  <https://metacpan.org/pod/distribution/Data-Tubes/lib/Data/Tubes/Util.pod#resolve_module>
        * [ETL::Yertl](http://preaction.me/yertl)
            * Yertl is an ETL framework written in Perl, designed to
              build ETL jobs in shell.
            * Written by preaction, so bias is likely
            * Yertl would need quite a bit of work before it could be
              used for CPAN Testers
                * A Perl-based API for use inside a Minion task
                * A document-storage database API which can be hooked
                  into the Metabase (Amazon SimpleDB)
        * [ETL::Pipeline](http://metacpan.org/pod/ETL::Pipeline)
            * ETL::Pipeline is a simple ETL for mapping input data to an
              output
            * Seems to be missing the SQL output referred to in the
              documentation
              <https://rt.cpan.org/Ticket/Display.html?id=117475>
            * No public repository for collaboration
              <https://rt.cpan.org/Ticket/Display.html?id=117473>

## Data APIs

* Status: <span style="color: blue">Planning</span>

### Requirements

There is a wide ecosystem of applications that depend on data from CPAN
Testers.

* [Metacpan](http://metacpan.org) displays a summary of CPAN Testers
  data in a release's infobox
* [Matrix](http://matrix.cpantesters.org) displays an alternate view of
  CPAN Testers data (even bypassing CPAN Testers when needed:
  <http://fast-matrix.cpantesters.org>
* [Analysis
  (CPAN::Testers::ParseReport)](http://analysis.cpantesters.org)
  analyzes the data to perform statistical analysis to find possible
  reasons behind new failures
* [TuX's CPAN Dashboard](http://tux.nl/perl.html)
* [CPAN::Dashboard](https://metacpan.org/pod/CPAN::Dashboard)
* [cpXXXan](http://cpxxxan.barnyard.co.uk) which uses CPAN Testers
  results to build CPAN indexes with the last known good release of
  a distribution for your Perl and OS.

These applications need APIs to get at the CPAN Testers data in various
forms, including the exact form in the database, and a summary form
aggregated by distribution+version useful for small views.

The APIs created must be stable, so versioned APIs is a requirement.
They must also be as fast as possible, so aggressive database indexing
and local caching should be used.

### Current Problems

* Performance
    * The existing APIs do not provide an efficient way to summarize the
      data, resulting in the Matrix pulling a 30MB JSON file for a very
      well-tested distribution like Test-Simple
        * <https://github.com/cpan-testers/cpantesters-project/issues/8>

* Stability
    * The current summary statistics API is a SQLite database, which has
      been error-prone to create, resulting in corrupted SQLite
      databases
        * This API is still being used by MetaCPAN, and we need a new
          API before we can shut this down
          * <https://github.com/cpan-testers/cpantesters-project/issues/9>

### Future Plans

* Provide a web API to retrieve summary data
    * Backend will generate and store the summary data locally
    * Create a bulk-read API for copying into other databases
        * Allow the API to accept an "updated since" to return only
          records that have changed
    * Create a REST-style API for returning data on single distributions
        * Use HTTP Cache-Control heavily for performance
    * Rely on a caching proxy or static files for performance
    * Possible Technologies
        * [Mojolicious](http://mojolicious.org)
            * Pure-perl, high performance, asynchronous web framework
        * [OpenAPI](https://openapis.org/specification)
            * An specification format for JSON APIs which helps document
              and validate
            * [Mojolicious::Plugin::OpenAPI](http://metacpan.org/pod/Mojolicious::Plugin::OpenAPI)

## Web App

* Status: <span style="color: red">Pending</span>

### Requirements

XXX

### Current Problems

XXX

### Future Plans

XXX

## Metabase

* Status: <span style="color: red">Under Discussion</span>

### Requirements

XXX

### Current Problems

XXX

### Future Plans

XXX

