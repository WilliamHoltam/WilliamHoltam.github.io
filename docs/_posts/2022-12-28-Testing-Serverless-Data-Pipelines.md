---
layout: post
title:  "5 Secrets That Will Make Your Engineering Team Fly"
date:   2022-12-28 19:53
categories: DevOps, Data Engineering
---

As cloud architectures abstract further from the underlying infrastructure upon which the code is running, more testing has to be automated. This is a natural consequence of:

1. Serverless architectures mean that developers don't have the ability to manually SSH onto the machine running the code and test functionality manually as they're developing.
2. A data pipeline which is made up of several different serverless functions/object stores/databases decoupled by queues is not able to be easily tested end to end locally.[^1]
3. Implementing defensive programming practices with the aim of reducing issues caused by human error.[^2]

In order to have a resilient development process with fast development cycles I like to implement the following tests:

1. Unit tests
2. Feature tests
3. Integration tests

_**Note:** as I'm most familiar with AWS, I'll use AWS services for my examples._

## Unit tests

Unit tests test the smallest piece of code which can be isolated in your source code. These are tests which are commonly written when following [Test Driven Development](https://en.wikipedia.org/wiki/Test-driven_development) (TDD). In practice, it means writing tests for a single function or method which can be run locally to verify that it behaves as expected and that when given inputs which are unexpected, the test handles those inputs in a way which makes sense given the context (e.g. raise an exception, raise a warning, log an event).

{% highlight python %}
import pytest


def calculate_energy(m: float) -> float:
    """Einstein's mass-energy equivalence equation.

    This function calculates Einstein's mass-energy
    equivalence equation, e = mc^2 where:
        e: energy
        m: mass
        c: speed of light
    """
    c = 3e8 # approximation of the speed of light
    return m*pow(3e8, 2)


def test_calculate_energy() -> None:
    """Unit test calculate_energy happy path."""
    energy = calculate_energy(m=10)
    assert energy == 9e17
{% endhighlight %}

_**Note:** When developing, as in life, you want to reduce barriers to following best practices and introduce barriers to bad practices. If you follow TDD then you will find that you will naturally implement aspects of functional programming. Since you write the test before you write the implementation, you must conceptually pre-define the scope of your function. This helps to limit the scope of your function to the initial intention and puts mental barriers in the way of one writing functions with side effects._

## Feature Tests

After one has gathered information and functional requirements and developed the various ELT/ETL code to implement the information requirements; one should have a test which runs a single component end to end, mocking any calls to external (often cloud) services, which can't be accessed from the local machine (queue, database, notification, SMTP etc.)[^3]

{% highlight python %}
from unittest.mock import patch

import pandas as pd
import pytest


def extract_object_information(
    event: dict[str, str]
) -> dict[str, str]:
    ...


def read_json_from_s3_into_dict(
    info: dict[str, str]
) -> dict[str, Any]:
    ...
    

def structure_semistructured_data(
    semistructured_data: dict[str, str]
) -> pd.DataFrame:
    ...


def write_data_to_database(
    structured_data: pd.DataFrame
) -> None:
    ...
    

def lambda_handler(event: dict[str, str], context: dict[str, str]) -> None:
    """Serverless component which forms part of this pipeline:
        JSON File -> S3 Bucket -> S3 Notification -> 
        Serverless_Function -> Database
    """
    info = extract_object_information(event)
    unstructured_data = read_json_from_s3_into_dict(info)
    structured_data = structure_json(unstructured_data)
    write_data_to_database(structured_data)


@patch('write_data_to_database')
def test_lambda_handler(mocked_write_data_to_database) -> None:
    """Test the serverless component in question."""
    event = {...}
    expected_result = ...
    lambda_handler(
        event=event, 
        context=dict(),
    )
    assert (
        mocked_write_data_to_database.call_args[0][0] 
        == expected_result
    )
{% endhighlight %}

## Integration Tests

Run your non-production CICD deploy pipeline to deploy your code to a non-production environment; at the end of this deploy pipeline, there should be a stage which runs integration tests. You should have at least one integration test per permutation of possible infrastructure; i.e. if you had a single lambda which could write to 3 different queues or 3 different database tables, you should have at least 3 integration tests. These tests take longer to run than feature tests or unit tests, and cannot be run locally. These tests should not be designed to poke holes in and test edge cases in your codebase, but instead should make sure that your various components can talk to each other, and that data will flow all the way through end to end. Edge cases should ideally be tested at a unit test level, or if required, a feature test level.

{% highlight python %}
import pytest
import backoff


def write_test_data_to_s3() -> None:
    """Write test data to S3 bucket"""
    ...


@backoff.on_exception(backoff.expo, AssertionError, max_time=60)
def read_data_from_rds_table() -> None:
    """Read data from RDS after it's flowed through the pipeline.
    ß
    If the assertion fails, try again using exponential backoff
    until either the assertion passes, or the max_time is reached.
    If the max_time is reached the last AssertionError captured is 
    raised.
    """
    ...


def test_integration():
    "Integration test for pipeline."
    expected_result = ...
    write_test_data_to_s3()
    output = read_data_from_rds_table()
    assert output == expected_result
{% endhighlight %}

I hope you find this post useful!!

## To End

How many programmers does it take to change a light bulb?  
None – It’s a hardware problem!

[^1]: It might be possible to implement local solutions to this using tools such as localstack with some tomfoolery.
[^2]: There are solutions to this, which may or may not be in place, depending on the maturity of your team.
[^3]: It's possible to run a database locally (and some other services), for the purposes of pipeline testing, using docker containers. This is particularly useful if you wish to test ELT logic locally.