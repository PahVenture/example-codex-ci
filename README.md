
# Automated Code Repair with Codex CLI and GitHub Actions


This project demonstrates how to leverage the Codex CLI (or a similar LLM command-line tool) integrated with GitHub Actions to automatically:

1.  **Detect** build or test failures in a CI pipeline.
2.  **Create** a GitHub Issue detailing the failure.
3.  **Trigger** a diagnostic workflow based on the issue.
4.  **Download** logs from the failed run.
5.  **Analyze** the logs and issue context using the Codex CLI.
6.  **Attempt** to automatically generate and apply a code fix.
7.  **Commit** the potential fix to a new branch.
8.  **Create** a Pull Request for review.
9.  **Comment** on the original issue with the outcome.

The repository includes an **intentional syntax error** in `Program.cs` to showcase the end-to-end process.

## Key Features

*   **.NET 9 Project:** A simple C# console application setup.
*   **Devcontainer:** Includes a `.devcontainer` configuration for a consistent development environment.
*   **CI Workflow (`build.yml`):** Builds and "tests" (runs) the .NET application on push/PR to `main`.
*   **Automated Issue Creation:** If the build/test fails, a job automatically creates a GitHub Issue with details and labels it for diagnosis.
*   **Diagnostic Workflow (`diagnosis.yml`):** Triggered by the creation or labeling of issues with `auto-diagnostic`.
*   **Log Retrieval:** Downloads the full logs from the specific failed workflow run identified in the issue.
*   **Codex CLI Integration:** Uses the Codex CLI (`@openai/codex`) to analyze the logs and attempt a fix.
    *   *(Note: While named Codex CLI, this could potentially be adapted for other compatible LLM CLI tools.)*
*   **Automated Patching:** Codex attempts to modify the source code directly based on its analysis.
*   **Git Integration:** Commits detected changes to a new branch (`fix/issue-<number>`).
*   **Pull Request Automation:** Creates a PR from the fix branch to `main` if changes were made.
*   **Feedback Loop:** Posts comments back to the originating issue indicating success, failure, or errors encountered during the diagnostic process.

## How it Works - The Automated Flow

1.  **Code Change & Push:** A developer pushes code to the `main` branch (or opens a PR targeting `main`).
2.  **Build & Test Failure:** The `build.yml` workflow triggers. The `build` step might pass, but the `test` step (which runs `dotnet run`) fails because of the intentional syntax error in `Program.cs`.
3.  **Issue Creation:** Since a job failed, the `create-issue` job in `build.yml` runs. It uses `actions/github-script` to create a new GitHub Issue, tagging the user who triggered the workflow, including details like the run ID and commit SHA, and adding the `bug`, `runtime-exception`, and crucially, the `auto-diagnostic` labels.
4.  **Diagnostic Trigger:** The creation of the issue with the `auto-diagnostic` label triggers the `diagnosis.yml` workflow.
5.  **Log Download:** The workflow parses the issue body to find the failed workflow `run_id`. It then uses the GitHub API (via `actions/github-script`) to download the logs for all jobs in that specific run, saving them to a `logs/` directory.
6.  **Codex Analysis & Fix Attempt:**
    *   The Codex CLI is installed (`npm install -g @openai/codex`).
    *   It's invoked with a prompt instructing it to use the downloaded logs (`logs/full-build-log.txt`) and the issue description to fix the problem.
    *   It's run in `--full-auto` mode with dangerous auto-approval enabled **(for demonstration purposes only!)**.
    *   Codex analyzes the error (likely identifying the missing comma in `Program.cs`) and modifies the file *in place*.
7.  **Change Detection & Commit:** The workflow checks if `git status` shows any modified files. If changes exist (i.e., Codex modified `Program.cs`):
    *   Git is configured.
    *   A new branch `fix/issue-<ISSUE_NUMBER>` is created.
    *   The changes are added and committed.
    *   The new branch is pushed to the origin repository.
8.  **Pull Request Creation:** If changes were committed and pushed, a Pull Request is automatically created from the `fix/issue-<ISSUE_NUMBER>` branch to `main`. The PR body links back to the original issue and the diagnostic workflow run.
9.  **Issue Commenting:** Regardless of whether a fix was applied, a final step comments on the original issue:
    *   If successful: Reports that a fix was attempted, provides a link to the new branch/PR, and links to the diagnostic run logs.
    *   If Codex made no changes: Reports that it couldn't automatically fix the issue and provides a link to the diagnostic logs.
    *   If errors occurred (log download failure, Codex error): Reports the specific error encountered.

## The Intentional Error

To demonstrate the workflow, a simple syntax error was intentionally introduced in `Program.cs`:

