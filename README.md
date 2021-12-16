# Elastic GitHub Runners in Google Cloud Platform
How to create elastic GitHub Self Hosted Runners on demand that scale to zero when idle.

Having idle runners waiting for jobs to be executed it's a waste of resources and can be very expensive for organisations that have hundreds of runners.

## The Goal
We want to automatically spin-up a GitHub Self-Hosted Runner in GCP when a workflow is triggered. We also want to tear it down at the end of the workflow.

## Requirements
You need to have access to GCP, the services we are going to use are: Secret Manager, CloudBuild and Compute Engine. You also need access to GitHub, and the permissions to generate a registration token. ([GitHub Docs](https://docs.github.com/en/rest/reference/actions#create-a-registration-token-for-a-repository)).

## Plan
We are going to configure a Cloud Build trigger to run when there is a change in our repo. The trigger will spin-up a VM that will register a new runner. The new runner will be executed using the `--ephemeral` flag ([GitHub Docs](https://docs.github.com/en/actions/hosting-your-own-runners/autoscaling-with-self-hosted-runners#using-ephemeral-runners-for-autoscaling)). The `--ephemeral` flag makes the runner available for a single job execution, after that the runner is de-registered and we can delete the instance.


cloudbuild:

Event -> Webhook event
Inline cloudbuild configuration
Substitution variables:
- `_ACTION` = $(body.action)
- `_JOB_NAME` = $(body.workflow_job.name)
- `_ORG_NAME` = $(body.organization.login)
- `_REPO_FULLNAME` = $(body.repository.full_name)
- `_REPO_NAME` = $(body.repository.name)
- `_RUNNER_LABELS` = $(body.workflow_job.labels)
- `_TIMEOUT` = 600

Create a secret with a github token 

```bash
PROJECT_ID="THE-PROJECT-ID"
SA="projects/${PROJECT_ID}/serviceAccounts/runner-bootstrap@${PROJECT_ID}.iam.gserviceaccount.com"
gcloud alpha builds triggers create webhook \
  --name=elastic-runner-webhook \
  --secret=projects/$PROJECT_ID/secrets/webhook-secret/versions/latest \
  --substitutions=_ACTION='$(body.action)',_JOB_NAME='$(body.workflow_job.name)',_ORG_NAME='$(body.organization.login)',_REPO_FULLNAME='$(body.repository.full_name)',_REPO_NAME='$(body.repository.name)',_RUNNER_LABELS='$(body.workflow_job.labels)',_TIMEOUT=600 \
  --filter='_ACTION == "queued"' \
  --service-account=$SA \
  --inline-config=build-config.yaml
```

Create a file named `build-config.yaml` with this content:
```yaml
steps:
  - name: gcr.io/cloud-builders/gcloud
    entrypoint: 'bash'
    args:
      - '-c'
      - |
        #### Create ci.yml
        cat > /workspace/ci.yml <<- EOF
        #cloud-config
        users:
        - name: ghr
          groups: docker
          homedir: /home/ghr
          uid: 2000
        package_upgrade: true
        yum_repos:
          docker-ce:
            name: Docker CE Stable - \$basearch
            baseurl: https://download.docker.com/linux/centos/\$releasever/\$basearch/stable
            enabled: true
            gpgcheck: true
            gpgkey: https://download.docker.com/linux/centos/gpg
        packages:
          - yum-utils
          - git
          - docker-ce
          - docker-ce-cli
          - containerd.io
        write_files:
          - path: /etc/systemd/system/shutdown.service
            permissions: 0644
            owner: root
            content: |
              [Unit]
              Description=Shutdown Service
              [Service]
              Type=simple
              Restart=no
              ExecStart=/bin/bash /etc/systemd/system/shutdown.sh
              [Install]
              WantedBy=multi-user.target
          - path: /etc/systemd/system/ghr.service
            permissions: 0644
            owner: root
            content: |
              [Unit]
              Description=GitHub Self-Hosted Runner
              Requires=network.target
              After=network.target
              [Service]
              User=ghr
              Type=oneshot
              Restart=no
              WorkingDirectory=/home/ghr
              Environment="HOME=/home/ghr"
              ExecStartPre=/bin/bash /home/ghr/pre-start.sh
              ExecStart=/bin/bash /home/ghr/run.sh
              ExecStartPost=/bin/bash /home/ghr/terminate.sh
              [Install]
              WantedBy=multi-user.target
          - path: /home/ghr/pre-start.sh
            permissions: 0755
            owner: root
            content: |
              #!/usr/bin/env bash
              set -euo pipefail
              REGISTRATION_TOKEN=\$(curl -H "Metadata-Flavor: Google" http://metadata.google.internal/computeMetadata/v1/instance/attributes/REGISTRATION_TOKEN)
              RUNNER_LABELS=\$(curl -H "Metadata-Flavor: Google" http://metadata.google.internal/computeMetadata/v1/instance/attributes/RUNNER_LABELS)
              REPO_FULLNAME=\$(curl -H "Metadata-Flavor: Google" http://metadata.google.internal/computeMetadata/v1/instance/attributes/REPO_FULLNAME)
              curl -o /home/ghr/actions-runner-linux-x64-2.285.1.tar.gz -L https://github.com/actions/runner/releases/download/v2.285.1/actions-runner-linux-x64-2.285.1.tar.gz
              tar xzf /home/ghr/actions-runner-linux-x64-2.285.1.tar.gz
              /home/ghr/config.sh --unattended --url https://github.com/\$${REPO_FULLNAME} --token \$${REGISTRATION_TOKEN} --labels \$${RUNNER_LABELS} --ephemeral
          - path: /home/ghr/terminate.sh
            permissions: 0755
            owner: root
            content: |
              #!/usr/bin/env bash
              set -euo pipefail
              NAME=\$(curl -X GET http://metadata.google.internal/computeMetadata/v1/instance/name -H 'Metadata-Flavor: Google')
              ZONE=\$(curl -X GET http://metadata.google.internal/computeMetadata/v1/instance/zone -H 'Metadata-Flavor: Google')
              gcloud --quiet compute instances delete \$$NAME --zone=\$$ZONE
          - path: /etc/systemd/system/shutdown.sh
            permissions: 0755
            owner: root
            content: |
              #!/usr/bin/env bash
              set -euo pipefail
              NAME=\$(curl -X GET http://metadata.google.internal/computeMetadata/v1/instance/name -H 'Metadata-Flavor: Google')
              ZONE=\$(curl -X GET http://metadata.google.internal/computeMetadata/v1/instance/zone -H 'Metadata-Flavor: Google')
              TIMEOUT=\$(curl -X GET http://metadata.google.internal/computeMetadata/v1/instance/attributes/TIMEOUT -H 'Metadata-Flavor: Google')
              re='^[0-9]+$'
              if [[ \$$TIMEOUT =~ \$$re ]] ; then
                echo "TIMEOUT is a valid number" >&2
                sleep \$$TIMEOUT
                shutdown +2
                gcloud --quiet compute instances delete \$$NAME --zone=\$$ZONE
              fi
        runcmd:
          - chown -R ghr /home/ghr
          - systemctl daemon-reload
          - systemctl start docker
          - systemctl start shutdown.service
          - systemctl start ghr.service
        EOF
  - name: gcr.io/cloud-builders/gcloud
    entrypoint: 'bash'
    args:
      - '-c'
      - |
        #### Create startup.sh
        cat > /workspace/startup.sh <<- EOF
        #!/bin/bash
        
        if ! type cloud-init > /dev/null 2>&1 ; then
          echo "Ran - `date`" >> /root/startup
          sleep 30
          yum install -y cloud-init
        
          if [ \$? == 0 ]; then
            echo "Ran - Success - `date`" >> /root/startup
            systemctl enable cloud-init
          else
            echo "Ran - Fail - `date`" >> /root/startup
          fi
        
          # Reboot either way
          reboot
        fi
        EOF
  - name: gcr.io/cloud-builders/gcloud
    entrypoint: 'bash'
    args:
      - '-c'
      - |
        #### Create shutdown.sh
        cat > /workspace/shutdown.sh <<- EOF
        #!/bin/bash

        export RUNNER_ALLOW_RUNASROOT=1
        REGISTRATION_TOKEN=\$(curl -H "Metadata-Flavor: Google" http://metadata.google.internal/computeMetadata/v1/instance/attributes/REGISTRATION_TOKEN)
        /home/ghr/config.sh remove --token \$${REGISTRATION_TOKEN}
        EOF
  - name: gcr.io/cloud-builders/gcloud
    entrypoint: 'bash'
    args:
      - '-c'
      - |
        apt update && apt install jq -y
        REGISTRATION_TOKEN=$(curl -H "Authorization: token $$GITHUB_TOKEN" -X POST https://api.github.com/repos/${_REPO_FULLNAME}/actions/runners/registration-token | jq -r .token)
        RUNNER_NAME=ghr-$(cat /proc/sys/kernel/random/uuid | sed 's/[-]//g' | head -c 6; echo;)
        RUNNER_LABELS=$(echo '${_RUNNER_LABELS}' | jq -r '@csv' | sed 's/"//g')
        gcloud compute instances create $$RUNNER_NAME \
        --project=$PROJECT_ID \
        --zone=europe-west1-b \
        --machine-type=e2-standard-2 \
        --network-interface=network-tier=PREMIUM,subnet=runners \
        --maintenance-policy=TERMINATE \
        --service-account=github-runner@$PROJECT_ID.iam.gserviceaccount.com \
        --scopes=https://www.googleapis.com/auth/cloud-platform \
        --create-disk=auto-delete=yes,boot=yes,device-name=instance-1,image=projects/centos-cloud/global/images/centos-7-v20211214,mode=rw,size=20,type=projects/$PROJECT_ID/zones/europe-west1-b/diskTypes/pd-ssd \
        --metadata=^:^REGISTRATION_TOKEN=$$REGISTRATION_TOKEN:RUNNER_LABELS=$$RUNNER_LABELS:REPO_FULLNAME=${_REPO_FULLNAME}:TIMEOUT=${_TIMEOUT} \
        --metadata-from-file user-data=/workspace/ci.yml,startup-script=/workspace/startup.sh,shutdown-script=/workspace/shutdown.sh \
        --no-shielded-secure-boot \
        --preemptible \
        --no-restart-on-failure \
        --shielded-vtpm \
        --shielded-integrity-monitoring \
        --reservation-affinity=any \
        --labels org_name=${_ORG_NAME},repo_name=${_REPO_NAME},job_name=${_JOB_NAME}
    secretEnv: ['GITHUB_TOKEN']
options:
  logging: CLOUD_LOGGING_ONLY
availableSecrets:
  secretManager:
  - versionName: projects/$PROJECT_NUMBER/secrets/github-token/versions/latest
    env: 'GITHUB_TOKEN'
```
