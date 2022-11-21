# azure-swa-hugo-blog

Code to deploy and maintain an Azure Static Webapp running a static web site generated with Hugo and GitHub Actions.

## Summary

The code in this repository is meant to deploy resources to host a static website generated by Hugo on Azure Web Apps with a custom domain. You will need the following to deploy successfully:

* Terraform installed locally
* Azure CLI installed locally
* Hugo installed locally for testing
* Azure subscription
* Registered domain name for website

## But Why?

Why what? Why manage your own blog and domain name? That's a much larger conversation, but I'll simply say that putting all of your content on someone else's platform also puts you at the mercy of their whims and in many cases you are relinquishing ownership of the content. Managing your own blog and domain gives you the independence to move to whatever platform supports HTML (hint: there's a lot) and also doesn't require you to cowtow to the overlords of any given platform.

### Why Azure Static Web Apps?

It doesn't have to be Azure SWA. In fact the basic website will nicely deploy wherever you want. I chose SWA for two reasons: they have a more than adequate free tier and I am already using it for my podcast [Chaos Lever](https://chaoslever.com).

### Why a Static Site?

Static sites are generally very fast to render. They don't require any compute on the hosting side. They can be deployed anywhere that can render HTML. Can you use Wordpress? Sure! I'm not going to litigate the point with you. My professional blog is on Wordpress for now and that's fine.

### Why Hugo?

Because I've used it before and it's fairly easy to pick up.

### Why GitHub and GitHub Actions

Two reasons for that one. First it's free, and second I think if you're working in tech it's a good idea to understand basic software development principles and workflows.

### Azure Infrastructure

The `infrastructure` folder has Terraform files in it to deploy the necessary resources to host the website. It includes the following:

* Resource group
* Static site
* Static site custom domain
* DNS zone, Alias, and TXT validation record
* API Token for website deployment and updates from GitHub Actions

