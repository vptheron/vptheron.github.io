---
layout: post
title: A tour of Elixir default values
---

I recently started playing seriously with [Elixir](http://elixir-lang.org/), an awesome language running on the [Erlang](https://www.erlang.org/) VM.  I also decided to use [GitLab](https://gitlab.com/) over [GitHub](https://github.com/).  While less "trendy" in the open source community, I find that GitLab offers much more interesting features than GitHub.  One of these features is the built-in continuous integration server [GitLab CI](https://about.gitlab.com/gitlab-ci/).  The server can be configured to automatically kick-off builds of your project upon commits.  It can even automate the deployment of the artifact.  It took me a couple tries to properly set up the configuration file to build and test my Elixir projects.  So I want to share my script here.

To configure the integration job, you need to add a file called `.gitlab-ci.yml` to your directory of your project.  This file describes the tasks to be executed by the GitLab Runner.  You can find the official quick start guide [here](http://doc.gitlab.com/ce/ci/quick_start/README.html).  My configuration can be find below:

{% highlight yml %}
test:
  script:
  - wget https://packages.erlang-solutions.com/erlang-solutions_1.0_all.deb && dpkg -i erlang-solutions_1.0_all.deb
  - apt-get update -qy
  - apt-get install -y elixir
  - mix local.hex --force
  - mix local.rebar --force
  - mix deps.get
  - mix test
{% endhighlight %}

Nothing really complicated for anyone used to working with Elixir, but let's walk through it.  The first three lines install Elixir itself, following the method detailed on the official [website](http://elixir-lang.org/install.html#unix-and-unix-like).  The remaining four lines use [Mix](http://elixir-lang.org/getting-started/mix-otp/introduction-to-mix.html), the prefered Elixir build tool.  We use Mix to install Hex and Rebar, respectively the package manage for the Erlang ecosystem and an Erlang build tool.  We then call `mix deps.get` to retrieve all the dependencies declared in our project build script, and `mix test` to compile and run the tests.

For now, I have not used the deploy stage of GitLab-CI.  An interesting thing to add at this point would be the generation of documentation.  When I get to play with that, I will complete the script with the required instructions and probably take the time to add a new post. 