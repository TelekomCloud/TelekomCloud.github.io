---
layout:   post
title:    "New Ways of Tempest Stress Testing"
date:     2013-09-11
author:   Marc Koderer
comments: true
---

# Overview

There are many stress test frameworks for OpenStack that are all pretty similar in nature. They follow fixed scenarios and fork many worker processes. Often they are difficult to enhance, since they were written for a single purpose.

With [blueprint stress-test](https://blueprints.launchpad.net/tempest/+spec/stress-tests) the community of Tempest developers focused to build a single and very flexible stress test framework, inside of Tempest.

# A Stress Test is not a Test Domain

In the past, stress tests had their own area inside Tempest. New tests were introduced mainly as clones of an exiting API, a scenario test or a mixture of both. But do stress tests really have their own testing domain? 

Two main purposes of stress test are obvious:

 * Having a framework to find/reproduce race conditions
 * Having a framework to simulate real-life load with a mixture of load profiles

Those two topics are already covered by Tempest tests today: grouping API test enables us to detect race conditions, using scenario test, we can simulate load profiles.

# Tempest Stress Test Core

The core of the stress test framework is quite simple: It's responsible for forking worker processes and summarizing results. How many processes should be forked is configurable in a JSON configuration file, which also can provide multiple arguments for each stress test.

# Integration of Existing Tests

In order to stop duplication code, we started to integrate existing Tempest tests into the stress test framework. A wrapper was build to call any kind of unit test and make it available to the framework. With that, it's easy to group existing tests. Here is an example of how this is done for two unit tests:


    [{"action": "tempest.stress.actions.unit_test.UnitTest",
      "threads": 8,
      "kwargs": {"test_method": "tempest.cli.simple_read_only.test_glance.\
                 SimpleReadOnlyGlanceClientTest.test_glance_fake_action",
                 "class_setup_per": "process"},
      "action": "tempest.stress.actions.unit_test.UnitTest",
      "threads": 8,
      "kwargs": {"test_method": "tempest.api.volume.test_volumes_actions.\
                 VolumesActionsTest.test_attach_detach_volume_to_instance",
                 "class_setup_per": "process"},
    }]


This will fork in total 16 processes that will conduct glance and cinder stress testing.

## The Stress Test Discovery

Test discovery is done like this:

<img src="/images/2013-09-11-new-ways-of-tempest-stress-testing/tempest_stress_discovery.png" alt="Drawing" style="width: 650px;"/>

Instead of manually adding exiting tests to the stress test framework, the next logical step was to introduce a decorator. It allows test developers to decide if a test is made available to the stress test framework. Or, to be more precise, it marks tests as being meaningful stress tests. The decorator can be used like this:

    @stresstest(class_setup_per='process')
    @attr(type='smoke')
    def test_attach_detach_volume_to_instance(self):

It can simply get added to any existing unit test or used as the only purpose for a test. Internally it is based on the existing mechanism of attribute discovery of `testtools` and adds the attribute `stress`. It has one mandatory parameter `class_setup_per`, since it must be decided when the `setUpClass` function should be called: For every process, for every action or just globally. This depends on the content of the `setUpClass` and must be decided by the developer. In many cases a call on a per process level is sufficient.

All existing test attributes like `smoke` or `gate` can also be used as filter within the discovery function.

## Which Tests are Good Candidates?

In fact, it's often easier to identify test cases that aren't good candidates:

 * Negative test
 * Single unit test function that cover only one little aspect (like listing volumes)
 * Tests that interfere each other (like changing quotas)
 
All others tests might be interesting candidates to get integrated and used from the test framework. So please feel free to identify new cases and contribute them to OpenStack/Tempest.