More information is found in the [deployment section](#deployment) for the infrastructure.

### Blog website

This repository uses the Hugo theme *ananke* to spin up a personal blog under a custom domain. You don't have to use that theme, you can remove it and select another theme from the [many, many, many themes out there](https://themes.gohugo.io/). You will need to remove the current theme and add the new one following the [process outlined in the Hugo docs](https://gohugo.io/getting-started/quick-start/).

You can deploy the website as-is and then customize it to your heart's content. If you want to render it locally, simply navigate to the `website` directory and run `hugo server -D` to start up a dev instance running on [http://localhost:1313](http://localhost:1313). Hey 1313? That's my thing! 😂

## Deployment

Here's the high-level process we're going to follow:

1. [Fork this repository to your GitHub account](#fork-the-repository)
1. [Clone your forked repo to your local workstation](#fork-the-repository)
1. [Deploy the Azure infrastructure](#infrastructure)
1. [Verify your domain](#verify-your-domain)
1. [Copy your API token](#prepare-your-repository)
1. [Publish your blog](#run-github-actions)

### Fork the Repository

Your shiny new website is going to live in a GitHub repository and be published through GitHub Actions. The bare bones to do so are included in this repository, so step one is forking the repo to your account. Click on the *Fork* button and do that now.

Once the fork is complete, go ahead and clone your fork of the repository down to your local workstation. That's where we'll be running our commands from.

### Infrastructure

We are going to deploy the Azure resources using Terraform, which means you have to decide where to store your state data. If you're just using this to deploy the infrastructure once and never touch it again, then no worries. You can use the local backend for state data and delete it when you're done. Go ahead and skip to the next section.

If you're going to manage the infrastructure with Terraform going forward, then you'll need somewhere to store state data. Since you're already using Azure for everything else, you might as well use it for the state data. The next section describes the steps to create a storage account and container with the Azure CLI that you can use for Terraform state data, and what you have to change in the configuration to use it.

#### Store State Data

We are going to create an Azure storage account and container to store our state data and then update the backend block in the `terraform.tf` file to use that storage account.

```bash
# Navigate to the infrastructure directory
cd infrastructure

# Store website name in a variable for naming, no capitals or punctuation
website_name=YOUR_WEBSITE_DOMAIN_NAME # e.g. 10bitpodcastcom
region=AZURE_REGION_TO_USE # e.g. eastus

## Log in with Azure CLI and select a subscription
az login
az account set -s SUBSCRIPTION_NAME

## Create an resource group
az group create -n "${website_name}tfdata" -l $region

## Create a storage account
az storage account create --name "${website_name}tfdata" --resource-group "${website_name}tfdata" \
  --location $region --sku Standard_LRS

## Create a storage account container
az storage container create -n tfdata --account-name "${website_name}tfdata" \
  --resource-group "${website_name}tfdata"

## Set up backend config file
cat << EOF > backend-config.txt
resource_group_name="${website_name}tfdata"
storage_account_name="${website_name}tfdata"
container_name="tfdata"

EOF
```

In the `terraform.tf` file uncomment the `backend` block so it looks like this:

```terraform
backend "azurerm" {
    key = "terraform.tfstate"
}
```

We will add in the rest of the values when we run `terraform init`.

#### Prepare your Terraform variable values

In the `infrastructure` directory is a file named `terraform.tfvars.example`, rename it to `terraform.tfvars` and fill out the values in the file. For my example, let's assume I want to publish my blog called "blog.10bitpodcast.com" in the Central US region. I would set the following values:

```terraform
region = "centralus"
website_name = "blog.10bitpodcast.com"
custom_domain_name = "10bitpodcast.com"
```

Terraform will create a DNS Zone for 10bitpodcast.com and add two records in the zone. One TXT record for validation under `blog` with a validation token generated by Azure Static Web Apps. And one Alias record pointing `blog` to the Azure Static Web App itself.

While Azure Static Web Apps supports either TXT or CNAME record validation, CNAME doesn't support using an apex domain (e.g. 10bitpodcast.com), so I chose to only use TXT. That means if you want to use an apex domain, go for it!

#### Deploy the Infrastructure

Time to get our web app up and running! Start by switching to the `infrastructure` directory if you're not already in it and run `terraform init`. I've provided two commands, one for local state data and the other if you're using Azure Storage:

```bash
# Run this command if you're using local state data
terraform init
```

```bash
# Run this command if you're using Azure Storage for state data
terraform init -backend-config=backend-config.txt
```

Now you can simply run `terraform apply` to stand up your resources:

```bash
terraform apply -auto-approve # YOLO!
```

The output you receive will include the Azure nameservers and API token for your web app. It should look something like this:

```bash
api_token = "8e857f7dd9e67fda7be48c55ace344ecf9f2a35ad15987500ebd352e6c2039252-0c848baf-da60-495a-a8ea-33177b28a76e010149011"
name_servers = toset([
  "ns1-09.azure-dns.com.",
  "ns2-09.azure-dns.net.",
  "ns3-09.azure-dns.org.",
  "ns4-09.azure-dns.info.",
])
```

You will use these values in the following sections.

### Verify your domain

To use a custom domain name with Azure Static Web Apps, you need to verify that you own the domain. We've already created the validation TXT record in the Azure DNS zone, but Static Web Apps can't find it until we update the name servers for domain.

#### Set up your domain nameservers

The instructions for this part will vary depending on your domain registrar. Essentially, we need to tell your registrar that Azure DNS is managing records for this domain. In the console of your registrar, specify that you are using custom name servers and enter all four nameservers delivered as part of the Terraform output.

A few important things to note. First, Azure correctly lists the name servers with a trailing `.`. Some registrars are not expecting the trailing period and will give you grief. Simply remove the offending period at the end.

Second, the Azure name servers will probably be different than what you see in my output. Don't just copy and paste what you see in this `README`, it won't work!

Lastly, it can take a bit of time for the changes to propagate the internet. In theory it should be fairly quick, since your web app is tapped directly into Azure DNS, but still these things take time. Go grab a cuppa and chill ☕.

### Deploy Your Site

The contents of the `website` directory are a simple Hugo website using the *ananke* theme. You can switch themes, customize them, and add your own static content. Hugo will allow you to render the site locally before you deploy it up to Azure. I'm not going to walk through that whole process, instead I will point you at the [Hugo getting started docs](https://gohugo.io/getting-started/).

#### Prepare Your Repository

We're now ready to use that other value from the Terraform output, the API token. On your forked GitHub repository, click on **Settings**, then **Secrets** -> **Actions**. Create a new repository secret called `AZURE_STATIC_WEB_APPS_API_TOKEN` and give it the value of the `api_token` Terraform output. This is how GitHub Actions is able to find your Static Web App and deploy the website.

### Run GitHub Actions

The required GitHub Actions are included in the file `azure-static-web-apps.yml` located in `.github/workflows`. The general workflow for publishing your site and making updates looks like this:

1. Create a new branch locally and make updates, create a new post, add images, etc.
1. Spin up a local copy to make sure it looks good and loads properly
1. Push your new branch up to GitHub
1. Create a Pull Request to merge your new branch to `main`

The Pull Request will kick off GitHub Actions workflow to deploy a temporary site on Azure for you to verify. The temporary site's URL will be added as a comment to the Pull Request.

1. Review the temporary site
1. If things look wrong, make an update locally and push to GitHub, updating the temporary site
1. If things look right, merge the Pull Request to `main`

Merging the Pull Request will create a Push on `main`, kicking off GitHub Actions again to delete the temporary site and deploy the update to your main site.

And that's it! When you want to make changes, simply follow the same workflow.
