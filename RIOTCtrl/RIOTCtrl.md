class: center, middle

# RIOTCtrl

## RIOT Summit 2021

---

# What where we doing before?

- [testrunner](https://github.com/RIOT-OS/RIOT/blob/master/dist/pythonlibs/testrunner/spawn.py)

- [releasetest](https://github.com/RIOT-OS/Release-Specs/pull/79)

- [riot-coap-pytest](https://github.com/kb2ma/riot-coap-pytest)

- [RobotFW-tests](https://github.com/RIOT-OS/RobotFW-tests)

- [iotlab_controller](https://github.com/miri64/iotlab_controller/blob/master/iotlab_controller/riot.py)

- and others...

---

# What were we doing before? (e.g.)

- flashing the application `$ make flash`

- writing a test script trying to match our shell commands...

```python
# code extacted from tests/shell/tests/01-run.py

def check_startup(child):
    child.sendline(CONTROL_C)
    child.expect_exact(PROMPT

def testfunc(child):
    # avoid sending an extra empty line on native.
    if BOARD == 'native':
        child.crlf = '\n'

    # loop other defined commands and expected output
    for cmd, expected in CMDS:
        check_cmd(child, cmd, expected)
```

- running the test `$ make test`

- shell commands changed -> scripts broken

---

# What were we trying to do?

- automate experiments

- automate testing scripts

- handle multiple RIOT devices

- Control read-eval-print-loops interfaces

---

# RIOTCtrl Base Class

```python
from riotctrl.ctrl import RIOTCtrl

env = {'BOARD': 'native'}
# if not running from the application directory the a path must be provided
ctrl = RIOTCtrl(env=env, application_directory='.')
# flash the application
ctrl.make_run(['flash'])
# run the terminal through a contextmanager
with ctrl.run_term():
    ctrl.term.expect('>')       # wait for shell to start
    ctrl.term.sendline("help")  # send the help command
    ctrl.term.expect('>')       # wait for the command result to finnish
    print(ctrl.term.before)     # print the command result
# run without a contextmanager
ctrl.start_term()               # start a serial terminal
ctrl.term.sendline("help")      # send the help command
ctrl.term.expect('>')           # wait for the command result to finnish
print(ctrl.term.before)         # print the command result
ctrl.stop_term()                # close the terminal
```

---

# ShellInteractions

```python
class Help(ShellInteraction):
    """Help ShellInteraction"""
    @ShellInteraction.check_term
    def help(self, timeout=-1, async_=False):
        """Sends the reboot command via the terminal"""
        return self.cmd("help", timeout, async_)
```

```python
from riotctrl.ctrl import RIOTCtrl
from riotctrl.shell import ShellInteraction

env = {'BOARD': 'native'}
# if not running from the application directory the a path must be provided
ctrl = RIOTCtrl(env=env, application_directory='.')
# flash the application
ctrl.flash()                     # alias for ctrl.make_run(['flash'])
# shell interaction instance
shell = ShellInteraction(ctrl)
shell.start_term()               # start a serial terminal
print(shell.cmd("help"))         # print the command result
shell.stop_term()                # close the terminal
```

---

# ShellInteractions

```python
from riotctrl.ctrl import RIOTCtrl
from riotctrl_shell.sys import Help

env = {'BOARD': 'native'}
# if not running from the application directory the a path must be provided
ctrl = RIOTCtrl(env=env, application_directory='.')
# flash the application
ctrl.flash()                     # alias for ctrl.make_run(['flash'])
# shell interaction instance, Help uses the @ShellInteraction.check_term
# decorator, it will start the terminal if its not yet running, and close
# it after the command ends
shell = Help(ctrl)              #
print(shell.help())             # print the command result
```

---

# Writing SAUL ShellInteraction

```bash
> saul help
saul help
usage: saul read|write
> saul read
saul read
usage: saul read <device id>|all
> saul write
saul write
usage: saul write <device id> <value 0> [<value 1> [<value 2]]
> saul
saul
ID    Class     Name
#0    SENSE_XX  x1
#1    SENSE_XX  x2
```

---

# Writing SAUL ShellInteraction

```python
from riotctrl.shell import ShellInteraction


class SaulShell(ShellInteraction):
    @ShellInteraction.check_term
    def saul_read(self, timeout=-1, async_=False):
        return self.cmd("saul", timeout, async_)
```


```python
from riotctrl.shell import ShellInteraction


class SaulShell(ShellInteraction):
    @ShellInteraction.check_term
    def saul_cmd(self, args=None, timeout=-1, async_=False):
        cmd = "saul"
        if args is not None:
            cmd += " {args}".format(args=" ".join(str(a) for a in args))
        return self.cmd(cmd, timeout=timeout, async_=False)

    def saul_read(self, dev_id="all", timeout=-1, async_=False):
        return self.saul_cmd(args=("read", f"{dev_id}",),
                             timeout=timeout, async_=async_)

    def saul_help(self, timeout=-1, async_=False):
        return self.saul_cmd(args=("help",), timeout=timeout, async_=async_)
```

---

# Writing SAUL ShellInteraction
## Parsing SAUL Interaction Results

```python
import re
from riotctrl.shell import ShellInteractionParser


class SaulShellCmdParser(ShellInteractionParser):
    pattern = re.compile(
        r"#(?P<id>\d+)\s*(?P<class>SENSE_[^\s]*)\s+(?P<name>[^\s].*)$")

    def parse(self, cmd_output):
        devices = None
        for line in cmd_output.splitlines():
            m = self.pattern.search(line)
            if m is not None:
                print("match")
                if devices is None:
                    devices = {}
                devices[m.group("id")] = {"class": m.group("class"),
                                          "name": m.group("name")}
        return devices
```

---

# Writing SAUL ShellInteraction
## Parsing SAUL Interaction Results

```python
env = {'BOARD': 'native'}
# if not running from the application directory the a path must be provided
ctrl = RIOTCtrl(env=env, application_directory='.')
# flash the application
ctrl.flash()                     # alias for ctrl.make_run(['flash'])
# shell interaction instance
shell = SaulShell(ctrl)
with ctrl.run_term():
    parser = SaulShellCmdParser()
    print(parser.parse(shell.saul_cmd()))
# > {'0': {'class': 'SENSE_XX', 'name': 'x2'},
#    '1': {'class': 'SENSE_XX', 'name': 'x1'}}
```

---

# Interacting with multiple RIOT devices

- [FIT IoT-LAB]:

```python
# first device using dwm1001-1 on the saclay site
env1 = {'BOARD': 'dwm10001', 'IOTLAB_ctrl': 'dwm1001-1.saclay.iot-lab.info'}
ctrl1 = RIOTCtrl(env=env1, application_directory='.')
# second device using dwm1001-2 on the saclay site
env2 = {'BOARD': 'dwm10001', 'IOTLAB_ctrl': 'dwm1001-2.saclay.iot-lab.info'}
ctrl2 = RIOTCtrl(env=env2, application_directory='.')
```

- locally:

```python
# first samr21-xpro
env1 = {'BOARD': 'samr21-xpro', 'DEBUG_ADAPTER_ID': 'ATML2127031800004957'}
ctrl1 = RIOTCtrl(env=env1, application_directory='.')
# second samr21-xpro
env2 = {'BOARD': 'samr21-xpro', 'DEBUG_ADAPTER_ID': 'ATML2127031800011458'}
ctrl2 = RIOTCtrl(env=env2, application_directory='.')
```

---

# Factories

```python
from contextlib import ContextDecorator
from riotctrl.ctrl import RIOTCtrl, RIOTCtrlBoardFactory
from riotctrl_ctrl import native

class RIOTCtrlAppFactory(RIOTCtrlBoardFactory, ContextDecorator):

    def __init__(self):
        super().__init__(board_cls={'native': native.NativeRIOTCtrl, })
        self.ctrl_list = list()

    def __enter__(self):
        return self

    def __exit__(self, *exc):
        for ctrl in self.ctrl_list:
            ctrl.stop_term()

    def get_ctrl(self, application_directory='.', env=None):
        # retrieve a RIOTCtrl Object
        ctrl = super().get_ctrl(
            env=env,
            application_directory=application_directory
        )
        self.ctrl_list.append(ctrl)    # append ctrl to list
        ctrl.flash()                   # flash
        ctrl.start_term()              # start terminal
        return ctrl                    # return ctrl with started terminal
```

---

# GNRC Networking example native

```python
from riotctrl_shell.gnrc import GNRCICMPv6Echo, GNRCICMPv6EchoParser
from riotctrl_shell.netif import Ifconfig


class Shell(ifconfig, GNRCICMPv6Echo):
  pass


with RIOTCtrlAppFactory() as factory:
    # Createa two native instances, specifying the tap inretace
    native_0 = factory.get_ctrl(env={'BOARD':'native', 'PORT':'tap0'})
    native_1 = factory.get_ctrl(env={'BOARD':'native', 'PORT':'tap1'})
    # `NativeRIOTCtrl` allows for `make reset` with `native`
    native_0.reset()
    native_1.reset()
    # Perform a multicast ping and parse results
    pinger = Shell(native_0)
    parser = GNRCICMPv6EchoParser()
    result = parser.parse(pinger.ping6("ff02::1"))
    assert result['stats']['packet_loss'] < 10    # assert packetloss is under 10%
    assert result['stats']['rx'] > 0              # assert at least one responde
```

---

# Support Tools

- IoT-LAB helpers providing a series of devices (see [utils/iotlab.py])

```python
@contextmanager
def get_iotlab_experiment_ctrls(boards, name='my-experiment', site='saclay'):
    envs = list()
    # populate environments based on provided boards
    for board in boards:
        if IoTLABExperiment.valid_board(board):
            env = {'BOARD': '{}'.format(board)}
        else:
            env = {'IOTLAB_ctrl': '{}'.format(board)}
        envs.append(env)
    exp = IoTLABExperiment(name=name, envs=envs, site=site)
    try:
        exp.start(duration=IOTLAB_EXPERIMENT_DURATION)
        yield exp, envs
    finally:
        exp.stop()
```

- others?

---

# Where is it being used?

- experiment-scripting: [riot-demo-apps]

- pytest: [ReleaseSpecs]

- unittests: [tests/turo], [tests/congure_test]

---

# TLDR

=> use common base

=> makes testing easy(er)

=> script complex RPC scenarios through ShellInteraction

=> start using it! we need more automation for easy maintenance!

[RIOTCtrl]: https://github.com/RIOT-OS/riotctrl
[FIT IoT-LAB]: https://www.iot-lab.info/
[multiple-boards-udev]: https://api.riot-os.org/advanced-build-system-tricks.html#multiple-boards-udev
[ReleaseSpecs]: https://github.com/RIOT-OS/Release-Specs
[tests/turo]: https://github.com/RIOT-OS/RIOT/blob/master/tests/turo/tests/01-run.py
[tests/congure_test]: https://github.com/RIOT-OS/RIOT/blob/master/tests/congure_test/tests/01-run.py
[04-single-hop-6lowpan-icmp]: https://github.com/RIOT-OS/Release-Specs/blob/master/04-single-hop-6lowpan-icmp/test_spec04.py
[utils/iotlab.py]: https://gitlab.inria.fr/pepper/riot-demo-apps/-/blob/master/dist/tools/utils/iotlab.py
[testutils/iotlab.py]: https://github.com/RIOT-OS/Release-Specs/blob/master/testutils/iotlab.py
[riot-demo-apps]: https://gitlab.inria.fr/pepper/riot-demo-apps/-/tree/master/dist/tools
