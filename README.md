 ![data flow](assets/data-flow.png)

# k6 Operator

> ### ⚠️ Experimental
>
> This project is **experimental** and changes a lot between commits.
> Use at your own risk. 

`k6io/operator` is a kubernetes operator for running distributed k6 tests in your cluster.

## Setup

### Deploying the operator
Install the operator by running the command below:

```bash
$ make deploy
``` 

### Installing the CRD

The k6 operator includes one custom resource called `K6`. This will be automatically installed when you do a
deployment, but in case you want to do it yourself, you may run the command below:

```bash
$ make install
```

## Usage

Two samples are available in `config/samples`, one for a test script and one for an actual test run.

### Adding test scripts

The operator utilises `ConfigMap`s to serve test scripts to the jobs. To upload your own test script, run the following:

```bash
$ kubectl create configmap my-test --from-file /path/to/my/test.js
``` 

For the execution segmentation to work, you'll need to adapt your k6 test slightly. First, import `getExecutionSegments` from the `k8s-distributed-execution` library available on https://jslib.k6.io.

 ```js
 import { getExecutionSegments } from 'https://jslib.k6.io/k8s-distributed-execution/0.0.1/index.js';
 ```
 
 Then modify your `options` constant to merge your configuration with the execution segment configuration:
 
```js
const myOptions = {
    stages: [
      { target: 200, duration: '30s' },
      { target: 0, duration: '30s' },
    ],
    threshold: {
      failed_requests: ['rate<=0'],
      http_req_duration: ['p(95)<500'],
    },
};

export const options = Object.assign(
  myOptions, 
  getExecutionSegments(),
);
```

### Executing tests
Tests are executed by applying the custom resource `K6` to a cluster where the operator is running. The properties
of a test run are few, but allow you to control some key aspects of a distributed execution.

```yaml
# k6-resource.yml

apiVersion: k6.io/v1alpha1
kind: K6
metadata:
  name: k6-sample
spec:
  parallelism: 4
  script: k6-test
  separate: false
```

The test configuration is applied using

```bash
$ kubectl apply -f /path/to/your/k6-resource.yml
```
     
#### Parallelism
How many instances of k6 you want to create. Each instance will be assigned an equal execution segment. For instance,
if your test script is configured to run 200 VUs and parallelism is set to 4, as in the example above, the operator will
create four k6 jobs, each running 50 VUs to achieve the desired VU count.

#### Script
The name of the config map that includes our test script. In the example in the [adding test scripts](#adding-test-scripts)
section, this is set to `my-test`.

#### Separate
Toggles whether the jobs created need to be distributed across different nodes. This is useful if you're running a
test with a really high VU count and want to make sure the resources of each node won't become a bottleneck.

### Cleaning up between test runs
After completing a test run, you need to clean up the test jobs created. This is done by running the following command:
```bash
$ kubectl delete -f /path/to/your/k6-resource.yml
```

## Uninstallation
Running the command below will delete all resources created by the operator.
```bash
$ make delete
```
