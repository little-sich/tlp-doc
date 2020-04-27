Battery Features
================

What are the 'Battery Features'?
--------------------------------
.. include:: ../include/expl-battery-features.rst
.. include:: ../include/disc-battery-features.rst

.. seealso::

    * :doc:`/settings/battery` (Settings)
    * :ref:`cmd-tlp-battery-features` (Commands)

How do I get 'Battery Features' support for my non-ThinkPad Laptop?
-------------------------------------------------------------------
Battery Features require a kernel driver matching your laptop brand or series.
Kernel driver development is not part of the project, TLP just probes active
drivers and provides a unified interface to them.

Thus *your* todos are:

* Reengineer your vendor's battery tool for the preloaded OS and document the
  ACPI calls used for the charge thresholds
* Implement the ACPI calls in a Linux kernel driver using the
  `natacpi framework <https://github.com/linrunner/TLP/issues/321>`_,
  exposing sysfiles `charge_start_threshold` and `charge_stop_threshold`
* Result: TLP supports your laptop out of the box

How to choose good battery charge thresholds?
---------------------------------------------
.. note::

    Newer ThinkPad models may not need charge thresholds due to dualmode battery
    firmware – refer to
    `Lenovo Forums [1] <https://forums.lenovo.com/t5/Windows-10/Power-Manager-for-Windows-10/m-p/2129075#M794>`_
    and `Lenovo Support <https://support.lenovo.com/de/de/solutions/ht104070>`_.

Factory settings for ThinkPad battery thresholds are as follows: when plugged in
the battery starts charging at 96%, and stops at 100%. These settings are optimized
for maximum runtime, but having a battery hold a lot of power will decrease its
capacity over the years. To alleviate this problem, the start/stop charge thresholds
can be adjusted – at the cost of a more or less reduced battery runtime.

It all depends on how you use your laptop, or more precisely, on the minimal
runtime you're ready to accept when you're on the road. In the end, it all comes
down to a runtime vs. lifespan trade-off.

If the laptop is plugged most of the time and rarely unplugged, maximizing battery
lifetime at the cost of a greatly reduced runtime may be acceptable, with values
like starting charge at 40% and stopping at 50%.

On the contrary, if you use it unplugged most of the time, starting charge at
85% and stopping at 90% would allow for a much longer runtime and still give a
lifespan benefit over the factory settings.

Source: `Lenovo Forums [2] <https://forums.lenovo.com/t5/Welcome-FAQs-Knowledge-Base/How-can-I-increase-battery-life-ThinkPad/ta-p/244800>`_

Default TLP settings (only if you uncomment the relevant lines) are slightly more
protective regarding lifespan, with 75/80% charge thresholds.

.. note::

    Please always consider that the start threshold is the critical constraint
    for runtime, because it defines the lowest charge level that can occur while
    plugged. Also, don't forget that TLP provides a command (:command:`tlp fullcharge`)
    to fully charge the battery, when you need to temporarily maximize runtime
    (for example in case of a trip).


.. _faq-which-kernel-module:

Which kernel module do I need for my hardware?
----------------------------------------------
.. note::

    Prerequisite: make shure to install the most recent version of TLP for
    accurate recommendations.

Check the bottom of the output of :command:`tlp-stat -b`, section 'Recommendations',
for the following lines

.. code-block:: none

    Install tp-smapi kernel modules for ThinkPad battery thresholds and recalibration
    Install acpi_call kernel module for ThinkPad battery thresholds and recalibration
    Install acpi_call kernel module for ThinkPad battery recalibration

and install the required external kernel module package as explained in
:doc:`/installation/index` for your distribution.

Almost all ThinkPad models need only one of the above kernel modules. You may
check the output of :command:`tlp-stat -b` for lines like:

.. code-block:: none

    tpacpi-bat = inactive (ThinkPad not supported)
    tp-smapi = inactive (ThinkPad not supported)

and remove the unnecessary module package (`tpacpi-bat` means `acpi_call`).
However it doesn't really hurt to keep both.

.. rubric:: natacpi – Ultimate solution at the horizon

.. note::

    The following requires at least version 1.2.2.

Starting with kernel 4.17 `tpacpi-bat` gets superseded by a new, native kernel
API called `natacpi` (contained in the ubiquitious kernel module `thinkpad_acpi`).
:command:`tlp-stat -b` indicates this as follows:

.. code-block:: none

    +++ Battery Features
    natacpi = active (data, thresholds)
    tpacpi-bat = active (recalibrate)
    tp-smapi = inactive (ThinkPad not supported)

