# Azure Federated Deploy Starter (Lab Repository)

Starter repository for the Pluralsight hands-on lab **Deploy an Azure App with GitHub Actions and Federated Identity**. **Fork** this repository into your own GitHub account, then follow the lab guide.

## What is in here

- `server.js`, `package.json`: the shipment tracker, a zero-dependency Node app the workflow deploys to Azure App Service. Its `/version` endpoint returns the deployed application version.
- `.github/workflows/deploy.yml`: the deployment workflow, shipped incomplete on purpose. Two marked sections (the Azure sign-in step and the deploy step) and an `AZURE_WEBAPP_NAME` placeholder are completed during the lab, and the lab walks you through running and troubleshooting the workflow.

## What the lab has you configure in your fork

- Repository secrets `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, and `AZURE_SUBSCRIPTION_ID`: the identity and subscription values for federated sign-in. None of them is a password or key; the trust itself comes from a federated credential you create on an Azure managed identity, scoped to your fork.
- The two marked workflow sections, using the exact steps the lab guide provides.

The application code is correct and never changes during the lab. The workflow file is the only file you edit.

----------------------------------------------------

## Background

You are an Azure delivery engineer on a team that is moving deployments off manually shared credentials. The Azure target and a managed identity have already been prepared by the platform team. Your job is to connect a GitHub repository to that identity using federated trust, then build and run a workflow that deploys the application without storing any client secret. Your job is not to write application code or create identities: the app is provided and correct, the identity and its role assignment already exist, and the one file you edit is the deployment workflow itself.

In this lab, you work across two browser surfaces. In the Azure portal, you review the pre-provisioned target and identity. In your own GitHub account, you fork the starter repository and read the fork's immutable identifiers from its OIDC settings. Back in the portal, you create the federated credential that pins trust to those identifiers and record the three identifier values the workflow needs; in your fork, you store those values as repository secrets and complete the workflow's two marked sections: the Azure sign-in step and the deploy step. Then you run it. The first run fails at the sign-in step, and diagnosing that failure is the heart of this lab: you read the error the run presents, see exactly which permission the workflow never requested, grant it with a fix-forward commit, and watch the push run to a clean deployment. You finish by confirming the application answers in place of the default App Service page.

By the end of this lab, you will have:

- Located the pre-provisioned App Service target and managed identity, forked the starter repository into your own account, and recorded your fork's immutable owner and repository IDs.
- Created a federated credential scoped to your fork's `main` branch and matching its immutable subject exactly, and stored the three authentication identifiers as repository secrets.
- Completed the deployment workflow by adding the OIDC sign-in step and the deploy step.
- Ran the workflow, diagnosed the failed sign-in from the error naming the missing `id-token` permission, granted it with a fix-forward commit, and watched the new run deploy cleanly.
- Validated the deployed application, its version endpoint, and its deployment record.

You will work with:

- A pre-provisioned Azure App Service web app, serving the default App Service page until your workflow deploys the app.
- A pre-provisioned user-assigned managed identity holding Contributor on the lab resource group, with no federated credential until you add one.
- A provided public starter repository containing the sample app and a deployment workflow with marked sections to complete.
- GitHub Actions and the App Service Deployment Center as the run and evidence surfaces.

> **Note:**
>
> - You will work in two browser windows: the Azure portal in an InPrivate or incognito window using the lab credentials, and GitHub in your normal window using your personal account. No command line or local tools are required.
> - You need your own GitHub account for this lab. The Azure credentials to sign in are in **Lab Credentials** on the lab page; GitHub sign-in uses your personal account.
> - The application code is correct and never changes. The only file you edit is the workflow file, `.github/workflows/deploy.yml`, and completing it is part of the point of the lab.
> - As you work, you may see informational pop-ups or guided tours on either platform that you can close by selecting **Close**, **Dismiss**, or **Skip**.

### Pre-configured resources

The following resources have been pre-configured for this lab:

| Resource            | Details                                                      |
| ------------------- | ------------------------------------------------------------ |
| App Service web app | A name beginning `app-fedid-`, Linux with Node 20, HTTPS-only. Nothing is deployed to it yet. |
| App Service plan    | A name beginning `plan-fedid-`, hosting the web app.         |
| Managed identity    | A user-assigned identity beginning `id-deploy-`, holding Contributor scoped to the lab resource group. The identity and its role assignment were created by the lab; you never create either. |
| Starter repository  | `pluralsight-cloud/Lab-Deploy-an-Azure-App-with-GitHub-Actions-and-Federated-Identity` on GitHub, public. You fork it into your own account. |

> **Note:** The identity has no federated credential at start, your fork has no secrets, and the workflow ships with two marked sections to complete. Each of those is your work, and the first workflow run is part of the lesson: you will diagnose and fix it rather than avoid it.

## Prepare the target and the repository

You will sign in to the lab subscription, review the empty deployment target and the identity the platform team prepared, and fork the starter repository into your GitHub account so everything you change from here on is yours.

### Sign in and review the target

1. In a new InPrivate or incognito browser window, sign in to the Azure portal using the username and password from **Lab Credentials** on the lab page.

   **Expected result**: The Azure portal opens on the lab resource group, listing the pre-configured resources: an App Service web app beginning `app-fedid-`, an App Service plan beginning `plan-fedid-`, and a managed identity beginning `id-deploy-`.

   > **Note:** A private window keeps the lab credentials separate from any Microsoft account you are already signed in to, so the portal opens in the lab subscription rather than your own.

2. Select the web app beginning `app-fedid-`, and on its **Overview**, in the **Essentials** section, select the **Default domain** link to browse it in a new tab.

   **Expected result**: The default App Service page loads, with the heading **Your web app is running and waiting for your content**, not an application. The target exists and runs, and nothing has been deployed to it.

   > **Note:** The first load can take a little while as the platform warms the app up; a slow or blank tab here is normal and not a sign that anything is wrong. Give it a moment and refresh once if it lingers. Keep this tab open when it loads: its address is your App Service name, and you copy it into the workflow in Objective 3.

3. Return to the Azure portal tab, which still shows the web app's **Overview**. In the breadcrumb path at the top of the page, select the resource group name.

   **Expected result**: The resource group opens, listing the three resources again.

4. Select the managed identity beginning `id-deploy-`.

   **Expected result**: The identity's overview opens.

5. In the identity's left menu, select **Azure role assignments**.

   **Expected result**: The list shows one assignment: **Contributor**, scoped to the lab resource group.

   > **Note:** This is the deployment permission your workflow will act with, and it was created for you. What does not exist yet is any way for GitHub to become this identity. That trust is the next objective, and it is yours to build.

### Fork the starter repository

1. In your normal browser window, not the InPrivate one, navigate to `https://github.com` and sign in with your own GitHub account.

   **Expected result**: You are signed in to GitHub. Keep this window and the InPrivate portal window open; you move between them throughout the lab.

