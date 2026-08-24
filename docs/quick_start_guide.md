# Green DevOps Plugin: Quick Start Guide

Welcome to the **Green DevOps Plugin**! This tool is designed to drastically reduce the carbon footprint of your CI/CD pipelines by introducing two powerful capabilities:
1. **Intelligent Build Optimization:** Analyzes your Git diffs and Maven dependencies to build and test *only* what is necessary.
2. **Carbon-Aware Scheduling & Deployment:** Uses machine learning and real-time grid carbon intensity data to automatically pause and reschedule your pipeline to a "greener" time window, while intelligently choosing between deployment strategies (Canary, Rolling, Recreate) based on risk and carbon impact.

---

## Prerequisites

- **Jenkins** with Docker pipeline support enabled.
- **Docker** installed on the Jenkins agent.
- A **Maven-based** Java repository.
- An **Electricity Maps API Key** for fetching real-time carbon intensity data.

---

### Configuring the API Key in Jenkins

Before adding the agent to your pipeline, you need to store your Electricity Maps API key securely in Jenkins:
1. Go to **Manage Jenkins** > **Credentials**.
2. Select your domain (e.g., `(global)`).
3. Click **Add Credentials**.
4. Set the **Kind** to `Secret text`.
5. Enter your API key in the **Secret** field.
6. Set the **ID** to `electricity-maps-key`.
7. Click **Create**.

---

## 1. Quick Integration

You can integrate the Green AI Agent directly into any existing Jenkins pipeline without needing to install heavy plugins. The agent runs completely contained inside a Docker image (`beliver247/build-optimizer-agent`).

### Pulling the Agent Image

Before running the agent, make sure to pull the latest image to your Jenkins runner:

```bash
docker pull beliver247/build-optimizer-agent:latest
```

### Add the Agent to your Jenkinsfile

Add the following stage to your Jenkinsfile right after your `Checkout` stage. This stage will run the optimizer, fetch the carbon forecast, and output a `dashboard_payload.json` file containing the build commands and deployment strategy.

```groovy
stage('Build Optimizer & Green Scheduling') {
    steps {
        script {
            // Define your environment variables
            env.ELECTRICITY_MAPS_API_KEY = credentials('electricity-maps-key')
            
            // Run the Green AI Agent Docker container
            sh '''
                docker run --rm -i \\
                  -e ELECTRICITY_MAPS_API_KEY=${ELECTRICITY_MAPS_API_KEY} \\
                  -e GIT_PREVIOUS_SUCCESSFUL_COMMIT=${GIT_PREVIOUS_SUCCESSFUL_COMMIT} \\
                  -e GIT_COMMIT=${GIT_COMMIT} \\
                  -v /var/run/docker.sock:/var/run/docker.sock \\
                  --entrypoint "" \\
                  beliver247/build-optimizer-agent:latest bash -lc "
                    mkdir -p /work
                    tar -xf - -C /work
                    cd /work
                    git config --global --add safe.directory /work
                    python3 -m optimizer \\
                        --project-root /work \\
                        --output-format json \\
                        --carbon-aware
                  "
            '''
        }
    }
}
```

---

## 2. Using the Optimizer's Decisions

The Docker agent outputs its decisions into the Jenkins environment. You can parse this JSON to dynamically control the rest of your pipeline.

### Selective Builds
The agent provides exact `mvn` commands that only build the modules affected by your Git commit.

```groovy
stage('Selective Build') {
    steps {
        script {
            // Assuming you parsed the JSON output into a `result` variable
            def buildCmd = result.actions.find { it.name == 'build' }?.command?.join(' ')
            if (buildCmd) {
                sh buildCmd
            } else {
                echo "No code changes detected. Skipping build!"
            }
        }
    }
}
```

### Green Rescheduling & Deployment Strategy
The agent's Machine Learning engine will recommend a deployment strategy (e.g., `rolling`, `canary`, `recreate`) based on the complexity of your changes and the current carbon intensity.

If the grid is currently dirty, the agent will recommend an optimal hour to reschedule the pipeline.

```groovy
stage('Green Deploy') {
    steps {
        script {
            def strategy = result.scheduling.strategy // e.g., 'rolling'
            def scheduledHour = result.scheduling.scheduled_hour
            
            if (scheduledHour != null) {
                echo "Grid is dirty! Rescheduling deployment for ${scheduledHour}:00"
                // Logic to reschedule pipeline...
            } else {
                echo "Grid is green. Deploying immediately using ${strategy} strategy."
                // Execute deployment...
            }
        }
    }
}
```

---

## 3. How It Works Under the Hood

### Build Optimization
1. **Module Discovery:** Dynamically scans your repository for `pom.xml` files.
2. **Dependency Graphing:** Parses `<dependencies>` to build a reverse dependency graph.
3. **Git Diff Mapping:** Checks exactly which files changed between commits.
4. **Blast Radius Calculation:** Safely computes which modules *must* be rebuilt based on the graph, skipping the rest to save CI compute time.

### Carbon-Aware Scheduling
1. **Grid Data:** Fetches a 24-hour carbon intensity forecast from Electricity Maps.
2. **ML Engine:** Analyzes the size of your Git diff (the "risk") and the current carbon intensity.
3. **Decision:** If the risk is high but the grid is green, it might choose a `canary` release. If the grid is extremely dirty, it will abort the current run and recommend the cleanest hour in the next 24 hours to automatically retry.

---

*Note: For a deeper dive into the specific algorithms behind the dependency graphing and blast radius calculation, see `HOW_IT_WORKS.md`.*