As of kernel 5.5 the patches for discharge/recalibrate haven't been merged.
The full implementation will look like this:

.. code-block:: none

    +++ Battery Features
    natacpi = active (data, thresholds, recalibrate)
    tpacpi-bat = inactive (superseded by natacpi)
    tp-smapi = inactive (ThinkPad not supported)

With full `natacpi` support, you won't need external kernel module packages anymore.

.. _faq-thresholds-ignored:

Why is my battery charged up to 100% – ignoring the charge thresholds?
----------------------------------------------------------------------
Possible causes are:

Laptop is not a ThinkPad
^^^^^^^^^^^^^^^^^^^^^^^^
Battery charge thresholds and recalibration work with ThinkPads only (see above).

External kernel module is not installed
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Symptom: :command:`tlp-stat -b` shows

.. code-block:: none

    tpacpi-bat = inactive (kernel module 'acpi_call' not installed)

or

.. code-block:: none

    tp-smapi = inactive (kernel module 'tp_smapi' not installed)

Solution: read :ref:`faq-which-kernel-module` and install the necessary packages.
If :command:`tlp-stat -b` still claims 'not installed' after installing
the appropriate package, reinstall the package via shell command and check
the output for errors. See below for possible causes.

Fedora release upgrade
^^^^^^^^^^^^^^^^^^^^^^
It may be necessary to rebuild the kernel modules (as root): ::

    akmods --force

Installation of package `acpi-call-dkms` failed
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Debian derivatives with kernel >= 4.12 will need at least package version 1.1.0-4
(from Debian Buster).

Kernel module `acpi-call` is not loaded
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Symptom: :command:`tlp-stat -b` shows

.. code-block:: none

    tpacpi-bat = inactive (kernel module 'acpi_call' load error)

Solution: try to load manually with

.. code-block:: sh

    sudo modprobe -v acpi_call

and use adequate forums to resolve your issue with `acpi-call`.

.. note::

    You may need to disable Secure Boot when `acpi-call` refuses to load.
    Check your distribution's :doc:`/installation/index` instructions.

Installation of package `tp-smapi-dkms` failed
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Symptom (Ubuntu): package install shows

.. code-block:: none

    Setting up tp-smapi-dkms (0.41-1) ...
    Creating symlink /var/lib/dkms/tp-smapi/0.41/source ->
    /usr/src/tp-smapi-0.41
    DKMS: add completed.
    Error! Your kernel headers for kernel 3.X.0-YY-generic cannot be found.
    Please install the linux-headers-3.X.0-YY-generic package,
    or use the --kernelsourcedir option to tell DKMS where it's located

Solution: install package **linux-generic-headers**.

Symptom (Ubuntu 16.04 HWE kernel 4.8): package install shows

.. code-block:: none

    Setting up tp-smapi-dkms (0.41-1) ...
    make KERNELRELEASE=4.8.0-46-generic -C /lib/modules/4.8.0-46-generic/build M=/var/lib/dkms/tp-smapi/0.41/build....(bad exit status: 2)
    Error! Bad return status for module build on kernel: 4.8.0-46-generic (x86_64)

Solution: either enable the TLP PPA (see :doc:`/installation/ubuntu`) and update
your packages (recommended) or download version
`0.43-1 from Focal <https://packages.ubuntu.com/focal/tp-smapi-dkms>`_
and install it manually.

Kernel module `tp-smapi` is not loaded
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Symptom: :command:`tlp-stat -b` shows

.. code-block:: none

    tp-smapi = inactive (kernel module 'tp_smapi' load error)

Solution: try to load manually with

.. code-block:: sh

    sudo modprobe -v tp_smapi

and check `tp-smapi Troubleshooting <http://www.thinkwiki.org/wiki/Tp_smapi#Troubleshooting>`_
for a solution matching the error message or use adequate forums to resolve your
issue with `tp-smapi`.

.. note::

    * You may need to disable Secure Boot when `tp-smapi` refuses to load,
      check your distribution's :doc:`/installation/index` instructions
    * `Libreboot` does not support `tp-smapi`
    * `tp-smapi` does not support newer models, check :ref:`faq-which-kernel-module`

ThinkPad T420(s)/T520/W520/X220
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
`tp-smapi` doesn't support start threshold and recalibration on Sandy Bridge
generation ThinkPads. Symptoms are:

:command:`tlp-stat -b` shows

.. code-block:: none

    /sys/devices/platform/smapi/BAT0/start_charge_thresh = (not available)

:command:`tlp setcharge/tlp fullcharge` show the message

