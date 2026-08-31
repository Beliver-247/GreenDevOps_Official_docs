# Full Jenkinsfile Walkthrough

This page walks through the complete, production-ready `Jenkinsfile` used in the GreenDevOps reference pipeline — stage by stage.

> [!NOTE]
> The full source lives in [`jenkinsfile.example`](https://github.com/Beliver-247/greenDevPlugin/blob/main/jenkinsfile.example) in the plugin repository.

---

## Pipeline Parameters & Environment

```groovy
parameters {
    booleanParam(
        name: 'DRY_RUN',
        defaultValue: false,
        description: 'When true, only run the optimizer analysis without building, testing, or deploying.'
    )
}

environment {
    DOCKER_HUB_CREDENTIALS = credentials('dockerhub-dumindu-credentials')
    DOCKER_IMAGE           = 'beliver247/green-release-app'
    DOCKER_TAG             = "${BUILD_NUMBER}"

    REMOTE_HOST            = '147.15.144.192'
    REMOTE_PORT            = '2510'
    REMOTE_USER            = 'dumindu'
    SSH_CREDENTIALS        = 'ubuntu-pc-ssh-dumindu'

    METRICS_URL            = 'http://192.168.9.127:5001'
}
```

**`DRY_RUN`** lets you trigger the pipeline to preview what *would* be built without actually building, pushing, or deploying anything.
All downstream stages check `!params.DRY_RUN` in their `when` conditions.

---

## Stage 1: Notify Start

```groovy
stage('Notify Start') {
    steps {
        sh """
            curl -s -X POST ${METRICS_URL}/deployment/start \
                -H "Content-Type: application/json" \
                -d '{"job_name":"${JOB_NAME}","build_number":"${BUILD_NUMBER}"}'
        """
    }
}
```

Sends a `POST /deployment/start` webhook to the GreenDevOps metrics server the moment the pipeline begins.
This enables the dashboard to track live build duration and carbon usage from the first second.

---

## Stage 2: Checkout

```groovy
stage('Checkout') {
    steps {
        deleteDir()
        checkout([$class: 'GitSCM',
            branches: [[name: '*/main']],
            userRemoteConfigs: [[
                url: 'https://github.com/Beliver-247/green-release-demo.git',
                credentialsId: 'github-dumindu-credentials'
            ]],
            extensions: [[ $class: 'CloneOption', shallow: false, depth: 0, noTags: false ]]
        ])
    }
}
```

**`shallow: false, depth: 0`** is critical. A shallow clone truncates Git history, which breaks `git diff` — the optimizer needs the full commit graph to compare `HEAD~1` against `HEAD`.

---

## Stage 3: Build Optimizer – Analyze

This is the core GreenDevOps stage. It runs the optimizer inside Docker, parses the JSON output, and populates all environment variables for subsequent stages.

### 3a. Running the Optimizer

```groovy
def output = sh(
    script: '''
        set -e
        tar -C "$PWD" -cf - . | docker run --rm -i --platform linux/amd64 \
          -v /var/run/docker.sock:/var/run/docker.sock \
          beliver247/build-optimizer-agent:latest \
          bash -lc '
            set -e
            mkdir -p /work
            tar -xf - -C /work
            cd /work
            git config --global --add safe.directory /work

            HEAD_SHA=$(git rev-parse --verify HEAD)
            if git rev-parse --verify HEAD~1 >/dev/null 2>&1; then
              BASE_SHA=$(git rev-parse --verify HEAD~1)
            else
              BASE_SHA=4b825dc642cb6eb9a060e54bf8d69288fbee4904
            fi

            python3 -m optimizer \
              --base "$BASE_SHA" \
              --head "$HEAD_SHA" \
              --project-root /work \
              --dry-run true \
              --output-format json \
              --carbon-aware
          '
    ''',
    returnStdout: true
).trim()
```

**Key points:**

- **`tar | docker run -i`** — The workspace is streamed into the container via stdin. This avoids Docker volume mounts that may not work on all Jenkins agents.
- **`git config --global --add safe.directory /work`** — Required because the `/work` directory is owned by a different user inside the container.
- **Empty tree SHA** (`4b825dc6...`) — Used as the base ref when there is no `HEAD~1` (first commit in the repository). This tells `git diff` to compare against an empty tree, effectively treating all files as new.
- **`--dry-run true`** in the *Analyze* stage — The optimizer analyses but does not execute Maven here. The actual Maven build happens in the next stage after the JSON is parsed.

### 3b. Parsing the JSON Output

```groovy
def jsonLine = output.readLines().find { it.startsWith('{"') }
if (jsonLine) {
    def result = new groovy.json.JsonSlurper().parseText(jsonLine)
    env.OPTIMIZER_STATUS = result.status ?: 'unknown'

    def buildCommands = []
    def testCommands = []
    for (action in result.actions ?: []) {
        if (action.name == 'build') buildCommands.add(action.command.join(' '))
        if (action.name == 'test')  testCommands.add(action.command.join(' '))
    }

    env.MAVEN_BUILD_COMMANDS = buildCommands.join('|||')
    env.MAVEN_TEST_COMMANDS  = testCommands.join('|||')
    env.AFFECTED_MODULES     = (result.affected_modules ?: []).join(',')

    if (result.scheduling) {
        env.CARBON_INTENSITY  = result.scheduling.current_intensity?.toString() ?: ''
        env.GREEN_PROBABILITY = result.scheduling.green_probability?.toString() ?: ''
        env.SCHEDULING_ACTION = result.scheduling.action ?: ''
        env.SCHEDULING_ENGINE = result.scheduling.engine ?: ''
    }
}
```

**The `|||` separator trick:** Jenkins environment variables are strings. A list of Maven commands is joined with `|||` (unlikely to appear in a real Maven command) so multiple commands survive serialisation into a single env var. The receiving stage splits on `\\|\\|\\|`.

---

## Stage 4: Selective Build

```groovy
stage('Selective Build') {
    when {
        expression { !params.DRY_RUN && env.OPTIMIZER_STATUS == 'success' && env.MAVEN_BUILD_COMMANDS?.trim() }
    }
    steps {
        script {
            echo "Running selective Maven build for modules: ${env.AFFECTED_MODULES}"
            env.MAVEN_BUILD_COMMANDS.split('\\|\\|\\|').each { cmd ->
                echo "Executing: ${cmd}"
                sh cmd
            }
        }
    }
}
```

This stage only runs if:
1. Not in dry-run mode.
2. The optimizer status is `success` (not `no_changes`, `docs_only`, etc.).
3. At least one build command was generated.

The `|||`-separated string is split back into individual commands and each is executed via `sh`.

---

## Stage 5: Selective Test

```groovy
stage('Selective Test') {
    when {
        expression { !params.DRY_RUN && env.OPTIMIZER_STATUS == 'success' && env.MAVEN_TEST_COMMANDS?.trim() }
    }
    steps {
        script {
            env.MAVEN_TEST_COMMANDS.split('\\|\\|\\|').each { cmd ->
                sh cmd
            }
        }
    }
}
```

Identical pattern to Selective Build, but runs the test commands.

---

## Stages 6–7: Docker Build & Push

```groovy
stage('Docker Build') {
    when { expression { !params.DRY_RUN && env.OPTIMIZER_STATUS == 'success' } }
    steps {
        dir('app') {
            sh "docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} -t ${DOCKER_IMAGE}:latest ."
        }
    }
}

stage('Docker Push') {
    when { expression { !params.DRY_RUN && env.OPTIMIZER_STATUS == 'success' } }
    steps {
        sh """
            echo "${DOCKER_HUB_CREDENTIALS_PSW}" | docker login -u "${DOCKER_HUB_CREDENTIALS_USR}" --password-stdin
            docker push ${DOCKER_IMAGE}:${DOCKER_TAG}
            docker push ${DOCKER_IMAGE}:latest
            docker logout
        """
    }
}
```

Builds and pushes the Docker image only when there was a successful selective build. Images are double-tagged: `:<BUILD_NUMBER>` for traceability and `:latest` for production deployment.

---

## Stage 8: Deploy to Server

```groovy
stage('Deploy to Server') {
    when { expression { !params.DRY_RUN && env.OPTIMIZER_STATUS == 'success' } }
    steps {
        sshagent(credentials: ["${SSH_CREDENTIALS}"]) {
            sh "ssh -o StrictHostKeyChecking=no -p ${REMOTE_PORT} ${REMOTE_USER}@${REMOTE_HOST} 'mkdir -p /home/${REMOTE_USER}/green-release-demo'"
            sh "scp -o StrictHostKeyChecking=no -P ${REMOTE_PORT} docker-compose.yml ${REMOTE_USER}@${REMOTE_HOST}:/home/${REMOTE_USER}/green-release-demo/docker-compose.yml"
            sh """
                ssh -o StrictHostKeyChecking=no -p ${REMOTE_PORT} ${REMOTE_USER}@${REMOTE_HOST} '
                    set -e
                    cd /home/${REMOTE_USER}/green-release-demo
                    docker pull ${DOCKER_IMAGE}:latest
                    docker-compose down || true
                    docker-compose up -d
                    sleep 15
                    docker-compose ps
                '
            """
        }
    }
}
```

Deploys via SSH using Jenkins's `sshagent` step:
1. Ensures the deploy directory exists on the remote host.
2. Copies the latest `docker-compose.yml`.
3. Pulls the new image, tears down the old container, and starts the new one.

`sleep 15` gives the container time to initialise before the smoke test.

---

## Stage 9: Smoke Test

```groovy
stage('Smoke Test') {
    when { expression { !params.DRY_RUN && env.OPTIMIZER_STATUS == 'success' } }
    steps {
        sshagent(credentials: ["${SSH_CREDENTIALS}"]) {
            sh """
                ssh -o StrictHostKeyChecking=no -p ${REMOTE_PORT} ${REMOTE_USER}@${REMOTE_HOST} \
                    'curl -sf http://localhost:8081/health && echo "SMOKE TEST PASSED" || exit 1'
            """
        }
    }
}
```

Runs a `curl` health check against the deployed container via SSH. A non-zero exit from `curl -sf` fails the stage and marks the build as failed.

---

## `post` Block — Cleanup & Metrics

```groovy
post {
    success {
        script {
            // Notify dashboard
            sh """
                curl -s -X POST ${METRICS_URL}/deployment/end \
                    -H "Content-Type: application/json" \
                    -d '{"status":"SUCCESS","build_number":"${BUILD_NUMBER}","image":"${DOCKER_IMAGE}:${DOCKER_TAG}"}' || true
            """
        }
    }
    failure {
        script {
            sh """
                curl -s -X POST ${METRICS_URL}/deployment/end \
                    -H "Content-Type: application/json" \
                    -d '{"status":"FAILURE","build_number":"${BUILD_NUMBER}"}' || true
            """
        }
    }
    always {
        sh "docker rmi ${DOCKER_IMAGE}:${DOCKER_TAG} || true"
        sh "docker image prune -f || true"
    }
}
```

- **Success/failure:** Sends `POST /deployment/end` to the metrics server with the final status, enabling the dashboard to record pipeline duration and outcome.
- **Always:** Removes the locally built image and prunes dangling images to keep the Jenkins agent's disk clean. The `|| true` ensures failures here don't mask the real build status.