```csharp
// Program.cs - Line 4
var myArray = new int[] { 1, 2, 3, 4 5 }; // <-- Missing comma between 4 and 5
```

This error will cause the `dotnet build` or `dotnet run` command in the `build.yml` workflow to fail, triggering the automated issue creation and subsequent diagnostic process. The expected outcome is that the `diagnosis.yml` workflow, using Codex, will identify and correct this line to:

```csharp
// Program.cs - Corrected Line 4
var myArray = new int[] { 1, 2, 3, 4, 5 }; // <-- Comma added
```

## Setup & Prerequisites

1.  **Fork or Clone:** Fork this repository or use it as a template.
2.  **Secrets:** You need to configure the following secrets in your repository (`Settings -> Secrets and variables -> Actions`):
    *   `OPENAI_API_KEY`: Your API key for the OpenAI service used by the Codex CLI.
    *   `GH_TOKEN`: A GitHub Personal Access Token (PAT) with permissions for:
        *   `repo`: Full control of private repositories (needed to push branches, create PRs).
        *   `workflow`: Read/write access to workflow runs (needed by `actions/github-script` potentially).
        *   `issues:write`: To create issues and comments.
        *   `pull-requests:write`: To create pull requests.
        *   *(Note: While `secrets.GITHUB_TOKEN` is often sufficient for many actions, creating issues/PRs reliably, especially under certain conditions or with specific permission needs, often requires a PAT stored as a separate secret like `GH_TOKEN`).*
3.  **Enable Actions:** Ensure GitHub Actions are enabled for your repository and have the necessary permissions to create issues, push branches, and create PRs (`Settings -> Actions -> General -> Workflow permissions -> Read and write permissions`).

## Usage / Seeing it in Action

1.  Ensure the prerequisites (Secrets, Permissions) are met.
2.  Make any small change to the repository (e.g., update this README) and push it to the `main` branch.
3.  Alternatively, go to the "Actions" tab, select the "Build .NET App" workflow, and run it manually using "Run workflow".
4.  **Observe:**
    *   The "Build .NET App" workflow will run and likely fail at the `test` step.
    *   The `create-issue` job will run and create a new issue in your repository.
    *   The "Auto Diagnostic" workflow will be triggered by the new issue.
    *   Follow the "Auto Diagnostic" run logs in the Actions tab.
    *   Check the repository's Issues tab for comments added by the workflow.
    *   Check the Pull Requests tab for a new PR created by the workflow (if the fix was successful).
    *   Examine the diff in the PR to see the change made by Codex.


## Customization & Adaptation

*   **Different LLM Tools:** Modify the `diagnosis.yml` workflow to use a different CLI tool if needed (e.g., tools interacting with Anthropic Claude, Google Gemini, etc., if they offer compatible CLI interfaces).
*   **Prompts:** Adjust the `CODEX_PROMPT` in `diagnosis.yml` for different types of analysis or desired output formats.
*   **Error Types:** This example uses a simple syntax error. The effectiveness for more complex runtime or logical errors will vary. The prompt might need significant refinement for different scenarios.
*   **Triggers:** Change the triggers in `diagnosis.yml` (e.g., trigger on specific comments, manual triggers).
*   **Target Languages:** Adapt the project setup, build steps, and potentially the prompt for languages other than C#.

## Limitations & Considerations

*   **LLM Reliability:** AI-generated fixes are not guaranteed to be correct or optimal. **Always review generated code carefully.**
*   **Cost:** Using the OpenAI API incurs costs based on token usage. Monitor your usage.
*   **Security:** Storing `OPENAI_API_KEY` and a powerful `GH_TOKEN` requires careful secret management. The workflow requires write permissions to your code, which has security implications.
*   **Complexity:** The current setup might struggle with complex bugs requiring multi-file changes, deep domain knowledge, or interactive debugging.
*   **Log Parsing:** The log download step relies on parsing the Run ID from the issue body, which could be fragile if the issue format changes.
*   **Codex CLI Behavior:** The specific behavior of the `@openai/codex` tool, especially regarding its patching mechanism (`--full-auto`, stdin interaction), might change or have quirks. The workflow includes robustness checks (timeout, exit code handling).
*   **Deterministic Fixes:** The same error might not always yield the exact same fix from the LLM.

## Contributing

Contributions are welcome! Feel free to open issues or submit pull requests for improvements, bug fixes, or new features.

## License

This project is licensed under the [MIT License](LICENSE). (Or choose another license like Apache 2.0 if preferred - remember to add the actual LICENSE file).# example-codex-ci
# example-codex-ci