2. Navigate to `https://github.com/pluralsight-cloud/Lab-Deploy-an-Azure-App-with-GitHub-Actions-and-Federated-Identity`.

   **Expected result**: The starter repository opens, showing `server.js`, `package.json`, and the `.github/workflows` folder.

3. Select **Fork**, keep your account as the owner, replace the pre-filled **Repository name** with `azure-federated-deploy-starter`, and select **Create fork**.

   **Expected result**: Your fork opens under your own account, named `azure-federated-deploy-starter`.

   > **Note:** The shorter working name keeps every string you build in this lab, from the OIDC prefix to the credential subject, compact and easy to verify. The fork's immutable IDs are assigned at creation regardless of the name you choose.

4. In your fork's file list, select the `.github/workflows` folder.

   **Expected result**: The folder opens, listing the workflow file `deploy.yml`.

5. Select `deploy.yml` and review the workflow without editing it.

   **Expected result**: The workflow has a push trigger for `main`, an `AZURE_WEBAPP_NAME` placeholder reading `<your-app-name>`, and one job with a checkout step and two marked TODO sections for the sign-in and deploy steps.

   > **Note:** The two marked TODO sections are your Objective 3 work. Nothing you were given contains a credential: no publish profile, no service principal secret, nothing to rotate or leak. That is the point of this lab's design.

