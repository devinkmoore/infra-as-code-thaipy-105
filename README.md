# ThaiPy 105: Infra as Code in Python with Pulumi

For introduction to infra as code, see the [slides](https://docs.google.com/presentation/d/1C0EjFw952fnU_X28p8u37oCGUNtGmh5apqHTdg-Pmis/edit?usp=sharing]).
You'll want to create your own repo and follow these steps since you have to reference your own GCP project ID, etc.

## How to

These are the steps I took to set everything up.

1. Create a Pulumi account, there's a free option
2. Create a GCP project, you may be prompted to enable billing and you'll.
3. Create a Python virtual environment
4. `pip install pulumi pulumi_gcp gcloud`
5. Authorize with Gcloud CLI, use the same user you created the GCP project with to avoid permissions issues
    ```
    gcloud auth login
    gcloud config set project <your-project-id>
    gcloud auth application-default login
    ```
6. `pulumi login`
7. `mkdir pulumi && cd pulumi`
8. `pulumi new gcp-python`, give the project a name and description, then choose the default `dev` when it asks you for
a stack name
9. In `pulumi/__main__.py` you can see there is already an example, add an arg `name="thaipy-12345"` to `storage.Bucket`
10. `pulumi up` and the bucket gets created
11. Now let's create a secret in Secret Manager, first create a config for the secret. If you look in 
`pulumi/Pulumi.dev.yaml` you'll see that it's been encrypted
    ```
    pulumi config set our_little_secret 123456 --secret
    ```
12. We'll need to reference some config values in our Pulumi program, on lines 6-8 put:
   ```
   project = pulumi.Config('gcp').require('project')
   secret_value = pulumi.Config('thaipy-123456').require_secret('our_little_secret')
   ```
13. Add a few imports at the top:
   ``` 
   from pulumi_gcp import projects, secretmanager, storage
   ```
14. In GCP you have to enable services before you can use them, add this to the bottom of the program
    ```
    secret_manager = projects.Service(
      'enable-secretmanager-api', service="secretmanager.googleapis.com", project=project
    )
    ```
    
15. No we can add the code to create the secret, note the `depends_on` argument in `pulumi.ResourceOptions`. If you
don't have that then it will fail because secret manager isn't enabled when Pulumi tries to create it.
    ``` 
    secret = secretmanager.Secret(
        "thaipy", secret_id="thaipy", replication={"auto": {}}, opts=pulumi.ResourceOptions(depends_on=[secret_manager])
    )
    secret_version = secretmanager.SecretVersion('thaipy', secret=secret.id, secret_data=secret_value)
    ```
16. `pulumi up` to create the secret
17. Let's say we want to change the secret value, we can do that with:
    ```
    pulumi config set our_little_secret 654321 --secret
    pulumi up
    ```
18. Finally, tear everything down with `pulumi destroy`