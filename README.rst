micropipenv example for OpenShift's Python s2i
----------------------------------------------


This repository contains all the materials needed to test and demo `micropipenv
<https://github.com/thoth-station/micropipenv>`_ in OpenShift's s2i container
image build process.

The application consists of a simple Flask server that is built and deployed
into OpenShift.

Installation instructions
=========================

To install this application into your OpenShift namespace, run the following
command:

.. code-block:: console

   oc project <YOUR-NAMESPACE-GOES-HERE>
   oc process -f https://raw.githubusercontent.com/fridex/s2i-example-micropipenv/master/openshift.yaml | oc apply -f -

After triggering the given command the application named
"s2i-example-micropipenv" should be built in your namespace.

micropipenv lock file discovery
===============================

Check `micropipenv documentation
<https://github.com/thoth-station/micropipenv>`_ for the automatic lock file
discovery.  micropipenv looks for files describing dependencies in the
following order:

1. Pipenv files - ``Pipfile.lock`` and ``Pipfile``
2. Poetry files - ``poetry.lock`` and ``pyproject.toml``
3. requirements.txt files for ``pip-tools`` style requirements or raw requirements.txt

As micropipenv unifies build log output across all the dependency managers, it
might be not directly visible what file is used (DEBUG mode can be helpful for
this case). To distinguish which file was used as a source for requirements,
this demo pins different versions of Flask for different cases.

Demo Pipenv support
===================

To test Python s2i with Pipenv functionality, run application without any
changes (Pipenv has the highest priority). Based on the lock file, you should
be able to see Flask in version 1.0.1 installed and the application should be
running.

.. figure:: https://raw.githubusercontent.com/fridex/s2i-example-micropipenv/master/fig/pipfile_lock.png
   :alt: Build log screen shot (part of it).
   :align: center

Demo Poetry support
===================

Now, let's rename our ``Pipfile.lock`` so micropipenv does not find it and
fallbacks to Poetry style requirements:

.. code-block:: console

  mv Pipfile.lock Pipfile.lock_bac

Let's start the build and use the content of the local directory as a package
source (to avoid committing to Git):

.. code-block:: console

  oc start-build --from-dir=. s2i-example-micropipenv

As Poetry style requirements are used, you should be able to see Flask in
version 1.0.3 being installad. The Python s2i build process prints the
installed stack onto standard error output (to unify logs, ``Pipfile.lock``
format is used). Mind the logs are identical with the scenario before where we
used Pipenv (the difference is in Flask version).

Demo pip-tools support
======================

Let's perform similar operation as before to let micropipenv install
dependencies from ``requirements.txt`` that have lowest priority:


.. code-block:: console

  mv poetry.lock poetry.lock_bac

And start the build process once again:

.. code-block:: console

  oc start-build --from-dir=. s2i-example-micropipenv

The ``requirements.txt`` file pins down Flask to version 1.1.1. You should see
this version of Flask being installed in the OpenShift's build logs.

Note the ``requirements.txt`` file was produced using ``pip-tools`` with
``--require-hashes`` being set. Note the build log has identical structure as
in the previous Pipenv and Poetry scenarios (the difference is in Flask version).

Demo pip support
================

Now, let's demo the last use case - ``requirements.txt`` does not state full
stack of Python libraries that should be installed. This is a bad practice as
the application deployed hasn't all stated dependencies. The packages
installed can be different if you trigger the build tomorrow or one year later.
It's also harder for maintainers (assuming source code only access) to track
down what packages the developer used to run the application.

.. code-block:: console

  mv requirements.txt requirements.txt{,_bac}
  mv requirements_unpinned.txt requirements.txt

After triggering the build, you should see a big warning about unpinned Python
application stack produced:

.. code-block:: console

  oc start-build --from-dir=. s2i-example-micropipenv

To notify Python s2i user what packages are installed, micropipenv populates
``pip freeze`` output to Python s2i build logs.

.. figure:: https://raw.githubusercontent.com/fridex/s2i-example-micropipenv/master/fig/requirements_unpinned.png
   :alt: Warning produced when Python requirements are not pinned down.
   :align: center

Explictly specifying requirements file
======================================

You can also provide ``MICROPIPENV_METHOD`` environment variable to the
OpenShift build process to make sure your desired installation method is used.
Refer to micropipenv docs for more info. See also ``openshift.yaml`` and
comments in the build configuration.

micropipenv vs Pipenv size
==========================

Let's compare size needed for installing Pipenv and micropipenv (size and
software shipped to prod matters) - the test was done on Fedora 31 with Python
3.7:

.. code-block:: console

  $ python3 -m venv venv_pipenv
  $ . venv_pipenv/bin/activate
  $ du -s -B1 venv_pipenv
  11210752	venv_pipenv
  $ pip3 install pipenv
  $ du -s -B1 venv_pipenv
  42049536	venv_pipenv
  $ deactivate


And now with micropipenv:

.. code-block:: console

  $ python3 -m venv venv_micropipenv
  $ . venv_micropipenv/bin/activate
  $ du -s -B1 venv_micropipenv
  11210752      venv_micropipenv
  $ pip3 install 'micropipenv[toml]'
  $ du -s -B1 venv_micropipenv
  11595776      venv_micropipenv
  $ deactivate

That's 30453760 bytes (30.4 Megabyte) difference!

Clean up
========

To clean up your name space after the demo, you can run the following command:


.. code-block:: console

  $ oc delete -l app=s2i-example-micropipenv bc,is,dc,route,svc
