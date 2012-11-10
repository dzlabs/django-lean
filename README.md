django-lean
===========

A blessed fork of <https://bitbucket.org/akoha/django-lean> with a new maintainer, contact @jdunck on twitter if you need support, or the django-lean Google Group.  
Existing, known issues, are here: <https://bitbucket.org/akoha/django-lean/issues?status=new&status=open> but please report new issues here on Github.


Welcome
-------

[django-lean][dl] aims to be a collection of tools for [Lean Startups](http://www.startuplessonslearned.com/search/label/lean%20startup) using the Django platform. Currently it provides a framework for implementing [split-test experiments](http://www.startuplessonslearned.com/search/label/split-test) in JavaScript, Python, or Django template code along with administrative views for analyzing the results of those experiments.

[django-lean][dl] is one of the more mature AB testing systems for Django, but if you have contributions or other improvements, they're welcome.

Discussion Group
----------------

For discussions related to the use or development of [django-lean][dl] please use our [Google group](http://groups.google.com/group/django-lean).

Features
--------

[django-lean][dl] allows you to perform split-test experiments on your users. In brief, this involves exposing 50% of your users to one implementation and 50% to another, then comparing the performance of these two groups with regards to certain metrics.  Multi-variate testing (that is, a single experiment with more than 2 tested outcomes) are not yet supported, but we'd welcome changes to support that.

Often you can achive the goals of multivariate testing through concurrent experiments with the same control but different treatments, or through iteration of winner-as-control vs new candidate treatmemnt.

### Experiments Types

[django-lean][dl] supports two kinds of experiments currently:

* Anonymous Conversion experiments compare the achievement of goals you define (i.e. "register" or "add to cart") amongst two groups of anonymous users.
* Registered Engagement experiments compare a quantitative measure of engagement that you define (i.e., activity, revenue, time on site, ...) amongst two groups of registered users.

There's no real reason why one couldn't measure engagement of anonymous users or conversions of registered users (i.e. "basic to pro") but we didn't need this, so they're not implemented (again, patches welcome!).

### Experiment Reports

[django-lean][dl] provides daily reports of experiment results, including confidence levels.

* For conversion experiments, results and confidence are displayed per conversion goal type (and for 'any' goal). Confidence is calculated using the chi-square method.
* For engagement experiments, confidence is calculated using the Student's t-test method.

Experiment reports are prepared using the `update_experiment_reports` management command. It's advisable to execute this command from a nightly cron-job.

### Bot Exclusion

[django-lean][dl] attempts to exclude non-human visitors from experiment reports by only recording data for visitors who have JavaScript enabled.

### Experiment Management

Experiments may be defined, enabled, disabled, or promoted via the `django-admin` interface. You may also define experiments in your source tree and have them automatically loaded into the database (see [experiments.loader.ExperimentLoader](https://github.com/dzlabs/django-lean/blob/master/src/django_lean/experiments/loader.py)).

Each experiment has a state, which affects whether visitors are enrolled in the experiment, and whether they see the control or test case of the experiment.

* `disabled`: No visitors are enrolled in the experiment. All visitors see the *control* case of the experiment, even if they were previously enrolled in the test group.
* `enabled`: All visitors who encounter the experiment are enrolled randomly in either the test or control group, and see the corresponding case.
* `promoted`: No visitors are enrolled in the experiment. All visitors see the *test* case of the experiment, even if they were previously enrolled in the control group.

New experiments start in the `disabled` state.

### Experiment Implementation

[django-lean][dl] makes it easy to implement experiments in Python, JavaScript, or Django templates. Here are some examples:

#### Python
<pre><code>
#!python
from experiments.models import Experiment
from experiments.utils import WebUser

...

def my_view_func(request, *args, **kwargs):
    if Experiment.test("my_experiment_name", WebUser(request)):
        view = edit_profile_test
    else:
        view = edit_profile_control
    return view(request, *args, **kwargs)
</code></pre>

Django Templates
----------------
<pre><code>
#!html+django

{% load experiments %}

...

<p>If you like what you see,

{% experiment change_buy_to_italics control  %}
<b>buy now!</b>
{% endexperiment %}

{% experiment change_buy_to_italics test  %}
<i>buy now!</i>
{% endexperiment %}

</p>
</code></pre>

#### JavaScript

**(In your HTML template:)**
<pre><code>
#!html+django

{% load experiments %}

<script>
<![CDATA[
{% include "experiments/include/experiments.js" %}
]]>
</script>

{% include "experiments/include/experiment_enrollment.html" %}


...

{% clientsideexperiment <experiment_name> %}
</code></pre>

**(In your JavaScript:)**
<pre><code>
#!javascript
if (experiment.test("<experiment_name>")) {
  // test case
} else {
  // control case
}
</code></pre>

### Conversion Tracking

Conversion experiments track the rate of conversion for their test and control groups. It is up to you to define and record the achievement of one or more project specific conversion goals.

#### Defining Conversion Goals

Conversion goals are defined by placing rows in the `experiments_goaltypes` table. This table is not currently exposed via `django-admin` but probably should be (patches welcome!). Alternative ways to populate it include manually via SQL, manually via the Django shell, via your `initial_data` fixture, or by defining a data migration in your database management tool (we use [django-south](http://south.aeracode.org/)).

Here is an example of defining a goal type using the Django shell:

<pre><code>
#!pycon
erik-wrights-macbook-pro:akoha erikwright$ ./manage.py shell --plain
Python 2.6.2 (r262:71600, Jul 16 2009, 12:11:28) 
[GCC 4.0.1 (Apple Inc. build 5490)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
(InteractiveConsole)
>>> from experiments.models import GoalType
>>> GoalType.objects.create(name="signup")
<GoalType: <GoalType: signup>>
>>> 
</code></pre>

#### Recording Conversions

Once you have defined a conversion goal type, you must record it when it is achieved place. This may be done either programatically, or using a tracking pixel.

##### Recording Conversions Programatically

<pre><code>
#!python
from experiments.models import GoalRecord
from experiments.utils import WebUser

...

def my_signup_view_func(request, *args, **kwargs):
    # ... process the signup request
    GoalRecord.record("signup", WebUser(request))
    # ...
</code></pre>

#### Recording Conversions Using a Tracking Pixel

Sometimes a conversion happens somewhere that you don't control (for example, an e-commerce platform). In this case, you can record the conversion by placing a transparent 1x1 pixel on the page that users see after the conversion occurs (i.e., the 'Thank You' page after purchase).

<pre><code>
#!html
<img src="http://example.com/experiments/goal/purchase" height="1" width="1" />
</code></pre>

### Engagement Tracking

Engagement experiments track an arbitrary engagement value for each user in their test and control groups. It is up to you to define a function that calculates an appropriate engagement value for your users.

Here is an example engagement calculator:

<pre><code>
#!python
class MyEngagementScoreCalculator(object):

    def calculate_user_engagement_score(self, user, start_date, end_date):
        """
        Define engagement as 'dollars spent per day'
        """

        days_in_period = (end_date - start_date).days + 1

        period_purchase_total = sum([p.subtotal for p in Purchase.objects.filter(
                                                        user=user, date__gte=start_date, date__lte=end_date)])
        engagement_score = ((float)(period_purchase_total) /
                                                        days_in_period)
        return engagement_score
</code></pre>

Your engagement calculator must be registered in `settings.py` as follows:

<pre><code>
#!python
...
LEAN_ENGAGEMENT_CALCULATOR = "mycompany.MyEngagementScoreCalculator"
...
</code></pre>

Dependencies
------------

[django-lean][dl] has a number of dependencies:

* [Mok](http://code.google.com/p/pymox/)
* [Beautiful Soup](http://www.crummy.com/software/BeautifulSoup/)
* [JQuery](http://jquery.com/)

Mox and Beautiful Soup are used exclusively by unit tests. JQuery is used only to execute a single, trivial AJAX request and could easily be removed from the dependency list if one were motivated (patch please!).

You may optionally use [South](http://south.aeracode.org/) in order to facilitate migrations of the [django-lean][dl] database schema, but it is not required:

Installation
------------

[django-lean][dl] has been developed with Django 1.0. Unit Tests run successfully with Django 1.1 but it has not been tried in production. If you successfully run it with another version, please update this documentation.

1. Install [django-lean][dl] using [easy_install][ez]
1. Add `experiments` to `INSTALLED_APPS` in `settings.py`
1. Ensure that `django.core.context_processors.request` is in `TEMPLATE_CONTEXT_PROCESSORS` in `settings.py`
1. Run `manage.py syncdb` to set up the [django-lean][dl] tables.
1. Run `manage.py test experiments` to see if everything is set up correctly.
1. For every page that will contain an experiment (or in the response after a server-side experiment):
1. Ensure that JQuery is included.
1. Ensure that `experiments/include/experiments.js` is somehow included (perhaps copy it where your static files go, include it as part of your existing generated JS files, map it from `urls.py`, include it directly in a `<script/>` tag, etc.).
1. Ensure that `experiments/include/experiment_enrollment.html` is rendered by your template.
1. Install the admin and public url mappings in your site `urls.py`
1. Register your engagement calculator in `settings.py`.
1. Define one or more conversion goal types.
1. Add conversion goal recording where appropriate.
1. Define, implement, and enable an experiment
1. Call `manage.py update_experiment_reports` nightly.
1. Experiment, learn, repeat!

### Installing URL Mappings

The following snippet added to `urls.py` should properly install the needed URL mappings (adjust to meet your needs):

<pre><code>
#!python
...
urlpatterns += patterns('',
    url(r'^admin/django-lean/', include('experiments.admin_urls')),
    url(r'^django-lean/', include('experiments.urls')),
)
...
</code></pre>

Development
-----------

<pre><code>
#!console
erik-wrights-macbook-pro:~ erikwright$ hg clone http://bitbucket.org/akoha/django-lean/
destination directory: django-lean
requesting all changes
adding changesets
adding manifests
adding file changes
added 46 changesets with 155 changes to 69 files
updating working directory
65 files updated, 0 files merged, 0 files removed, 0 files unresolved
erik-wrights-macbook-pro:~ erikwright$ cd django-lean
erik-wrights-macbook-pro:django-lean erikwright$ /usr/bin/python bootstrap.py 
Creating directory '/Users/erikwright/django-lean/bin'.
Creating directory '/Users/erikwright/django-lean/parts'.
Creating directory '/Users/erikwright/django-lean/eggs'.
Creating directory '/Users/erikwright/django-lean/develop-eggs'.
Generated script '/Users/erikwright/django-lean/bin/buildout'.
erik-wrights-macbook-pro:django-lean erikwright$ ./bin/buildout 
Develop: '/Users/erikwright/django-lean/.'
package init file 'experiments/tests/data/__init__.py' not found (or not a regular file)
Getting distribution for 'djangorecipe'.
Got djangorecipe 0.20.
Getting distribution for 'zc.recipe.egg'.
Got zc.recipe.egg 1.2.2.
Installing django.
django: Downloading Django from: http://www.djangoproject.com/download/1.1.1/tarball/
Getting distribution for 'mox'.
zip_safe flag not set; analyzing archive contents...
Got mox 0.5.0.
Getting distribution for 'BeautifulSoup'.
Got BeautifulSoup 3.1.0.1.
Generated script '/Users/erikwright/django-lean/bin/django'.
Generated script '/Users/erikwright/django-lean/bin/test'.
erik-wrights-macbook-pro:django-lean erikwright$ ./bin/test
Creating test database...
Creating table auth_permission
Creating table auth_group
Creating table auth_user
Creating table auth_message
Creating table django_content_type
Creating table django_session
Creating table django_site
Creating table experiments_anonymousvisitor
Creating table experiments_goaltype
Creating table experiments_goalrecord
Creating table experiments_experiment
Creating table experiments_participant
Creating table experiments_dailyengagementreport
Creating table experiments_dailyconversionreport
Creating table experiments_dailyconversionreportgoaldata
Installing index for auth.Permission model
Installing index for auth.Message model
Installing index for experiments.AnonymousVisitor model
Installing index for experiments.GoalRecord model
Installing index for experiments.Experiment model
Installing index for experiments.Participant model
Installing index for experiments.DailyEngagementReport model
Installing index for experiments.DailyConversionReport model
Installing index for experiments.DailyConversionReportGoalData model
..............................
----------------------------------------------------------------------
Ran 30 tests in 24.305s

OK
Destroying test database...
erik-wrights-macbook-pro:django-lean erikwright$ 
</code></pre>

Other Resources
---------------

The following links might be of interest to those wanting to learn more about Lean Startup, split testing, and related concepts.

* <http://elem.com/~btilly/effective-ab-testing/>
* <http://www.startuplessonslearned.com/search/label/lean%20startup>
* <http://www.startuplessonslearned.com/search/label/split-test>
* <http://www.slideshare.net/erikwright/djangolean-akohas-opensource-ab-experimentation-framework-montreal-python-9>

[dl]: https://github.com/dzlabs/django-lean
[ez]: http://pypi.python.org/pypi/setuptools

