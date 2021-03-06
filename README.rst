Dockerized Prelude SIEM
=======================

This repository contains a dockerized version of Prelude SIEM.


Requirements
------------

This repository relies on the following dependencies:

* docker.io >= 1.13.1
* docker-compose >= 1.11.0 (optional)

It has been tested on Debian 10.0 (Buster) against the following
versions of these dependencies:

* docker.io 18.09
* docker-compose 1.21.0

In addition, the host should have at least 2 GB of available RAM.


Installation and start/stop instructions
----------------------------------------

Using git and docker-compose
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Clone this repository:

..  sourcecode:: console

    git clone https://github.com/fpoirotte/docker-prelude-siem.git

To start the SIEM, go to the newly created folder and run ``docker-compose``:

..  sourcecode:: console

    cd docker-prelude-siem
    docker-compose up -f docker-compose.yml -f docker-composer.prod.yml \
                      --build --force-recreate --abort-on-container-exit
    # or if "make" is installed on your system, you can just run "make"

``docker-compose`` will recreate the containers, start them and wait for
further instructions.

The following containers will be spawned during this process:

* ``db-alerts``: database server for IDMEF alerts
* ``db-gui``: database server for the user interface (Prewikka)
* ``manager``: Prelude SIEM's manager
* ``correlator``: alert correlator
* ``lml``: log management lackey
* ``prewikka``: web user interface
* ``prewikka-crontab``: periodic scheduler used by prewikka

To stop the SIEM, hit Ctrl+C in the terminal where ``docker-compose``
was run.


Using only docker
~~~~~~~~~~~~~~~~~

Installation using only docker is a little bit more tedious.
The following steps will ensure you get an installation identical to the
``docker-compose`` one presented above. The same container names will be used
as well.

1.  Create the networks:

    ..  sourcecode:: console

        docker network create prelude_alerts
        docker network create prelude_gui
        docker network create prelude_agents

2.  Create the volumes:

    ..  sourcecode:: console

        docker volume create --driver local --name prelude_db-alerts
        docker volume create --driver local --name prelude_db-gui

3.  Create the various containers based on their respective images:

    ..  sourcecode:: console

        docker create \
            -v prelude_db-alerts:/var/lib/pgsql/data:ro \
            -v $(pwd)/secrets/alerts_db:/run/secrets/alerts_db:ro \
            -e POSTGRES_DB=prelude \
            -e POSTGRES_USER=prelude \
            -e POSTGRES_PASSWORD_FILE=/run/secrets/alerts_db \
            --net=none --name prelude_db-alerts_1         postgres:latest

        docker create \
            -v prelude_db-gui:/var/lib/pgsql/data:ro \
            -v $(pwd)/secrets/gui_db:/run/secrets/gui_db:ro \
            -e POSTGRES_DB=prewikka \
            -e POSTGRES_USER=prewikka \
            -e POSTGRES_PASSWORD_FILE=/run/secrets/gui_db \
            --net=none --name prelude_db-gui_1            postgres:latest

        docker create \
            -p 5553:5553 -p 4690:4690 \
            -v $(pwd)/secrets/alerts_db:/run/secrets/alerts_db:ro \
            -v $(pwd)/secrets/sensors:/run/secrets/sensors:ro \
            -e ALERTS_DB_PASSWORD_FILE=/run/secrets/alerts_db \
            -e SENSORS_PASSWORD_FILE=/run/secrets/sensors \
            --net=none --name prelude_manager_1           fpoirotte/prelude-manager

        docker create \
            -v $(pwd)/secrets/alerts_db:/run/secrets/alerts_db:ro \
            -v $(pwd)/secrets/sensors:/run/secrets/sensors:ro \
            -e ALERTS_DB_PASSWORD_FILE=/run/secrets/alerts_db \
            -e SENSORS_PASSWORD_FILE=/run/secrets/sensors \
            --net=none --name prelude_correlator_1        fpoirotte/prelude-correlator

        docker create \
            -p 80:80 \
            -v $(pwd)/secrets/alerts_db:/run/secrets/alerts_db:ro \
            -v $(pwd)/secrets/gui_db:/run/secrets/gui_db:ro \
            -e ALERTS_DB_PASSWORD_FILE=/run/secrets/alerts_db \
            -e GUI_DB_PASSWORD_FILE=/run/secrets/gui_db \
            --net=none --name prelude_prewikka_1          fpoirotte/prewikka

        docker create \
            -v $(pwd)/secrets/alerts_db:/run/secrets/alerts_db:ro \
            -v $(pwd)/secrets/gui_db:/run/secrets/gui_db:ro \
            -e ALERTS_DB_PASSWORD_FILE=/run/secrets/alerts_db \
            -e GUI_DB_PASSWORD_FILE=/run/secrets/gui_db \
            --net=none --name prelude_prewikka-crontab_1  fpoirotte/prewikka-crontab

        # Use the following command to enable the syslog receiver for TCP only.
        # This is recommended for most installations to avoid potential conflicts
        # with the host's own syslog server.
        docker create \
            -p 514:514/tcp \
            -v $(pwd)/secrets/sensors:/run/secrets/sensors:ro \
            -e SENSORS_PASSWORD_FILE=/run/secrets/sensors \
            --net=none --name prelude_lml_1         fpoirotte/prelude-lml

        # Otherwise, use the following command to enable it for both TCP and UDP.
        docker create \
            -p 514:514/tcp -p 514:514/udp \
            -v $(pwd)/secrets/sensors:/run/secrets/sensors:ro \
            -e SENSORS_PASSWORD_FILE=/run/secrets/sensors \
            --net=none --name prelude_lml_1         fpoirotte/prelude-lml

