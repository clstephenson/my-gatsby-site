---
title: Storing Spring Boot Properties in Environment Variables
date: "2019-06-24T22:57:32.169Z"
template: "post"
draft: false
slug: "/posts/spring-boot-properties-in-environment-variables/"
category: "Spring Boot"
tags:
  - "Spring Boot"
description: "Should database credentials be stored in the application.properties configuration file for a Spring Boot application? If not, where should they live? Find out how to easily store Spring Boot properties as OS environment variables."
---

Should database credentials be stored in the application.properties configuration file for a Spring Boot application? If not, where should they live? These were questions I had when starting to learn Spring Boot. The question of storing database credentials can also apply to API keys and other secure access information, and is applicable to software applications in general. Application configuration files like the application.properties file in a Spring Boot application often get checked in to source control along as part of the project. In many cases, like mine, the source is in a public Github repository. Needless to say, it doesn't seem like a good idea to push that information for all to see.

```yaml
# DB Connection Info
spring.datasource.url=url
spring.datasource.username=username
spring.datasource.password=password
```

The above configuration will get the job done, but places confidential information in a properties file that may get pushed to a public repository. Another downside is that there may be (or probably should be) different connection details for various environments, such as development, staging, production, etc.

## Environment Variables to the Rescue!

Using the operating system's environment variables to store certain information is one solution for these issues. The information can be stored on the host where the application is running, not hard-coded into the application config files. Therefore, the information is not part of version control. Also, each application environment can have different values for those environment variables, allowing the application to run seamlessly between environments without needing to change configs or profiles.

Spring Boot makes it very easy to externalize application properties to environment variables. Simply create environment variables for each property to be externalized, and that variable can be reference in the properties file as `${ENV_VAR}`. A default value can also be provided, and will be used if the environment variable doesn't exist. Here are some examples...

```yaml
# DB_USER is the environment variable
spring.datasource.username=${DB_USER}

# Set a default value in case the DB_USER environment variable isn't set
spring.datasource.username=${DB_USER:someUsername}
```

As a bonus, if the environment variable's name is the same as the property, then the property and variable do not even need to be referenced in the application.properties file. Spring Boot will see the environment variable matching the property name. If the operating system disallows periods (.) in environment variable names, then they can be replaced with underscores (\_). For the example shown above, the environment variable could be named `SPRING_DATASOURCE_USERNAME`.

There may be other ways to externalize the configuration data for a Spring Boot application, but using environment variables is pretty simple. For more information, see the Spring Boot documentation for [Externalized Configuration](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-external-config.html).