6. At the top of the repository, select the **Actions** tab.

   **Expected result**: A notice states **Workflows aren't being run on this forked repository**.

7. Select **I understand my workflows, go ahead and enable them**.

   **Expected result**: A confirmation strip reads **Actions Enabled.** and the **Deploy to Azure** workflow is listed.

   > **Note:** Forks ship with Actions paused as a safety default. Enabling them now matters: later in the lab, a commit to `main` must trigger a run, and a paused fork would swallow it silently.

### Record the fork's immutable identifiers

1. Still at the top of the repository, select the **Settings** tab.

   **Expected result**: The repository settings open on the **General** page.

2. In the left menu, under **Code, planning, and automation**, expand **Actions** and select **OIDC**.

   **Expected result**: The **OIDC configuration** page opens with a **Subject claim** section showing a **Use default template** checkbox and, below it, the **Use immutable subject claim** checkbox.

3. Review the **Use immutable subject claim** checkbox.

   **Expected result**: The checkbox is selected and grayed out, with a note that it is automatically enabled for this repository, and that repositories created or renamed after July 15, 2026 use immutable subject claims.

   > **Note:** Your fork was created moments ago, so every token its workflows mint carries the immutable subject form. GitHub assigns the owner ID and repository ID once and never reuses them: renaming or transferring the repository does not change them, and a deleted repository re-created under the same name receives new ones. The credential you create next records both IDs, which is what pins the trust to this exact fork. Leave every setting on this page as it is; you never select **Save subject claim** in this lab.

4. Read the **Default subject claim prefix** and note the two numbers in it: the owner ID after your username and the repository ID after the repository name, each following an `@`.

   **Expected result**: The prefix reads `repo:<your-username>@<owner-id>/azure-federated-deploy-starter@<repository-id>` with your two numeric IDs in place, and the page states: Use this prefix when configuring trust policies in your cloud provider.

   > **Note:** Keep this page open or note the two IDs somewhere transient. You enter both in the Azure portal in the next objective, and you return to this prefix to verify the credential you build matches it exactly.

## Configure federated trust

You will create the trust that replaces a stored secret: a federated credential on the managed identity, built from your fork's names and immutable IDs and scoped to exactly your fork and its `main` branch. Then you record the three identifier values the workflow uses to name that identity, and store them as repository secrets.

### Create the federated credential

1. Switch to the portal window, where the managed identity is still open on its **Azure role assignments**. In the left menu under **Settings**, select **Federated credentials**.

   **Expected result**: The federated credentials list opens and is empty, showing **0 of 20 configured**.

2. Select **Add Credential**.

   **Expected result**: The **Add Federated Credential** pane opens with a **Federated credential scenario** selector.

3. In the scenario selector, select **Configure a GitHub issued token to impersonate this managed identity and deploy to Azure**.

   **Expected result**: The pane shows a **Connect your GitHub account** section with fields for the organization, repository, their numeric IDs, and the entity, and an information banner recommending the new subject identifier format with immutable organization and repository IDs. The **Issuer URL** and **Audience** fields are pre-filled and need no changes.

4. Fill in the trust exactly: **Organization** is your GitHub username, **Organization ID** is the owner ID from your fork's OIDC page, **Repository** is `azure-federated-deploy-starter`, **Repository ID** is the repository ID from the same page, **Entity** is **Branch**, and **Branch** is `main`.

   **Expected result**: The **Subject identifier** field previews `repo:<your-username>@<owner-id>/azure-federated-deploy-starter@<repository-id>:ref:refs/heads/main`.

5. Compare the preview with the **Default subject claim prefix** on your fork's **OIDC** page.

   **Expected result**: The preview is exactly that prefix plus `:ref:refs/heads/main`, character for character.

   > **Note:** Read that subject string carefully, because it is the entire security model. Azure will only exchange a token whose subject matches what this credential records, and the matching is exact and case-sensitive. Nothing here is secret; the trust is in the match, and the immutable IDs pin the match to this exact fork. Later in the lab, a run prints the subject it presented, and you will see the token, this credential, and your fork's OIDC page all carrying the same string.

