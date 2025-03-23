# github-workflows

ui-ci.yml

```bash
name: Run CI Process

on:
  workflow_call:
    inputs:
      environment:
        description: 'Deployment environment (dev, test, stag, prod)'
        required: true
        type: string

jobs:
  ci-process:
    name: CI process
    runs-on: ubuntu-latest
    # This is option 2: run on the runner of inputs
    # runs-on: ["${{ inputs.RUNNER }}"]

    # Prevents parallel execution of the same workflow on the same branch.
    # If a new workflow run starts for the same branch/ref, the previous run is canceled.
    concurrency:
      group: ${{ github.workflow }}-ci-process-${{ github.ref }}
      cancel-in-progress: true

    steps:
      - name: Checkout Repository
      - uses: actions/checkout@v4
        with:
          fetch-depth: 1 # ensures only the latest commit is pulled (faster performance).

      - name: Setup nodeJs
      - uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Install yarn
        run: npm install -g yarn

      # Retrieves the directory where Yarn stores cached dependencies.
      # Saves this path in a GitHub Actions output variable.
      - name: Get yarn cache directory path
        id: yarn-cache-dir-path
        run: |
          echo "dir=$(yarn cache dir)" >> $GITHUB_OUTPUT

      # Uses GitHub's cache action to store and restore Yarn dependencies.
      # Cache key is based on: The OS and The yarn.lock file (to detect dependency changes).
      # The cache prevents unnecessary reinstallation of dependencies in future runs.
      - name: Cache Yarn Dependencies
      - uses: actions/cache@v4
        id: yarn-cache
        env:
          cache-name: cache-node-modules
        with:
          path: ${{ steps.yarn-cache-dir-path.outputs.dir }}
          key: ${{ runner.os }}-build-${{ env.cache-name }}-${{ hashFiles('**/yarn.lock') }}
          restore-keys: |
            ${{ runner.os }}-build-${{ env.cache-name }}-
            ${{ runner.os }}-build-
            ${{ runner.os }}-

      # If the cache wasn't restored, it prints 'cache not hit'
      - name: Check cache
        if: steps.yarn-cache.outputs.cache-hit != 'true'
        run: echo 'cache not hit'

      # Installs project dependencies without modifying yarn.lock.
      - name: Install Dependencies
        run: yarn install --frozen-lockfile

      # Runs the linting process (e.g., ESLint or similar tools).
      - name: Lint Check
        run: yarn run lint

      # Build the project: Compiles/transpiles the project.
      - name: Build
        run: yarn run build

      # Runs unit/integration tests by Jest
      - name: Test
        run: yarn test

      # Run Linting & Generate Lint Report
      # - name: Lint Code
      #   id: execute-lint
      #   # Creates an ESLint JSON & Markdown report in report/eslint/.
      #   run: |
      #     mkdir -p report/eslint
      #     echo "{}" > report/eslint/eslint.json
      #     echo "ESlint: No issue, Great!" > report/eslint/eslint.md
      #     yarn run lint:report

      # # Run Tests & Generate Coverage Report
      # - name: Test Coverage
      #   if: always()
      #   run: yarn test:ci

      # Determines if the workflow is running on a PR branch or a normal branch.
      # Saves the branch name to SONAR_BRANCH.
      # - name: Set SONAR_BRANCH environment variable
      #   run: |
      #     if [[ -z "$GITHUB_BASE_REF" ]]; then
      #       echo "SONAR_BRANCH=${GITHUB_REF_NAME}" >> $GITHUB_ENV
      #     fi

      # Runs SonarQube analysis with SonarCloud.
      # Uses secrets for authentication (SONAR_CLOUD_TOKEN).
      # Waits for quality gate results before proceeding.
      # - name: SonarCloud Scan
      #   id: sonarqube-scan
      #   if: always()
      #   uses: SonarSource/sonarqube-scan-action@master
      #   env:
      #     GITHUB_TOKEN: ${{ secrets.READ_ONLY_PAT }}
      #     SONAR_TOKEN: ${{ secrets.SONAR_CLOUD_TOKEN }}
      #     SONAR_BRANCH: ${{ env.SONAR_BRANCH }}
      #   with:
      #     args: >
      #       -Dsonar.projectBaseDir=.
      #       -Dsonar.qualitygate.wait=true
      #       -Dsonar.organization=${{ secrets.SONAR_ORGANIZATION_KEY }}
      #       -Dsonar.projectKey=${{ secrets.SONAR_PROJECT_KEY }}
      #       ${{ env.SONAR_BRANCH && format('-Dsonar.branch.name={0}', env.SONAR_BRANCH) || '' }}

      # Publishes unit test results using Jest JUnit reports.
      # - name: Unit Test Report
      #   uses: dorny/test-reporter@v1
      #   if: ${{ always() }}
      #   with:
      #     name: Unit Test Report
      #     path: report/unit/*.xml
      #     reporter: jest-junit

      # # Generates test coverage reports from Jest.
      # - name: Generate Coverage Report
      #   uses: ArtiomTr/jest-coverage-report-action@v2
      #   id: coverage
      #   if: ${{ always() }}
      #   with:
      #     skip-step: all
      #     output: report-markdown
      #     coverage-file: report/coverage/jest.json
      #     base-coverage-file: report/coverage/jest.json
      #     annotations: none

      # # Annotates PRs with ESLint errors and warnings.
      # - name: Annotate Code Linting Results
      #   uses: ataylorme/eslint-annotate-action@1.2.0
      #   if: ${{ always() }}
      #   with:
      #     repo-token: '${{ secrets.GITHUB_TOKEN }}'
      #     report-json: 'report/eslint/eslint.json'

      # # Converts ESLint JSON results into Markdown.
      # # Appends the report to GitHub Actions job summary.
      # - name: Write Eslint Report To Workflow Job Summary
      #   id: write-eslint-report
      #   if: ${{ always() }}
      #   run: |
      #     FILE=report/eslint/eslint.json
      #     if [ -f "$FILE" ]; then
      #       npm_config_yes=true npx github:10up/eslint-json-to-md --path $FILE --output ./report/eslint/eslint.md
      #       cat ./report/eslint/eslint.md >> $GITHUB_STEP_SUMMARY
      #     else
      #       echo "report=No issue, Great!" >> $GITHUB_OUTPUT

      # # Adds test coverage and linting results as sticky comments in PRs.
      # - name: Attached Coverage Report To PR
      #   uses: marocchino/sticky-pull-request-comment@v2
      #   if: ${{ always() }}
      #   with:
      #     header: coverage-report
      #     message: ${{ steps.coverage.outputs.report }}

      # - name: Attached Eslint Report To PR
      #   uses: marocchino/sticky-pull-request-comment@v2
      #   if: ${{ always() }}
      #   with:
      #     header: eslint-report
      #     path: report/eslint/eslint.md

      # # Marks ESLint and test coverage results as status checks.
      # - name: Attach Coverage Check Status
      #   uses: LouisBrunner/checks-action@v2.0.0
      #   if: ${{ always() }}
      #   with:
      #     token: ${{ secrets.GITHUB_TOKEN }}
      #     name: Coverage Report
      #     conclusion: ${{ steps.coverage.outcome }}

      # - name: Attach Eslint Check Status
      #   uses: LouisBrunner/checks-action@v2.0.0
      #   if: ${{ always() }}
      #   with:
      #     token: ${{ secrets.GITHUB_TOKEN }}
      #     name: ESlint Report
      #     conclusion: ${{ steps.execute-lint.outcome }}

```
