---
title: How to run a chunk of code when your Elixir/Phoenix app starts
date: 2017-07-03 12:48:15
tags:
- elixir
- phoenix
- gen_stage
- startup
---

There are ~~two~~ three ways in which you can run some code whenever your Elixir/Phoenix
application starts.

I have actually found there is an easy way to do this using the [Task](https://hexdocs.pm/elixir/Task.html) module

  1. Put the code directly in your `application.ex`

    ```
    defmodule Danny.Application do
      use Application

      # ...
      def start(_type, _args) do
        # ...
        {:ok, pid} = Supervisor.start_link(children, opts)
        # code to run at startup
        MyStartupCode.run
        {:ok, pid}
      end

    end
    ```

  2. Use a `:temporary` worker. Your supervision tree can even contain workers
    which run once and then stop `:normal`ly. This code gives you more control
    over how you can handle failures when the startup code misbehaves. In my app https://sprymesh.com
    I have a cache warmer which is executed on startup. Here is the relevant code:

    ```
    defmodule Danny.Application do
      use Application

      # ...
      def start(_type, _args) do
        children = [
          supervisor(Danny.Repo, []),
          # ...
          worker(CacheWarmer, [], restart: :temporary) # this worker starts, does its thing and dies
          # ...
          ]

        opts = [strategy: :one_for_one, name: Danny.Supervisor]
        Supervisor.start_link(children, opts)
      end

    end

    defmodule CacheWarmer do
      import Logger, only: [debug: 1]

      use GenServer

      def start_link do
        GenServer.start_link(__MODULE__, [:ok], name: __MODULE__)
      end

      def init(state) do
        send(self(), :work)
        {:ok, state}
      end

      def handle_info(:work, state) do
        # warming the caches
        debug "warming the cache"
        # ...
        debug "finished warming the cache, shutting down"
        {:stop, :normal, state}
      end
    end

    ```

    That's all! Now, when your application starts, the supervisor will start the `CacheWarmer` gen server with a restart policy of `:temporary`.
    From the erlang [documenation for supervisors](http://erlang.org/doc/design_principles/sup_princ.html)

    > A temporary child process is never restarted (not even when the supervisor restart strategy is rest_for_one or one_for_all and a sibling death causes the temporary process to be terminated).
    >
    > A transient child process is restarted only if it terminates abnormally, that is, with an exit reason other than normal, shutdown, or `{shutdown,Term}`

    You can use the `:temporary` restart or the `:transient` restart policy based on your needs.

    The other thing to note here is the code for our `CacheWarmer` GenServer.
    It sends itself a message in the `init` callback, so as soon as it starts up it enters the `handle_info(:work, state)` callback.
    This is where you would have the code which would run at startup. And once your code is executed, our `handle_info` callback returns
    a `{:stop, :normal, state}` which signals the GenServer to exit nicely. The supervisor on seeing the CacheWarmer GenServer exit normally accepts that the CacheWarmer worker's time has come and doesn't try to restart it.

  3. Use the [Task](https://hexdocs.pm/elixir/Task.html) module to start a supervised async worker using the following code:

    ```
    defmodule Danny.Application do
      use Application

      # ...
      def start(_type, _args) do
        children = [
          supervisor(Danny.Repo, []),
          # ...
          worker(Task, [&CacheWarmer.warm/0], restart: :temporary) # this worker starts, does its thing and dies
          # ...
          ]

        opts = [strategy: :one_for_one, name: Danny.Supervisor]
        Supervisor.start_link(children, opts)
      end

    end

    defmodule CacheWarmer do
      import Logger, only: [debug: 1]

      def warm do
        # warming the caches
        debug "warming the cache"
        # ...
        debug "finished warming the cache, shutting down"
      end
    end

    ```

    In this version, the only additional code you need to write to is a one liner in your application.ex

    ```
    worker(Task, [&CacheWarmer.warm/0], restart: :temporary) # this worker starts, does its thing and dies
    ```

    This is much simpler, I'll be using this moving forward :)
