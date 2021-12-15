# Elastic GitHub Runners in Google Cloud Platform
How to create elastic GitHub Self Hosted Runners on demand that scale to zero when idle.

Having idle runners waiting for jobs to be executed it's a waste of resources and can be very expensive for organisations that have hundreds of runners.

## The Goal
We want to automatically spin-up a GitHub Self-Hosted Runner in GCP when a workflow is triggered. We also want to tear it down at the end of the workflow.

## Requirements
You need to have access to GCP, the services we are going to use are: Secret Manager, CloudBuild and Compute Engine. You also need access to GitHub, we need to create a runner for a repository, so you need an access token with the `repo` scope to generate a registration token ([GitHub Docs](https://docs.github.com/en/rest/reference/actions#create-a-registration-token-for-a-repository)).

## Plan
We are going to configure a Cloud Build trigger to run when there is a change in our repo. The trigger will spin-up a VM that will register a new runner. The new runner will be executed using the `--ephemeral` flag ([GitHub Docs](https://docs.github.com/en/actions/hosting-your-own-runners/autoscaling-with-self-hosted-runners#using-ephemeral-runners-for-autoscaling)). The `--ephemeral` flag makes the runner available for a single job execution, after that the runner is de-registered and we can delete the instance.
