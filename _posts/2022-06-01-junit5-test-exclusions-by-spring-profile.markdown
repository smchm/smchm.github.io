---
layout: post
title:  "JUnit5 test exclusions based on Spring profile names"
date:   2022-06-01 14:30:48 +0100
categories: [java, spring, junit5]
---
To prevent some `JUnit5` tests from running when specific `Spring Profiles` are active simply implement the [ExecutionCondition](https://junit.org/junit5/docs/5.0.0/api/org/junit/jupiter/api/extension/ExecutionCondition.html) interface and interrogate the `spring.profiles.active` system property;

{% highlight java %}
package com.example.project.config;

import org.junit.jupiter.api.extension.ConditionEvaluationResult;
import org.junit.jupiter.api.extension.ExecutionCondition;
import org.junit.jupiter.api.extension.ExtensionContext;

public class DisabledInProductionExtension implements ExecutionCondition {
  @Override
  public ConditionEvaluationResult evaluateExecutionCondition(ExtensionContext extensionContext) {
    final String springProfileKey = "spring.profiles.active";
    if (System.getProperties().containsKey(springProfileKey)) {
      if (System.getProperties().get(springProfileKey).toString().contains("integration-test-production")) {
        return ConditionEvaluationResult.disabled("Test disabled in production environment");
      }
      return ConditionEvaluationResult.enabled("Test enabled in non-production environment");
    }
    return ConditionEvaluationResult.enabled("No profile active, test enabled");
    }
}
{% endhighlight %}

And to use the Extension

{% highlight java %}
package com.example.project;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import com.example.project.config.DisabledInProductionExtension;

@ExtendWith({DisabledInProductionExtension.class})
public class IntegrationTestForNonProductionEnvironment {
}
{% endhighlight %}

To define multiple Extensions for different profiles we can create complimentary `ExecutionCondition` implementations. The framework will chain the invocations *until one evaluates to disabled* and exclude the annotated test accordingly.

{% highlight java %}
package com.example.project;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import com.example.project.config.DisabledInProductionExtension;
import com.example.project.config.DisabledInStagingExtension;

@ExtendWith({DisabledInProductionExtension.class, DisabledInStagingExtension.class})
public class IntegrationTestForDevelopmentEnvironment {
}
{% endhighlight %}