6. Under **Credential details**, in **Name**, enter `fork-main-deploy`.

   **Expected result**: Every required field is filled and the **Add** button becomes available.

7. Select **Add**.

   **Expected result**: A notification confirms the credential was successfully added, and `fork-main-deploy` appears in the list with its issuer and subject identifier, the counter reading **1 of 20 configured**.

### Record the authentication values

1. In the identity's left menu, select **Overview**, and copy the **Client ID** value.

   **Expected result**: You have the identity's client ID, a GUID.

   > **Note:** Keep the three values from this section available in the portal windows or paste them somewhere transient; you store each one as a repository secret in a moment.

2. Still on the identity's **Overview**, copy the **Subscription ID** value.

   **Expected result**: You have the subscription ID, a GUID.

3. In the top search bar, enter `Microsoft Entra ID` and select **Microsoft Entra ID** from the results. On its overview, copy the **Tenant ID** value.

   **Expected result**: You have the tenant ID, a GUID.

   > **Note:** Notice what you are collecting: three identifiers, no keys. The client ID names the identity, the tenant and subscription name where it lives. None of them can authenticate anything by itself; the federated credential you created is what turns a workflow run into this identity.

### Store the repository secrets

1. Switch to the GitHub window, which still shows your fork's settings from the **OIDC** page. In the left menu, select **Secrets and variables**, then **Actions**.

   **Expected result**: The **Actions secrets and variables** page opens.

2. Select **New repository secret**, name it `AZURE_CLIENT_ID`, paste the client ID as the value, and select **Add secret**.

   **Expected result**: `AZURE_CLIENT_ID` is listed under **Repository secrets**.

3. Repeat for the second value: name `AZURE_TENANT_ID`, value the tenant ID, and select **Add secret**.

   **Expected result**: `AZURE_TENANT_ID` is listed.

4. Repeat for the third value: name `AZURE_SUBSCRIPTION_ID`, value the subscription ID, and select **Add secret**.

   **Expected result**: All three secrets are listed: `AZURE_CLIENT_ID`, `AZURE_SUBSCRIPTION_ID`, `AZURE_TENANT_ID`.

   > **Note:** Storing identifiers in secrets is convention and tidiness, not necessity: even if all three leaked, a stranger's repository still could not authenticate, because its runs would carry the wrong subject. Compare that with a publish profile or a client secret, where a leak is the whole ballgame. That difference is why your team is moving to federation.

## Build the deployment workflow

You will complete the workflow's two marked sections in the GitHub web editor: the sign-in step that authenticates over OIDC using your three secrets, and the deploy step that packages the repository and ships it to your App Service target.

### Add the sign-in step

1. In your fork, select the **Code** tab, open `.github/workflows/deploy.yml` as before (the `.github/workflows` folder, then the file), and select the pencil icon to edit the file.

   **Expected result**: The workflow opens in the web editor.

2. Replace the entire line reading `# ==== TODO 1: Azure sign-in step (added during the lab) ====` with the following, keeping the indentation exactly as shown so `- name:` aligns with the checkout step's `- name:` above it.

   ```yaml
     - name: Azure sign-in
       uses: azure/login@v2
       with:
         client-id: ${{ secrets.AZURE_CLIENT_ID }}
         tenant-id: ${{ secrets.AZURE_TENANT_ID }}
         subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
   ```

   **Expected result**: The sign-in step sits under the checkout step at the same indentation level.

   > **Note:** YAML is whitespace-sensitive, and indentation is the most common way this step goes wrong. There is no password field in what you pasted: the step presents the run's own GitHub-issued token plus these three identifiers, and Azure checks that token against the federated credential you created.

### Add the deploy step and commit

1. Replace the entire line reading `# ==== TODO 2: Deploy step (added during the lab) ====` with the following, at the same indentation as the step above.

   ```yaml
     - name: Deploy to Azure App Service
       uses: azure/webapps-deploy@v3
       with:
         app-name: ${{ env.AZURE_WEBAPP_NAME }}
         package: .
   ```

   **Expected result**: The workflow now has three steps: checkout, Azure sign-in, and the deploy step.

