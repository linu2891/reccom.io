# ScaldingUnit - TDD utils for Scalding developers

![Scalding logo BW](./logo/scaldingBW.png "Scalding")

## NOTE: ScaldingUnit merged into Scalding

ScaldingUnit has been merged with some small changes (mostly the name of the class to extend has been renamed from TestInfrastructure to BddDsl)
into scalding-core development branch in date 30th Jan 2014.
Current project will be kept active to be used for version of scalding prior to 0.9.
ScaldingUnit is also not working with Scalding 0.9.1 so please switch to using the BddDsl trait instead of the TestInfrastructure for project using Scalding version > 0.9

## Aim

The aim of this project is to allow user to write Scalding (https://github.com/twitter/scalding) map-reduce jobs in a more modular and test-driven way.
It is based on the experience done in the Big Data unity at BSkyB where it originated and is currently used and maintained.
It essentially provides a test harness to support the decomposition of a Scalding Map-Reduce Job into a series of smaller steps,
each one testable independently before being assembled into the main Job that will then be tested as a whole using Scalding-based
tests.

[![Build Status](https://api.travis-ci.org/galarragas/ScaldingUnit.png)](http://travis-ci.org/galarragas/ScaldingUnit)

## What does it look like

A test written with scalding unit look as shown below:

With ScalaTest

```scala
class SampleJobPipeTransformationsSpec extends FlatSpec with ShouldMatchers with TupleConversions with TestInfrastructure {
  "A sample job pipe transformation" should "add user info" in {
    Given {
      List(("2013/02/11", 1000002l, 1l)) withSchema EVENT_COUNT_SCHEMA
    } And {
      List( (1000002l, "stefano@email.com", "10 Downing St. London") ) withSchema USER_DATA_SCHEMA
    } When {
      (eventCount: RichPipe, userData: RichPipe) => eventCount.addUserInfo(userData)
    } Then {
      buffer: mutable.Buffer[(String, Long, String, String, Long)] =>
        buffer.toList shouldEqual List( ("2013/02/11", 1000002l, "stefano@email.com", "10 Downing St. London", 1l) )
    }
  }
}
```

or with Specs2

```scala
import org.specs2.{mutable => mutableSpec}

class SampleJobPipeTransformationsSpec2Spec extends mutableSpec.SpecificationWithJUnit with TupleConversions with TestInfrastructure {

  // See: https://github.com/twitter/scalding/wiki/Frequently-asked-questions

  "A sample job pipe transformation" should {
     Given {
      List(("2013/02/11", 1000002l, 1l)) withSchema EVENT_COUNT_SCHEMA
    } And {
      List((1000002l, "stefano@email.com", "10 Downing St. London")) withSchema USER_DATA_SCHEMA
    } When {
      (eventCount: RichPipe, userData: RichPipe) => eventCount.addUserInfo(userData)
    } Then {
      buffer: mutable.Buffer[(String, Long, String, String, Long)] =>
        "add user info" in {
          buffer.toList shouldEqual List(("2013/02/11", 1000002l, "stefano@email.com", "10 Downing St. London", 1l))
        }
    }

  }
}
```


Where `addUserInfo` is a function joining two richPipes to generate an enriched one.

## Usage

To add the dependency on maven include:

```xml
<dependency>
  <groupId>com.pragmasoft</groupId>
  <artifactId>scalding-unit</artifactId>
  <version>0.6</version>
</dependency>
```

and repository:

```xml
<repository>
  <id>conjars.org</id>
  <url>http://conjars.org/repo</url>
</repository>
```

The code is not currently working correctly if you are using Scalding 0.9.rc4.
The code supporting the 0.9 version is currently in a branch until 0.9 will be GA. To use it include the following dependency:

```xml
<dependency>
  <groupId>com.pragmasoft</groupId>
  <artifactId>scalding-unit</artifactId>
  <version>0.5-scalding0.9</version>
</dependency>
```

Support for 0.9.rc4 hasn't been updated to version 0.6 yet.

## Using ScaldingUnit with Different Versions of Scala

ScaldingUnit is built with Scala 2.10 but has been tested with Scala 2.10 and from version 0.5 with 2.9.2.
Refer to the example directories (examples and examples2.9) to see how to use it with ScalaTest and Specs2.

The explanation below refers to Scala 2.10 code but the changes to apply to use 2.9 are minimal.

## Motivation and details

## Writing and testing Scalding Jobs without ScaldingUnit

A Scalding job consists in a series of transformations applied to one or more sources in order to create one or more
output resources or sinks. A very simple example taken from the Scalding documentations is as follows.

```scala
package com.twitter.scalding.examples

import com.twitter.scalding._

class WordCountJob(args : Args) extends Job(args) {
    TextLine( args("input") )
     .flatMap('line -> 'word) { line : String => tokenize(line) }
     .groupBy('word) { _.size }
     .write( Tsv( args("output") ) )

    // Split a piece of text into individual words.
    def tokenize(text : String) : Array[String] = {
     // Lowercase each word and remove punctuation.
     text.toLowerCase.replaceAll("[^a-zA-Z0-9\\s]", "").split("\\s+")
    }
}
```

The transformations are defined as operations on a `cascading.pipe.Pipe` class or on the richer wrapper `com.twitter.scalding.RichPipe`.
Scalding provides a way of testing Jobs via the com.twitter.scalding.JobTest class. This class allows to specify values for the different
Job sources and to specify assertions on the different job sinks.
This approach works very well to do end to end test on the Job and is good enough for small jobs as the one described above
but doesn't encourage modularisation when writing more complex Jobs. That's the reason we started working on ScaldingUnit.

## Checking your Pipes, modular unit test of your Scalding Job

When the Job logic become more complex it is very helpful to decompose its work in simpler functions to be tested independently before being
aggregated into the Job.

Let's consider the following Scalding Job (it is still a simple one for reason of space but should give the idea):

```scala
class SampleJob(args: Args) extends Job(args) {
  val INPUT_SCHEMA = List('date, 'userid, 'url)
  val WITH_DAY_SCHEMA = List('date, 'userid, 'url, 'day)
  val EVENT_COUNT_SCHEMA = List('day, 'userid, 'event_count)
  val OUTPUT_SCHEMA = List('day, 'userid, 'email, 'address, 'event_count)

  val USER_DATA_SCHEMA = List('userid, 'email, 'address)

  val INPUT_DATE_PATTERN: String = "dd/MM/yyyy HH:mm:ss"

  Osv(args("eventsPath")).read
    .map('date -> 'day) {
           date: String => DateTimeFormat.forPattern(INPUT_DATE_PATTERN).parseDateTime(date).toString("yyyy/MM/dd");
         }
    .groupBy(('day, 'userid)) { _.size('event_count) }
    .joinWithLarger('userid -> 'userid, Osv(args("userInfoPath")).read).project(OUTPUT_SCHEMA)
    .write(Tsv(args("outputPath")))
}
```

It is possible to identify three main operation performed during this task. Each one is providing a identifiable and
 autonomous transformation to the pipe, just relying on a specific input schema and generating a transformed pipe with a
 potentially different output schema. Following this idea it is possible to write the Job in this way:

```scala
import SampleJobPipeTransformations._

class SampleJob(args: Args) extends Job(args) {

  Osv(args("eventsPath")).read
    .addDayColumn
    .countUserEventsPerDay
    .addUserInfo(Osv(args("userInfoPath")).read)
    .write(Tsv(args("outputPath")))
}
```

Where the single operations have been extracted as basic functions into a separate class that is not a Scalding Job:

```scala
package object SampleJobPipeTransformations  {

  val INPUT_SCHEMA = List('date, 'userid, 'url)
  val WITH_DAY_SCHEMA = List('date, 'userid, 'url, 'day)
  val EVENT_COUNT_SCHEMA = List('day, 'userid, 'event_count)
  val OUTPUT_SCHEMA = List('day, 'userid, 'email, 'address, 'event_count)

  val USER_DATA_SCHEMA = List('userid, 'email, 'address)

  val INPUT_DATE_PATTERN: String = "dd/MM/yyyy HH:mm:ss"

  implicit def wrapPipe(pipe: Pipe): SampleJobPipeTransformationsWrapper = new SampleJobPipeTransformationsWrapper(new RichPipe(pipe))
  implicit class SampleJobPipeTransformationsWrapper(val self: RichPipe) extends PipeOperations {

    /**
     * Input schema: INPUT_SCHEMA
     * Output schema: WITH_DAY_SCHEMA
     * @return
     */
    def addDayColumn = self.map('date -> 'day) {
      date: String => DateTimeFormat.forPattern(INPUT_DATE_PATTERN).parseDateTime(date).toString("yyyy/MM/dd");
    }

    /**
     * Input schema: WITH_DAY_SCHEMA
     * Output schema: EVENT_COUNT_SCHEMA
     * @return
     */
    def countUserEventsPerDay = self.groupBy(('day, 'userid)) { _.size('event_count) }

    /**
     * Joins with userData to add email and address
     *
     * Input schema: WITH_DAY_SCHEMA
     * User data schema: USER_DATA_SCHEMA
     * Output schema: OUTPUT_SCHEMA
     */
    def addUserInfo(userData: Pipe) = self.joinWithLarger('userid -> 'userid, userData).project(OUTPUT_SCHEMA)
  }
}
```

The main Job class is responsible of essentially dealing with the configuration, opening the input and output richPipes and
combining those macro operations. Using implicit transformations and value classes as shown above is possible to use those
operations as if they were part of the RichPipe class.

It is now possible to test all this method independently and without caring about the source and sinks of the job and of
the way the configuration is given.

The specification of the transformation class is shown below:

```scala
class SampleJobPipeTransformationsSpec extends FlatSpec with ShouldMatchers with TupleConversions with TestInfrastructure {
  "A sample job pipe transformation" should "add column with day of event" in {
    Given {
      List( ("12/02/2013 10:22:11", 1000002l, "http://www.youtube.com") ) withSchema INPUT_SCHEMA
    } When {
      pipe: RichPipe => pipe.addDayColumn
    } Then {
      buffer: mutable.Buffer[(String, Long, String, String)] =>
        buffer.toList(0) shouldEqual (("12/02/2013 10:22:11", 1000002l, "http://www.youtube.com", "2013/02/12"))
    }
  }

  it should "count user events per day" in {
    def sortedByDateAndIdAsc( left: (String, Long, Long), right: (String, Long, Long)): Boolean =
      (left._1 < right._1) || ((left._1 == right._1) && (left._2 < left._2))

    Given {
      List(
          ("12/02/2013 10:22:11", 1000002l, "http://www.youtube.com", "2013/02/12"),
          ("12/02/2013 10:22:11", 1000002l, "http://www.youtube.com", "2013/02/12"),
          ("11/02/2013 10:22:11", 1000002l, "http://www.youtube.com", "2013/02/11"),
          ("15/02/2013 10:22:11", 1000002l, "http://www.youtube.com", "2013/02/15"),
          ("15/02/2013 10:22:11", 1000001l, "http://www.youtube.com", "2013/02/15"),
          ("15/02/2013 10:22:11", 1000003l, "http://www.youtube.com", "2013/02/15"),
          ("15/02/2013 10:22:11", 1000001l, "http://www.youtube.com", "2013/02/15"),
          ("15/02/2013 10:22:11", 1000002l, "http://www.youtube.com", "2013/02/15")
        ) withSchema WITH_DAY_SCHEMA
    } When {
      pipe: RichPipe => pipe.countUserEventsPerDay
    } Then {
      buffer: mutable.Buffer[(String, Long, Long)] =>
        buffer.toList.sortWith(sortedByDateAndIdAsc(_, _)) shouldEqual List(
                ("2013/02/11", 1000002l, 1l),
                ("2013/02/12", 1000002l, 2l),
                ("2013/02/15", 1000001l, 2l),
                ("2013/02/15", 1000002l, 2l),
                ("2013/02/15", 1000003l, 1l)
              )
    }
  }


  it should "add user info" in {
    Given {
      List(("2013/02/11", 1000002l, 1l)) withSchema EVENT_COUNT_SCHEMA
    } And {
      List( (1000002l, "stefano@email.com", "10 Downing St. London") ) withSchema USER_DATA_SCHEMA
    } When {
      (eventCount: RichPipe, userData: RichPipe) => eventCount.addUserInfo(userData)
    } Then {
      buffer: mutable.Buffer[(String, Long, String, String, Long)] =>
        buffer.toList shouldEqual List( ("2013/02/11", 1000002l, "stefano@email.com", "10 Downing St. London", 1l) )
    }
  }
}
```

The TestInfrastructure trait is providing a BDD-like syntax to specify the Input to supply to the operation to test and
to write the expectations on the results (the upper case syntax is caused by the fact that
the `then` keyword since it is deprecated from Scala 2.10).

Once the different steps have been tested thoroughly it is possible to combine them in the main Job and test the end to end
behavior using the JobTest class provided by Scalding.

## Testing in Local and Hadoop Mode

Until version 0.5 all tests were executed in Local mode. From version 0.6 is possible to decide if testing in local or
in Hadoop mode (the runHadoop method in JobTest).

Check the sample code for details but, in summary, mixing-in with the `TestInfrastructure` trait will cause the test to be
executed locally as before. Using the trait `HadoopTestInfrastructure` will run the test on Hadoop.
If you want to execute selected test on Hadoop or to decide checking some configuration or execution parameter you can
mix with the `MultiTestModeTestInfrastructure` and define the test mode in the test method or in the test class as follows:

```scala
class SampleJobPipeConfigurableRunModeTransformationsSpec2Spec extends mutableSpec.SpecificationWithJUnit with TupleConversions with MultiTestModeTestInfrastructure {

  // See: https://github.com/twitter/scalding/wiki/Frequently-asked-questions

  import TestRunMode._

  "You can run on hadoop" should {

    implicit val _ = testOnHadoop

    Given {
      List(("12/02/2013 10:22:11", 1000002l, "http://www.youtube.com")) withSchema INPUT_SCHEMA
    } When {
      pipe: RichPipe => pipe.addDayColumn
    } Then {
      buffer: mutable.Buffer[(String, Long, String, String)] =>
        "add column with day of event" in {
          buffer.toList(0) shouldEqual (("12/02/2013 10:22:11", 1000002l, "http://www.youtube.com", "2013/02/12"))
        }
    }
  }

  "You can run locally" should {

    implicit val _ = testLocally

    Given {
      List(("12/02/2013 10:22:11", 1000002l, "http://www.youtube.com")) withSchema INPUT_SCHEMA
    } When {
      pipe: RichPipe => pipe.addDayColumn
    } Then {
      buffer: mutable.Buffer[(String, Long, String, String)] =>
        "add column with day of event" in {
          buffer.toList(0) shouldEqual (("12/02/2013 10:22:11", 1000002l, "http://www.youtube.com", "2013/02/12"))
        }
    }
  }

  "You can run decide using some configuration parameter or randomly" should {

    implicit val _ = if( (System.currentTimeMillis % 2) == 0) testLocally else testOnHadoop

    Given {
      List(("12/02/2013 10:22:11", 1000002l, "http://www.youtube.com")) withSchema INPUT_SCHEMA
    } When {
      pipe: RichPipe => pipe.addDayColumn
    } Then {
      buffer: mutable.Buffer[(String, Long, String, String)] =>
        "add column with day of event" in {
          buffer.toList(0) shouldEqual (("12/02/2013 10:22:11", 1000002l, "http://www.youtube.com", "2013/02/12"))
        }
    }
  }
}

```

## Content

The repository contains two projects:

 - scalding-unit: the main project, providing the test framework
 - examples: a set of examples to describe the design approach described here

## What's next

At the moment we are covering only the Fields-based API since is the one used in most of the project we are working on.
We are planning to start providing a similar infrastructure for the type-safe API too.
We are also planning to add support for job acceptance validation comparing sample data process against an external
executable specification (such as a R program)

## License

Copyright 2013 PragmaSoft Ltd.

Licensed under the Apache License, Version 2.0: http://www.apache.org/licenses/LICENSE-2.0


