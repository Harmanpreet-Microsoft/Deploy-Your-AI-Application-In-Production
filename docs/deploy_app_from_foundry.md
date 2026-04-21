# Deploy an Application from Microsoft Foundry

This guide explains how to deploy a chat application directly from the Microsoft Foundry playground to Azure App Service.

## Overview

Microsoft Foundry provides a built-in capability to publish playground experiences as web applications. This accelerator deploys the required infrastructure (App Service, managed identity, networking) so you can publish directly from the Foundry playground.

> **UI note:** The new Foundry experience (the "New Foundry" toggle) does not currently show **Deploy to a web app**. Use the classic Foundry UI to publish for now. We will update this guide once the new UI supports web app deployment.

## Prerequisites

- Completed deployment of this accelerator (`azd up`)
- An AI Search index with your data (created via the workspace documents indexer or manually)
- For network-isolated (WAF) deployments: access to the Jump VM via Bastion

---

## Step 1: Upload Documents to the Bronze Lakehouse

Before deploying the web app, upload the files you want the chat application to use as grounding data.

1. Go to [app.fabric.microsoft.com](https://app.fabric.microsoft.com)
2. Open your deployment workspace (e.g., `workspace-<envname>`)
3. Open the **bronze** lakehouse
4. Navigate to **Files** → **documents**
5. Upload your files (PDFs, text files, etc.) into this folder

---

## Step 2: Re-trigger the AI Search Indexer

After uploading documents, re-run the indexer so the new files are processed and searchable.

1. Navigate to **Azure Portal** → your resource group
2. Open the **AI Search** service
3. Go to **Indexers** → select the documents indexer (named `workspace-<envname>-documents-indexer`)
4. Click **Run** and wait for the indexer to complete
5. Verify the **Document count** under **Indexes** has increased to reflect your uploaded files

Or use the CLI:

```bash
az search indexer run --name workspace-<envname>-documents-indexer --service-name <search-name> --resource-group <rg-name>
```

---

## Step 3: Access Microsoft Foundry

### Network-Isolated Deployments (WAF)

Since all resources are deployed with private endpoints, you must access Microsoft Foundry through the Jump VM:

1. Go to the [Azure Portal](https://portal.azure.com)
2. Navigate to your resource group
3. Select the **Jump VM** (Windows Virtual Machine) and ensure it is **Running**
4. Click **Connect** → **Bastion**
5. Enter the VM credentials (`VM_ADMIN_USERNAME` / `VM_ADMIN_PASSWORD`, or as set in [infra/main.bicepparam](../infra/main.bicepparam))
6. Once connected, open a browser inside the VM and navigate to [Azure Portal](https://portal.azure.com)
7. Navigate to your resource group → select the **AI Foundry** project → click **Launch AI Foundry**

### Non-WAF Deployments

Access Microsoft Foundry directly from your browser:

1. Navigate to [ai.azure.com](https://ai.azure.com)
2. Sign in and select your Microsoft Foundry project

---

## Step 4: Configure the Chat Playground

1. Navigate to **Playgrounds** → **Chat playground**
2. Click **Add your data** (in the data source section)
3. Select your **AI Search** service (the one in your resource group)
4. Select the search index — choose the index that starts with your **workspace name** (e.g., `workspace-<envname>-documents-index`)
5. Complete the data source configuration

---

## Step 5: Deploy a Base Model

1. In Microsoft Foundry, go to **Deploy** → **Deploy base model**
2. Select **gpt-4o** or **gpt-4o-mini**
3. Configure the deployment name and settings, then deploy
4. Wait for the deployment to complete
5. Once deployed, select that model in the **model dropdown** at the top of the chat playground

### Test Your Configuration

1. Test the chat experience in the playground
2. Verify responses are grounded in your uploaded documents
3. Adjust system prompts and parameters as needed

---

## Step 6: Deploy to Web App

> **UI note:** If the "New Foundry" toggle is enabled, the **Deploy to a web app** option may be hidden. Switch to the classic Foundry UI to publish the web app.

1. On the top panel of the chat playground, click **Deploy** → **Deploy to a web app**
2. Configure deployment options:
   - **Create new** or **Update existing** web app
   - Select your **Subscription** and **Resource group**
   - Choose your deployment **Region** (see region guidance below)
3. Review authentication settings (Entra ID is recommended)
4. Click **Deploy** and wait for the deployment to complete

### Region Guidance

| Deployment Type | Region Requirement |
|----------------|-------------------|
| **Network-isolated (WAF)** | Deploy in the **same region** as the VNet used by your deployment |
| **Non-WAF** | Deploy in **any region** |

---

## Step 7: Configure VNet Integration (WAF Only)

For network-isolated deployments, the web app must be integrated with the deployment VNet to access private resources:

1. Navigate to **Azure Portal** → your resource group
2. Find the newly deployed **App Service** (web app)
3. Go to the **Networking** tab
4. Under **Outbound traffic configuration**, select **VNet integration**
5. Configure the VNet integration with the VNet from your deployment
6. Save the configuration

> **Non-WAF deployments** do not require VNet integration. Skip this step.

---

## Step 8: Access Your Application

After deployment and networking configuration:

1. Navigate to the **App Service** in Azure Portal
2. Copy the **Default domain** URL
3. Open the URL in a browser (authentication may be required)
4. For WAF deployments, you can access the web app from the Jump VM or from any machine if the App Service has public access enabled with VNet integration for backend connectivity

---

## Additional Resources

- [Deploy a web app for chat on your data](https://learn.microsoft.com/en-us/azure/ai-foundry/how-to/deploy-web-app)
- [Microsoft Foundry documentation](https://learn.microsoft.com/en-us/azure/ai-foundry/)
- [Customize the web app](https://learn.microsoft.com/en-us/azure/ai-foundry/how-to/deploy-web-app#customize-the-web-app)

## Troubleshooting

### App not accessible from internet

If your App Service is deployed with private endpoints, you'll need to access it through the Jump VM or configure Azure Front Door for public access. For WAF deployments, ensure VNet integration is configured (see [Step 7](#step-7-configure-vnet-integration-waf-only)).

### Authentication errors

Ensure the App Service has the correct managed identity permissions to access:
- Azure OpenAI / AI Services
- Azure AI Search
- Azure Storage (if applicable)

### Data not appearing in responses

Verify that:
1. Your documents were uploaded to the bronze lakehouse (`Files/documents/`)
2. The AI Search indexer has been re-triggered and completed successfully (see [Step 2](#step-2-re-trigger-the-ai-search-indexer))
3. The playground is configured to use the correct index (workspace-prefixed, not onelake-prefixed)
4. The deployed app has the same index configuration

### Web app cannot reach backend services (WAF)

If the web app returns errors about being unable to connect to Azure OpenAI or AI Search:
1. Verify VNet integration is configured on the App Service (see [Step 7](#step-7-configure-vnet-integration-waf-only))
2. Ensure the web app is deployed in the same region as the VNet
