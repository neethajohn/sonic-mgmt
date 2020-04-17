# Pytest organization proposal

This proposal intends to achieve the following
  - Have a standard way of categorizing tests
  - Have some guidelines around test file organization
  - Complicated test sequences can be written as single standalone scripts
  - Have a master wrapper for test execution
  - Follow common documentation style
  - Test result collection

## Test categorization
Leverage pytest markers to group tests based on topology, platform and features. Test cases for particular feature can be invoked using '-m featurename' command line option. For platform specific tests and topology groups, we intend to follow the pattern outlined below

```
pytest.ini
[pytest]
markers:
    topology(topo_name): The topologies this particular testcase can run against. topo_name can be individual topology names like 't0', 't1', 'ptf', 'any' or a comma separated like ('t0', 't1') if supported on multiple topologies
    platform(platform_name): used for platform specific test(broadcom, mellanox, vs etc)
    acl: ACL tests
    nat: NAT tests

```
conftest.py

```
def pytest_addoption(parser):
        parser.addoption("--topology", action="store", metavar="NAME",
                        help="only run tests matching the topology NAME")

def pytest_runtest_setup(item):
    toponames = [mark.args for mark in item.iter_markers(name="topology")]
    if toponames:
        if item.config.getoption("--topology") not in toponames[0]:
            pytest.skip("test requires topology in {!r}".format(toponames))
    else:
        if item.config.getoption("--topology"):
            pytest.skip("test does not match topology")

```

Sample test file: test_topo.py

```
@pytest.mark.topology('t0', 't1')
def test_all():
   assert 1 == 1

@pytest.mark.topology('t0')
def test_t0():
   assert 1 == 1


@pytest.mark.topology('any')
def test_any():
   assert 1 == 1

```

Sample test file: test_notopo.py

```
def test_notopo():
   assert 1 == 1

```

Test run

```
py.test --inventory inv --host-pattern dut1 --module-path ../ansible/library/ --testbed tb --testbed_file tb.csv --topology t1 test_topo.py test_notopo.py -rA

platform linux2 -- Python 2.7.12, pytest-4.6.9, py-1.8.1, pluggy-0.13.1
ansible: 2.8.7
rootdir: /var/nejo/Networking-acs-sonic-mgmt/tests, inifile: pytest.ini
plugins: ansible-2.2.2
collected 4 items                                                                                                                                                                                                                       

test_topo.py .ss                                                                                                                                                                                                                  [ 75%]
test_notopo.py s   

....

.... 
PASSED test_topo.py::test_all
SKIPPED [1] /var/nejo/Networking-acs-sonic-mgmt/tests/conftest.py:293: test requires topology in [('t0',)]
SKIPPED [1] /var/nejo/Networking-acs-sonic-mgmt/tests/conftest.py:293: test requires topology in [('any',)]
SKIPPED [1] /var/nejo/Networking-acs-sonic-mgmt/tests/conftest.py:295: test does not match topology

```

## Test file organization
- Feature specific tests and their helpers go into specific feature folders
  
- Any reusable code needs to go under tests/common

- File naming convention


## Complicated test sequences

Currently nightly tests make use of Jenkins templates to piece together individual test scripts to test various flows like upgrade path (upgrading from a set of image versions to another and performing basic sanity after each upgrade). Have a single standalone script for these tasks to enable sharing with community and better test result organization.

## Master wrapper
Make it easier to run a nightly test against a feature/platform/topology from the command line. Have something similar to the 'ansible/testbed-cli.sh' script which can be invoked with just the basic parameters (dut name, testbed name, what flavor of test to run)


## Documentation style 
Follow a common style of documentation for test methods which can be used by some tool to generate html content


## Test result collection
Use the --junitxml attribute to collect test results. Can leverage the existing format used in sonic-utilities/sonic-swss repo for reporting test results.