2. In the `env:` block near the top of the file, replace `<your-app-name>` with your App Service name, the part of the default domain before `.azurewebsites.net`, such as `app-fedid-<hash>`, keeping the quotes.

   **Expected result**: The env block reads `AZURE_WEBAPP_NAME: "app-fedid-<hash>"` with your actual name.

   > **Note:** The name is one tab away: the default-page tab you opened in the private window in Objective 1 is still there, and its address reads `app-fedid-<hash>.azurewebsites.net`. Copy the part before `.azurewebsites.net`. If you closed that tab, the same name is the web app's title on its **Overview** in the portal.

3. Select **Commit changes**, keep the option to commit directly to `main`, and confirm **Commit changes**.

   **Expected result**: The commit lands on `main`.

   > **Note:** The workflow triggers on every push to `main`, so this commit starts your first run. Leave the editor and follow it in the next section; the run's result is part of the lesson.

## Run, diagnose, and fix forward

Your commit has already triggered the workflow. This first run fails at the Azure sign-in step, and the failure is the teaching moment: the run's log names the permission the workflow never requested. You read the error, grant exactly that permission with one commit, and the push runs the corrected workflow to a clean deployment.

### Watch the run fail at sign-in

1. Select the **Actions** tab and open the newest run, the one named after your commit.

   **Expected result**: The run fails in about twenty seconds. In the **Build and deploy** job, the checkout step succeeded, the **Azure sign-in** step shows the failure, and the **Deploy to Azure App Service** step shows a slash icon, meaning it was skipped.

2. Select the **Build and deploy** job and expand the **Azure sign-in** step's log.

   **Expected result**: The log shows `Running Azure CLI Login.`, then an error reading `Error: Failed to fetch federated token from GitHub. Please make sure to give write permissions to id-token in the workflow.`, followed by a second error mentioning the `ACTIONS_ID_TOKEN_REQUEST_URL` environment variable.

   > **Note:** You may also see Node.js deprecation warnings at the top of the log and in the run's annotations. They come from the actions' runtimes, appear on every run, and can be ignored throughout this lab.

3. Read the first error line again and consider what it is and is not saying.

   **Expected result**: The run failed before a token existed: GitHub refused to issue one, so Azure was never contacted. Your secrets, your sign-in step, and the federated credential were never reached and cannot be the cause.

   > **Note:** A workflow job can request its short-lived OIDC token only when it declares `id-token: write` in a `permissions` block, and this workflow never declares one. The error names the fix outright: give write permissions to `id-token`. The cause lives in the workflow file itself, in your repository, and that placement decides your next move.

### Grant the permission and fix forward

1. Select the **Code** tab, open `.github/workflows/deploy.yml` as before, and select the pencil icon to edit the file.

   **Expected result**: The workflow opens in the web editor.

2. On the blank line between `workflow_dispatch:` and `env:`, add the following block, flush left so `permissions:` starts at the beginning of the line, aligned with `on:` and `env:`.

   ```yaml
   permissions:
     id-token: write
     contents: read
   ```

   **Expected result**: The workflow now declares `permissions:` at the top level, between its triggers and its `env:` block.

   > **Note:** `id-token: write` lets each run request its own short-lived identity token, which is what the sign-in step presents instead of a password. `contents: read` must ride along: declaring a `permissions` block replaces the job's default permissions instead of adding to them, and without read access the checkout step would fail next.

3. Select **Commit changes**, keep the option to commit directly to `main`, and confirm **Commit changes**.

   **Expected result**: The commit lands on `main`.

   > **Note:** Committing the fix is the correct move here, and it is worth pausing on why. The failure's cause lives in the repository, so you fix forward with a new commit, and the push triggers a fresh run of the corrected workflow. When a failure's cause lives on the Azure side instead, such as a federated credential that does not match the token, you fix it there and re-run the same commit, because the commit was never wrong. Knowing which side the cause lives on tells you which move is correct.

### Watch the new run deploy

