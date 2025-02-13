(howto-install-appliance-github)=
# How to install the appliance within a GitHub Action workflow

For use within a [GitHub Action workflow](https://github.com/features/actions), a simple action named [`canonical/anbox-cloud-github-action`](https://github.com/canonical/anbox-cloud-github-action) is available. This action can be easily integrated into existing workflows and it will install and configure the Anbox Cloud Appliance for direct use on a GitHub runner.

## Prerequisites

* A repository on GitHub which can host GitHub Action workflows
* The token for your Ubuntu Pro subscription

## Create a new workflow

One example of where this GitHub Action could be helpful is when setting up a new workflow to run integration tests for an Android application. In this case, the action can be used as one of the first steps before the actual test is executed. The example workflow below implements the following steps:

1. Set up Anbox Cloud
2. Create a new Android 13 instance and configure it to allow external ADB access
3. Connect the instance to ADB running on the host of the runner
4. Perform a simple test to check the `ro.product.model` property for a specified value

The workflow runs only on `push` trigger because access to a repository secret is required, which should not be granted on pull requests for security reasons. See [here](https://securitylab.github.com/resources/github-actions-preventing-pwn-requests/) for more details.

```yaml
name: Run integration tests
on: push
jobs:
  run-tests:
    runs-on: ubuntu-latest
    steps:
    - name: Setup Anbox Cloud
      uses: canonical/anbox-cloud-github-action@main
      with:
        channel: 1.23/stable
        ubuntu-pro-token: ${{ secrets.UBUNTU_PRO_TOKEN }}
    - name: Create Android instance
      id: create-instance
      run: |
        set -x
        id="$(amc launch -r -s adb jammy:android13:amd64)"
        amc wait -c status=running "$id"
        echo "id=$id" >> "$GITHUB_OUTPUT"
    - name: Access Android over ADB
      run: |
        sudo apt install -y adb
        id=${{ steps.create-instance.outputs.id }}
        addr="$(amc show "$id" --format=json | jq -r .network.address)"
        adb connect "$addr":5559
    - name: Run tests
      run: |
        test "$(adb shell getprop ro.product.model)" = Anbox
```

Alternatively, you can also integrate the execution of actual integration tests in the test step. As the Android instance is connected over ADB, you can perform any operation as with any other connected Android device.