.. code-block:: none

    start => Warning: cannot set threshold.

:command:`tlp discharge/recalibrate` show the message

.. code-block:: none

    Error: discharge function not available for this laptop.

Solutions:

* Install a kernel ≥ 4.19 to make natacpi available
* TLP automatically uses `tpacpi-bat` when the kernel module `acpi_call` is
  available, see :ref:`faq-which-kernel-module`

ThinkPad T430(s)/T530/W530/X230 (and all later models)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Solutions:

* Install a kernel ≥ 4.19 to make natacpi available
* TLP automatically uses `tpacpi-bat` when the kernel module `acpi_call` is
  available, see :ref:`faq-which-kernel-module`

ThinkPad E495
^^^^^^^^^^^^^
Symptom: thresholds have been written to the Embedded Controller (EC),
:command:`tlp-stat -b` reads them back as configured
(see `Issue #454 <https://github.com/linrunner/TLP/issues/454>`_):

.. code-block:: none

    /sys/class/power_supply/BAT0/charge_start_threshold = 45 [%]
    /sys/class/power_supply/BAT0/charge_stop_threshold = 60 [%]

Yet they do not work.

Cause: bug in Lenovo's Embedded Controller (EC) firmware.

Workaround: remove thresholds completely with

.. code-block:: sh

    sudo tlp fullcharge

then continue with the workaround for :ref:`faq-disabling-thresholds-does-not-work`.

ThinkPad L420/520, SL300/400/500, X121e
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
These models are neither supported by `tp-smapi` nor by `tpacpi-bat` or `natacpi`.
Please refrain from opening issues for this.

Battery has been removed
^^^^^^^^^^^^^^^^^^^^^^^^
By removing (and re-inserting) the battery the charge thresholds are reset to
factory settings (96 / 100%) for some models. To restore TLP's settings use

.. code-block:: sh

    sudo tlp setcharge

(see :ref:`cmd-tlp-battery-features`) or

* Restart system
* Shutdown and power off system

.. _faq-wrong-threshold-values:

Charge thresholds shown by `tlp-stat -b` do not correspond to the configured ones
---------------------------------------------------------------------------------
Possible causes are:

.. rubric:: Configuration was not activated

After changes to the configuration it is necessary to reboot. Alternatively use

.. code-block:: sh

    sudo tlp start

or

.. code-block:: sh

    sudo tlp setcharge

to activate the thresholds.

.. rubric:: ThinkPad Edge, E / L / S series, SL410/510, Yoga series

On these models the threshold values shown by :command:`tlp-stat -b` do not
correspond to the written values. For example the setting `START_CHARGE_THRESH_BATx=75`
/ `STOP_CHARGE_THRESH_BATx=80` shows 75 / 74. The described behavior is caused
by the firmware (UEFI/BIOS), not by TLP. Nonetheless the charge thresholds work
as configured.

.. _faq-start-thresholds-does-not-apply:

Start threshold does not apply after change
-------------------------------------------
Affected hardware: ThinkPads X240, Yoga 12 (based on user feedback)

Workaround: activating a new start threshold may require to discharge the battery
below the old start threshold after writing the new threshold, i.e. via
:command:`tlp setcharge` or reboot
(see `Issue #173 <https://github.com/linrunner/TLP/issues/183#issuecomment-175228097>`_).

.. _faq-charging-stops-at-80:

`tlp fullcharge BAT1` stops at ~80%
------------------------------------
Affected hardware: ThinkPad T440s (based on user feedback)

Symptom: despite the stop threshold is set to 100% either by configuration or by
:command:`tlp fullcharge/setcharge`, charging of BAT1 stops at around 80%.

Cause: this is hardcoded into Lenovo's embedded controller (EC) firmware.
After BAT1 reaches 80% charging commences with BAT0 until 100%, afterwards BAT1
continues until 100%. If a stop threshold is set for BAT0, the last step may never
happen.

No solution: Lenovo's firmware offers no possibilty to change the behaviour.

.. _faq-erratic-battery-behavior:

Erratic battery behavior on ThinkPad T420(s)/T520/W520/X220 (and all later models)
----------------------------------------------------------------------------------
Symptom: some users report severely reduced battery capacity or sudden drops of
the charge level from around 30% to zero when employing charge thresholds.

Probable cause: conflict with dualmode battery firmware.

Solution: remove battery thresholds completely or use only the start threshold;
then recalibrate battery once.

.. note::

    This is a software only issue, no harm is done to the battery.

.. _faq-panel-applet-soc:

In contrast, for ThinkPad models supporting `tp-smapi` :command:`tlp-stat -b`
shows the correct state in **/sys/devices/platform/smapi/BATx/state**.

Do charge thresholds work even when TLP is not running or the laptop is powered off?
------------------------------------------------------------------------------------
Yes. For ThinkPads the charging process is not controlled by software running on
the operating system but by the embedded controller (EC) firmware. TLP just writes
the thresholds to the corresponding EC registers (via `tp-smapi`, `tpacpi-bat` or
`natacpi`). Once stored the charge thresholds stay effective permanently.
See below for removal.

.. _faq-start-threshold:

What exactly does the start charge threshold `START_CHARGE_THRESH_BATx` do?
---------------------------------------------------------------------------
The start charge threshold ensures that the battery is not recharged immediately
after every short discharge process. The charging process starts only when the
previous discharge was below the value of `START_CHARGE_THRESH_BATx`.

How to designate the battery to discharge when battery powered?
---------------------------------------------------------------
ThinkPad battery balancing i.e. selecting the active battery for
models with more than one battery is not possible because the Lenovo
firmware offers no way to do this, neither manual nor automatic.

.. _faq-discharging-misconception:

Why does the battery not start to discharge when the stop threshold is reached during charging?
-----------------------------------------------------------------------------------------------
*Author's remark: sometimes users trap into this misconception without me having
understood how it happens. This is the attempt to lead them out again.*

The task of the stop threshold is to reduce battery wear by limiting the charge
level below 100%. So charging stops at the threshold value and the battery will
not be discharged as long as the charger remains connected.

This is the behaviour intended by the manufacturer. It is hard-coded into the EC
firmware (see above) and behaves identically for the pre-loaded OS.

In contrast, repeated discharge of the battery during operation on AC power,
when it has reached the desired maximum state of charge, would lead to absurdly
high wear (i.e. charging cycles), without any benefit being derived from it.

Can I prevent discharging the battery by setting the start threshold?
---------------------------------------------------------------------
No. Discharging the battery can be prevented only by connecting the charger
or switching off your ThinkPad.

.. _faq-how-to-disable-thresholds:

How do I disable the charge thresholds?
---------------------------------------
Remove the charge thresholds from the configuration by inserting a leading `#`

.. code-block:: none

    #START_CHARGE_THRESH_BAT0=75
    #STOP_CHARGE_THRESH_BAT0=80

and use

.. code-block:: sh

    sudo tlp fullcharge

to immediately activate the factory settings 96/100%.

.. _faq-disabling-thresholds-does-not-work:

Disabling the charge thresholds does not work
----------------------------------------------
Affected hardware: ThinkPads E580, T480s, X1 Carbon 6th (based on user feedback)

Symptom: after resetting the thresholds as described above, :command:`tlp-stat -b`
shows the stop threshold unchanged.

Cause: after applying a stop threshold value < 100, Lenovo's embedded controller
(EC) firmware will not accept values higher than the previously set value.

Solution: update EC firmware (contained in BIOS update)

* T480s: ECP 1.13 (BIOS 1.31)
* X1 Carbon 6th: ECP 1.12 (BIOS 1.37)

Workaround (without BIOS update):

* Disable the threshold configuration as decribed above
* Power off the laptop via shutdown
* Unplug AC power
* Power on the laptop
* At the Lenovo logo: press :kbd:`Enter`, :kbd:`F1` to enter the BIOS setup
* Go to: `Config → Power → Disable Built-in Battery`, :kbd:`Enter`, :kbd:`Y`
  laptop will power off
* Connect AC
* Power on laptop
* Check with :command:`tlp-stat -b` – thresholds should be at factory settings
  96/100% now
* When unsuccessful, repeat the whole procedure

My battery does not charge anymore after recalibration showing X% remaining capacity constantly
-----------------------------------------------------------------------------------------------
Most probable cause: battery is defect – and was it even before the recalibration
attempt.

.. _faq-cycle-count:

Why does the panel applet show the battery state 'charging' despite charge thresholds are effective?
----------------------------------------------------------------------------------------------------
Existing panel applets query `upowerd` or the standard kernel interface which do
not reflect the charging condition correctly as soon as charge thresholds intervene.
In this situation :command:`tlp-stat -b` shows 'Unknown (threshold effective)' for
**/sys/class/power_supply/BATx/status**. There is no solution at the moment.

Why does `tlp-stat -b` display 'cycle_count = (not supported)'?
---------------------------------------------------------------
Cycle count is not available for all laptops. Positive exceptions are older
ThinkPads supporting `tp-smapi` and some newer hardware.