1. Select the **Actions** tab.

   **Expected result**: A second run is listed, named after your fix commit, above the failed first run.

2. Open the new run and expand the **Azure sign-in** step's log once the step completes.

   **Expected result**: The log shows a **Federated token details:** block listing the issuer, the subject claim, the audience, and the job workflow reference, then `Attempting Azure CLI login by using OIDC...`, then `Subscription is set successfully.` and `Azure CLI login succeeds by using OIDC.`

   > **Note:** Read the `subject claim` line: it is the immutable subject your fork presented, matching the credential you created and the prefix on your fork's OIDC page character for character. Token, credential, and OIDC settings all carry one string, and that three-way agreement is what healthy federated trust looks like. If the sign-in instead fails with an error containing `AADSTS700213: No matching federated identity record found for presented assertion subject`, one of the IDs in the credential was mistyped: copy the full `subject claim` value from this block in the failed log, open `fork-main-deploy` under the identity's **Federated credentials**, select **Edit (optional)** under **Subject identifier**, paste the presented subject, select **Update**, and re-run the previous run.

3. Wait for the **Deploy to Azure App Service** step to finish.

   **Expected result**: The deploy step logs `Successfully deployed web package to App Service.` followed by the application URL, and the run completes successfully end to end.

   > **Note:** The sign-in completes in a few seconds and the deploy step takes about 20 to 30 seconds. Scan the run log when it finishes: expand the `Run azure/login@v2` line inside the **Azure sign-in** step, and your three identifiers echo only as masked `***` values in its inputs; no credential appears anywhere, because nothing was ever provided that could deploy from outside this trust.

## Validate the deployment

You will verify the result the way an operator would: the application answering in place of the default page, the version endpoint reporting the deployed version, and the deployment record on the App Service side.

### Browse the application and confirm the record

1. Switch to the portal window. In the top search bar, enter `app-fedid` and select your web app from the results.

   **Expected result**: The web app's **Overview** opens.

2. On the **Overview**, in the **Essentials** section, select the **Default domain** link to browse the application in a new tab.

   **Expected result**: The Shipment Tracker page loads in place of the default App Service page, and it states the deployment used federated identity.

   > **Note:** The first load after a deployment typically appears within a few seconds once the run is green; refresh if the default page lingers.

3. In that tab, append `/version` to the address and load it.

   **Expected result**: The endpoint returns JSON with `"application": "shipment-tracker"` and `"version": "1.0.0"`, the deployed application version.

4. Return to the portal window. In the left menu under **Deployment**, select **Deployment Center**, then the **Logs** tab.

   **Expected result**: The deployment from your workflow run is listed with its time, a **Deployment ID**, **Author** `N/A`, a status of `Succeeded (Active)`, and **Message** `OneDeploy`.

   > **Note:** An information banner may state that CI/CD is not configured; that is expected, because the workflow pushes each deployment directly rather than registering a pipeline here. The **Deployment ID** is the deployment record's own identifier, not your commit hash; the timestamp is what ties this record to your run.

   > **Note:** Take stock of the credential inventory of what you built: zero client secrets stored anywhere, three plain identifiers in your fork's secrets, one federated credential pinning trust to your exact fork and branch, one pre-provisioned identity holding Contributor, and a deployment that authenticated with a per-run token. Nothing in that chain can be stolen and reused outside your repository, and the only thing the workflow ever needed was permission to request that token.

## Conclusion

You are now done with the lab. You connected a GitHub fork to a pre-provisioned Azure managed identity: you recorded the fork's immutable identifiers, created a federated credential matching them exactly, stored three identifiers as repository secrets, and completed the deployment workflow with an OIDC sign-in step and a deploy step. Your first run failed at sign-in because the workflow was not permitted to request its identity token; you read the error, granted `id-token: write` with a fix-forward commit, and the new run deployed the Shipment Tracker cleanly, confirmed by the page, the `/version` endpoint, and the Deployment Center record. The pattern transfers directly to your own delivery environment: federate the trust to your repository's immutable identity, grant runs only the permission to request their own token, and never store a secret that a leak could turn into a deployment. Confirm by selecting **End Lab**.
