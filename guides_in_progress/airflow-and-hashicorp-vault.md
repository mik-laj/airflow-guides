---
title: "Integrating Airflow and Hashicorp Vault"
description: "Pull connection information from your Hashicorp Vault to use in your Airflow DAGs."
date: 2020-03-21T00:00:00.000Z
slug: "airflow-and-hashicorp-vault"
tags: ["Connections", "Airflow"]
---

> Note: This guide is beta, as this feature is not included in stable Airflow release yet. We'll walk you through how to use this feature with a test build we've pushed out that includes this feature.

## Overview

A feature we often get asked about here at Astronomer is a native sync with Hashicorp Vault for secrets and connection information. The Airflow community has rallied around the need for more robust sync options from external secrets stores, and one of our very own commiters, [Kaxil Naik](https://www.linkedin.com/in/kaxil?originalSubdomain=uk), has built out a feature to sync connection information from Vault. Below is a step-by-step guide on how to leverage this functionality to import your connection info from your Vault store.

## Pre-Requisites

In this example, we're going to be using [Virtualenvwrapper](https://virtualenvwrapper.readthedocs.io/en/latest/) to manage virtual environments. You can install this package and set up your shell to work nicely with it by following the [instructions in their installation docs](https://virtualenvwrapper.readthedocs.io/en/latest/install.html).

## Setting up the Vault Dev Server

1. **Create a virtual environment.**

        mkvirtualenv test-secrets-backend

    >Note: If you ever need to use this virtual env from a blank terminal window, you can run `workon test-secrets-backend` to re-instantiate it.

2. **Install Airflow and the Hashicorp dependency to your virtual environment.** Note that this is currently pulling a test build that our team at Astronomer has pushed out to allow users to test this feature before it's included in an official Airflow release.

        pip install apache_airflow-1\!1.10.10.dev1+astro.1-py2.py3-none-any.whl 
        pip install hvac

3. **Install Hashicorp Vault using Homebrew.**

        # Option 1. Official (run with no UI)
        brew install vault

        ## Option 2. Vault CLI and GUI (reccomended becuase the Vault UI is a nice feature)
        brew tap petems/vault
        brew install petems/vault-prebuilt/vault

4. **Run the Vault server.**

        vault server -dev

    Now, get the vault token from the output initially printed to your terminal- this should be directly under your `Unseal Key`.

        Root Token: <TOKEN>

    The Vault UI is now accessible at http://127.0.0.1:8200/ui.

5. **Create your Vault Secret.** With your Vault server running, open a new terminal window and run `workon test-backend-secrets` to re-initialize your virtual environment.Then, create a Vault secret:

        vault kv put secret/connections/smtp_default conn_uri=smtps://user:host@relay.example.com:465

    > Note: if you get a `server gave HTTP response to HTTPS client` error, you'll need to export an env var to set the address via `export VAULT_ADDR='http://127.0.0.1:8200'`

    The secret here is referred to as a `mount_point`. Generally, a user might create a separate mount points for each application consuming secrets from Vault.
    To organize our secrets, we specify the `/connection` path and put all Airflow connections in this path.

    > Note: This path is fully configurable and you're more than welcome to change it to suit your needs.

    For the purposes of this example, `smtp_default` is the secret name we're using. You can store arbitrary key/value pairs in this secret. By default, Airflow will look for the `conn_uri` inside the `smtp_default` key.

## Retreiving Connections from Vault

1. **Add an Example DAG.** Add an Example DAG to the Airflow distribution that you've installed to your virtual environment so that you can verify connections are being successfully retrieved from the Vault. Here's one you can use:

        from airflow import DAG
        from airflow.operators.python_operator import PythonOperator
        from datetime import datetime
        from airflow.hooks.base_hook import BaseHook
        ​
        ​
        def get_secrets(**kwargs):
            conn = BaseHook.get_connection(kwargs['my_conn_id'])
            print(f"Password: {conn.password}, Login: {conn.login}, URI: {conn.get_uri()}, Host: {conn.host}")
        ​
        with DAG('zz_example_secrets_dags', start_date=datetime(2020, 1, 1), schedule_interval=None) as dag:
        ​
        
            test_task = PythonOperator(
                task_id='test-task',
                python_callable=get_secrets,
                op_kwargs={'my_conn_id': 'smtp_default'},
            )

    To get this example DAG running in your local Airflow environment, add it to your virtual environment's `airflow/example_dags` folder (ie. `/Users/username/.virtualenvs/test-secrets-backend/lib/python3.7/site-packages/airflow/example_dags/vault_dag`).

2. **Initialize the Airflow database.** Run `airflow initdb` in your virtual environment to initialize the default SQLite DB that ships with stock Airflow.

3. **Create your Scheduler environment.** Open a new terminal window and re-instantiate your virtual env with `workon test-backend-secrets`- this will be your Airflow Scheduler environment.  Export the following env vars in this environment so that your scheduler can access your Vault secrets, then run the scheduler.

        export AIRFLOW__SECRETS__BACKEND="airflow.contrib.secrets.hashicorp_vault.VaultSecrets"
        export AIRFLOW__SECRETS__BACKEND_KWARGS='{"url":"http://127.0.0.1:8200","token":"<YOUR-ROOT-TOKEN>","connections_path": "connections"}'
        airflow scheduler

    The `AIRFLOW__SECRETS__BACKEND` var is the backend we want to use to fetch secrets. The default options are `EnvironmentVariables` and `Metastore`.

    Other available secrets backends include AWS SSM and Google Secrets Manager.

    > Note: Run `export OBJC_DISABLE_INITIALIZE_FORK_SAFETY=YES` in your scheduler environment if python quits unexpectedly while running the DAG.

4. **Create your Webserver Environment** Open a new terminal window and re-instantiate your virtual env with `workon test-backend-secrets`- this will be your Airflow Webserver environment. Spin up the webserver:

            airflow webserver

    You should now be able to access your local Airflow environment at localhost:8080.
        ​

5. **Trigger the DAG and verify that you're getting the expected output.** You can do this by checking for the following output in your task logs:

        [2020-03-24 01:34:44,824] {logging_mixin.py:112} INFO - [2020-03-24 01:34:44,823] {base_hook.py:87} INFO - Using connection to: id: smtp_default. Host: relay.example.com, Port: 465, Schema: , Login: user, Password: XXXXXXXX, extra: None
        [2020-03-24 01:34:44,824] {logging_mixin.py:112} INFO - Password: host, Login: user, URI: smtps://user:host@relay.example.com:465, Host: relay.example.com

## Debugging

If you're running into issues at any step in the process or think this guide could be improved to catch any snags, we'd love to hear from you. Feel free to open issues on our [Airflow Guides repo](https://github.com/astronomer/airflow-guides) directly. If you'd prefer to dive in yourself, here are a few debugging steps that we found helpful when building out and testing this feature.

First, run `ipython` in your terminal to create the shell.

Then, use the Vault Python client directly to check for the correct outputs (shown below).

        In [1]: import hvac                                                                                                  

        In [2]: client=hvac.Client(url="http://127.0.0.1:8200")                                                      

        In [3]: client.token = "s.2hMdO4NDz9SuMuenpvZb2bA7"                                                                  

        In [4]: client.is_authenticated()                                                                                    
        Out[4]: True

        In [5]: client.secrets.kv.v2.read_secret_version(path="connections/smtp_default")                                    
        Out[5]: 
        {'request_id': '04857d45-a960-011c-f1f1-ba4011a1cccc',
        'lease_id': '',
        'renewable': False,
        'lease_duration': 0,
        'data': {'data': {'conn_uri': 'smtps://user:host@relay.example.com:465'},
        'metadata': {'created_time': '2020-03-26T16:09:34.412794Z',
        'deletion_time': '',
        'destroyed': False,
        'version': 1}},
        'wrap_info': None,
        'warnings': None,
        'auth': None}