安装
============

安装
------------


使用 pip_ 从 PyPI_ 安装Motor::

  $ pip install motor

Pip 自动安装 Motor 需要的包, PyMongo_, Greenlet_,
和 Tornado_.

Prerequisites
-------------

* CPython 2.6, 2.7, or 3.3
* Although Motor works with PyPy 2.0-beta1, limitations with greenlets and
  PyPy's JIT compiler make PyPy applications that use Motor too slow for
  regular use
* PyMongo_ 2.5.1 or later
* Tornado_
* Greenlet_

Tests require Nose_ and generating the docs_ requires Sphinx_.

.. _PyPI: http://pypi.python.org/pypi/motor

.. _pip: http://pip-installer.org

.. _PyMongo: https://pypi.python.org/pypi/pymongo/

.. _Tornado: http://www.tornadoweb.org

.. _Greenlet: http://pypi.python.org/pypi/greenlet/

.. _Nose: http://pypi.python.org/pypi/nose/

.. _docs: http://motor.readthedocs.org

.. _Sphinx: http://sphinx-doc.org/