4.  Reconnect the containers to their respective networks:

    ..  sourcecode:: console

        docker network disconnect none prelude_db-alerts_1
        docker network disconnect none prelude_db-gui_1
        docker network disconnect none prelude_manager_1
        docker network disconnect none prelude_correlator_1
        docker network disconnect none prelude_lml_1
        docker network disconnect none prelude_prewikka_1
        docker network disconnect none prelude_prewikka-crontab_1
        docker network connect --alias=db-alerts            prelude_alerts   prelude_db-alerts_1
        docker network connect --alias=manager              prelude_alerts   prelude_manager_1
        docker network connect --alias=correlator           prelude_alerts   prelude_correlator_1
        docker network connect --alias=prewikka             prelude_alerts   prelude_prewikka_1
        docker network connect --alias=prewikka-crontab     prelude_alerts   prelude_prewikka-crontab_1
        docker network connect --alias=db-gui               prelude_gui      prelude_db-gui_1
        docker network connect --alias=prewikka             prelude_gui      prelude_prewikka_1
        docker network connect --alias=prewikka-crontab     prelude_gui      prelude_prewikka-crontab_1
        docker network connect --alias=manager              prelude_agents   prelude_manager_1
        docker network connect --alias=correlator           prelude_agents   prelude_correlator_1
        docker network connect --alias=lml                  prelude_agents   prelude_lml_1

That's it for the installation.

Now, to start the SIEM, run:

..  sourcecode:: console

    docker start prelude_db-alerts_1 prelude_db-gui_1 prelude_manager_1 prelude_correlator_1 prelude_lml_1 prelude_prewikka_1 prelude_prewikka-crontab_1

To stop it, run:

..  sourcecode:: console

    docker stop prelude_prewikka_1 prelude_prewikka-crontab_1 prelude_lml_1 prelude_correlator_1 prelude_manager_1 prelude_db-gui_1 prelude_db-alerts_1


Uninstallation
--------------

Before you install the SIEM, make sure the containers are stopped (see above).
The following commands will remove most objects used by the SIEM,
only leaving behind images related to the base OS (``centos``)
and databases (``centos/postgresql-95-centos7``):

..  sourcecode:: console

    docker          rm  prelude_prewikka_1 prelude_prewikka-crontab_1 prelude_lml_1 prelude_correlator_1 prelude_manager_1 prelude_db-gui_1 prelude_db-alerts_1
    docker network  rm  prelude_agents prelude_alerts prelude_gui
    docker volume   rm  prelude_db-alerts prelude_db-gui
    docker          rmi fpoirotte/prelude-lml fpoirotte/prelude-correlator fpoirotte/prelude-manager fpoirotte/prewikka fpoirotte/prewikka-crontab


Usage
-----

To access the SIEM, open a web browser and go to http://localhost/

To start analyzing syslog entries, send them to port 514 (TCP, unless you
also enabled the UDP port in the configuration file).

You can also use external sensors. In that case, the sensor must first
be registered against this machine (see
https://www.prelude-siem.org/projects/prelude/wiki/InstallingAgentThirdparty
for instructions on how to do that for the most commonly used sensors).
When asked for a password during the registration process, input the
contents from the file at ``secrets/sensors``.

..  note::

    Since the containers are meant to be ephemeral, information about
    the external sensors' registrations is lost when the ``manager``
    container is restarted. You may need to register the sensors again
    in that case.


Exposed services
----------------

The following services get exposed to the host:

* ``514/tcp`` (``lml`` container): syslog receiver

* ``514/udp`` (``lml`` container): syslog receiver (disabled by default
  as it usually conflicts with the host's syslog server)

* ``80/tcp`` (``prewikka`` container): web interface

* ``5553/tcp`` (``manager`` container): sensors' registration server
  (to connect external sensors like Suricata, OSSEC, ...)

* ``4690/tcp`` (``manager`` container): IDMEF alert receiver
  (for external sensors)


Test the SIEM
-------------

To test the SIEM, send syslog entries to ``localhost:514`` (TCP).

For example, the following command will produce a ``Remote Login`` alert
using the predefined rules:

..  sourcecode:: console

    logger --stderr -i -t sshd --tcp --port 514 --priority auth.info --rfc3164 --server localhost Failed password for root from ::1 port 45332 ssh2


Customizations
--------------

Detection rules
~~~~~~~~~~~~~~~

You can customize detection rules by mounting your own folder into the ``lml``
container to use in place of ``/etc/prelude-lml/ruleset/``.
See ``https://github.com/Prelude-SIEM/prelude-lml-rules/tree/master/ruleset``
to get a sense of the contents of this folder.

Correlation rules
~~~~~~~~~~~~~~~~~

You can enable/disable/customize correlation rules by mounting your own folder
containing the rules' configuration files into the ``correlator`` container
in place of ``/etc/prelude-correlator/conf.d/``.


Known caveats
-------------

The following limitations have been observed while using this project:

* The sensors are re-registered every time the containers are restarted,
  meaning new entries get created on the ``Agents`` page every time a
  sensor is restarted.


Developer mode
--------------

In developer mode, the containers will use images which are recreated from
the Dockerfiles contained in the git repository, rather than reusing pre-built
images published on Docker Hub.

This mode is only useful for myself and others who may want to fork this
repository.

To start Prelude SIEM in developer mode, use this command:

..  sourcecode:: console

    make run ENVIRONMENT=dev


License
-------

This project is released under the MIT license.
See `LICENSE`_ for more information.

..  _`LICENSE`:
    https://github.com/fpoirotte/docker-prelude-siem/blob/master/LICENSE
